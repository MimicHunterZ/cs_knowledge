Glide的缓存可以分为两种，第一种是内存缓存，第二种是硬盘缓存。
其中内存缓存又包括活动缓存（WeakReference Cache）和 内存缓存（LruCache）。硬盘缓存就是DiskLruCache。有关缓存的链路如下：
![[Pasted image 20261001001436.png]]
活动缓存（ActiveResources）：弱引用队列（用于回收） + 弱引用缓存（Map，value为弱引用）
内存缓存（LruResourceCache）：lru缓存实现
磁盘缓存（DiskLruCacheWrapper）：lru缓存实现；缓存区分为：磁盘（结果缓存，ResourceCacheGenerator）、磁盘（原始数据缓存，DataCacheGenerator），两者共用一个Map，根据key的组成来区分（ResourceCacheGenerator、DataCacheGenerator中的startNext方法可以看出获取缓存时key的区别）。如果不存在则访问网络/文件（原始来源，SourceGenerator）。
- 结果缓存：比如请求一张图片，指定了 `centerCrop` + 圆形变换，最终生成的是一张**已经裁剪成圆形、尺寸已经适配目标 View 的 Bitmap**。这个 Bitmap 被编码后存进磁盘，就是结果缓存。
- 原始数据缓存：比如从网络下载了一张 2000×2000 的 JPEG，Glide 把**这个 JPEG 的原始字节**存进磁盘。这就是原始数据缓存。
- 来源缓存：严格意义上，就算图片文件不是在服务器上而是在设备上，只要图片对应的文件目录不属于 Glide 管理的范围，那么也是外部来源，Glide 中定义了 5 种数据源，分别是 `LOCAL`、`REMOTE`、`DATA_DISK_CACHE`、`RESOURCE_DISK_CACHE` 和 `MEMORY_CACHE` ，其中 LOCAL 和 REMOTE 就是`外部来源`，REMOTE 数据源一般就是由 `HttpGlideUrlLoader` 来加载，而 LOCAL 数据源则是由 UriLoader 和 MediaStoreFileLoader 等 ModelLoader 来加载。