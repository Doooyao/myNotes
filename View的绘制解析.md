#### View的绘制解析

##### ViewRootImpl
在performTraversals方法中,调用了performMeasure,performLayout,performDraw方法,下面看一下是如何实现的
1. performMeasure
```java
private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
        if (mView == null) {
            return;
        }
        Trace.traceBegin(Trace.TRACE_TAG_VIEW, "measure");
        try {
            mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);//这里可以看到,直接调用了decorview的measure方法
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
    }
```
2. performLayout
```java
private void performLayout(WindowManager.LayoutParams lp, int desiredWindowWidth,
            int desiredWindowHeight) {
        mLayoutRequested = false;
        mScrollMayChange = true;
        mInLayout = true;//layout开始
            ```省略部分代码```
        try {
            host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());//这里我们可以看到,调用了顶层view的layout
            mInLayout = false;//结束
            int numViewsRequestingLayout = mLayoutRequesters.size();//这是个什么数组呢,一会儿可以看到
            if (numViewsRequestingLayout > 0) {
                // requestLayout() was called during layout.
                // If no layout-request flags are set on the requesting views, there is no problem.
                // If some requests are still pending, then we need to clear those flags and do
                // a full request/measure/layout pass to handle this situation.
                ArrayList<View> validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters,
                        false);//有效测量的子view
                if (validLayoutRequesters != null) {
                    // Set this flag to indicate that any further requests are happening during
                    // the second pass, which may result in posting those requests to the next
                    // frame instead
                    mHandlingLayoutInLayoutRequest = true;

                    // Process fresh layout requests, then measure and layout
                    int numValidRequests = validLayoutRequesters.size();
                    for (int i = 0; i < numValidRequests; ++i) {
                        final View view = validLayoutRequesters.get(i);
                        Log.w("View", "requestLayout() improperly called by " + view +
                                " during layout: running second layout pass");
                        view.requestLayout();//子view调用requestLayout
                    }
                    measureHierarchy(host, lp, mView.getContext().getResources(),
                            desiredWindowWidth, desiredWindowHeight);//再次Measure
                    mInLayout = true;
                    host.layout(0, 0, host.getMeasuredWidth(), host.getMeasuredHeight());//再次layout

                    mHandlingLayoutInLayoutRequest = false;

                    // Check the valid requests again, this time without checking/clearing the
                    // layout flags, since requests happening during the second pass get noop'd
                    validLayoutRequesters = getValidLayoutRequesters(mLayoutRequesters, true);
                    if (validLayoutRequesters != null) {
                        final ArrayList<View> finalRequesters = validLayoutRequesters;
                        // Post second-pass requests to the next frame
                        getRunQueue().post(new Runnable() {
                            @Override
                            public void run() {
                                int numValidRequests = finalRequesters.size();
                                for (int i = 0; i < numValidRequests; ++i) {
                                    final View view = finalRequesters.get(i);
                                    Log.w("View", "requestLayout() improperly called by " + view +
                                            " during second layout pass: posting in next frame");
                                    view.requestLayout();//最后一次requestlayout
                                }
                            }
                        });
                    }
                }

            }
        } finally {
            Trace.traceEnd(Trace.TRACE_TAG_VIEW);
        }
        mInLayout = false;
    }
```
3. performDraw 这一部分的代码有点多,我挑重要的上啦
```java
private void performDraw() {
       ```省略一些本地变量设置和不需要绘制情况的return```
        mIsDrawing = true; //开始draw
        boolean usingAsyncReport = false;
        ```省略通知```
        try {
            boolean canUseAsync = draw(fullRedrawNeeded);//调用draw方法
            if (usingAsyncReport && !canUseAsync) {
                mAttachInfo.mThreadedRenderer.setFrameCompleteCallback(null);
                usingAsyncReport = false;
            }
        } finally {
            mIsDrawing = false;//绘制结束
        }
        ```省略剩下的```
    }
```
再来看看viewrootimpl中的draw方法 里面省略了一些代码
```java
private boolean draw(boolean fullRedrawNeeded) {
        if (!sFirstDrawComplete) {
            synchronized (sFirstDrawHandlers) {
                sFirstDrawComplete = true;
                final int count = sFirstDrawHandlers.size();
                for (int i = 0; i< count; i++) {
                    mHandler.post(sFirstDrawHandlers.get(i));//初次绘制
                }
            }
        }

        final Rect dirty = mDirty;//需要绘制的区域
      
        if (fullRedrawNeeded) {
            mAttachInfo.mIgnoreDirtyState = true;
            dirty.set(0, 0, (int) (mWidth * appScale + 0.5f), (int) (mHeight * appScale + 0.5f));//如果全都需要重绘
        }
        boolean useAsyncReport = false;
        if (!dirty.isEmpty() || mIsAnimating || accessibilityFocusDirty) {
            if (mAttachInfo.mThreadedRenderer != null && mAttachInfo.mThreadedRenderer.isEnabled()) {
                // If accessibility focus moved, always invalidate the root.
                boolean invalidateRoot = accessibilityFocusDirty || mInvalidateRootRequested;
                mInvalidateRootRequested = false;
                // Draw with hardware renderer.
                mIsAnimating = false;
                if (mHardwareYOffset != yOffset || mHardwareXOffset != xOffset) {
                    mHardwareYOffset = yOffset;
                    mHardwareXOffset = xOffset;
                    invalidateRoot = true;
                }
                if (invalidateRoot) {
                    mAttachInfo.mThreadedRenderer.invalidateRoot();//更新root
                }
                dirty.setEmpty();
                // draw(...) might invoke post-draw, which might register the next callback already.
                final FrameDrawingCallback callback = mNextRtFrameCallback;
                mNextRtFrameCallback = null;
                mAttachInfo.mThreadedRenderer.draw(mView, mAttachInfo, this, callback);//draw
            } else {
                if (mAttachInfo.mThreadedRenderer != null &&
                        !mAttachInfo.mThreadedRenderer.isEnabled() &&
                        mAttachInfo.mThreadedRenderer.isRequested() &&
                        mSurface.isValid()) {

                    try {
                        mAttachInfo.mThreadedRenderer.initializeIfNeeded(
                                mWidth, mHeight, mAttachInfo, mSurface, surfaceInsets);//初始化
                    } catch (OutOfResourcesException e) {
                        handleOutOfResourcesException(e);
                        return false;
                    }

                    mFullRedrawNeeded = true;
                    scheduleTraversals();
                    return false;
                }
                if (!drawSoftware(surface, mAttachInfo, xOffset, yOffset,
                        scalingRequired, dirty, surfaceInsets)) {//绘制
                    return false;
                }
            }
        }
    }
```
1. mThreadedRenderer.invalidateRoot
```java
    void invalidateRoot() {
        mRootNodeNeedsUpdate = true;//可以看到,这个方法只是设置这个根节点需要更新的变量
    }
```
2. mThreadedRenderer.draw
```java
void draw(View view, AttachInfo attachInfo, DrawCallbacks callbacks,
            FrameDrawingCallback frameDrawingCallback) {
        attachInfo.mIgnoreDirtyState = true;

        final Choreographer choreographer = attachInfo.mViewRootImpl.mChoreographer;
        choreographer.mFrameInfo.markDrawStart();

        updateRootDisplayList(view, callbacks);

        attachInfo.mIgnoreDirtyState = false;

        // register animating rendernodes which started animating prior to renderer
        // creation, which is typical for animators started prior to first draw
        if (attachInfo.mPendingAnimatingRenderNodes != null) {
            final int count = attachInfo.mPendingAnimatingRenderNodes.size();
            for (int i = 0; i < count; i++) {
                registerAnimatingRenderNode(
                        attachInfo.mPendingAnimatingRenderNodes.get(i));//注册动画节点
            }
            attachInfo.mPendingAnimatingRenderNodes.clear();
            // We don't need this anymore as subsequent calls to
            // ViewRootImpl#attachRenderNodeAnimator will go directly to us.
            attachInfo.mPendingAnimatingRenderNodes = null;
        }

        final long[] frameInfo = choreographer.mFrameInfo.mFrameInfo;
        if (frameDrawingCallback != null) {
            nSetFrameCallback(mNativeProxy, frameDrawingCallback);
        }
        int syncResult = nSyncAndDrawFrame(mNativeProxy, frameInfo, frameInfo.length);//drawframe
        if ((syncResult & SYNC_LOST_SURFACE_REWARD_IF_FOUND) != 0) {
            setEnabled(false);
            attachInfo.mViewRootImpl.mSurface.release();
            attachInfo.mViewRootImpl.invalidate();//可以看到,这里又调回了invalidate方法
        }
        if ((syncResult & SYNC_INVALIDATE_REQUIRED) != 0) {
            attachInfo.mViewRootImpl.invalidate();
        }
    }
```
再回来看viewrootimpl的invalidate
```java
void invalidate() {
        mDirty.set(0, 0, mWidth, mHeight);
        if (!mWillDrawSoon) {
            scheduleTraversals();//这里调用了刷新的方法 这里最后会调用performTraversals方法刷新页面
        }
    }
```
另一方面 来看看开始绘制的方法
```java
private boolean drawSoftware(Surface surface, AttachInfo attachInfo, int xoff, int yoff,
            boolean scalingRequired, Rect dirty, Rect surfaceInsets) {
        final Canvas canvas;
        int dirtyXOffset = xoff;
        int dirtyYOffset = yoff;
        if (surfaceInsets != null) {
            dirtyXOffset += surfaceInsets.left;
            dirtyYOffset += surfaceInsets.top;
        }
        try {
            dirty.offset(-dirtyXOffset, -dirtyYOffset);
            final int left = dirty.left;
            final int top = dirty.top;
            final int right = dirty.right;
            final int bottom = dirty.bottom;
            canvas = mSurface.lockCanvas(dirty);//锁定要绘制的这一块画布
            if (left != dirty.left || top != dirty.top || right != dirty.right
                    || bottom != dirty.bottom) {
                attachInfo.mIgnoreDirtyState = true;
            }
            canvas.setDensity(mDensity);
        } catch (Surface.OutOfResourcesException e) {
            handleOutOfResourcesException(e);
            return false;
        } catch (IllegalArgumentException e) {
            mLayoutRequested = true;  // ask wm for a new surface next time.
            return false;
        } finally {
            dirty.offset(dirtyXOffset, dirtyYOffset);  // Reset to the original value.
        }
        mView.draw(canvas);  // 这里就可以看到对根布局的draw进行调用,关于view内部onDraw的调用,我们下面再说
        return true;
    }
```


