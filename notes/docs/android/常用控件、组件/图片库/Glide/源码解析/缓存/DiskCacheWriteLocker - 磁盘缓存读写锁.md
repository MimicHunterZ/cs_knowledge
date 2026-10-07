锁对象池 + 加锁时机：
- `WriteLockPool`中维护了一个优先队列，存放被回收的锁对象（锁对象是通用的，`obtain`方法中调用`pool.poll()`），用于复用
- `acquire()`和`release()`方法中都加了两次锁，一次为`synchronized`用于引用计数，另一次为`ReentrantLock`用于保护真正的读写操作。（事实上`synchronized`中的操作无法用`ConcurrentHashMap`替代，因为是复合逻辑。可以加更细粒度的锁，但是影响不大）

```java
package com.bumptech.glide.load.engine.cache;

import com.bumptech.glide.util.Preconditions;
import com.bumptech.glide.util.Synthetic;
import java.util.ArrayDeque;
import java.util.HashMap;
import java.util.Map;
import java.util.Queue;
import java.util.concurrent.locks.Lock;
import java.util.concurrent.locks.ReentrantLock;

/**
 * Keeps a map of keys to locks that allows locks to be removed from the map when no longer in use
 * so the size of the collection is bounded.
 *
 * <p>This class will be accessed by multiple threads in a thread pool and ensures that the number
 * of threads interested in each lock is updated atomically so that when the count reaches 0, the
 * lock can safely be removed from the map.
 */
final class DiskCacheWriteLocker {
  private final Map<String, WriteLock> locks = new HashMap<>();
  private final WriteLockPool writeLockPool = new WriteLockPool();

  void acquire(String safeKey) {
    WriteLock writeLock;
    synchronized (this) {
      writeLock = locks.get(safeKey);
      if (writeLock == null) {
        writeLock = writeLockPool.obtain();
        locks.put(safeKey, writeLock);
      }
      writeLock.interestedThreads++;
    }

    writeLock.lock.lock();
  }

  void release(String safeKey) {
    WriteLock writeLock;
    synchronized (this) {
      writeLock = Preconditions.checkNotNull(locks.get(safeKey));
      if (writeLock.interestedThreads < 1) {
        throw new IllegalStateException(
            "Cannot release a lock that is not held"
                + ", safeKey: "
                + safeKey
                + ", interestedThreads: "
                + writeLock.interestedThreads);
      }

      writeLock.interestedThreads--;
      if (writeLock.interestedThreads == 0) {
        WriteLock removed = locks.remove(safeKey);
        if (!removed.equals(writeLock)) {
          throw new IllegalStateException(
              "Removed the wrong lock"
                  + ", expected to remove: "
                  + writeLock
                  + ", but actually removed: "
                  + removed
                  + ", safeKey: "
                  + safeKey);
        }
        writeLockPool.offer(removed);
      }
    }

    writeLock.lock.unlock();
  }

  private static class WriteLock {
    final Lock lock = new ReentrantLock();
    // 引用计数
    int interestedThreads;

    @Synthetic
    WriteLock() {}
  }

  private static class WriteLockPool {
    private static final int MAX_POOL_SIZE = 10;
    private final Queue<WriteLock> pool = new ArrayDeque<>();

    @Synthetic
    WriteLockPool() {}

    WriteLock obtain() {
      WriteLock result;
      synchronized (pool) {
        result = pool.poll();
      }
      if (result == null) {
        result = new WriteLock();
      }
      return result;
    }

    void offer(WriteLock writeLock) {
      synchronized (pool) {
        if (pool.size() < MAX_POOL_SIZE) {
          pool.offer(writeLock);
        }
      }
    }
  }
}
```