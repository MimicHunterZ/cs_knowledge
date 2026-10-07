> `DecorView` 是 `PhoneWindow` 的一个**内部类**（`android.internal.policy.PhoneWindow.DecorView`），直接继承 `FrameLayout`。

它是**窗口（Window）这个抽象概念在 View 层的载体**：每个 `Window`（Activity 的、Dialog 的、PopupWindow 底层的）都持有一个 DecorView，它是这棵 View 树的根节点，也是 `WindowManager.addView()` 被传入的那个 View。

关键点：**DecorView 本身不是"窗口"，它只是窗口里的顶层 View。** 真正让它变成一个系统窗口的是 `WindowManager.LayoutParams` + `ViewRootImpl` + WMS 里的 `WindowState`。

```
Activity
 └── Window (抽象类，唯一实现 PhoneWindow)
      └── DecorView (FrameLayout)          ← View 树的根
           └── ... 你的布局 (android.R.id.content)
      
ViewRootImpl  ──(mView 指向)──> DecorView   ← ViewRootImpl 不是 View，是 ViewParent
```

## 它的内部布局结构

`PhoneWindow.installDecor()` 会 new 一个 DecorView，然后 `generateLayout()` 根据主题属性 `windowLayout` 和 feature 标志（`FEATURE_ACTION_BAR`、`FEATURE_NO_TITLE`、`FEATURE_ACTION_MODE_OVERLAY` 等）从一批 framework 布局里挑一个 inflate 进去。

最典型的 `screen_simple.xml`（NoActionBar 场景）：

```xml
<LinearLayout orientation="vertical">          <!-- mContentRoot -->
    <ViewStub android:id="@+id/action_mode_bar_stub" />
    <FrameLayout android:id="@android:id/content" />   <!-- mContentParent -->
</LinearLayout>
```

带 ActionBar 时是 `screen_action_bar.xml`：

```xml
<ActionBarOverlayLayout>
    <LinearLayout orientation="vertical">       <!-- mContentRoot -->
        <DecorContentParent (ActionBarContainer)>  <!-- 装 ActionBar -->
        <ViewStub id="action_mode_bar_stub"/>
        <FrameLayout id="android:id/content"/>      <!-- mContentParent -->
    </LinearLayout>
</ActionBarOverlayLayout>
```

所以完整的层级大致是：

```
DecorView (FrameLayout)
 ├── [可选的 DecorCaptionView]              // 自由窗口/多窗口的标题栏
 ├── LinearLayout / ActionBarOverlayLayout  // mContentRoot
 │    ├── ActionBarContainer / title 容器
 │    ├── ViewStub(action_mode_bar_stub)    // ActionMode 的上下文栏
 │    └── FrameLayout(android:id/content)   // mContentParent ← setContentView 目标
 └── (绘制层) BarColorView                  // 状态栏/导航栏背景色块
```

`setContentView()` 最终就是 `mLayoutInflater.inflate(layoutResID, mContentParent)`，所以你在布局里用 `findViewById(android.R.id.content)` 就能拿到"你自己写的布局的父容器"。

## 创建与生命周期

1. `Activity.attach()` → `mWindow = new PhoneWindow(this, ...)`
2. `setContentView()` → `PhoneWindow.setContentView()` → `installDecor()`
   - `mDecor = new DecorView(getContext(), featureId, this, params)`
   - `generateLayout(mDecor)` → inflate 上面的 framework 布局
   - 找 `mContentParent`、`mDecorContentParent`，应用主题里的 `windowBackground`、`windowTitle`
3. `ActivityThread.handleResumeActivity()` → `wm.addView(mDecor, l)`
4. `WindowManagerGlobal.addView()` → 创建 `ViewRootImpl` → `root.setView(decor, params, panelParent)`
   - 这一步 DecorView 才被 attach，触发了 `onAttachedToWindow()`
5. `ViewRootImpl.performTraversals()` 完成第一次 measure/layout/draw
6. `Activity.makeVisible()` → `mDecor.setVisibility(VISIBLE)`
7. 销毁时 `WindowManagerGlobal.removeView(decor)` → `dispatchDetachedFromWindow()`

顺序很重要：**在 `onResume` 完成、`addView` 之前，DecorView 是没有 `mAttachInfo`、没有真实 Surface、拿不到窗口尺寸的。**

## 四、DecorView 承担的特殊职责

它不只是个容器，还是一大堆窗口级行为的落点：

**1. 输入事件与 Window.Callback 转发**
- `dispatchKeyEvent` → 先给 `mWindow.getCallback()`（Activity）处理
- `onTouchEvent`：手指按在内容区外的"边界"上时，会尝试关闭 popup / panel，或触发 `startMovingTask()`（自由窗口拖动）
- `onWindowFocusChanged` → 转发给 `Activity.onWindowFocusChanged`

**2. Insets（窗口内边距）的分发**
- 系统状态栏/导航栏/输入法的高度通过 `WindowInsets` 从 `ViewRootImpl` 往下传
- DecorView 的 `onApplyWindowInsets` 决定内容是否"避开"系统栏；`fitsSystemWindows`、`decorFitsSystemWindows` 都在这一层生效

**3. 系统栏颜色与可见性**
- `window.setStatusBarColor()` / `setNavigationBarColor()` 的实现，是设置 DecorView 内部的 `BarColorView`（`mStatusColorViewState` / `mNavigationColorViewState`），在 `onDraw`/`drawColorViews` 里直接画一条颜色
- `SYSTEM_UI_FLAG_FULLSCREEN`、`LAYOUT_FULLSCREEN`、`LAYOUT_HIDE_NAVIGATION`、`LIGHT_STATUS_BAR` 等最终都作用在 DecorView 上

**4. ActionBar / ActionMode**
- `mDecorContentParent` 指向 ActionBar 容器，`startActionMode`、`showContextMenuForChild`、`onWindowStartingActionMode` 都从这里走

**5. 窗口背景**
- 主题的 `windowBackground` 实际是设成 DecorView 的 drawable，这就是为什么启动时先看到一片纯色


一句话：**Activity 是控制器，PhoneWindow 是窗口壳子，DecorView 是这个壳子的顶层 View，ViewRootImpl 是它和系统（WMS/SurfaceFlinger）之间的桥。**

## 版本演进

- **4.4**：`windowTranslucentStatus` 出现，状态栏可透明
- **5.0**：`setStatusBarColor()`，颜色由 DecorView 自己画
- **8.0**：`SYSTEM_UI_FLAG_LIGHT_NAVIGATION_BAR` 等陆续加齐
- **10**：`View.systemUiVisibility` 废弃，改用 `WindowInsetsController`
- **11**：`WindowInsets.Type.*` 新模型；`setDecorFitsSystemWindows()`
- **15**：**强制 edge-to-edge**，`decorFitsSystemWindows` 默认 false，`setStatusBarColor` 失效——以前那套靠 DecorView 画状态栏颜色的方案基本退场

## 调试手段

- Android Studio 的 **Layout Inspector** 直接能看到 DecorView 这层
- `adb shell dumpsys window windows` 看窗口参数和 token
- `decorView.toString()` / 递归打印可看到 `android:id/content`、`action_mode_bar_stub` 这些 framework id

> DecorView 是"窗口 = 一棵 View 树的根"这个映射的具体实现，它继承 FrameLayout 负责装内容，同时兼任输入转发、Insets 分发、系统栏颜色绘制、ActionBar/ActionMode 宿主这几件事。