##### MeasureSpec
这个类在View里面,直接上源码看看
```java
public static class MeasureSpec {
        private static final int MODE_SHIFT = 30;//模式偏移位数
        private static final int MODE_MASK  = 0x3 << MODE_SHIFT;//mask 11100000000这种

        /** @hide */
        @IntDef({UNSPECIFIED, EXACTLY, AT_MOST})
        @Retention(RetentionPolicy.SOURCE)
        public @interface MeasureSpecMode {}
        
        public static final int UNSPECIFIED = 0 << MODE_SHIFT;//前两位是0
        
        public static final int EXACTLY     = 1 << MODE_SHIFT;//前两位是1

        public static final int AT_MOST     = 2 << MODE_SHIFT;//

        public static int makeMeasureSpec(@IntRange(from = 0, to = (1 << MeasureSpec.MODE_SHIFT) - 1) int size,
                                          @MeasureSpecMode int mode) {
            if (sUseBrokenMakeMeasureSpec) {
                return size + mode;//把size和mode合成
            } else {
                return (size & ~MODE_MASK) | (mode & MODE_MASK);//清除边界
            }
        }

        @MeasureSpecMode
        public static int getMode(int measureSpec) {
            //noinspection ResourceType
            return (measureSpec & MODE_MASK);
        }

        public static int getSize(int measureSpec) {
            return (measureSpec & ~MODE_MASK);
        }

        static int adjust(int measureSpec, int delta) { //调节大小
            final int mode = getMode(measureSpec);
            int size = getSize(measureSpec);
            if (mode == UNSPECIFIED) {
                // No need to adjust size for UNSPECIFIED mode.
                return makeMeasureSpec(size, UNSPECIFIED);
            }
            size += delta;
            if (size < 0) {
                Log.e(VIEW_LOG_TAG, "MeasureSpec.adjust: new size would be negative! (" + size +
                        ") spec: " + toString(measureSpec) + " delta: " + delta);
                size = 0;
            }
            return makeMeasureSpec(size, mode);
        }
        ```省略部分代码```
    }
```
下面看一下ViewRootImpl计算MeasureSpec
```java
 private static int getRootMeasureSpec(int windowSize, int rootDimension) {
        int measureSpec;
        switch (rootDimension) {
        case ViewGroup.LayoutParams.MATCH_PARENT:
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.EXACTLY);
            break;
        case ViewGroup.LayoutParams.WRAP_CONTENT:
            measureSpec = MeasureSpec.makeMeasureSpec(windowSize, MeasureSpec.AT_MOST);
            break;
        default:
            measureSpec = MeasureSpec.makeMeasureSpec(rootDimension, MeasureSpec.EXACTLY);
            break;
        }
        return measureSpec;
    }
```
好像没啥好解释的=.=直接看代码就行
下面再看一下在普通的View中是怎么计算MeasureSpec的
首先,在父View ViewGroup中
```java
public static int getChildMeasureSpec(int spec, int padding, int childDimension) {
        int specMode = MeasureSpec.getMode(spec);
        int specSize = MeasureSpec.getSize(spec);

        int size = Math.max(0, specSize - padding);

        int resultSize = 0;
        int resultMode = 0;

        switch (specMode) {
        // Parent has imposed an exact size on us
        case MeasureSpec.EXACTLY:
            if (childDimension >= 0) {
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size. So be it.
                resultSize = size;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent has imposed a maximum size on us
        case MeasureSpec.AT_MOST:
            if (childDimension >= 0) {
                // Child wants a specific size... so be it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size, but our size is not fixed.
                // Constrain child to not be bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size. It can't be
                // bigger than us.
                resultSize = size;
                resultMode = MeasureSpec.AT_MOST;
            }
            break;

        // Parent asked to see how big we want to be
        case MeasureSpec.UNSPECIFIED:
            if (childDimension >= 0) {
                // Child wants a specific size... let him have it
                resultSize = childDimension;
                resultMode = MeasureSpec.EXACTLY;
            } else if (childDimension == LayoutParams.MATCH_PARENT) {
                // Child wants to be our size... find out how big it should
                // be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            } else if (childDimension == LayoutParams.WRAP_CONTENT) {
                // Child wants to determine its own size.... find out how
                // big it should be
                resultSize = View.sUseZeroUnspecifiedMeasureSpec ? 0 : size;
                resultMode = MeasureSpec.UNSPECIFIED;
            }
            break;
        }
        //noinspection ResourceType
        return MeasureSpec.makeMeasureSpec(resultSize, resultMode);
    }
```
几种情况的判断,不用解释了吧
##### Measure
下面我们看一下Measure相关的流程
从开始ViewRootImpl的代码来看 逐级测量是从调用measure开始的,我们看看他干了些啥=.=
首先看view.measure方法
```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
        boolean optical = isLayoutModeOptical(this);//布局模式是否可见的 这个方法一会儿再看
        if (optical != isLayoutModeOptical(mParent)) { //和父布局不同
            Insets insets = getOpticalInsets();//获取光学贴图的宽高
            int oWidth  = insets.left + insets.right;
            int oHeight = insets.top  + insets.bottom;
            widthMeasureSpec  = MeasureSpec.adjust(widthMeasureSpec,  optical ? -oWidth  : oWidth);//相应调整
            heightMeasureSpec = MeasureSpec.adjust(heightMeasureSpec, optical ? -oHeight : oHeight);
        }
        
        long key = (long) widthMeasureSpec << 32 | (long) heightMeasureSpec & 0xffffffffL;//把宽高拼接成64位的key
        if (mMeasureCache == null) mMeasureCache = new LongSparseLongArray(2);//mMeasureCache缓存

        final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;//强制重新测量

        // Optimize layout by avoiding an extra EXACTLY pass when the view is
        // already measured as the correct size. In API 23 and below, this
        // extra pass is required to make LinearLayout re-distribute weight.
        final boolean specChanged = widthMeasureSpec != mOldWidthMeasureSpec
                || heightMeasureSpec != mOldHeightMeasureSpec;//是否修改过测量宽高
        final boolean isSpecExactly = MeasureSpec.getMode(widthMeasureSpec) == MeasureSpec.EXACTLY
                && MeasureSpec.getMode(heightMeasureSpec) == MeasureSpec.EXACTLY;//是不是精确
        final boolean matchesSpecSize = getMeasuredWidth() == MeasureSpec.getSize(widthMeasureSpec)
                && getMeasuredHeight() == MeasureSpec.getSize(heightMeasureSpec);//宽高是否匹配
        final boolean needsLayout = specChanged//宽高发生了变化并且()
                && (sAlwaysRemeasureExactly || !isSpecExactly || !matchesSpecSize);//是否需要测量布局
        //在sAlwaysRemeasureExactly == false 并且传进来的是精确的宽高和当前测量结果相同时,不需要重新测量
        if (forceLayout || needsLayout) {//需要重新布局测量
            // first clears the measured dimension flag
            mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET; //清除此符号位 测量尺寸设置

            resolveRtlPropertiesIfNeeded(); //解决所有与RTL相关的属性

            int cacheIndex = forceLayout ? -1 : mMeasureCache.indexOfKey(key);//缓存
            if (cacheIndex < 0 || sIgnoreMeasureCache) {//没有或者忽略缓存
                // measure ourselves, this should set the measured dimension flag back
                onMeasure(widthMeasureSpec, heightMeasureSpec);//onmeasure~~~
                mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;//清除此符号位,layout之前不需要measure了
            } else {
                long value = mMeasureCache.valueAt(cacheIndex);//缓存中获取
                // Casting a long to int drops the high 32 bits, no mask needed
                setMeasuredDimensionRaw((int) (value >> 32), (int) value);//设置measure,设置测量尺寸设置的符号位
                mPrivateFlags3 |= PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;//设置此符号位,layout之前需要measure
            }

            // flag not set, setMeasuredDimension() was not invoked, we raise
            // an exception to warn the developer
            if ((mPrivateFlags & PFLAG_MEASURED_DIMENSION_SET) != PFLAG_MEASURED_DIMENSION_SET) {
                throw new IllegalStateException("View with id " + getId() + ": "
                        + getClass().getName() + "#onMeasure() did not set the"
                        + " measured dimension by calling"
                        + " setMeasuredDimension()");
            }

            mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;//设置需要layout的符号位
        }

        mOldWidthMeasureSpec = widthMeasureSpec;
        mOldHeightMeasureSpec = heightMeasureSpec;

        mMeasureCache.put(key, ((long) mMeasuredWidth) << 32 |
                (long) mMeasuredHeight & 0xffffffffL); // suppress sign extension //缓存结果
    }
```
由此可见onmeasure中根据当前及传入的数据判断是否需要重新measure并修改相应符号位,及使用并添加缓存
我们先看看刚刚的一个方法isLayoutModeOptical
```java
    /**
     * Return true if o is a ViewGroup that is laying out using optical bounds.
     * 这里只能看出来如果view的一个viewgroup并且这个方法true则true,反之false 我们去viewgroup里面看看
     */
    public static boolean isLayoutModeOptical(Object o) {
        return o instanceof ViewGroup && ((ViewGroup) o).isLayoutModeOptical();
    }
    
    //下面这个是viewgroup里面的方法
    boolean isLayoutModeOptical() {
        return mLayoutMode == LAYOUT_MODE_OPTICAL_BOUNDS;//这里就是简单的判断了一下布局模式,这个LAYOUT_MODE_OPTICAL_BOUNDS是啥呢
        //我百度搜了一下,有个老哥是这么说的,我直接粘贴过来:
        //"如果view有一个光学的边界比如阴影，发光等等，则optical的值为true"
        //那就比较清晰了,我们回到上面的代码,看看干了啥
    }
```
measure的方法就走完了,关于光学布局我还是不太懂=.=有机会再回来看看//todo
下面看一下onmeasure
###### onMeasure
这个方法比较常见了,我们自定义view的时候经常重写他~~
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {
        setMeasuredDimension(getDefaultSize(getSuggestedMinimumWidth(), widthMeasureSpec),
                getDefaultSize(getSuggestedMinimumHeight(), heightMeasureSpec));
    }
    哈哈哈 这个方法是不是很少,在这里我们就设置测量好的宽高了,你可以重写view的onmeasure,在super里把你想要设置的宽高传进来进行设置
    我们看看这里面调用的几个方法
