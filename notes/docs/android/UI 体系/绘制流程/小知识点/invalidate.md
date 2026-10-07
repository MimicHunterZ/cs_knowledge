> `invalidate` 是 `View` 类的一个核心方法，用于**通知系统当前视图的内容已失效，需要在下一个绘制周期重新绘制**。它是自定义 View 实现动态效果（如动画、数据更新）的基础手段之一。

### 方法签名与重载

`invalidate` 定义在 `android.view.View` 类中，常见形式有以下几种：

| 方法 | 说明 | 状态 |
|---|---|---|
| `void invalidate()` | 使整个 View 失效，触发 `onDraw()` 重绘。 | 推荐使用 |
| `void invalidate(Rect dirty)` | 指定需要重绘的脏矩形区域。 | 已废弃（API 28） |
| `void invalidate(int l, int t, int r, int b)` | 指定需要重绘的矩形区域。 | 已废弃（API 28） |

两个带参数的版本在 API 28 被废弃，原因是[[硬件加速渲染]]在 API 14 引入后，脏矩形的重要性大大降低；**从 API 21 开始，传入的矩形区域会被完全忽略**，系统内部会自行计算需要重绘的区域。因此官方建议直接使用无参的 `invalidate()`。

### 线程规则：必须在 UI 线程调用

`invalidate()` 有一个严格的限制：**只能在 UI 线程（主线程）中调用**。如果在工作线程中直接调用，会抛出 `CalledFromWrongThreadException`。如果需要从非 UI 线程刷新视图，应改用 `postInvalidate()`，它会将刷新操作封装成消息，通过 `Handler` 切换到 UI 线程后再执行 `invalidate()`。

### 与 `postInvalidate()` 的对比

| | `invalidate()` | `postInvalidate()` |
|---|---|---|
| **调用线程** | 必须为 UI 线程 | 可在任意线程调用 |
| **执行方式** | 同步标记失效，等待下一帧重绘 | 通过 `ViewRootImpl` 的 `Handler` 发送消息，在 UI 线程中执行 `invalidate()` |
| **适用场景** | 已在 UI 线程中更新视图 | 在工作线程中需要刷新 UI |

`postInvalidate()` 的本质就是将 `invalidate` 操作“投递”到 UI 线程的消息队列中，因此它最终调用的仍然是 `invalidate()`。

### 工作原理：不只是“重绘”

`invalidate()` 并不直接触发 `onDraw()` 的立即调用。它的实际作用是**将 View 标记为“脏”（dirty）**，并请求父容器进行重绘调度。流程大致如下：

1. 调用 `invalidate()` 后，View 被标记为需要重绘。
2. 请求会沿着 View 树向上传递，最终到达 `ViewRootImpl`。
3. `ViewRootImpl` 在下一个垂直同步（VSync）信号到来时，统一执行 `performTraversals()`，完成测量、布局和绘制流程。
4. 在绘制阶段，系统只重绘那些被标记为“脏”的 View，而非整个视图树，这是一种性能优化。

### 与 `requestLayout()` 的区别

- **`invalidate()`**：仅触发 **重绘（draw）**，不会重新测量（measure）或重新布局（layout）。适用于视图外观变化但尺寸和位置不变的情况，如改变文字颜色、进度条进度。
- **`requestLayout()`**：触发 **完整的测量、布局、绘制** 流程。适用于视图的尺寸或位置发生变化的情况，如改变 View 的宽高、边距。

如果 View 的边界和外观都变了，通常需要同时调用两者（或仅调用 `requestLayout()`，因为布局流程本身就包含绘制）。

### 相关辅助方法

除了 `invalidate()` 本身，`View` 还提供了几个相关的失效方法：

- **`invalidateDrawable(Drawable)`**：使指定的 Drawable 失效，触发其重绘。
- **`invalidateOutline()`**：重建 View 的 Outline（轮廓），用于阴影、裁剪等效果。当 View 的轮廓提供者发生变化时需要调用。

### 典型使用场景

- **自定义 View 中数据变化**：例如一个自定义的柱状图，数据更新后调用 `invalidate()` 让图表重新绘制。
- **动画驱动**：在 `ValueAnimator` 的更新回调中调用 `invalidate()`，逐帧改变视图外观。
- **触摸反馈**：在 `onTouchEvent` 中根据触摸位置改变绘制内容，调用 `invalidate()` 刷新高亮区域。
- **属性动画或状态切换**：改变 View 的绘制属性（如 alpha、颜色过滤器）后，系统通常会自动触发 `invalidate()`，但手动调用可以确保刷新时机可控。