---
title: Android独立窗口自绘渲染实现
date: 2025-05-03 19:19:19
categories: 
	- Android
tags: 
	- 渲染
---
# 引言
在Android中，独立窗口自绘渲染，典型应用场景比如SurfaceView与手机双边侧滑返回动画面板（WMS.addView）。最近入职一家新公司，涉及对接更底层渲染实现，具体表现在NDK层，获取一个独立Window窗口，上层用Skia进行绘制，并在Android系统中渲染出来。本文旨在分析WMS.addView链路，明淅渲染关键路径，为后面自定义渲染作支持。
<!-- more -->
# 核心类说明

在 Android 系统中，独立窗口自绘渲染，绕不开`Surface` 和 `ANativeWindow` ，二者是与图形渲染和窗口管理相关的核心类，它们的关系和功能可以总结如下：

## **类结构解释**
```cpp
class Surface : public ANativeObjectBase<ANativeWindow, Surface, RefBase>{

}
```
- **继承关系**：  
  `Surface` 继承自模板类 `ANativeObjectBase<ANativeWindow, Surface, RefBase>`。
- **模板参数含义**：
  - **`ANativeWindow`**：表示底层的原生窗口接口。
  - **`Surface`**：子类自身（使用 CRTP 模式，允许基类调用子类方法）。
  - **`RefBase`**：Android 的引用计数基类，用于对象生命周期管理。

---

## **Surface 的作用**
- **图形渲染的抽象层**：  
  `Surface` 是 Android 应用与显示系统之间的桥梁，代表一个可绘制的表面。应用程序通过 `Surface` 进行 UI 渲染（例如通过 Canvas 或 OpenGL ES）。
- **跨进程通信**：  
  `Surface` 实现了 `Parcelable` 接口，可跨进程传递（如从应用进程传递到 SurfaceFlinger 服务进程）。
- **缓冲区管理**：  
  管理图形缓冲区队列（`BufferQueue`），协调生产者和消费者（如应用和 SurfaceFlinger）之间的缓冲区交换。

---

## **ANativeWindow 的作用**
- **原生窗口的 C 接口**：  
  `ANativeWindow` 是 Android NDK 中定义的抽象，提供对底层窗口系统的访问（如通过 `ANativeWindow_fromSurface()` 获取）。
- **跨平台兼容性**：  
  封装了不同硬件/平台的窗口操作（例如设置缓冲区大小、格式，提交渲染结果）。
- **与 Surface 的关系**：  
  `ANativeWindow` 是 `Surface` 的底层接口，`Surface` 类通过继承 `ANativeObjectBase` 实现了 `ANativeWindow` 的功能。

---

## **两者关系**
1. **继承与封装**：  
   `Surface` 是 `ANativeWindow` 的高层封装，提供更易用的 C++ API，而 `ANativeWindow` 是底层的 C 风格接口。
2. **功能实现**：  
   `Surface` 通过 `ANativeObjectBase` 模板类继承 `ANativeWindow` 的接口，并实现其方法（如 `dequeueBuffer`/`queueBuffer`），最终通过 `RefBase` 管理生命周期。
3. **使用场景**：
   - **Java/Kotlin 层**：通过 `Surface` 类进行 UI 渲染。
   - **Native 层（NDK）**：通过 `ANativeWindow` 直接操作窗口（例如 Vulkan/OpenGL ES 渲染）。


---

## **总结**
- **`ANativeWindow`**：底层原生窗口接口，提供跨平台的窗口操作（NDK 使用）。
- **`Surface`**：高层封装，整合 `ANativeWindow` 功能并提供 Android 框架级的渲染管理。
- **关系**：`Surface` 是 `ANativeWindow` 的面向对象实现，二者共同服务于图形渲染，前者用于 Java/C++ 框架层，后者用于更接近硬件的 NDK 层。

# WMS.addView自绘链路