```
1. setMeasuredDimension 
```java
protected final void setMeasuredDimension(int measuredWidth, int measuredHeight) {//设置测量宽高
        boolean optical = isLayoutModeOptical(this);
        if (optical != isLayoutModeOptical(mParent)) {
            Insets insets = getOpticalInsets();
            int opticalWidth  = insets.left + insets.right;
            int opticalHeight = insets.top  + insets.bottom;
            measuredWidth  += optical ? opticalWidth  : -opticalWidth;
            measuredHeight += optical ? opticalHeight : -opticalHeight;
        }//关于光学布局 不懂=.=
        setMeasuredDimensionRaw(measuredWidth, measuredHeight);
    }
    
private void setMeasuredDimensionRaw(int measuredWidth, int measuredHeight) {
    mMeasuredWidth = measuredWidth;
    mMeasuredHeight = measuredHeight;
    mPrivateFlags |= PFLAG_MEASURED_DIMENSION_SET;
    //设置测量宽高,设置测量尺寸已设置的符号位
}
```
2. getDefaultSize
```java
public static int getDefaultSize(int size, int measureSpec) {
        int result = size;
        int specMode = MeasureSpec.getMode(measureSpec);
        int specSize = MeasureSpec.getSize(measureSpec);
        switch (specMode) {
        case MeasureSpec.UNSPECIFIED://返回第一个
            result = size;
            break;
        case MeasureSpec.AT_MOST:
        case MeasureSpec.EXACTLY:
            result = specSize;//返回第二个
            break;
        }
        return result;
    }
    简单理解
