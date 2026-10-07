它的本质是：**父 ViewGroup 内部有一个开关，子 View 请求修改这个开关，父 ViewGroup 在分发事件时根据开关决定是否调用 `onInterceptTouchEvent()`。**

---

## 1. 核心 API

```java
parent.requestDisallowInterceptTouchEvent(true);
```

- `true`：请求父 ViewGroup 不要拦截后续事件。
- `false`：恢复父 ViewGroup 的拦截能力。

这个方法是 `ViewParent` 接口的方法，`ViewGroup` 实现了它。

---

## 2. ViewGroup 内部的标志位

`ViewGroup` 中有一个标志：

```java
FLAG_DISALLOW_INTERCEPT
```

它保存在 `mGroupFlags` 里。

当子 View 调用：

```java
parent.requestDisallowInterceptTouchEvent(true);
```

`ViewGroup` 会：

1. 设置自己的 `FLAG_DISALLOW_INTERCEPT`。
2. 递归调用自己的父容器：

```java
@Override
public void requestDisallowInterceptTouchEvent(boolean disallowIntercept) {
    if (disallowIntercept == ((mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0)) {
        return;
    }

    if (disallowIntercept) {
        mGroupFlags |= FLAG_DISALLOW_INTERCEPT;
    } else {
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    }

    if (mParent != null) {
        mParent.requestDisallowInterceptTouchEvent(disallowIntercept);
    }
}
```

所以这个标志会沿着父链一直往上传递，直到顶层 `ViewRootImpl`。

---

## 3. 父 ViewGroup 分发时怎么用这个标志？

在 `ViewGroup.dispatchTouchEvent()` 中，关键逻辑是：

```java
final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

if (!disallowIntercept) {
    intercepted = onInterceptTouchEvent(ev);
} else {
    intercepted = false;
}
```

含义：

- 如果 `disallowIntercept == true`，父 ViewGroup **直接跳过 `onInterceptTouchEvent()`**，认为不拦截。
- 如果 `disallowIntercept == false`，才会调用 `onInterceptTouchEvent()` 决定是否拦截。

所以准确来说并不是子 View “抢走”了父 View 的权力，而是让父 ViewGroup 主动跳过了拦截判断。

---

## 4. ACTION_DOWN 会重置标志

注意：`ACTION_DOWN` 时，`ViewGroup` 会重置触摸状态：

```java
if (actionMasked == MotionEvent.ACTION_DOWN) {
    cancelAndClearTouchTargets(ev);
    resetTouchState(); // 清除 FLAG_DISALLOW_INTERCEPT
}
```

因此：

- 子 View 必须在 `ACTION_DOWN` 时或之后立刻调用 `requestDisallowInterceptTouchEvent(true)`。
- 如果等到父 View 已经拦截了 `ACTION_MOVE`，再调用就晚了，子 View 会收到 `ACTION_CANCEL`。

---

## 5. 限制与注意点

1. **不能阻止父 View 在 ACTION_DOWN 时拦截**  
   因为 DOWN 会重置标志。如果父 View 对 DOWN 就返回 `true`，子 View 根本没机会。

2. **不能阻止父 View 重写 dispatchTouchEvent 强行处理**  
   如果父 View 完全重写 `dispatchTouchEvent()`，不调用 `onInterceptTouchEvent()`，或者不尊重 `requestDisallowInterceptTouchEvent()`，那这个机制就失效。

3. **父 View 可以不尊重这个请求**  
   `requestDisallowInterceptTouchEvent()` 只是请求。标准 `ViewGroup` 会尊重，但自定义父容器可以重写该方法忽略它。

4. **只影响拦截，不影响分发顺序**  
   事件仍然是从父到子分发，子 View 只是让父 View 不拦截。