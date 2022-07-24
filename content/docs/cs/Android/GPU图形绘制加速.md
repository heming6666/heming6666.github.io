---
title: GPU 图形绘制加速原理
---
## 一、整体架构
- 软件加速/硬件加速：CPU / GPU
- 两步走：构建 + 渲染
- 绘制内存的分配，CPU/GPU都一样，请求 SurfaceFlinger，但硬件加速可能会从FrameBuffer硬件缓冲区直接分配内存。
![](https://upload-images.jianshu.io/upload_images/1460468-22e4b5bba04b472b.jpg?imageMogr2/auto-orient/strip|imageView2/2/format/webp)
## 二、构建阶段
#### 2.1 方法
- 构建：递归遍历所有 view, 把需要的操作缓存下来
- 每个 View 有自己的 DrawOp List，同时递归遍历子 View，就能遍历到所有的绘制 Op，然后把 Op 树交给 RenderThread。
- View 即 RenderNode 节点
- View 的绘制会被抽象成一个个 DrawOp(DisplayListOp),每个DrawOp有对应的OpenGL绘制命令。子 View 的绘制：DrawRenderNodeOp
![](https://upload-images.jianshu.io/upload_images/1460468-d546b6e86e2a1f30.png?imageMogr2/auto-orient/strip|imageView2/2/w/1200/format/webp)
![](https://upload-images.jianshu.io/upload_images/1460468-1f3c83ffb4e74889.jpg?imageMogr2/auto-orient/strip|imageView2/2/format/webp)

#### 2.2 实现
`ThreadedRenderer` 对象，作用：
- 在 UI 线程中完成 `DrawOp` 树构建
- 和 `RenderThread` 通信。(是一个单例线程)

以下代码，`ThreadedRenderer` 中：
- 用 `RootNode` 标记`DrawOp` 树的根节点
- `RenderProxy` 是用来和 `RenderThread` 通信的 `fd`.

```Java
ThreadedRenderer(Context context, boolean translucent) {
    ...
    <!--新建native node-->
    long rootNodePtr = nCreateRootRenderNode();
    mRootNode = RenderNode.adopt(rootNodePtr);
    mRootNode.setClipToBounds(false);
    <!--新建NativeProxy-->
    mNativeProxy = nCreateProxy(translucent, rootNodePtr);
    ProcessInitializer.sInstance.init(context, mNativeProxy);
    loadSystemProperties();
}
```
`ThreadedRenderer`的`draw`函数如下：
```Java
@Override
void draw(View view, AttachInfo attachInfo, HardwareDrawCallbacks callbacks) {
    attachInfo.mIgnoreDirtyState = true;

    final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
    choreographer.mFrameInfo.markDrawStart();
    <!--关键点1：构建View的DrawOp树-->
    updateRootDisplayList(view, callbacks);

    <!--关键点2：通知RenderThread线程绘制-->
    int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);
    ...
}
```
`updateRootDisplayList`函数如下。流程：
```Java
private void updateRootDisplayList(View view, HardwareDrawCallbacks callbacks) {
    <!--更新-->
    updateViewTreeDisplayList(view);
   if (mRootNodeNeedsUpdate || !mRootNode.isValid()) {
      <!--获取DisplayListCanvas-->
        DisplayListCanvas canvas = mRootNode.start(mSurfaceWidth, mSurfaceHeight);
        try {
        <!--利用canvas缓存Op-->
            final int saveCount = canvas.save();
            canvas.translate(mInsetLeft, mInsetTop);
            callbacks.onHardwarePreDraw(canvas);

            canvas.insertReorderBarrier();
            canvas.drawRenderNode(view.updateDisplayListIfDirty());
            canvas.insertInorderBarrier();

            callbacks.onHardwarePostDraw(canvas);
            canvas.restoreToCount(saveCount);
            mRootNodeNeedsUpdate = false;
        } finally {
        <!--将所有Op填充到RootRenderNode-->
            mRootNode.end(canvas);
        }
    }
}
```