```
3. getSuggestedMinimumWidth/getSuggestedMinimumHeight
```java
protected int getSuggestedMinimumWidth() {
        return (mBackground == null) ? mMinWidth : max(mMinWidth, mBackground.getMinimumWidth());
    }
    最小宽度,具体不解释啦
```
view中的onmeasure就结束了,在viewgroup中并没有重写onmeasure方法,我们找一个子类看看
就选一个比较简单的viewgroup做代表吧~那就是~framelayout~当当当当~~
另一方面,DecorView也是继承自framelayout
framelayout中的onmeasure
```java
protected void onMeasure(int widthMeasureSpec, int heightMeasureSpec) {     
    int count = getChildCount();//获取子view的数量
    final boolean measureMatchParentChildren =
            MeasureSpec.getMode(widthMeasureSpec) != MeasureSpec.EXACTLY ||
            MeasureSpec.getMode(heightMeasureSpec) != MeasureSpec.EXACTLY;//如果自己的宽高不是精确地,则要测量matchparentchild
    mMatchParentChildren.clear();//清除存储的matchparentchild

    int maxHeight = 0;
    int maxWidth = 0;
    int childState = 0;

    for (int i = 0; i < count; i++) {//遍历child
        final View child = getChildAt(i);
        if (mMeasureAllChildren || child.getVisibility() != GONE) {//child不GONE并且mMeasureAllChildren
            measureChildWithMargins(child, widthMeasureSpec, 0, heightMeasureSpec, 0);//传入自己的测量宽高测量child
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            maxWidth = Math.max(maxWidth,
                    child.getMeasuredWidth() + lp.leftMargin + lp.rightMargin);//计算最大child的宽高
            maxHeight = Math.max(maxHeight,
                    child.getMeasuredHeight() + lp.topMargin + lp.bottomMargin);
            childState = combineMeasuredStates(childState, child.getMeasuredState());//记录测量状态
            if (measureMatchParentChildren) {
                if (lp.width == LayoutParams.MATCH_PARENT ||
                        lp.height == LayoutParams.MATCH_PARENT) {
                    mMatchParentChildren.add(child);//child有依赖父布局的时存起来
                }
            }
        }
    }

    // Account for padding too
    maxWidth += getPaddingLeftWithForeground() + getPaddingRightWithForeground();//加上padding
    maxHeight += getPaddingTopWithForeground() + getPaddingBottomWithForeground();

    // Check against our minimum height and width
    maxHeight = Math.max(maxHeight, getSuggestedMinimumHeight());//最大,第二个参数方法上面有讲过
    maxWidth = Math.max(maxWidth, getSuggestedMinimumWidth());

    // Check against our foreground's minimum height and width
    final Drawable drawable = getForeground();
    if (drawable != null) {
        maxHeight = Math.max(maxHeight, drawable.getMinimumHeight());//和前景比谁更大?
        maxWidth = Math.max(maxWidth, drawable.getMinimumWidth());
    }

    setMeasuredDimension(resolveSizeAndState(maxWidth, widthMeasureSpec, childState),//重新计算size和state
            resolveSizeAndState(maxHeight, heightMeasureSpec,
                    childState << MEASURED_HEIGHT_STATE_SHIFT));//设置自己的宽高啦,里面的方法等下再讲

    count = mMatchParentChildren.size();
    if (count > 1) {//如果只有一个,为啥不管?rodo
        for (int i = 0; i < count; i++) {//遍历matchparent的child
            final View child = mMatchParentChildren.get(i);
            final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

            final int childWidthMeasureSpec;
            if (lp.width == LayoutParams.MATCH_PARENT) {//matchparent的情况
                final int width = Math.max(0, getMeasuredWidth()//自己的宽度
                        - getPaddingLeftWithForeground() - getPaddingRightWithForeground()//计算自己的剩余空间
                        - lp.leftMargin - lp.rightMargin);
                childWidthMeasureSpec = MeasureSpec.makeMeasureSpec(
                        width, MeasureSpec.EXACTLY);//设置精确宽度给child
            } else {
                childWidthMeasureSpec = getChildMeasureSpec(widthMeasureSpec,
                        getPaddingLeftWithForeground() + getPaddingRightWithForeground() +
                        lp.leftMargin + lp.rightMargin,
                        lp.width);//计算child测量尺寸
            }

            final int childHeightMeasureSpec;
            if (lp.height == LayoutParams.MATCH_PARENT) {//高度,同上
                final int height = Math.max(0, getMeasuredHeight()
                        - getPaddingTopWithForeground() - getPaddingBottomWithForeground()
                        - lp.topMargin - lp.bottomMargin);
                childHeightMeasureSpec = MeasureSpec.makeMeasureSpec(
                        height, MeasureSpec.EXACTLY);
            } else {
                childHeightMeasureSpec = getChildMeasureSpec(heightMeasureSpec,
                        getPaddingTopWithForeground() + getPaddingBottomWithForeground() +
                        lp.topMargin + lp.bottomMargin,
                        lp.height);
            }

            child.measure(childWidthMeasureSpec, childHeightMeasureSpec);//测量child
        }
    }
}
```
可见在他的onmeasure里,分别计算了自己以及所有的子view,甚至会计算多次
上面的代码里有几个方法,我们一一看一下
1. combineMeasuredStates|getMeasuredState
```java
public static int combineMeasuredStates(int curState, int newState) {
        return curState | newState; //实际上是做了个或
    }
