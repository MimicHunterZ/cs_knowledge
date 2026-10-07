> ViewRootImpl 是 **View 树与 WindowManagerService（WMS）之间的桥梁**，也是**每次 UI 遍历（measure / layout / draw）和输入事件分发的总调度器**。它自己不是 View，但你所有 View 的 `requestLayout()`、`invalidate()` 最终都会走它。

源码位置：`frameworks/base/core/java/android/view/ViewRootImpl.java`

---

## 在体系中的位置

```
ActivityThread / WindowManagerGlobal
        │  addView(decorView, params)
        ▼
   ViewRootImpl  ──── mWindow (IWindow.Stub, 即 ViewRootImpl$W) ──► WMS
        │        ◄─── mWindowSession (IWindowSession) ────────────  WMS
        │
        │  mView (DecorView)   ← 它实现 ViewParent，是 DecorView 的 parent
        ▼
   DecorView → ViewGroup → ... → View
```

关键类签名：

```java
public final class ViewRootImpl implements
        ViewParent,                          // 所以能当 DecorView 的 parent
        View.AttachInfo.Callbacks,
        ThreadedRenderer.DrawCallbacks,      // 硬件加速绘制回调
        Window.OnWindowDismissedCallback,
        AttachedSurfaceControl
```

对应的成员：

| 成员 | 含义 |
|---|---|
| `mView` | 顶层 DecorView |
| `mWindow` | `IWindow.Stub` 的 Binder 对象（`ViewRootImpl$W`），WMS 通过它回调 App |
| `mWindowSession` | `IWindowSession`，App 向 WMS 发请求的通道 |
| `mAttachInfo` | 分发给整棵 View 树的上下文（窗口位置、可见性、Insets、HardwareRenderer…） |
| `mChoreographer` | 接收 vsync，驱动 `mTraversalRunnable` |
| `mHandler` | `ViewRootHandler`（主线程），处理 WMS 回调消息 |
| `mSurface` | 这个窗口的画布 |
| `mInputChannel` / `mInputEventReceiver` | 接收 InputDispatcher 派发的输入事件 |

---

## 生命周期

1. **创建**：`WindowManagerGlobal.addView()` 里 `new ViewRootImpl(context, display)`，然后 `root.setView(decorView, params, panelParentView)`。
2. **setView()** 里做的事：
   - `mAdded = true`，`requestLayout()`
   - `session.addToDisplay(mWindow, ..., mInputChannel)` → **WMS 真正创建窗口**，并把 `InputChannel` 的接收端回传
   - `mView.dispatchAttachedToWindow(mAttachInfo, 0)` → 整棵树 attach
   - 准备 `mChoreographer`、`mHandler`、输入阶段链
3. **运行期**：反复 `performTraversals()` + 输入分发。
4. **销毁**：`WindowManagerGlobal.removeView()` → `ViewRootImpl.die()` → `session.remove(mWindow)` → `doDie()` → `mView.dispatchDetachedFromWindow()`，从 `mRoots` 移除。

> **一个窗口 = 一个 ViewRootImpl**。Activity 主窗口、每个 Dialog、每个 PopupWindow/Toast（走 WindowManager 的那些）各自持有一个。

---

## 职责一：View 遍历（performTraversals）

这是它的核心，一次遍历大致是：

```java
performTraversals() {
    relayoutWindow(...)        // 问 WMS：我的 frame / insets / surface 变了吗？
    // 1. measure
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
    // 2. layout
    mView.layout(0, 0, childWidth, childHeight);
    // 3. 回调 ViewTreeObserver
    dispatchOnGlobalLayout();
    if (dispatchOnPreDraw()) { // OnPreDrawListener 返回 false 可取消这次绘制
        // 4. draw
        performDraw();
    }
}
```

**触发方式**：View 树里任何人调 `requestLayout()` / `invalidate()` → 沿 `mParent` 往上冒泡 → 到 ViewRootImpl：

- `requestLayout()` → `scheduleTraversals()`
- `invalidateChildInParent()` → 记 `mDirty` 矩形 → `scheduleTraversals()`

**scheduleTraversals() 的两个精髓**：

```java
mTraversalScheduled = true;
mHandler.getLooper().getQueue().postSyncBarrier();   // ① 同步屏障
mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null); // ② 等 vsync
```

- **[[同步屏障]]**：让之后的主线程消息（普通同步消息）先别执行，**优先把这次 traversal 跑完**，降低延迟。(todo: messageQueue 补充)
- **Choreographer**：把遍历对齐到 vsync，一帧内多次 `requestLayout` 会被**合并成一次**遍历（`mTraversalScheduled` 去重）。

到了 vsync：`mTraversalRunnable → doTraversal() → performTraversals() → unscheduleTraversals()`（移除屏障）。

**布局期间再 requestLayout**：会有 `mLayoutRequesters`（HandlerActionQueue）暂存，遍历结束后再补偿执行；如果已经处于 layout 中，会 `requestLayoutDuringLayout`，这就是有时打日志 "requestLayout() improperly called ... during layout" 的来源。

**measure 的根 MeasureSpec**（`getRootMeasureSpec`）：

