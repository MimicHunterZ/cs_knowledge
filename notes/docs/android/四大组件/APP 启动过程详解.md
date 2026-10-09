## 1. 启动分类
- **冷启动**：进程不存在，需创建新进程。最完整、最慢。
- **温启动**：进程存在，但 Activity 已销毁或被回收。
- **热启动**：进程和 Activity 均存在，仅需从后台切回前台。

## 2. 冷启动全流程 (Android 10+ 逻辑)

### 第一阶段：点击图标 (IPC 1)
1. 用户点击 [[App启动全流程：从Zygote到ActivityThread|Launcher]] 图标。
2. [[App启动全流程：从Zygote到ActivityThread|Launcher]] 进程通过 [[Binder机制：从mmap到AIDL|Binder]] 向 [[AMS与系统调度机制|SystemServer]] 进程的 [[AMS与系统调度机制|AMS]] 发送 `startActivity` 请求。

### 第二阶段：进程创建 (Socket 通信)
1. [[AMS与系统调度机制|AMS]] 检查目标进程是否存在。
2. 若不存在，[[AMS与系统调度机制|AMS]] 通过 **Socket** 向 [[App启动全流程：从Zygote到ActivityThread|Zygote]] 进程发送创建进程的请求。
3. **[[App启动全流程：从Zygote到ActivityThread|Zygote]] 孵化**：
    - [[App启动全流程：从Zygote到ActivityThread|Zygote]] 进程 fork 出子进程（App 进程）。
    - **优势**：[[App启动全流程：从Zygote到ActivityThread|Zygote]] 预加载了常用的 Java 类和资源，子进程通过“写时复制”（Copy-on-Write）技术共享这些资源，极大加快启动速度。

### 第三阶段：App 进程初始化
1. 新进程启动后，进入 `RuntimeInit`，最终调用 **[[App启动全流程：从Zygote到ActivityThread|ActivityThread]]** 的 `main()` 方法。
2. **[[App启动全流程：从Zygote到ActivityThread|ActivityThread]].main()**：
    - 创建主线程的 [[线程消息机制|Looper]] (`Looper.prepareMainLooper()`)。
    - 创建 **[[线程消息机制|Handler]]** (H 类) 用于处理系统指令。
    - 调用 `attach()` 方法向 [[AMS与系统调度机制|AMS]] 报到。

### 第四阶段：绑定 Application (IPC 2)
1. [[AMS与系统调度机制|AMS]] 通过 [[Binder机制：从mmap到AIDL|Binder]] 回调 App 进程的 `bindApplication`。
2. App 进程发送 `BIND_APPLICATION` 消息给 `H`。
3. `H` 处理消息：
    - 创建 `LoadedApk` 对象。
    - 创建 `ContextImpl`。
    - **反射创建 `Application` 实例**。
    - 调用 `Application.onCreate()`。

### 第五阶段：启动 Activity (IPC 3)
1. [[AMS与系统调度机制|AMS]] 发送 `EXECUTE_TRANSACTION` 指令。
2. App 进程通过 `ClientTransactionHandler` 执行事务。
3. 依次触发：
    - `onCreate()`：加载布局，初始化 [[Bundle与Parcelable序列化|Bundle]] 数据。
    - `onStart()`：界面可见。
    - **[[Android核心枢纽：从onResume到焦点获取|onResume]]**：界面进入前台。

## 3. 渲染与显示
1. [[Android核心枢纽：从onResume到焦点获取|onResume]] 执行完后，[[App启动全流程：从Zygote到ActivityThread|ActivityThread]] 调用 `wm.addView()`。
2. 创建 **[[ViewRootImpl：UI系统的总指挥|ViewRootImpl]]**。
3. [[ViewRootImpl：UI系统的总指挥|ViewRootImpl]] 触发 `requestLayout()`，等待 [[屏幕刷新与SurfaceFlinger合成|Vsync]] 信号进行 [[View绘制全流程|measure]]、[[View绘制全流程|layout]]、[[View绘制全流程|draw]]。
4. 最终通过 [[屏幕刷新与SurfaceFlinger合成|SurfaceFlinger]] 将画面呈现在屏幕上。