public final int getMeasuredState() {
        return (mMeasuredWidth&MEASURED_STATE_MASK)//这个MEASURED_STATE_MASK的值为0xff000000,32
                | ((mMeasuredHeight>>MEASURED_HEIGHT_STATE_SHIFT)//MEASURED_HEIGHT_STATE_SHIFT的值为16
                        & (MEASURED_STATE_MASK>>MEASURED_HEIGHT_STATE_SHIFT));
        //可以看到这个MeasuredState就是把测量宽高的前16位拼接在一起并且分别只保留前8位
        //具体是干啥用呢,暂时不清楚,下面就会讲到啦
    }
```
2. resolveSizeAndState 这个方法实际上是view的
```java
/**
 * 这个方法参数说明很好
 * @param size How big the view wants to be.
 * @param measureSpec Constraints imposed by the parent.
 * @param childMeasuredState Size information bit mask for the view's
 *                           children.
 * @return Size information bit mask as defined by
 *         {@link #MEASURED_SIZE_MASK} and
 *         {@link #MEASURED_STATE_TOO_SMALL}.
 */
public static int resolveSizeAndState(int size, int measureSpec, int childMeasuredState) {
    final int specMode = MeasureSpec.getMode(measureSpec);//测量模式
    final int specSize = MeasureSpec.getSize(measureSpec);//测量大小的值
    final int result;//返回值
    switch (specMode) {
        case MeasureSpec.AT_MOST://最大不能超过specSize
            if (specSize < size) {
                result = specSize | MEASURED_STATE_TOO_SMALL;//设置太小啦的状态 这个状态码是0x01000000 
                // 这里就知道前面为啥要 前八位了,可见这些状态都存在前八位 我们下面还有一个方法更加清晰
            } else {
                result = size;//返回想要的大小
            }
            break;
        case MeasureSpec.EXACTLY:
            result = specSize;//精确,那就直接返回测量值
            break;
        case MeasureSpec.UNSPECIFIED:
        default:
            result = size;
    }
    return result | (childMeasuredState & MEASURED_STATE_MASK);//再算上子view的状态
}
这里可以看到,实际上是根据子view的测量结果再次调整自己并返回
//这个就是刚才提到的方法
public final int getMeasuredWidth() {
        return mMeasuredWidth & MEASURED_SIZE_MASK;//这个MEASURED_SIZE_MASK的值是0x00ffffff,由此可见当获取width时,我们
        //只返回后面的24位,因为前面两位代表MeasureMode,中间六位代表MeasuredState
    }

```
3. measureChildWithMargins //这里顾名思义,就是带着margins测量child啦 这个方法是viewgroup提供的~
```java
protected void measureChildWithMargins(View child,
            int parentWidthMeasureSpec, int widthUsed,
            int parentHeightMeasureSpec, int heightUsed) {
        final MarginLayoutParams lp = (MarginLayoutParams) child.getLayoutParams();

        final int childWidthMeasureSpec = getChildMeasureSpec(parentWidthMeasureSpec,
                mPaddingLeft + mPaddingRight + lp.leftMargin + lp.rightMargin
                        + widthUsed, lp.width);//这里可以看到,调用这个方法获取child的测量尺寸
        final int childHeightMeasureSpec = getChildMeasureSpec(parentHeightMeasureSpec,
                mPaddingTop + mPaddingBottom + lp.topMargin + lp.bottomMargin
                        + heightUsed, lp.height);

        child.measure(childWidthMeasureSpec, childHeightMeasureSpec);//对child内部进行测量
    }