- `MATCH_PARENT` → `EXACTLY`（填满父容器可用空间）
- `WRAP_CONTENT` → `AT_MOST`（尺寸由内容决定，自适应内容，但不会超过父容器允许的范围）
- 固定值 → `EXACTLY`

**draw**：

- 硬件加速：`mAttachInfo.mThreadedRenderer.draw(...)` → RenderThread 真正渲染
- 软件绘制：`drawSoftware()` → `mSurface.lockCanvas(mDirty)` → `mView.draw(canvas)` → `unlockCanvasAndPost()`
- 现代版本还有 `reportNextDraw()` / `finishDrawing()`，用于和 WMS 做绘制同步（Activity 转场、`reportFullyDrawn` 依赖这个）

> 具体遍历流程查看：[[绘制流程|here]]

---

## 职责二：输入事件

ViewRootImpl 自身实现 `InputEventReceiver`，从 `mInputChannel` 收事件，然后走一条**责任链（InputStage）**：

```
NativePreImeInputStage
   → ViewPreImeInputStage        (View 的 onKeyPreIme)
      → ImeInputStage            (输入法)
         → EarlyPostImeInputStage
            → NativePostImeInputStage
               → ViewPostImeInputStage   (dispatchTouchEvent / dispatchKeyEvent / hover / scroll)
                  → SyntheticInputStage  (摇杆/轨迹球)
                     → fallback: mFallbackEventHandler (返回键、媒体键等)
```

- 触摸/按键事件最终落到 `mView.dispatchPointerEvent()` / `dispatchKeyEvent()`，即读者熟悉的 `onTouchEvent` / `onKeyDown`。
- 触摸模式变化（`isInTouchMode`）由它感知并 `dispatchOnTouchModeChanged`。
- 输入分发超时 / ANR 判定，也和这个链路相关。

---

## 职责三：与 WMS 的双向通信

**App → WMS**（通过 `mWindowSession`）：
`addToDisplay`、`relayout`、`remove`、`setInsets`、`setInputFocus`、`finishDrawing`、`performHapticFeedback`…

**WMS → App**（通过 `mWindow`，即 `ViewRootImpl$W`，最终变成 `ViewRootHandler` 消息）：

| IWindow 回调 | ViewRootImpl 处理 |
|---|---|
| `resized(...)` | `MSG_RESIZED` → 更新 `mWinFrame`/Insets → 再 `requestLayout` |
| `dispatchAppVisibility()` | `MSG_DISPATCH_APP_VISIBILITY` → 窗口可见性 |
| `dispatchDetachedFromWindow()` | `MSG_DISPATCH_DETACHED_FROM_WINDOW` → 窗口被移除 |
| `moved()` | `MSG_WINDOW_MOVED` |
| `dispatchScreenState()` | 屏幕开关 |
| `dispatchWallpaperCommand()` | 壁纸命令 |

其它它统一管理的横切能力：

- **WindowFocus**：`mAttachInfo.mHasWindowFocus`，`mView.dispatchWindowFocusChanged()`
- **Insets**：`mAttachInfo.mContentInsets/mStableInsets`，`dispatchApplyWindowInsets()`；Android 10+ 内置 `InsetsController`（状态栏/导航栏/IME 动画）
- **配置变化**：`updateConfiguration()` → `dispatchConfigurationChanged()`
- **无障碍**：`AccessibilityInteractionController`
- **Activity 转场**：`mPendingTransitions`、`setPausedForTransition()`
- **WindowTreeObserver** 的 `OnScrollChanged`、`OnGlobalLayout`、`OnPreDraw` 派发
- **硬件加速开关**：`enableHardwareAcceleration()`、`mAttachInfo.mThreadedRenderer`

---

## 注意点
1. **线程约束**：ViewRootImpl 记住创建时的 `mThread`，`checkThread()` 在 `requestLayout` / `invalidate` 等处校验 , 如果不匹配会有如下报错。
   ```
   android.view.ViewRootImpl$CalledFromWrongThreadException:
   Only the original thread that created a view hierarchy can touch its views.
   ```
   注意：它**不是**在 `setText` 之类的 setter 里检查，而是在真的要走布局/绘制流程时才检查——所以偶发时看起来"有时崩有时不崩"。
> 这里涉及到 [[UI是否只能在主线程修改这个问题]] ，因为：创建ViewRootImpl的子线程也能更新UI

2. **它是 requestLayout 的终点**：`View.requestLayout()` 一路 `mParent.requestLayout()`，DecorView 的 `mParent` 就是 ViewRootImpl。

3. **它不画 View，也不测量 View**：它只提供 MeasureSpec、调用入口、Surface 和 vsync 时机；算法都在 View/ViewGroup 里。

4. **常见坑**：
   - `ViewRootImpl: The specified message queue synchronization barrier token has not been posted or has already been removed` —— [[同步屏障]]管理异常（一般是遍历被多次调度/取消）。相关博客： https://juejin.cn/post/7310571761044160575> 
   - `ViewRootImpl: sendUserActionEvent() returned.` —— detach 相关。
   - **泄漏**：ViewRootImpl 持有 `mView` → Context（Activity），若窗口没正确移除（Dialog 没 dismiss、Toast 未结束等），就会漏 Activity。LeakCanary 里经常看到 `ViewRootImpl$W` / `DecorView` 路径。