在Android 10及以后，WMS.addView可以以侧滑返回动画面板实现代码作说明，代码主要在[systemui/navigationbar/gestural](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/packages/SystemUI/src/com/android/systemui/navigationbar/gestural/)，核心类EdgeBackGestureHandler、BackPanelController、BackPanel，这三个组件共同构成了 Android 的边缘返回手势系统，提供流畅的用户体验，是一个WMS.addView应用场景。
其中ViewCaptureAwareWindowManager以一个独立的Window添加BackPanel（View），BackPanel响应相应滑动事件，用Canvas做自绘实现,具体链路包括：

## **[BackPanelController#setLayoutParams](https://cs.android.com/android/platform/superproject/main/+/main:frameworks/base/packages/SystemUI/src/com/android/systemui/navigationbar/gestural/EdgeBackGestureHandler.java;drc=61197364367c9e404c7da6900658f1b16c42d0da;bpv=0;bpt=1;l=813)**

```java
//SystemUI/src/com/android/systemui/navigationbar/gestural/EdgeBackGestureHandler
override fun setLayoutParams(layoutParams: WindowManager.LayoutParams) {
        this.layoutParams = layoutParams
        windowManager.addView(mView, layoutParams)
    }
```

layoutParams来自EdgeBackGestureHandler#createLayoutParams
```java
//SystemUI/src/com/android/systemui/navigationbar/gestural/EdgeBackGestureHandler
    private WindowManager.LayoutParams createLayoutParams() {
        Resources resources = mContext.getResources();
        //TYPE_NAVIGATION_BAR_PANEL: 表示这是一个导航栏面板窗口,优先级低于系统UI但高于普通应用
        //FLAG_NOT_FOCUSABLE: 窗口不能获得焦点
        //FLAG_NOT_TOUCHABLE: 窗口不接收触摸事件
        //FLAG_LAYOUT_IN_SCREEN: 窗口布局在屏幕坐标系中
        WindowManager.LayoutParams layoutParams = new WindowManager.LayoutParams(
                resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_width),
                resources.getDimensionPixelSize(R.dimen.navigation_edge_panel_height),
                WindowManager.LayoutParams.TYPE_NAVIGATION_BAR_PANEL,
                WindowManager.LayoutParams.FLAG_NOT_FOCUSABLE
                        | WindowManager.LayoutParams.FLAG_NOT_TOUCHABLE
                        | WindowManager.LayoutParams.FLAG_LAYOUT_IN_SCREEN,
                PixelFormat.TRANSLUCENT); //半透明
        layoutParams.accessibilityTitle = mContext.getString(R.string.nav_bar_edge_panel);
        layoutParams.windowAnimations = 0;
        layoutParams.privateFlags |=
                (WindowManager.LayoutParams.SYSTEM_FLAG_SHOW_FOR_ALL_USERS
                | PRIVATE_FLAG_EXCLUDE_FROM_SCREEN_MAGNIFICATION);
        layoutParams.setTitle(TAG + mContext.getDisplayId());
        layoutParams.setFitInsetsTypes(0 /* types */);
        layoutParams.setTrustedOverlay();
        return layoutParams;
    }
```
## **WindowManager.addView 绑定Window**
WindowManager.addView与WMS处理Window绑定。
1. **WindowManager.addView 调用链**：
```java
// frameworks/base/core/java/android/view/WindowManagerImpl.java
public void addView(View view, ViewGroup.LayoutParams params) {
    mGlobal.addView(view, params, mContext.getDisplay(), mParentWindow);
}
```

2. **WindowManagerGlobal 中的处理**：
```java
// frameworks/base/core/java/android/view/WindowManagerGlobal.java
public void addView(View view, ViewGroup.LayoutParams params,
        Display display, Window parentWindow) {
    
    // 1. 创建 ViewRootImpl
    if (windowlessSession == null) {
       root = new ViewRootImpl(view.getContext(), display);
    } else {
       root = new ViewRootImpl(view.getContext(), display,
                        windowlessSession, new WindowlessWindowLayout());
    }
    
    // 2. 保存引用
    mViews.add(view);
    mRoots.add(root);
    mParams.add(wparams);

     // 3. 设置参数
    try {
        root.setView(view, wparams, panelParentView, userId);
    } catch (RuntimeException e) {
        throw e;
    }
}
```