```
getChildMeasureSpec 之前已经讲过了,实际上就是根据父亲的测量尺寸和子view的dimension(就是layoutparams的宽高)生成一个measurespec
在上面代码的最后一行可以看到,我们使用这个生成的测量尺寸再给子view进行测量

到这里measure方法就结束啦 下面我们来看看layout方法~
-measure end-

##### Layout
先看看view的layout方法
```java
public void layout(int l, int t, int r, int b) {//这个参数是相对父布局的位置
        if ((mPrivateFlags3 & PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT) != 0) {//需要测量,上面的代码中有显示,当测量直接从缓存中取
            onMeasure(mOldWidthMeasureSpec, mOldHeightMeasureSpec);//时,就会设置这个符号位
            mPrivateFlags3 &= ~PFLAG3_MEASURE_NEEDED_BEFORE_LAYOUT;//清除此符号位
        }

        int oldL = mLeft;
        int oldT = mTop;
        int oldB = mBottom;
        int oldR = mRight;

        boolean changed = isLayoutModeOptical(mParent) ?
                setOpticalFrame(l, t, r, b) : setFrame(l, t, r, b);//设置光学布局并返回是否改变了 这两个方法下面会讲

        if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {//有改变或者需要进行layout
            //这个符号位上面也有,如果第一次或者重新测量了布局就会设置此符号位
            onLayout(changed, l, t, r, b);//直接调用onlayout

            if (shouldDrawRoundScrollbar()) {//需要绘制圆形滚动条
                if(mRoundScrollbarRenderer == null) {//生成一个滚动条渲染者
                    mRoundScrollbarRenderer = new RoundScrollbarRenderer(this);
                }
            } else {
                mRoundScrollbarRenderer = null;
            }

            mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;//清除此标记位置

            ListenerInfo li = mListenerInfo;//监听者信息 这是啥呢,一会儿再说
            if (li != null && li.mOnLayoutChangeListeners != null) {//回调所有的onLayoutChange
                ArrayList<OnLayoutChangeListener> listenersCopy =
                        (ArrayList<OnLayoutChangeListener>)li.mOnLayoutChangeListeners.clone();
                int numListeners = listenersCopy.size();
                for (int i = 0; i < numListeners; ++i) {
                    listenersCopy.get(i).onLayoutChange(this, l, t, r, b, oldL, oldT, oldR, oldB);
                }
            }
        }

        final boolean wasLayoutValid = isLayoutValid();//布局是否合法?这个方法下面再说

        mPrivateFlags &= ~PFLAG_FORCE_LAYOUT;//清除强制layout的标记 这个标记怎么打上去的暂时不清楚
        mPrivateFlags3 |= PFLAG3_IS_LAID_OUT;//添加布局完成的标记

        if (!wasLayoutValid && isFocused()) {//如果之前的布局不合法并且已经获取焦点了
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;//去掉请求焦点的标记
            if (canTakeFocus()) {//可以获取焦点
                // We have a robust focus, so parents should no longer be wanting focus.
                clearParentsWantFocus();//逐级去掉所有parent的请求焦点标记
            } else if (getViewRootImpl() == null || !getViewRootImpl().isInLayout()) {
                //这里的源码注释我直接翻译一下:这是一种奇怪的情况
                //最有可能的原因是由用户而不是ViewRootImpl调用了layout方法
                //在这种情况下，无法保证parent的layout被评估,所以因此最安全的操作是清除焦点。
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);//清除内部焦点,这个方法啥意思不是很清楚,下面再说
                clearParentsWantFocus();//逐级去掉所有parent的请求焦点标记
            } else if (!hasParentWantsFocus()) {//没有parent请求焦点
                // original requestFocus was likely on this view directly, so just clear focus
                clearFocusInternal(null, /* propagate */ true, /* refocus */ false);//view自己请求的
            }
            // otherwise, we let parents handle re-assigning focus during their layout passes.
        } else if ((mPrivateFlags & PFLAG_WANTS_FOCUS) != 0) {//请求焦点了
            mPrivateFlags &= ~PFLAG_WANTS_FOCUS;//清除标记
            View focused = findFocus();//查找焦点
            if (focused != null) {
                // Try to restore focus as close as possible to our starting focus.
                if (!restoreDefaultFocus() && !hasParentWantsFocus()) {//重新处理焦点
                    // Give up and clear focus once we've reached the top-most parent which wants
                    // focus.
                    focused.clearFocusInternal(null, /* propagate */ true, /* refocus */ false);
                }
            }
        }

        if ((mPrivateFlags3 & PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT) != 0) {//这个暂时不知道是啥 todo
            mPrivateFlags3 &= ~PFLAG3_NOTIFY_AUTOFILL_ENTER_ON_LAYOUT;
            notifyEnterOrExitForAutoFillIfNeeded(true);
        }
    }
```
下面看看上面有调用的几个方法
1. setFrame/setOpticalFrame
```java
private boolean setOpticalFrame(int left, int top, int right, int bottom) {
        Insets parentInsets = mParent instanceof View ?
                ((View) mParent).getOpticalInsets() : Insets.NONE;
        Insets childInsets = getOpticalInsets();
        return setFrame(
                left   + parentInsets.left - childInsets.left,
                top    + parentInsets.top  - childInsets.top,
                right  + parentInsets.left + childInsets.right,
                bottom + parentInsets.top  + childInsets.bottom);//加上父亲光学布局的宽高调用setframe
    }
protected boolean setFrame(int left, int top, int right, int bottom) {//这个方法也挺多,我省略下不重要的代码
        boolean changed = false;
        if (mLeft != left || mRight != right || mTop != top || mBottom != bottom) {//新旧不同,changed为true
            changed = true;
            int drawn = mPrivateFlags & PFLAG_DRAWN;
            int oldWidth = mRight - mLeft;
            int oldHeight = mBottom - mTop;
            int newWidth = right - left;
            int newHeight = bottom - top;
            boolean sizeChanged = (newWidth != oldWidth) || (newHeight != oldHeight);//大小变没变
            // Invalidate our old position
            invalidate(sizeChanged);//调用这个方法重绘view
            mLeft = left;//保存新的参数
            mTop = top;
            mRight = right;
            mBottom = bottom;
            mRenderNode.setLeftTopRightBottom(mLeft, mTop, mRight, mBottom);//设置渲染节点
            mPrivateFlags |= PFLAG_HAS_BOUNDS;//边界设置完毕的标记
            if (sizeChanged) {
                sizeChange(newWidth, newHeight, oldWidth, oldHeight);//大小改变
            }
            if ((mViewFlags & VISIBILITY_MASK) == VISIBLE || mGhostView != null) {//这个ghostview以后再说 反正这里表示有东西可见
                // If we are visible, force the DRAWN bit to on so that
                // this invalidate will go through (at least to our parent).
                // This is because someone may have invalidated this view
                // before this call to setFrame came in, thereby clearing
                // the DRAWN bit.
                mPrivateFlags |= PFLAG_DRAWN;//添加标记
                invalidate(sizeChanged);//刷新
                // parent display list may need to be recreated based on a change in the bounds
                // of any child
                invalidateParentCaches();//刷新父亲缓存 这些讲到绘制再说
            }

            // Reset drawn bit to original value (invalidate turns it off)
            mPrivateFlags |= drawn;
            //设置各种changed参数为true
            mBackgroundSizeChanged = true;
            mDefaultFocusHighlightSizeChanged = true;
            if (mForegroundInfo != null) {
                mForegroundInfo.mBoundsChanged = true;
            }

            notifySubtreeAccessibilityStateChangedIfNeeded();//通知tree更新 这个不清楚,先不管
        }
        return changed;//返回是否改变了
    }    
