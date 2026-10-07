采样流程：
1. 从原图尺寸和目标尺寸算出理想缩放比
2. 用 `highestOneBit` 变成 2 的幂 → `inSampleSize`
3. 模拟 BitmapFactory 各格式的舍入规则，精确预测输出尺寸
4. 用密度缩放（与普通缩放不同）做第二级精调，弥补 2 的幂不够精确的缺陷
5. 把参数写进 `BitmapFactory.Options`，交给系统解码
```java
private static void calculateScaling(
      ImageType imageType,
      ImageReader imageReader,
      DecodeCallbacks decodeCallbacks,
      BitmapPool bitmapPool,
      DownsampleStrategy downsampleStrategy,
      int degreesToRotate,
      int sourceWidth,
      int sourceHeight,
      int targetWidth,
      int targetHeight,
      BitmapFactory.Options options)
      throws IOException {
    // We can't downsample source content if we can't determine its dimensions.
    if (sourceWidth <= 0 || sourceHeight <= 0) {
      if (Log.isLoggable(TAG, Log.DEBUG)) {
        Log.d(
            TAG,
            "Unable to determine dimensions for: "
                + imageType
                + " with target ["
                + targetWidth
                + "x"
                + targetHeight
                + "]");
      }
      return;
    }

	// 处理旋转
    int orientedSourceWidth = sourceWidth;
    int orientedSourceHeight = sourceHeight;
    // If we're rotating the image +-90 degrees, we need to downsample accordingly so the image
    // width is decreased to near our target's height and the image height is decreased to near
    // our target width.
    //noinspection SuspiciousNameCombination
    if (isRotationRequired(degreesToRotate)) {
      orientedSourceWidth = sourceHeight;
      orientedSourceHeight = sourceWidth;
    }

	// 计算精确缩放因子
    final float exactScaleFactor =
        downsampleStrategy.getScaleFactor(
            orientedSourceWidth, orientedSourceHeight, targetWidth, targetHeight);

    if (exactScaleFactor <= 0f) {
      throw new IllegalArgumentException(
          "Cannot scale with factor: "
              + exactScaleFactor
              + " from: "
              + downsampleStrategy
              + ", source: ["
              + sourceWidth
              + "x"
              + sourceHeight
              + "]"
              + ", target: ["
              + targetWidth
              + "x"
              + targetHeight
              + "]");
    }

    SampleSizeRounding rounding =
        downsampleStrategy.getSampleSizeRounding(
            orientedSourceWidth, orientedSourceHeight, targetWidth, targetHeight);
    if (rounding == null) {
      throw new IllegalArgumentException("Cannot round with null rounding");
    }

	// 计算"理想输出尺寸"
    int outWidth = round(exactScaleFactor * orientedSourceWidth);
    int outHeight = round(exactScaleFactor * orientedSourceHeight);

	// 算出"实际需要的倍数"
    int widthScaleFactor = orientedSourceWidth / outWidth;
    int heightScaleFactor = orientedSourceHeight / outHeight;

    // TODO: This isn't really right for both CenterOutside and CenterInside. Consider allowing
    // DownsampleStrategy to pick, or trying to do something more sophisticated like picking the
    // scale factor that leads to an exact match.
    int scaleFactor =
        rounding == SampleSizeRounding.MEMORY
            ? Math.max(widthScaleFactor, heightScaleFactor)
            : Math.min(widthScaleFactor, heightScaleFactor);

	// 转换成 2 的幂 —— 第一次"砍"
    int powerOfTwoSampleSize;
    // BitmapFactory does not support downsampling wbmp files on platforms <= M. See b/27305903.
    if (Build.VERSION.SDK_INT <= 23
        && NO_DOWNSAMPLE_PRE_N_MIME_TYPES.contains(options.outMimeType)) {
      // WBMP/ICO，且 Android M 及以下  
      powerOfTwoSampleSize = 1;
    } else {
      // scaleFactor 在二进制下的最高位
      powerOfTwoSampleSize = Math.max(1, Integer.highestOneBit(scaleFactor));
      if (rounding == SampleSizeRounding.MEMORY
          && powerOfTwoSampleSize < (1.f / exactScaleFactor)) {
        // 还可以再左移一位
        powerOfTwoSampleSize = powerOfTwoSampleSize << 1;
      }
    }

    // Here we mimic framework logic for determining how inSampleSize division is rounded on various
    // versions of Android. The logic here has been tested on emulators for Android versions 15-26.
    // PNG - Always uses floor
    // JPEG - Always uses ceiling
    // Webp - Prior to N, always uses floor. At and after N, always uses round.
    options.inSampleSize = powerOfTwoSampleSize;
    int powerOfTwoWidth;
    int powerOfTwoHeight;
    if (imageType == ImageType.JPEG) {
      // libjpegturbo can downsample up to a sample size of 8. libjpegturbo uses ceiling to round.
      // After libjpegturbo's native rounding, skia does a secondary scale using floor
      // (integer division). Here we replicate that logic.
      // libjpegturbo 单次最多采样 8 倍，用 ceil
      int nativeScaling = Math.min(powerOfTwoSampleSize, 8);
      powerOfTwoWidth = (int) Math.ceil(orientedSourceWidth / (float) nativeScaling);
      powerOfTwoHeight = (int) Math.ceil(orientedSourceHeight / (float) nativeScaling);
      // 超过 8 的部分
      int secondaryScaling = powerOfTwoSampleSize / 8;
      if (secondaryScaling > 0) {
        powerOfTwoWidth = powerOfTwoWidth / secondaryScaling;
        powerOfTwoHeight = powerOfTwoHeight / secondaryScaling;
      }
    } else if (imageType == ImageType.PNG || imageType == ImageType.PNG_A) {
      powerOfTwoWidth = (int) Math.floor(orientedSourceWidth / (float) powerOfTwoSampleSize);
      powerOfTwoHeight = (int) Math.floor(orientedSourceHeight / (float) powerOfTwoSampleSize);
    } else if (imageType.isWebp()) {
      if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.N) {
        powerOfTwoWidth = Math.round(orientedSourceWidth / (float) powerOfTwoSampleSize);
        powerOfTwoHeight = Math.round(orientedSourceHeight / (float) powerOfTwoSampleSize);
      } else {
        powerOfTwoWidth = (int) Math.floor(orientedSourceWidth / (float) powerOfTwoSampleSize);
        powerOfTwoHeight = (int) Math.floor(orientedSourceHeight / (float) powerOfTwoSampleSize);
      }
    } else if (orientedSourceWidth % powerOfTwoSampleSize != 0
        || orientedSourceHeight % powerOfTwoSampleSize != 0) {
      // If we're not confident the image is in one of our types, fall back to checking the
      // dimensions again. inJustDecodeBounds decodes do obey inSampleSize.
      int[] dimensions = getDimensions(imageReader, options, decodeCallbacks, bitmapPool);
      // Power of two downsampling in BitmapFactory uses a variety of random factors to determine
      // rounding that we can't reliably replicate for all image formats. Use ceiling here to make
      // sure that we at least provide a Bitmap that's large enough to fit the content we're going
      // to load.
      powerOfTwoWidth = dimensions[0];
      powerOfTwoHeight = dimensions[1];
    } else {
      powerOfTwoWidth = orientedSourceWidth / powerOfTwoSampleSize;
      powerOfTwoHeight = orientedSourceHeight / powerOfTwoSampleSize;
    }

	// 计算二次缩放的密度比
    double adjustedScaleFactor =
        downsampleStrategy.getScaleFactor(
            powerOfTwoWidth, powerOfTwoHeight, targetWidth, targetHeight);

    // Density scaling is only supported if inBitmap is null prior to KitKat. Avoid setting
    // densities here so we calculate the final Bitmap size correctly.
    // 设置密度字段（Android 4.4+）
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.KITKAT) {
      options.inTargetDensity = adjustTargetDensityForError(adjustedScaleFactor);
      options.inDensity = getDensityMultiplier(adjustedScaleFactor);
    }
    if (isScaling(options)) {
      options.inScaled = true;
    } else {
      options.inDensity = options.inTargetDensity = 0;
    }

    if (Log.isLoggable(TAG, Log.VERBOSE)) {
      Log.v(
          TAG,
          "Calculate scaling"
              + ", source: ["
              + sourceWidth
              + "x"
              + sourceHeight
              + "]"
              + ", degreesToRotate: "
              + degreesToRotate
              + ", target: ["
              + targetWidth
              + "x"
              + targetHeight
              + "]"
              + ", power of two scaled: ["
              + powerOfTwoWidth
              + "x"
              + powerOfTwoHeight
              + "]"
              + ", exact scale factor: "
              + exactScaleFactor
              + ", power of 2 sample size: "
              + powerOfTwoSampleSize
              + ", adjusted scale factor: "
              + adjustedScaleFactor
              + ", target density: "
              + options.inTargetDensity
              + ", density: "
              + options.inDensity);
    }
  }
```