3. **`ViewRootImpl.setView()` 触发窗口创建**
在 `setView()` 中，通过 IPC 调用 `WindowManagerService` 创建窗口：
```java
// ViewRootImpl.java
public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView, int userId) {
    // ... 其他逻辑
    try {
        // 通过 mWindowSession （WindowManagerGlobal.getWindowSession()）与 WMS 通信
        mWindowSession.addToDisplayAsUser(
            mWindow, mWindowAttributes, getHostVisibility(), mDisplay.getDisplayId(), userId,
            mInsetsController.getRequestedVisibility(), inputChannel, mTempInsets, mControls);
    } catch (RemoteException e) {
        // 处理异常
    }
}
//frameworks/base/services/core/java/com/android/server/wm/Session.java
    @Override
    public int addToDisplayAsUser(IWindow window, WindowManager.LayoutParams attrs,
            int viewVisibility, int displayId, int userId, @InsetsType int requestedVisibleTypes,
            InputChannel outInputChannel, InsetsState outInsetsState,
            InsetsSourceControl.Array outActiveControls, Rect outAttachedFrame,
            float[] outSizeCompatScale) {
        return mService.addWindow(this, window, attrs, viewVisibility, displayId, userId,
                requestedVisibleTypes, outInputChannel, outInsetsState, outActiveControls,
                outAttachedFrame, outSizeCompatScale);
    }
```
- **`mWindowSession`** 是 `IWindowSession` 的实例，由 `WindowManagerGlobal` 创建，是 App 进程与 WMS 通信的代理。
- **`addToDisplayAsUser`** 是 IPC 调用，通知 WMS 创建窗口并分配资源。

在 WMS 服务端，`Session.addToDisplayAsUser()` 最终会创建 `WindowState`：
```java
// WindowManagerService.java
public int addWindow(Session session, IWindow client, ...) {
    // ... 权限校验、参数处理
    final WindowState win = new WindowState(this, session, client, token, parentWindow,
            appOp[0], seq, attrs, viewVisibility, session.mUid, userId);
    win.mSession.onWindowAdded(win);
    mWindowMap.put(client.asBinder(), win);
}
```
- **`WindowState`** 是 WMS 中窗口的抽象，管理窗口的层级、可见性等。