```
2. ListenerInfo 这个不上代码了,是view的一个内部类,持有view添加的各种listener
3. isInLayout 上面代码调用的是viewrootimpl中的方法,但view中也有同名方法,不过很巧,方法实现也是调用的viewrootimpl哈哈哈哈
下面上一下viewrootimpl中的isInLayout
```java
boolean isInLayout() {
        return mInLayout;//哈哈,实际上是一个变量,在之前的performlayout中是有的,在调用host.layout之前设置true,之后设置false
    }
```
4. 最最最重要了 onlayout
```java
protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
    }//这个方法view并没有实现,因为实际这个方法是用来调用子view的layout的,我简单理解 layout作用于自己,onlayout调用子view的layout
```
下面我们看看viewgroup的相关方法
1. layout
```java
    public final void layout(int l, int t, int r, int b) {
        if (!mSuppressLayout && (mTransition == null || !mTransition.isChangingLayout())) {
            if (mTransition != null) {
                mTransition.layoutChange(this);//有动画时
            }
            super.layout(l, t, r, b);//这里可以看到还是调用了view的layout
        } else {
            // record the fact that we noop'd it; request layout when transition finishes
            mLayoutCalledWhileSuppressed = true;
        }
    }
```
2. onlayout
//这里实际可以看到,viewgroup依然没有实现这个方法
```java
@Override
protected abstract void onLayout(boolean changed,
            int l, int t, int r, int b);
```
所以我们找一个父布局来看看都干了啥
还是我们的framelayout
```java
    protected void onLayout(boolean changed, int left, int top, int right, int bottom) {
        layoutChildren(left, top, right, bottom, false /* no force left gravity */);//调用方法对children进行布局
    }
    
    void layoutChildren(int left, int top, int right, int bottom, boolean forceLeftGravity) {
        final int count = getChildCount();
        final int parentLeft = getPaddingLeftWithForeground();
        final int parentRight = right - left - getPaddingRightWithForeground();
        final int parentTop = getPaddingTopWithForeground();
        final int parentBottom = bottom - top - getPaddingBottomWithForeground();
        for (int i = 0; i < count; i++) {//遍历child
            final View child = getChildAt(i);
            if (child.getVisibility() != GONE) {//要显示的view
                final LayoutParams lp = (LayoutParams) child.getLayoutParams();
                final int width = child.getMeasuredWidth();
                final int height = child.getMeasuredHeight();
                int childLeft;
                int childTop;
                int gravity = lp.gravity;
                if (gravity == -1) {
                    gravity = DEFAULT_CHILD_GRAVITY;
                }
                final int layoutDirection = getLayoutDirection();
                final int absoluteGravity = Gravity.getAbsoluteGravity(gravity, layoutDirection);
                final int verticalGravity = gravity & Gravity.VERTICAL_GRAVITY_MASK;
                switch (absoluteGravity & Gravity.HORIZONTAL_GRAVITY_MASK) {
                    case Gravity.CENTER_HORIZONTAL:
                        childLeft = parentLeft + (parentRight - parentLeft - width) / 2 +
                        lp.leftMargin - lp.rightMargin;
                        break;
                    case Gravity.RIGHT:
                        if (!forceLeftGravity) {
                            childLeft = parentRight - width - lp.rightMargin;
                            break;
                        }
                    case Gravity.LEFT:
                    default:
                        childLeft = parentLeft + lp.leftMargin;
                }
                switch (verticalGravity) {
                    case Gravity.TOP:
                        childTop = parentTop + lp.topMargin;
                        break;
                    case Gravity.CENTER_VERTICAL:
                        childTop = parentTop + (parentBottom - parentTop - height) / 2 +
                        lp.topMargin - lp.bottomMargin;
                        break;
                    case Gravity.BOTTOM:
                        childTop = parentBottom - height - lp.bottomMargin;
                        break;
                    default:
                        childTop = parentTop + lp.topMargin;
                }//计算出可以分配给child的剩余空间,调用child的onlayout
                child.layout(childLeft, childTop, childLeft + width, childTop + height);
            }
        }
    }
```
##### Draw
先看view.draw方法
```java
    @CallSuper
    public void draw(Canvas canvas) {
        final int privateFlags = mPrivateFlags;//拿到flags
        final boolean dirtyOpaque = (privateFlags & PFLAG_DIRTY_MASK) == PFLAG_DIRTY_OPAQUE &&
                (mAttachInfo == null || !mAttachInfo.mIgnoreDirtyState);//脏标记
        mPrivateFlags = (privateFlags & ~PFLAG_DIRTY_MASK) | PFLAG_DRAWN;//去掉脏标记
        int saveCount;

        if (!dirtyOpaque) {
            drawBackground(canvas);//绘制背景
        }

        final int viewFlags = mViewFlags;
        boolean horizontalEdges = (viewFlags & FADING_EDGE_HORIZONTAL) != 0;//是否需要绘制水平边缘
        boolean verticalEdges = (viewFlags & FADING_EDGE_VERTICAL) != 0;//是否需要绘制垂直边缘
        if (!verticalEdges && !horizontalEdges) {//都不需要
            // Step 3, draw the content
            if (!dirtyOpaque) onDraw(canvas);//ondraw 绘制内容

            // Step 4, draw the children
            dispatchDraw(canvas);//绘制子view

            drawAutofilledHighlight(canvas);//绘制自适应高光

            // Overlay is part of the content and draws beneath Foreground
            if (mOverlay != null && !mOverlay.isEmpty()) {
                mOverlay.getOverlayView().dispatchDraw(canvas);//绘制Overlay
            }

            // Step 6, draw decorations (foreground, scrollbars)
            onDrawForeground(canvas);//绘制前景

            // Step 7, draw the default focus highlight
            drawDefaultFocusHighlight(canvas);//焦点高光

            if (debugDraw()) {
                debugDrawFocus(canvas);
            }

            // we're done...
            return;
        }

      // 需要绘制边缘阴影的话，则执行全部2-6的流程，不过这个流程并不常见，而且性能和速度上也不是很优秀。 所以我下面就给省略了嘿嘿
    }
