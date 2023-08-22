# 

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903778303361038)

1.  [RecyclerView 缓存机制 | 如何复用表项？](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")
    
2.  [RecyclerView 缓存机制 | 回收些什么？](https://juejin.cn/post/6844903778303361038 "https://juejin.cn/post/6844903778303361038")
    
3.  [RecyclerView 缓存机制 | 回收到哪去？](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")
    
4.  [RecyclerView 缓存机制 | scrap view 的生命周期](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")
    
5.  [RecyclerView 动画原理 | pre-layout，post-layout 与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")
    
6.  [RecyclerView 面试题 | 哪些情况下表项会被回收到缓存池？](https://juejin.cn/post/6930412704578404360/ "https://juejin.cn/post/6930412704578404360/")
    

如果想直接看结论可以移步到[第四篇](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")末尾（你会后悔的，过程更加精彩）。

上一篇文章讲述了 “从哪里获得回收的表项”，这一篇会结合实际回收场景分析下 “回收哪些表项？”。

回收场景
----

_**在众多回收场景中最显而易见的就是 “滚动列表时移出屏幕的表项被回收”。滚动是由`MotionEvent.ACTION_MOVE`事件触发的，就以`RecyclerView.onTouchEvent()`为切入点寻觅 “回收表项” 的时机**_：

```
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild2 {
    @Override
    public boolean onTouchEvent(MotionEvent e) {
            ...
            case MotionEvent.ACTION_MOVE: {
                    ...
                    // 内部滚动
                    if (scrollByInternal(
                            canScrollHorizontally ? dx : 0,
                            canScrollVertically ? dy : 0,
                            vtev)) {
                        getParent().requestDisallowInterceptTouchEvent(true);
                    }
                    ...
                }
            } break;
            ...
    }
}
复制代码

```

去掉了大量位移赋值逻辑后，一个处理滚动的函数出现在眼前：

```
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild2 {
   ...
   LayoutManager mLayout;// 处理滚动的LayoutManager
   ...
   boolean scrollByInternal(int x, int y, MotionEvent ev) {
        ...
        if (mAdapter != null) {
            ...
            if (x != 0) { // 水平滚动
                consumedX = mLayout.scrollHorizontallyBy(x, mRecycler, mState);
                unconsumedX = x - consumedX;
            }
            if (y != 0) { // 垂直滚动
                consumedY = mLayout.scrollVerticallyBy(y, mRecycler, mState);
                unconsumedY = y - consumedY;
            }
            ...
        }
        ...
}
复制代码

```

`RecyclerView`把滚动委托给`LayoutManager`来处理：

```
public class LinearLayoutManager extends RecyclerView.LayoutManager implements ItemTouchHelper.ViewDropHandler, RecyclerView.SmoothScroller.ScrollVectorProvider {

    @Override
    public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler,
            RecyclerView.State state) {
        if (mOrientation == HORIZONTAL) {
            return 0;
        }
        return scrollBy(dy, recycler, state);
    }

    int scrollBy(int dy, RecyclerView.Recycler recycler, RecyclerView.State state) {
    	...
        //更新LayoutState（这个函数对于“回收哪些表项”来说很关键，待会会提到）
        updateLayoutState(layoutDirection, absDy, true, state);
        //滚动时向列表中填充新的表项
        final int consumed = mLayoutState.mScrollingOffset + fill(recycler, mLayoutState, state, false);
		...
        return scrolled;
    }
    ...
}
复制代码

```

沿着调用链往下找，发现了一个[上一篇](https://juejin.cn/post/6844903778303361038 "https://juejin.cn/post/6844903778303361038")中介绍过的函数`LinearLayoutManager.fill()`，列表滚动的同时会不断的向其中填充表项。

上一遍只关注了其中填充的逻辑，里面还有回收逻辑：

```
public class LinearLayoutManager extends RecyclerView.LayoutManager {
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState, RecyclerView.State state, boolean stopOnFocusable) {
        ...
        int remainingSpace = layoutState.mAvailable + layoutState.mExtra;
        LayoutChunkResult layoutChunkResult = mLayoutChunkResult;
        //不断循环获取新的表项用于填充，直到没有填充空间
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            ...
            //填充新的表项
            layoutChunk(recycler, state, layoutState, layoutChunkResult);
			...
            if (layoutState.mScrollingOffset != LayoutState.SCROLLING_OFFSET_NaN) {
                //在当前滚动偏移量基础上追加因新表项插入增加的像素（这句话对于“回收哪些表项”来说很关键）
                layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
                ...
                //回收表项
                recycleByLayoutState(recycler, layoutState);
            }
            ...
        }
        ...
        return start - layoutState.mAvailable;
    }
}
复制代码

```

在不断获取新表项用于填充的同时也在回收表项，就好比滚动着的列表，有表项插入的同时也有表项被移出，移步到回收表项的函数：

```
public class LinearLayoutManager extends RecyclerView.LayoutManager {
    ...
    private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
        if (!layoutState.mRecycle || layoutState.mInfinite) {
            return;
        }
        if (layoutState.mLayoutDirection == LayoutState.LAYOUT_START) {
            // 从列表头回收
            recycleViewsFromEnd(recycler, layoutState.mScrollingOffset);
        } else {
            // 从列表尾回收
            recycleViewsFromStart(recycler, layoutState.mScrollingOffset);
        }
    }
    ...
    /**
     * 当向列表尾部滚动时回收滚出屏幕的表项
     * @param dt（该参数被用于检测滚出屏幕的表项）
     */
    private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,int noRecycleSpace) {
        final int limit = scrollingOffset - noRecycleSpace;
        //从头开始遍历 LinearLayoutManager，以找出应该会回收的表项
        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            // 如果表项的下边界 > limit 这个阈值
            if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
                //回收索引为 0 到 i-1 的表项
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
    ...
}
复制代码

```

`RecyclerView`的回收分两个方向：1. 从列表头回收 2. 从列表尾回收。

就以 “从列表头回收” 为研究对象分析下`RecyclerView`在滚动时到底是怎么判断 “哪些表项应该被回收？”。 （“从列表头回收表项” 所对应的场景是：手指上滑，列表向下滚动，新的表项逐个插入到列表尾部，列表头部的表项逐个被回收。）

回收哪些表项
------

要回答这个问题，刚才那段代码中套在`recycleChildren(recycler, 0, i)`外面的判断逻辑是关键：`mOrientationHelper.getDecoratedEnd(child) > limit`。

其中的`mOrientationHelper.getDecoratedEnd(child)`代码如下：

```
// 屏蔽方向的抽象接口，用于减少关于方向的 if-else
public abstract class OrientationHelper {
    // 获取当前表项相对于列表头部的坐标
    public abstract int getDecoratedEnd(View view);
    // 垂直布局对该接口的实现
    public static OrientationHelper createVerticalHelper(RecyclerView.LayoutManager layoutManager) {
        return new OrientationHelper(layoutManager) {
            @Override
            public int getDecoratedEnd(View view) {
                final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams)view.getLayoutParams();
                return mLayoutManager.getDecoratedBottom(view) + params.bottomMargin;
            }
}
复制代码

```

`mOrientationHelper.getDecoratedEnd(child)` 表示当前表项的尾部相对于列表头部的坐标，`OrientationHelper`这层抽象屏蔽了列表的方向，所以这句话在纵向列表中可以翻译成 “当前表项的底部相对于列表顶部的纵坐标”。

判断条件`mOrientationHelper.getDecoratedEnd(child) > limit`中的`limit`又是什么意思？

_**在纵向列表中，“表项底部纵坐标> 某个值” 意味着表项位于某条线的下方，即 limit 是列表中隐形的线，所有在这条线上方的表项都应该被回收。**_

那这条线是如何被计算的？

```
public class LinearLayoutManager extends RecyclerView.LayoutManager {
    private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,int noRecycleSpace) {
        final int limit = scrollingOffset - noRecycleSpace;
        ...
    }
}
复制代码

```

`limit`的值由 2 个变量决定，其中`noRecycleSpace`的值为 0（这是断点告诉我的，详细过程可移步 [RecyclerView 动画原理 | 换个姿势看源码（pre-layout）](https://juejin.cn/post/6890288761783975950/ "https://juejin.cn/post/6890288761783975950/")）

而`scrollingOffset`的值由外部传入：

```
public class LinearLayoutManager {
    private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
        int scrollingOffset = layoutState.mScrollingOffset;
        ...
        recycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}
复制代码

```

问题转换为`layoutState.mScrollingOffset`的值由什么决定？全局搜索下它被赋值的地方：

```
public class LinearLayoutManager {
    private void updateLayoutState(int layoutDirection, int requiredSpace,boolean canUseExistingSpace, RecyclerView.State state) {
        ...
        int scrollingOffset;
        // 获取末尾的表项视图
        final View child = getChildClosestToEnd();
        // 计算在不往列表里填充新表项的情况下，列表最多可以滚动多少像素
        scrollingOffset = mOrientationHelper.getDecoratedEnd(child) - mOrientationHelper.getEndAfterPadding();
        ...
        mLayoutState.mScrollingOffset = scrollingOffset;
    }
}
复制代码

```

`updateLayoutState()`方法中先获取了列表末尾表项的视图，并通过`mOrientationHelper.getDecoratedEnd(child)`计算出该表项底部到列表顶部的距离，然后在减去列表长度。这个差值可以理解为**在不往列表里填充新表项的情况下，列表最多可以滚动多少像素**。略抽象，图示如下：

![][img-0] 图中蓝色边框表示列表，灰色矩形表示表项。

`LayoutManager`只会加载可见表项，图中表项 6 有一半露出了屏幕，所以它会被加载到列表中，而表项 7 完全不可见，所以不会被加载。这种情况下，如果不继续往列表中填充表项 7，那列表最多滑动的距离就是半个表项 6 的距离，表项在代码中即是`mLayoutState.mScrollingOffset`的值。

若非常缓慢地滑动列表，并且只滑动 “半个表项 6” 的距离（即表项 7 没有机会展示）。在这个理想的场景下`limit`的值 = 半个表项 6 的长度。也就是说`limit`这根隐形的线应该在如下位置：

![][img-1]

回看一下，回收表项的代码：

```
public class LinearLayoutManager {
    private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,int noRecycleSpace) {
        final int limit = scrollingOffset - noRecycleSpace;
        //从头开始遍历 LinearLayoutManager，以找出应该会回收的表项
        final int childCount = getChildCount();
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            // 如果表项的下边界 > limit 这个阈值
            if (mOrientationHelper.getDecoratedEnd(child) > limit
                    || mOrientationHelper.getTransformedEndWithDecoration(child) > limit) {
                //回收索引为 0 到 i-1 的表项
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
}
复制代码

```

回收逻辑从头开始遍历 `LinearLayoutManager`，当遍历到表项 1 的时候，发现它的下边界 > limit，所以触发表项回收，回收表项的索引区间为 0 到 0，即没有任何表项被回收。（想想也是，表项 1 还未完整地被移出屏幕）。

若滑动速度和距离更大会发生什么？

计算`limit`值的方法`updateLayoutState()`在`scrollBy()`中被调用：

```
public class LinearLayoutManager {
    int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
        ...
        // 将滚动距离的绝对值传入 updateLayoutState()
        final int absDelta = Math.abs(delta);
        updateLayoutState(layoutDirection, absDelta, true, state);
        ...
    }
    
    private void updateLayoutState(int layoutDirection, int requiredSpace,boolean canUseExistingSpace, RecyclerView.State state) {
        ...
        // 计算在不往列表里填充新表项的情况下，列表最多可以滚动多少像素
        scrollingOffset = mOrientationHelper.getDecoratedEnd(child)- mOrientationHelper.getEndAfterPadding();
        ...
        // 将列表因滚动而需要的额外空间存储在 mLayoutState.mAvailable
        mLayoutState.mAvailable = requiredSpace;
        mLayoutState.mScrollingOffset = scrollingOffset;
        ...
    }
}
复制代码

```

至此，两个重要的值被分别存储在`mLayoutState.mScrollingOffset`和`mLayoutState.mAvailable`，分别是 “在不往列表里填充新表项的情况下，列表最多可以滚动多少像素”，及 “滚动总像素值”。

`srollBy()`在调用`updateLayoutState()`存储了这两个重要的值之后，立马进行了填充表项的操作：

```
public class LinearLayoutManager {
    int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
        ...
        final int absDelta = Math.abs(delta);
        updateLayoutState(layoutDirection, absDelta, true, state);
        final int consumed = mLayoutState.mScrollingOffset + fill(recycler, mLayoutState, state, false);
        ...
    }
}
复制代码

```

填充表项
----

其中的`fill()`即是向列表填充表项的方法：

```
public class LinearLayoutManager {
    // 根据剩余空间填充表项
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,RecyclerView.State state, boolean stopOnFocusable) {
        ...
        // 计算剩余空间 = 可用空间 + 额外空间(=0)
        int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
        // 循环，当剩余空间 > 0 时，继续填充更多表项
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            ...
            // 填充单个表项
            layoutChunk(recycler, state, layoutState, layoutChunkResult)
            ...
            // 从剩余空间中扣除新表项占用像素值
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            remainingSpace -= layoutChunkResult.mConsumed;
            ...
        }
    }
}
复制代码

```

填充表项是一个`while`循环，循环结束条件是 “列表剩余空间是否> 0”，每次循环调用`layoutChunk()`将单个表项填充到列表中：

```
public class LinearLayoutManager {
    // 填充单个表项
    void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,LayoutState layoutState, LayoutChunkResult result) {
        // 1.获取下一个该被填充的表项视图
        View view = layoutState.next(recycler);
        // 2.使表项成为 RecyclerView 的子视图
        addView(view);
        ...
        // 3.测量表项视图（把 RecyclerView 内边距和表项装饰考虑在内）
        measureChildWithMargins(view, 0, 0);
        // 获取填充表项视图需要消耗的像素值
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
        ...
        // 4.布局表项
        layoutDecoratedWithMargins(view, left, top, right, bottom);
    }
}
复制代码

```

`layoutChunk()`先从缓存池中获取下一个该被填充表项的视图（关于复用的详细分析可以移步 [RecyclerView 缓存机制 | 如何复用表项？](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")）。

紧接着调用了`addView()`使表项视图成为 RecyclerView 的子视图，调用链如下：

```
public class RecyclerView {
    ChildHelper mChildHelper;
    public abstract static class LayoutManager {
        public void addView(View child) {
            addView(child, -1);
        }
        
        public void addView(View child, int index) {
            addViewInt(child, index, false);
        }
        
        private void addViewInt(View child, int index, boolean disappearing) {
            ...
            mChildHelper.attachViewToParent(child, index, child.getLayoutParams(), false);
            ...
        }
    }
}

class ChildHelper {
    final Callback mCallback;
    void attachViewToParent(View child, int index, ViewGroup.LayoutParams layoutParams,boolean hidden) {
        ...
        mCallback.attachViewToParent(child, offset, layoutParams);
    }
}
复制代码

```

调用链从`RecyclerView`到`LayoutManager`再到`ChildHelper`，最后又回到了`RecyclerView`：

```
public class RecyclerView {
    ChildHelper mChildHelper;
    private void initChildrenHelper() {
        mChildHelper = new ChildHelper(new ChildHelper.Callback() {
            @Override
            public void attachViewToParent(View child, int index,ViewGroup.LayoutParams layoutParams) {
                ...
                RecyclerView.this.attachViewToParent(child, index, layoutParams);
            }
            ...
        }
    }
}
复制代码

```

`addView()`的最终落脚点是`ViewGroup.attachViewToParent()`：

```
public abstract class ViewGroup {
    protected void attachViewToParent(View child, int index, LayoutParams params) {
        child.mLayoutParams = params;

        if (index < 0) {
            index = mChildrenCount;
        }

        // 将子视图添加到数组中
        addInArray(child, index);
        // 子视图和父亲关联
        child.mParent = this;
        child.mPrivateFlags = (child.mPrivateFlags & ~PFLAG_DIRTY_MASK
                        & ~PFLAG_DRAWING_CACHE_VALID)
                | PFLAG_DRAWN | PFLAG_INVALIDATED;
        this.mPrivateFlags |= PFLAG_INVALIDATED;

        if (child.hasFocus()) {
            requestChildFocus(child, child.findFocus());
        }
        dispatchVisibilityAggregated(isAttachedToWindow() && getWindowVisibility() == VISIBLE
                && isShown());
        notifySubtreeAccessibilityStateChangedIfNeeded();
    }
}
复制代码

```

`attachViewToParent()`中包含了 “添加子视图” 最具标志性的两个动作：1. 将子视图添加到数组中 2. 子视图和父亲关联。

使表项成为 RecyclerView 子视图之后，对其进行了测量:

```
public class LinearLayoutManager {
    // 填充单个表项
    void layoutChunk(RecyclerView.Recycler recycler, RecyclerView.State state,LayoutState layoutState, LayoutChunkResult result) {
        // 1.获取下一个该被填充的表项视图
        View view = layoutState.next(recycler);
        // 2.使表项成为 RecyclerView 的子视图
        addView(view);
        ...
        // 3.测量表项视图（把 RecyclerView 内边距和表项装饰考虑在内）
        measureChildWithMargins(view, 0, 0);
        // 获取填充表项视图需要消耗的像素值
        result.mConsumed = mOrientationHelper.getDecoratedMeasurement(view);
        ...
        // 4.布局表项
        layoutDecoratedWithMargins(view, left, top, right, bottom);
    }
}
复制代码

```

测量之后，有了视图的尺寸，就可以知道填充该表项会消耗掉多少像素值，将该数值存储在`LayoutChunkResult.mConsumed`中。

有了尺寸后，就可以布局表项了，即确定表项上下左右四个点相对于 RecyclerView 的位置：

```
public class RecyclerView {
    public abstract static class LayoutManager {
        public void layoutDecoratedWithMargins(@NonNull View child, int left, int top, int right,
                int bottom) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();
            final Rect insets = lp.mDecorInsets;
            // 为表项定位
            child.layout(left + insets.left + lp.leftMargin, top + insets.top + lp.topMargin,
                    right - insets.right - lp.rightMargin,
                    bottom - insets.bottom - lp.bottomMargin);
        }
    }
}
复制代码

```

调用控件的`layout()`方法即是为控件定位，关于定位子控件的详细介绍可以移步 [Android 自定义控件 | View 绘制原理（画在哪？）](https://juejin.cn/post/6844904070394707981 "https://juejin.cn/post/6844904070394707981")。

填充完一个表项后，会从`remainingSpace`中扣除它所占用的空间（这样 while 循环才能结束）

```
public class LinearLayoutManager {
    // 根据剩余空间填充表项
    int fill(RecyclerView.Recycler recycler, LayoutState layoutState,RecyclerView.State state, boolean stopOnFocusable) {
        ...
        // 计算剩余空间 = 可用空间 + 额外空间(=0)
        int remainingSpace = layoutState.mAvailable + layoutState.mExtraFillSpace;
        // 循环，当剩余空间 > 0 时，继续填充更多表项
        while ((layoutState.mInfinite || remainingSpace > 0) && layoutState.hasMore(state)) {
            ...
            // 填充单个表项
            layoutChunk(recycler, state, layoutState, layoutChunkResult)
            ...
            // 从剩余空间中扣除新表项占用像素值
            layoutState.mAvailable -= layoutChunkResult.mConsumed;
            remainingSpace -= layoutChunkResult.mConsumed;
            ...
            // 在 limit 上追加新表项所占像素值
            layoutState.mScrollingOffset += layoutChunkResult.mConsumed;
            ...
            // 根据当前状态回收表项
            recycleByLayoutState(recycler, layoutState);
            }
        }
    }
}
复制代码

```

`layoutState.mScrollingOffset`会追加新表项所占用的像素值，即它的值在不断增大（`limit 隐形线`在不断下移）。

在一次`while`循环的最后，会根据当前`limit 隐形线`的位置回收表项：

```
public class LinearLayoutManager {
    private void recycleByLayoutState(RecyclerView.Recycler recycler, LayoutState layoutState) {
        ...
        ecycleViewsFromStart(recycler, scrollingOffset, noRecycleSpace);
    }
}
    
        private void recycleViewsFromStart(RecyclerView.Recycler recycler, int scrollingOffset,int noRecycleSpace) {
        final int limit = scrollingOffset - noRecycleSpace;
        final int childCount = getChildCount();
        // 从头遍历表项
        for (int i = 0; i < childCount; i++) {
            View child = getChildAt(i);
            // 当某表项底部位于 limit 隐形线之后时，回收它以上的所有表项
            if (mOrientationHelper.getDecoratedStart(child) > limit || mOrientationHelper.getTransformedStartWithDecoration(child) > limit) {
                recycleChildren(recycler, 0, i);
                return;
            }
        }
    }
}
复制代码

```

每向列表尾部填充一个表项，`limit隐形线`的位置就往下移动表项占用的像素值，这样列表头部也就有更多的表项符合被回收的条件。

关于回收细节的分析，可以移步 [RecyclerView 缓存机制 | 回收到哪去？](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")。

预计的滑动距离被传入`scrollBy()`，`scrollBy()`把即将滑入屏幕的表项填充到列表中，同时把即将移出屏幕的表项回收到缓存池，最后它会比较预计滑动值和计算滑动值的大小，取其中的较小者返回：

```
public class LinearLayoutManager {
    // 第一个参数是预计的滑动距离
    int scrollBy(int delta, RecyclerView.Recycler recycler, RecyclerView.State state) {
        ...
        final int absDelta = Math.abs(delta);
        updateLayoutState(layoutDirection, absDelta, true, state);
        // 经过计算的滚动值
        final int consumed = mLayoutState.mScrollingOffset + fill(recycler, mLayoutState, state, false);
        // 最终返回的滚动值
        final int scrolled = absDelta > consumed ? layoutDirection * consumed : delta;
        ...
        return scrolled;
    }
}
复制代码

```

沿着`scrollBy()`调用链网上寻找：

```
public class LinearLayoutManager {
    @Override
    public int scrollVerticallyBy(int dy, RecyclerView.Recycler recycler,
            RecyclerView.State state) {
        if (mOrientation == HORIZONTAL) {
            return 0;
        }
        return scrollBy(dy, recycler, state);
    }
}
    
public class RecyclerView {
    void scrollStep(int dx, int dy, @Nullable int[] consumed) {
        ...
        if (dy != 0) {
            consumedY = mLayout.scrollVerticallyBy(dy, mRecycler, mState);
        }
        ...
    }
    
    boolean scrollByInternal(int x, int y, MotionEvent ev) {
        ...
       scrollStep(x, y, mReusableIntPair);
       ...
       dispatchNestedScroll(consumedX, consumedY, unconsumedX, unconsumedY, mScrollOffset,TYPE_TOUCH, mReusableIntPair);
        ...
    }
}
复制代码

```

只有当执行了`dispatchNestedScroll()`才会真正触发列表的滚动，也就说 RecyclerView 在列表滚动发生之前就预先计算好了，哪些表项会移入屏幕，哪些表项会移出屏幕，并分别将它们填充到列表或回到到缓存池。而做这两件事的依据即是`limit隐形线`，最后用一张图来概括下这条线的意义： ![][img-2]

`limit`的值表示这一次滚动的总距离。（图中是一种理想情况，即当滚动结束后新插入表项 7 的底部正好和列表底部重叠）

`limit隐形线`可以理解为：**隐形线当前所在位置，在滚动完成后会和列表顶部重合**

推荐阅读
----

RecyclerView 系列文章目录如下:

1.  [RecyclerView 缓存机制 | 如何复用表项？](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")
    
2.  [RecyclerView 缓存机制 | 回收些什么？](https://juejin.cn/post/6844903778303361038 "https://juejin.cn/post/6844903778303361038")
    
3.  [RecyclerView 缓存机制 | 回收到哪去？](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")
    
4.  [RecyclerView 缓存机制 | scrap view 的生命周期](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")
    
5.  [读源码长知识 | 更好的 RecyclerView 点击监听器](https://juejin.cn/post/6844903862361391117 "https://juejin.cn/post/6844903862361391117")
    
6.  [代理模式应用 | 每当为 RecyclerView 新增类型时就很抓狂](https://juejin.cn/post/6876967151975006221 "https://juejin.cn/post/6876967151975006221")
    
7.  [更好的 RecyclerView 表项子控件点击监听器](https://juejin.cn/post/6881427923316768776 "https://juejin.cn/post/6881427923316768776")
    
8.  [更高效地刷新 RecyclerView | DiffUtil 二次封装](https://juejin.cn/post/6882531923537707015 "https://juejin.cn/post/6882531923537707015")
    
9.  [换一个思路，超简单的 RecyclerView 预加载](https://juejin.cn/post/6885146484791050247 "https://juejin.cn/post/6885146484791050247")
    
10.  [RecyclerView 动画原理 | 换个姿势看源码（pre-layout）](https://juejin.cn/post/6890288761783975950 "https://juejin.cn/post/6890288761783975950")
    
11.  [RecyclerView 动画原理 | pre-layout，post-layout 与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")
    
12.  [RecyclerView 动画原理 | 如何存储并应用动画属性值？](https://juejin.cn/post/6895523568025600014 "https://juejin.cn/post/6895523568025600014")
    
13.  [RecyclerView 面试题 | 列表滚动时，表项是如何被填充或回收的？](https://juejin.cn/post/6903290882095579143 "https://juejin.cn/post/6903290882095579143")
    
14.  [RecyclerView 面试题 | 哪些情况下表项会被回收到缓存池？](https://juejin.cn/post/6930412704578404360/ "https://juejin.cn/post/6930412704578404360/")
    
15.  [RecyclerView 性能优化 | 把加载表项耗时减半 (一)](https://juejin.cn/post/6939666015500369950 "https://juejin.cn/post/6939666015500369950")
    
16.  [RecyclerView 性能优化 | 把加载表项耗时减半 (二)](https://juejin.cn/post/6942276625090215943 "https://juejin.cn/post/6942276625090215943")
    
17.  [RecyclerView 性能优化 | 把加载表项耗时减半 (三)](https://juejin.cn/post/6954892589539524615 "https://juejin.cn/post/6954892589539524615")
    
18.  [RecyclerView 的滚动是怎么实现的？（一）| 解锁阅读源码新姿势](https://juejin.cn/post/6958962329220284453 "https://juejin.cn/post/6958962329220284453")
    
19.  [RecyclerView 的滚动时怎么实现的？（二）| Fling](https://juejin.cn/post/6960457567680069645/ "https://juejin.cn/post/6960457567680069645/")
    
20.  [RecyclerView 刷新列表数据的 notifyDataSetChanged() 为什么是昂贵的?](https://juejin.cn/post/6965633977960890381 "https://juejin.cn/post/6965633977960890381")

[img-0]:data:image/webp;base64,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

[img-1]:data:image/webp;base64,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

[img-2]:data:image/webp;base64,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
