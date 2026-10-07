老版本 Glide 需要自己往 Activity/Fragment 里塞一个隐藏 Fragment，靠它的 `onStart/onStop/onDestroy` 来感知生命周期。

新版本改成了：**直接拿宿主的 `androidx.lifecycle.Lifecycle`**。`FragmentActivity` 和 `Fragment` 本身就已经实现了 `LifecycleOwner`，所以 Glide 不需要再注入任何东西，直接监听它们的 Lifecycle 即可。

这也是为什么 `RequestManagerRetriever` 里的 `get(FragmentActivity)` 和 `get(Fragment)` 都把 `activity.getLifecycle()` / `fragment.getLifecycle()` 传进来。