## **`ViewRootImpl.relayoutWindow` 绑定Surface**
`SurfaceControl` 的创建实际发生在 **窗口的首次布局（`performTraversals`）阶段**，由 `WindowManagerService` 触发。关键流程如下：
在客户端（应用进程）的 `ViewRootImpl` 中，通过 IPC 调用 `WindowManagerService` 的 `relayoutWindow` 方法，请求更新窗口布局并创建 `Surface`：
```java
// frameworks/base/core/java/android/view/ViewRootImpl.java
private final WindowRelayoutResult mRelayoutResult = new WindowRelayoutResult(
            mTmpFrames, mPendingMergedConfiguration, mSurfaceControl, mTempInsets, mTempControls);
private int relayoutWindow(WindowManager.LayoutParams params, ...) throws RemoteException {
    // 调用 WMS 的 relayoutWindow 方法
    relayoutResult = mWindowSession.relayout(mWindow, params,
                    requestedWidth, requestedHeight, viewVisibility,
                    insetsPending ? WindowManagerGlobal.RELAYOUT_INSETS_PENDING : 0,
                    mRelayoutSeq, mLastSyncSeqId, mRelayoutResult);
   if (mSurfaceControl.isValid()) {
      updateBlastSurfaceIfNeeded();
   }
    // ...
}

// Android 图形系统的底层优化,引入 BLAST (BufferQueue Layer State Traversal) 机制
void updateBlastSurfaceIfNeeded() {
        mBlastBufferQueue = new BLASTBufferQueue(mTag, mSurfaceControl,
                mSurfaceSize.x, mSurfaceSize.y, mWindowAttributes.format);
        mBlastBufferQueue.setTransactionHangCallback(sTransactionHangCallback);
        mBlastBufferQueue.setApplyToken(mBbqApplyToken);
        Surface blastSurface;
        if (addSchandleToVriSurface()) {
            blastSurface = mBlastBufferQueue.createSurfaceWithHandle();
        } else {
            blastSurface = mBlastBufferQueue.createSurface();
        }
        // 更新 Surface
        mSurface.transferFrom(blastSurface);
}
//createSurfaceWithHandle或createSurface调用nativeGetSurface
// frameworks/base/libs/hwui/BLASTBufferQueue.cpp

// JNI 转换
//通过 nativePtr 获取 Native 层的 BLASTBufferQueue 实例。
//调用 getSurface() 获取 sp<Surface>。
//使用 android_view_Surface_createFromSurface 将 Native Surface 转换为 Java Surface 对象。
static jobject BLASTBufferQueue_getSurface(JNIEnv* env, jclass clazz, jlong nativePtr) {
    BLASTBufferQueue* bbq = reinterpret_cast<BLASTBufferQueue*>(nativePtr);
    sp<Surface> surface = bbq->getSurface();
    return android_view_Surface_createFromSurface(env, surface);
}

// frameworks/native/libs/gui/BLASTBufferQueue.cpp
//Surface 是 ANativeWindow 的子类，其底层通过 IGraphicBufferProducer（生产者接口）与 BufferQueue 绑定。在 BLASTBufferQueue 中，生产者接口 mProducer 被传递给 Surface，使其能够通过 dequeueBuffer 和 queueBuffer 管理图形缓冲区
//Surface 内部持有 IGraphicBufferProducer
//BBQSurface继承Surface
sp<Surface> BLASTBufferQueue::getSurface(bool includeSurfaceControlHandle) {
    std::lock_guard _lock{mMutex};
    sp<IBinder> scHandle = nullptr;
    if (includeSurfaceControlHandle && mSurfaceControl) {
        scHandle = mSurfaceControl->getHandle();
    }
    return new BBQSurface(mProducer, true, scHandle, this);
}
```
在服务端（`WindowManagerService`），`mWindowSession.relayout` 最终会调用 `WindowStateAnimator.createSurfaceLocked()` 创建 `SurfaceControl`：
```java
//frameworks/base/services/core/java/com/android/server/wm/Session.java
 @Override
public int relayout(IWindow window, WindowManager.LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewFlags, int flags, int seq,
            int lastSyncSeqId, WindowRelayoutResult outRelayoutResult) {
        int res = mService.relayoutWindow(this, window, attrs, requestedWidth,
                requestedHeight, viewFlags, flags, seq, lastSyncSeqId, outRelayoutResult);
        return res;
}
// frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
public int relayoutWindow(Session session, IWindow client, LayoutParams attrs,
            int requestedWidth, int requestedHeight, int viewVisibility, int flags, int seq,
            int lastSyncSeqId, WindowRelayoutResult outRelayoutResult) {
   result = createSurfaceControl(outSurfaceControl, result, win, winAnimator);  
}

// frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java
private int createSurfaceControl(SurfaceControl outSurfaceControl, int result,
            WindowState win, WindowStateAnimator winAnimator) {
        if (!win.mHasSurface) {
            result |= RELAYOUT_RES_SURFACE_CHANGED;
        }

        SurfaceControl surfaceControl;
        try {
            Trace.traceBegin(TRACE_TAG_WINDOW_MANAGER, "createSurfaceControl");
            surfaceControl = winAnimator.createSurfaceLocked();
        } finally {
            Trace.traceEnd(TRACE_TAG_WINDOW_MANAGER);
        }
        return result;
}
// frameworks/base/services/core/java/com/android/server/wm/WindowStateAnimator.java
void createSurfaceLocked() {
   mSurfaceControl = mWin.makeSurface()
                    .setParent(mWin.mSurfaceControl)
                    .setName(mTitle)
                    .setFormat(format)
                    .setFlags(flags)
                    .setMetadata(METADATA_WINDOW_TYPE, attrs.type)
                    .setMetadata(METADATA_OWNER_UID, mSession.mUid)
                    .setMetadata(METADATA_OWNER_PID, mSession.mPid)
                    .setCallsite("WindowSurfaceController")
                    .setBLASTLayer().build();
}
```

