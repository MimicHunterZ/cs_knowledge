## 1. Scrap 缓存：mAttachedScrap / mChangedScrap

**作用**：布局过程中临时缓存屏幕内、被 detach 的 ViewHolder。  
**特点**：不是跨滚动复用的缓存，主要服务于 `onLayoutChildren` 的布局过程。

- `mAttachedScrap`：存放**位置未变、数据未变**的 ViewHolder。复用时不重新创建、不重新绑定，直接 attach 回去。
- `mChangedScrap`：存放**发生变化**的 ViewHolder，主要在 **pre-layout** 阶段使用，比如动画前的布局。  它通常需要重新绑定或更新。

**容量**：没有固定容量，基本等于当前 attached 的 ViewHolder 数量。  
**是否重新绑定**：`mAttachedScrap` 通常不需要；`mChangedScrap` 可能需重新绑定。  
**生命周期**：一次布局过程内，布局结束后会清理。

---

## 2. mCachedViews

**作用**：缓存**最近滑出屏幕**的 ViewHolder，用于快速回滚复用。  
比如向上滑动，item 0 滑出屏幕，会先进入 `mCachedViews`；如果马上滑回来，可以直接取出，不需要重新绑定。

**默认容量**：`2`，可通过 `setItemViewCacheSize(int)` 修改。  
**是否重新绑定**：如果 position 匹配且数据未失效，**不需要重新绑定**。  
**回收规则**：  
- 滑出屏幕时，优先放入 `mCachedViews`。  
- 如果满了，移除最旧的 ViewHolder，放入 `RecycledViewPool`。  
- 从 `mCachedViews` 取时，按 position / id 匹配，匹配不到就移除并放入池中。

---

## 3. ViewCacheExtension

**作用**：开发者自定义的缓存扩展，位于 `mCachedViews` 和 `RecycledViewPool` 之间。  
**特点**：RecyclerView 不负责管理它的缓存，需要开发者自己维护。  
**接口**：`getViewForPositionAndType(Recycler recycler, int position, int type)`，返回 View 或 null。  
**是否重新绑定**：由开发者自己保证，通常返回已经绑定好的 View。  
**实际使用**：绝大多数场景不用，除非有非常特殊的复用需求。默认是 `null`。

---

## 4. RecycledViewPool

**作用**：真正的“回收池”，按 **viewType** 缓存 ViewHolder。  
当 `mCachedViews` 满了之后，ViewHolder 会被放入这里。

**默认容量**：每个 viewType 默认最多 `5` 个，可通过 `setMaxRecycledViews(int viewType, int max)` 修改。  
**是否重新绑定**：**必须重新绑定**，即会调用 `onBindViewHolder`。  
**特点**：  
- 从池中取出时会 `resetInternal()`，清除 position、flag 等信息。  
- 可以多个 RecyclerView 共享同一个 `RecycledViewPool`，通过 `setRecycledViewPool()` 设置。  
- 适合嵌套 RecyclerView 或同类型列表复用。

---

## 获取 ViewHolder 的顺序

在 `Recycler.tryGetViewHolderForPositionByDeadline()` 中大致顺序是：

1. 如果是 pre-layout，先查 `mChangedScrap`；
2. 查 `mAttachedScrap`；
3. 查 `mCachedViews`；
4. 查 `ViewCacheExtension`；
5. 查 `RecycledViewPool`；
6. 都没有，才 `createViewHolder()` 创建新的，然后 `bindViewHolder()`。

---

## 回收流程

当 ViewHolder 滑出屏幕或需要回收时：

1. 如果 ViewHolder 有效，优先放入 `mCachedViews`；
2. `mCachedViews` 满了，最旧的移入 `RecycledViewPool`；
3. `RecycledViewPool` 也满了，就丢弃。

---

## 总结

- **Scrap**：布局时临时复用，不重新绑定；
- **mCachedViews**：最近离屏缓存，默认 2 个，不重新绑定；
- **ViewCacheExtension**：自定义缓存，一般不用；
- **RecycledViewPool**：按类型回收，默认每类 5 个，必须重新绑定。

所以 RecyclerView 的缓存核心思想是：**尽量复用 ViewHolder，尽量少调用 onCreateViewHolder 和 onBindViewHolder**。