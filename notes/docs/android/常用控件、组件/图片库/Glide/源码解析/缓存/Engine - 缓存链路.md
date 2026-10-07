```java
// 入口方法
public <R> LoadStatus load(...) {
    long startTime = VERBOSE_IS_LOGGABLE ? LogTime.getLogTime() : 0;
	// 缓存key
    EngineKey key = keyFactory.buildKey(model,signature,width,height,
	    transformations,resourceClass,transcodeClass,options);

    EngineResource<?> memoryResource;
    synchronized (this) {
      memoryResource = loadFromMemory(key, isMemoryCacheable, startTime);

      if (memoryResource == null) {
        return waitForExistingOrStartNewJob(...);
      }
    }

    // Avoid calling back while holding the engine lock, doing so makes it easier for callers to
    // deadlock.
    cb.onResourceReady(
        memoryResource, DataSource.MEMORY_CACHE, /* isLoadedFromAlternateCacheKey= */ false);
    return null;
  }

// 缓存链路
@Nullable
private EngineResource<?> loadFromMemory(
      EngineKey key, boolean isMemoryCacheable, long startTime) {
    if (!isMemoryCacheable) {
      return null;
    }

    EngineResource<?> active = loadFromActiveResources(key);
    if (active != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from active resources", startTime, key);
      }
      if (Log.isLoggable(GLIDE_MEMORY_TRACKING_TAG, Log.DEBUG)) {
        logCacheHit("active", key, active);
      }
      return active;
    }

    EngineResource<?> cached = loadFromCache(key);
    if (cached != null) {
      if (VERBOSE_IS_LOGGABLE) {
        logWithTimeAndKey("Loaded resource from cache", startTime, key);
      }
      if (Log.isLoggable(GLIDE_MEMORY_TRACKING_TAG, Log.DEBUG)) {
        logCacheHit("cache", key, cached);
      }
      return cached;
    }

    return null;
}

// 活动缓存
@Nullable
private EngineResource<?> loadFromActiveResources(Key key) {
    EngineResource<?> active = activeResources.get(key);
    if (active != null) {
      active.acquire();
    }

    return active;
}

// lru缓存
private EngineResource<?> loadFromCache(Key key) {
    EngineResource<?> cached = getEngineResourceFromCache(key);
    if (cached != null) {
      cached.acquire();
      activeResources.activate(key, cached);
    }
    return cached;
}
```