## **视图通过 `Surface` 绘制内容**
在 `ViewRootImpl` 的绘制流程中，通过 `Surface` 提交帧数据：
```java
// ViewRootImpl.java -> performTraversals()
private void performTraversals() {
    // 1. 测量、布局
    measureHierarchy(...);
    layout(...);
    // 2. 绘制到 Surface
    performDraw(...)
}

private boolean performDraw(@Nullable SurfaceSyncGroup surfaceSyncGroup) {
   usingAsyncReport = draw(fullRedrawNeeded, surfaceSyncGroup, mSyncBuffer);
}

private boolean draw(boolean fullRedrawNeeded, @Nullable SurfaceSyncGroup activeSyncGroup,
            boolean syncBuffer) {
      if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {
                    return false;
      }
}

private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
    Surface surface = mSurface;
    Canvas canvas = surface.lockCanvas(dirty);
    // 通过 View 系统绘制内容
    mView.draw(canvas);
    surface.unlockCanvasAndPost(canvas);
}
```
- **`surface.lockCanvas()`** 获取 `Canvas` 对象，用于绘制。
- **`unlockCanvasAndPost()`** 提交绘制内容到 `Surface`，最终由 SurfaceFlinger 合成显示。

## **总结流程**
1. **`addView` 触发 `ViewRootImpl.setView()`**  
   客户端通过 IPC 调用 `WMS.addWindow()`，创建 `WindowState`。

2. **首次布局请求**  
   `ViewRootImpl.relayoutWindow()` 调用 `WMS.relayoutWindow()`，触发 `SurfaceControl` 的创建。

3. **`SurfaceControl` 的创建**  
   服务端通过 `SurfaceSession` 创建 `SurfaceControl`，并通过 Binder 将句柄返回客户端。

4. **客户端 `Surface` 绑定**  
   客户端 `ViewRootImpl` 通过 `updateBlastSurfaceIfNeeded` 将 `Surface` 与 `SurfaceControl` 绑定。

5. **绘制提交**  
   客户端通过 `Surface.lockCanvas()` 和 `unlockCanvasAndPost()` 将内容绘制到 `Surface`，由 SurfaceFlinger 合成显示。

---

## **关键绑定关系总结**
| **组件**          | **作用**                                                                 |
|--------------------|-------------------------------------------------------------------------|
| `ViewRootImpl`     | 连接视图系统与 WMS，管理 `Surface` 生命周期和绘制流程。                  |
| `WindowState`      | WMS 中的窗口抽象，持有 `SurfaceControl` 控制 Surface 属性。             |
| `SurfaceControl`   | 服务端（WMS）对 `Surface` 的控制句柄，管理缓冲区分配和属性，对应 SurfaceFlinger 的 Layer。 |
| `Surface`          | 客户端（应用进程）的绘制接口，通过 Binder 持有服务端 `SurfaceControl` 的引用。 |

---

## **流程总结**
1. **窗口创建**  
   `addView` 触发 `ViewRootImpl` 创建，并通过 IPC 调用 WMS 的 `addToDisplayAsUser`，在服务端生成 `WindowState` 和 `SurfaceControl`。

2. **Surface 分配**  
   WMS 通过 `SurfaceControl` 创建 `Surface`，并将句柄传递给应用进程的 `ViewRootImpl`。

3. **绘制绑定**  
   `ViewRootImpl` 通过 `relayoutWindow` 更新 `Surface`，在 `performTraversals` 中完成视图的测量、布局、绘制，最终通过 `Surface` 提交帧数据。

4. **显示合成**  
   SurfaceFlinger 根据 `Surface` 的缓冲区内容，合成到屏幕。

---