```
上面方法中我们重点看一下分发子view的绘制
这个方法在viewgroup中实现
```java
protected void dispatchDraw(Canvas canvas) {//分发绘制
        boolean usingRenderNodeProperties = canvas.isRecordingFor(mRenderNode);//记录
        final int childrenCount = mChildrenCount;
        final View[] children = mChildren;
        int flags = mGroupFlags;
        if ((flags & FLAG_RUN_ANIMATION) != 0 && canAnimate()) {
            final boolean buildCache = !isHardwareAccelerated();
            for (int i = 0; i < childrenCount; i++) {
                final View child = children[i];
                if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE) {
                    final LayoutParams params = child.getLayoutParams();
                    attachLayoutAnimationParameters(child, params, i, childrenCount);//布局动画相关
                    bindLayoutAnimation(child);
                }
            }

            final LayoutAnimationController controller = mLayoutAnimationController;
            if (controller.willOverlap()) {
                mGroupFlags |= FLAG_OPTIMIZE_INVALIDATE;
            }

            controller.start();

            mGroupFlags &= ~FLAG_RUN_ANIMATION;//标记位
            mGroupFlags &= ~FLAG_ANIMATION_DONE;

            if (mAnimationListener != null) {
                mAnimationListener.onAnimationStart(controller.getAnimation());
            }
        }

        int clipSaveCount = 0;
        final boolean clipToPadding = (flags & CLIP_TO_PADDING_MASK) == CLIP_TO_PADDING_MASK;
        if (clipToPadding) {
            clipSaveCount = canvas.save(Canvas.CLIP_SAVE_FLAG);//是否进行padding剪裁canvas
            canvas.clipRect(mScrollX + mPaddingLeft, mScrollY + mPaddingTop,
                    mScrollX + mRight - mLeft - mPaddingRight,
                    mScrollY + mBottom - mTop - mPaddingBottom);
        }

        // We will draw our child's animation, let's reset the flag
        mPrivateFlags &= ~PFLAG_DRAW_ANIMATION;
        mGroupFlags &= ~FLAG_INVALIDATE_REQUIRED;

        boolean more = false;
        final long drawingTime = getDrawingTime();

        if (usingRenderNodeProperties) canvas.insertReorderBarrier();
        final int transientCount = mTransientIndices == null ? 0 : mTransientIndices.size();
        int transientIndex = transientCount != 0 ? 0 : -1;
        // Only use the preordered list if not HW accelerated, since the HW pipeline will do the
        // draw reordering internally
        final ArrayList<View> preorderedList = usingRenderNodeProperties
                ? null : buildOrderedChildList();
        final boolean customOrder = preorderedList == null
                && isChildrenDrawingOrderEnabled();
        for (int i = 0; i < childrenCount; i++) {//遍历child
            while (transientIndex >= 0 && mTransientIndices.get(transientIndex) == i) {
                final View transientChild = mTransientViews.get(transientIndex);
                if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                        transientChild.getAnimation() != null) {
                    more |= drawChild(canvas, transientChild, drawingTime);//绘制临时child
                }
                transientIndex++;
                if (transientIndex >= transientCount) {
                    transientIndex = -1;
                }
            }

            final int childIndex = getAndVerifyPreorderedIndex(childrenCount, i, customOrder);
            final View child = getAndVerifyPreorderedView(preorderedList, children, childIndex);
            if ((child.mViewFlags & VISIBILITY_MASK) == VISIBLE || child.getAnimation() != null) {
                more |= drawChild(canvas, child, drawingTime);//绘制child
            }
        }
        while (transientIndex >= 0) {
            // there may be additional transient views after the normal views
            final View transientChild = mTransientViews.get(transientIndex);
            if ((transientChild.mViewFlags & VISIBILITY_MASK) == VISIBLE ||
                    transientChild.getAnimation() != null) {
                more |= drawChild(canvas, transientChild, drawingTime);//再次绘制
            }
            transientIndex++;
            if (transientIndex >= transientCount) {//临时view绘制完毕
                break;
            }
        }
        if (preorderedList != null) preorderedList.clear();//清空

        // Draw any disappearing views that have animations
        if (mDisappearingChildren != null) {//消失动画
            final ArrayList<View> disappearingChildren = mDisappearingChildren;
            final int disappearingCount = disappearingChildren.size() - 1;
            // Go backwards -- we may delete as animations finish
            for (int i = disappearingCount; i >= 0; i--) {
                final View child = disappearingChildren.get(i);
                more |= drawChild(canvas, child, drawingTime);
            }
        }
        if (usingRenderNodeProperties) canvas.insertInorderBarrier();

        if (debugDraw()) {
            onDebugDraw(canvas);
        }

        if (clipToPadding) {
            canvas.restoreToCount(clipSaveCount);
        }

        // mGroupFlags might have been updated by drawChild()
        flags = mGroupFlags;

        if ((flags & FLAG_INVALIDATE_REQUIRED) == FLAG_INVALIDATE_REQUIRED) {
            invalidate(true);//刷新
        }

        if ((flags & FLAG_ANIMATION_DONE) == 0 && (flags & FLAG_NOTIFY_ANIMATION_LISTENER) == 0 &&
                mLayoutAnimationController.isDone() && !more) {
            // We want to erase the drawing cache and notify the listener after the
            // next frame is drawn because one extra invalidate() is caused by
            // drawChild() after the animation is over
            mGroupFlags |= FLAG_NOTIFY_ANIMATION_LISTENER;
            final Runnable end = new Runnable() {
               @Override
               public void run() {
                   notifyAnimationListener();
               }
            };
            post(end);
        }
    }
```
上面有几个变量,我们看一下
1. TransientViews
这是一个用于保存临时view的list,全局搜索了一下用的地方不是很多?注释说明这里面的view要在影响之前移除
下面再看一下drawChild方法
```java
protected boolean drawChild(Canvas canvas, View child, long drawingTime) {
        return child.draw(canvas, this, drawingTime);//可见直接调用了view.draw对子view进行绘制
    }
```

#### END


