1、清理机制
- `get()` 里的懒清理：在访问到`Map`中`value`（`ResourceWeakReference`）的属性（`EngineResource`）时，如果为空，则调用`cleanupActiveReference`进行清理。（如果某个资源被回收后，再也没人用同样的 key 去查，这条记录就会**永远留在 `activeEngineResources` 这个 Map 里**，造成内存泄漏。）
-  `Executor`**持续监听 `ReferenceQueue`，调用`cleanReferenceQueue`处理已经被 GC 回收的弱引用**。`resourceReferenceQueue.remove()`是阻塞的，只有`EngineResource`入队，才会唤醒，类似`Object.notifyAll()`，没有性能影响。
2、弱引用
- 把"外部是否还在使用"这个判断**交给了 GC**，而不是靠人工维护的计数或轮询。GC 是唯一能准确知道"这个对象还有没有强引用"的权威。
- 引用链：`Map`的`value`强引用`ResourceWeakReference`，`ResourceWeakReference`弱引用`EngineResource`（会被GC回收），`EngineResource`强引用`Bitmap`等底层资源
3、内存泄漏
- `ResourceWeakReference` 里可能还持有 `Resource<?> resource` 的**强引用**（当 `isActiveResourceRetentionAllowed=true` 时），这意味着**底层 Bitmap 也被这条链强引用着，无法回收**。
- 上面提到的`Map`强引用`value`
```java
package com.bumptech.glide.load.engine;

import android.os.Process;
import androidx.annotation.NonNull;
import androidx.annotation.Nullable;
import androidx.annotation.VisibleForTesting;
import com.bumptech.glide.load.Key;
import com.bumptech.glide.load.engine.EngineResource.ResourceListener;
import com.bumptech.glide.util.Executors;
import com.bumptech.glide.util.Preconditions;
import com.bumptech.glide.util.Synthetic;
import java.lang.ref.ReferenceQueue;
import java.lang.ref.WeakReference;
import java.util.HashMap;
import java.util.Map;
import java.util.concurrent.Executor;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.ThreadFactory;

final class ActiveResources {
  private final boolean isActiveResourceRetentionAllowed;
  
  // 单线程后台执行器，唯一职责就是运行cleanReferenceQueue()，也就是持续监听 ReferenceQueue，处理已经被 GC 回收的弱引用，详见类构造方法。
  private final Executor monitorClearedResourcesExecutor;
  
  // 缓存实现
  @VisibleForTesting final Map<Key, ResourceWeakReference> activeEngineResources = new HashMap<>();
  
  // 弱引用队列
  private final ReferenceQueue<EngineResource<?>> resourceReferenceQueue = new ReferenceQueue<>();

  private ResourceListener listener;

  private volatile boolean isShutdown;
  @Nullable private volatile DequeuedResourceCallback cb;

  ActiveResources(boolean isActiveResourceRetentionAllowed) {
    this(
        isActiveResourceRetentionAllowed,
        java.util.concurrent.Executors.newSingleThreadExecutor(
            new ThreadFactory() {
              @Override
              public Thread newThread(@NonNull final Runnable r) {
                return new Thread(
                    new Runnable() {
                      @Override
                      public void run() {
                        Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
                        r.run();
                      }
                    },
                    "glide-active-resources");
              }
            }));
  }

  @VisibleForTesting
  ActiveResources(
      boolean isActiveResourceRetentionAllowed, Executor monitorClearedResourcesExecutor) {
    this.isActiveResourceRetentionAllowed = isActiveResourceRetentionAllowed;
    this.monitorClearedResourcesExecutor = monitorClearedResourcesExecutor;

    monitorClearedResourcesExecutor.execute(
        new Runnable() {
          @Override
          public void run() {
            cleanReferenceQueue();
          }
        });
  }

  void setListener(ResourceListener listener) {
    synchronized (listener) {
      synchronized (this) {
        this.listener = listener;
      }
    }
  }

  synchronized void activate(Key key, EngineResource<?> resource) {
    ResourceWeakReference toPut =
        new ResourceWeakReference(
            key, resource, resourceReferenceQueue, isActiveResourceRetentionAllowed);

    ResourceWeakReference removed = activeEngineResources.put(key, toPut);
    if (removed != null) {
      removed.reset();
    }
  }

  synchronized void deactivate(Key key) {
    ResourceWeakReference removed = activeEngineResources.remove(key);
    if (removed != null) {
      removed.reset();
    }
  }

  @Nullable
  synchronized EngineResource<?> get(Key key) {
    ResourceWeakReference activeRef = activeEngineResources.get(key);
    if (activeRef == null) {
      return null;
    }

    EngineResource<?> active = activeRef.get();
    if (active == null) {
      cleanupActiveReference(activeRef);
    }
    return active;
  }

  @SuppressWarnings({"WeakerAccess", "SynchronizeOnNonFinalField"})
  @Synthetic
  void cleanupActiveReference(@NonNull ResourceWeakReference ref) {
    synchronized (this) {
      activeEngineResources.remove(ref.key);

      if (!ref.isCacheable || ref.resource == null) {
        return;
      }
    }

    EngineResource<?> newResource =
        new EngineResource<>(
            ref.resource,
            /* isMemoryCacheable= */ true,
            /* isRecyclable= */ false,
            ref.key,
            listener);
    listener.onResourceReleased(ref.key, newResource);
  }

  @SuppressWarnings("WeakerAccess")
  @Synthetic
  void cleanReferenceQueue() {
    while (!isShutdown) {
      try {
        ResourceWeakReference ref = (ResourceWeakReference) resourceReferenceQueue.remove();
        cleanupActiveReference(ref);

        // This section for testing only.
        DequeuedResourceCallback current = cb;
        if (current != null) {
          current.onResourceDequeued();
        }
        // End for testing only.
      } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
      }
    }
  }

  @VisibleForTesting
  void setDequeuedResourceCallback(DequeuedResourceCallback cb) {
    this.cb = cb;
  }

  @VisibleForTesting
  interface DequeuedResourceCallback {
    void onResourceDequeued();
  }

  @VisibleForTesting
  void shutdown() {
    isShutdown = true;
    if (monitorClearedResourcesExecutor instanceof ExecutorService) {
      ExecutorService service = (ExecutorService) monitorClearedResourcesExecutor;
      Executors.shutdownAndAwaitTermination(service);
    }
  }

  @VisibleForTesting
  static final class ResourceWeakReference extends WeakReference<EngineResource<?>> {
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final Key key;

    @SuppressWarnings("WeakerAccess")
    @Synthetic
    final boolean isCacheable;

    @Nullable
    @SuppressWarnings("WeakerAccess")
    @Synthetic
    Resource<?> resource;

    @Synthetic
    @SuppressWarnings("WeakerAccess")
    ResourceWeakReference(
        @NonNull Key key,
        @NonNull EngineResource<?> referent,
        @NonNull ReferenceQueue<? super EngineResource<?>> queue,
        boolean isActiveResourceRetentionAllowed) {
      super(referent, queue);
      this.key = Preconditions.checkNotNull(key);
      this.resource =
          referent.isMemoryCacheable() && isActiveResourceRetentionAllowed
              ? Preconditions.checkNotNull(referent.getResource())
              : null;
      isCacheable = referent.isMemoryCacheable();
    }

    void reset() {
      resource = null;
      clear();
    }
  }
}
```