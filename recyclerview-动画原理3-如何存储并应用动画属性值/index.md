# 



这是 RecyclerView 动画原理的第三篇，系列文章目录如下：

1.  [RecyclerView 动画原理 | 换个姿势看源码（pre-layout）](https://juejin.cn/post/6890288761783975950 "https://juejin.cn/post/6890288761783975950")
2.  [RecyclerView 动画原理 | pre-layout，post-layout 与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")
3.  [RecyclerView 动画原理 | 如何存储并应用动画属性值？](https://juejin.cn/post/6895523568025600014 "https://juejin.cn/post/6895523568025600014")

引子
--

这一篇源码分析还是基于下面这个 Demo 场景：

![][img-0]

列表中有两个表项（1、2），删除 2，此时 3 会从屏幕底部平滑地移入并占据原来 2 的位置。

为了实现该效果，`RecyclerView`的策略是：为动画前的表项先执行一次`pre-layout`，将不可见的表项 3 也加载到布局中，形成一张布局快照（1、2、3）。再为动画后的表项执行一次`post-layout`，同样形成一张布局快照（1、3）。比对两张快照中表项 3 的位置，就知道它该如何做动画了。

在此援引上一篇已经得出的结论：

> 1.  `RecyclerView`为了实现表项动画，进行了 2 次布局（预布局 + 后布局），在源码上表现为`LayoutManager.onLayoutChildren()`被调用 2 次。
>     
> 2.  `State.mInPreLayout`用于标识是否在预布局阶段。预布局的生命周期始于`RecyclerView.dispatchLayoutStep1()`，终于`RecyclerView.dispatchLayoutStep2()`。
>     
> 3.  在预布局阶段，循环填充表项时，若遇到被移除的表项，则会忽略它占用的空间，多余空间被用来加载额外的表项，这些表项在屏幕之外，本来不会被加载。
>     

其中第三点表现在源码上，是这样的：

```
public class LinearLayoutManager {
    // 布局表项
    public void onLayoutChildren() {
        // 不断填充表项
        fill() {
            while(列表有剩余空间){
                // 填充单个表项
                layoutChunk(){
                    // 让表项成为子视图
                    addView(view)
                }
                if (表项没有被移除) {
                    剩余空间 -= 表项占用空间	
                }
                ...
            }
        }
    }
}
复制代码

```

这是`RecyclerView`填充表项的伪码。以 Demo 为例，预布局阶段，第一次执行`onLayoutChildren()`，因表项 2 被删除，所以它占用的空间不会被扣除，导致`while`循环多执行一次，这样表项 3 就被填充进列表。

存后布局动画属性值
---------

`RecyclerView`用一个 Int 值`mLayoutStep`标记布局阶段，它有三种可能的取值。

```
public class RecyclerView {
    public static class State {
        static final int STEP_START = 1;
        static final int STEP_LAYOUT = 1 << 1;
        static final int STEP_ANIMATIONS = 1 << 2; // 布局的动画阶段
        int mLayoutStep = STEP_START; // 当前布局阶段
    }
}
复制代码

```

只要全局查找下`mLayoutStep`什么时候被赋值为`STEP_ANIMATIONS`，就可以知道表项动画什么时候开始：

```
public class RecyclerView {
    final State mState = new State();
    private void dispatchLayoutStep2() {// 布局子表项第二阶段
        mState.mInPreLayout = false; // 预布局结束
        mLayout.onLayoutChildren(mRecycler, mState); // 开始后布局

        mState.mLayoutStep = State.STEP_ANIMATIONS; // 标记为布局的动画阶段
        ...
    }
}
复制代码

```

RecyclerView 在后布局结束后，将`mState.mLayoutStep`置为`State.STEP_ANIMATIONS`，表示表项动画即将开始。

在紧接着的 “布局子表项第三阶段” 的开头，就断言：

```
public class RecyclerView {
    @Override
    protected void onLayout(boolean changed, int l, int t, int r, int b) {
        ...
        dispatchLayout();// 开始布局 RecyclerView 的子表项
        ...
    }
    
    void dispatchLayout() {
        ...
        if (mState.mLayoutStep == State.STEP_START) {
            dispatchLayoutStep1();// 布局子表项第一阶段
            ...
            dispatchLayoutStep2(); // 布局子表项第二阶段
        }
        ...
        dispatchLayoutStep3(); // 布局子表项第三阶段
    }
    
    private void dispatchLayoutStep3() {
        // 断言“在布局的动画阶段”
        mState.assertLayoutStep(State.STEP_ANIMATIONS);
        ...
    }
    
    public static class State {
        // 断言 mLayoutStep 是否为 accepted，否则抛异常
        void assertLayoutStep(int accepted) {
            if ((accepted & mLayoutStep) == 0) {
                throw new IllegalStateException("Layout state should be one of "
                        + Integer.toBinaryString(accepted) + " but it is "
                        + Integer.toBinaryString(mLayoutStep));
            }
        }
    }
}
复制代码

```

由此可以断定，触发动画的逻辑将会出现在`RecyclerView.dispatchLayoutStep3()`中，继续往下走读源码：

```
public class RecyclerView {
    private void dispatchLayoutStep3() {
            // 遍历表项
            for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
                // 获取表项对应 ViewHolder
                ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                // 获取表项动画信息
                final ItemHolderInfo animationInfo = mItemAnimator.recordPostLayoutInformation(mState, holder);
                ...
            }
    }                        
}
复制代码

```

RecyclerView 在布局子表项的第三阶段中遍历了当前所有的表项（对于 Demo 场景，会遍历表项 1、3），调用`ItemAnimator.recordPostLayoutInformation()`逐个构建表项动画信息`ItemHolderInfo`：

```
public class RecyclerView {
    public abstract static class ItemAnimator {
        // 记录后布局信息
        public ItemHolderInfo recordPostLayoutInformation(State state,ViewHolder viewHolder) {
            return obtainHolderInfo().setFrom(viewHolder);
        }
        // 构建表项信息
        public ItemHolderInfo obtainHolderInfo() {
            return new ItemHolderInfo();
        }
        // 表项信息实体类
        public static class ItemHolderInfo {
            // 上下左右相对于列表的距离
            public int left;
            public int top;
            public int right;
            public int bottom;
            
            public ItemHolderInfo setFrom(RecyclerView.ViewHolder holder) {
                return setFrom(holder, 0);
            }
            // 记录表项位置
            public ItemHolderInfo setFrom(RecyclerView.ViewHolder holder,int flags) {
                final View view = holder.itemView;
                this.left = view.getLeft();
                this.top = view.getTop();
                this.right = view.getRight();
                this.bottom = view.getBottom();
                return this;
            }
        }
    }
}
复制代码

```

构建的`ItemHolderInfo`实例记录了表项相对于列表左上角的位置（上下左右），然后调用`addToPostLayout()`将其添加到`ViewInfoStore`：

```
public class RecyclerView {
    // 用于存放表项动画信息
    final ViewInfoStore mViewInfoStore = new ViewInfoStore();
    private void dispatchLayoutStep3() {
            // 遍历表项
            for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
                ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                final ItemHolderInfo animationInfo = mItemAnimator.recordPostLayoutInformation(mState, holder);
                // 将后布局表项动画信息保存到 mViewInfoStore
                mViewInfoStore.addToPostLayout(holder, animationInfo);
                ...
            }
    }                        
}
复制代码

```

`ViewInfoStore`专门用于存放表项动画信息：

```
class ViewInfoStore {
    // 存放 ViewHolder 与其对应动画信息 的 ArrayMap 结构
    final ArrayMap<RecyclerView.ViewHolder, InfoRecord> mLayoutHolderMap = new ArrayMap<>();
    // 存储后布局表项与其动画信息
    void addToPostLayout(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) {
            // 从池中获取 InfoRecord 实例
            record = InfoRecord.obtain();
            // 将 ViewHolder 和 InfoRecord 绑定
            mLayoutHolderMap.put(holder, record);
        }
        record.postInfo = info; // 将后布局表项动画信息存储在 postInfo 字段中
        record.flags |= FLAG_POST; // 追加 FLAG_POST 到标志位
    }
    
    static class InfoRecord {
        int flags; // 标记位
        static final int FLAG_PRE = 1 << 2; // pre-layout 标记
        static final int FLAG_POST = 1 << 3; // post-layout 标记
        static final int FLAG_APPEAR = 1 << 1; // 表项出现标志
        RecyclerView.ItemAnimator.ItemHolderInfo preInfo;// pre-layout 表项位置信息
        RecyclerView.ItemAnimator.ItemHolderInfo postInfo;// post-layout 表项位置信息
        // 池：为避免内存抖动
        static Pools.Pool<InfoRecord> sPool = new Pools.SimplePool<>(20);
        // 从池中获取 InfoRecord 实例
        static InfoRecord obtain() {
            InfoRecord record = sPool.acquire();
            return record == null ? new InfoRecord() : record;
        }
        ...
    }
}
复制代码

```

表项动画信息被包装成`InfoRecord`实例并用一个`int`类型的标志位来标识**表项经历过哪些布局阶段**。若表项动画信息是在 post-layout 阶段被添加的，其标志位会追加`FLAG_POST`（该标记位用于判断做什么类型的动画）。最后将表项动画信息和对应的 ViewHolder 相互绑定并存储到 ArrayMap 结构中。

至此可以得出如下结论：

> RecyclerView 在布局的第三个阶段会遍历后布局中填充的所有表项，为每个表项构建动画信息实例，该实例不仅保存了表项与列表的相对位置，还用一个标记位记录了表项经历过的布局阶段，并将表项与其动画信息的对应关系存储在`ViewInfoStore`中的`mLayoutHolderMap`结构中。

将结论应用到 Demo 的场景中：列表在布局第三阶段，会遍历表项 1、3，为它们构建动画信息实例，该实例的标志位被追加了`FLAG_POST`标志。这些信息都被存储在`ViewInfoStore`中的`mLayoutHolderMap`结构中。

存预布局动画属性值
---------

`InfoRecord`中除了`postInfo`还有一个`preInfo`，分别表示后布局和预布局表项的动画信息。想必还有一个`addToPreLayout()`与`addToPostLayout()`对应：

```
class ViewInfoStore {
    // 存储预布局表项与其动画信息
    void addToPreLayout(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) {
            record = InfoRecord.obtain();
            mLayoutHolderMap.put(holder, record);
        }
        record.preInfo = info; // 将后布局表项动画信息存储在 preInfo 字段中
        record.flags |= FLAG_PRE; // 追加 FLAG_PRE 到标志位
    }
}
复制代码

```

`addToPreLayout()`在预布局阶段被调用：

```
public class RecyclerView {
    private void dispatchLayoutStep1() {
            ...
            // 遍历可见表项
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                ...
                // 构建表项动画信息
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                // 将表项动画信息保存到 mViewInfoStore
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                ...
            }
            ...
            // 预布局
            mLayout.onLayoutChildren(mRecycler, mState);
    }
}
复制代码

```

RecyclerView 布局的第一个阶段中，在第一次执行`onLayoutChildren()`之前，即预布局之前，遍历了所有的表项并逐个构建动画信息。以 Demo 为例，预布局之前，表项 1、2 的动画信息被构建并且标志位追加了`FLAG_PRE`，这些信息都被保存到`mViewInfoStore`实例中。

紧接着`RecyclerView`执行了`onLayoutChildren()`，即进行预布局。

```
public class RecyclerView {
    private void dispatchLayoutStep1() {
            // 遍历预布局前所有表项
            int count = mChildHelper.getChildCount();
            for (int i = 0; i < count; ++i) {
                final ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                ...
                final ItemHolderInfo animationInfo = mItemAnimator
                        .recordPreLayoutInformation(mState, holder,
                                ItemAnimator.buildAdapterChangeFlagsForAnimations(holder),
                                holder.getUnmodifiedPayloads());
                mViewInfoStore.addToPreLayout(holder, animationInfo);
                ...
            }
            ...
            // 预布局
            mLayout.onLayoutChildren(mRecycler, mState);
            // 遍历预布局之后所有的表项
            for (int i = 0; i < mChildHelper.getChildCount(); ++i) {
                final View child = mChildHelper.getChildAt(i);
                final ViewHolder viewHolder = getChildViewHolderInt(child);
                ...
                // 如果 ViewInfoStore 中没有对应的 ViewHolder 信息
                if (!mViewInfoStore.isInPreLayout(viewHolder)) {
                    ...
                    // 构建表项动画信息
                    final ItemHolderInfo animationInfo = mItemAnimator.recordPreLayoutInformation(mState, viewHolder, flags, viewHolder.getUnmodifiedPayloads());
                    ...
                    // 将表项 ViewHolder 和其动画信息绑定并保存在 mViewInfoStore 中
                    mViewInfoStore.addToAppearedInPreLayoutHolders(viewHolder, animationInfo);
                    
                }
            }
    }
}
复制代码

```

RecyclerView 在预布局之后再次遍历了所有表项。因为预布局会把表项 3 也填充到列表中，所以表项 3 的动画信息也会被存入`mViewInfoStore`，不过调用的是`ViewInfoStore.addToAppearedInPreLayoutHolders()`：

```
class ViewInfoStore {
    void addToAppearedInPreLayoutHolders(RecyclerView.ViewHolder holder, RecyclerView.ItemAnimator.ItemHolderInfo info) {
        InfoRecord record = mLayoutHolderMap.get(holder);
        if (record == null) {
            record = InfoRecord.obtain();
            mLayoutHolderMap.put(holder, record);
        }
        record.flags |= FLAG_APPEAR; // 追加 FLAG_APPEAR 到标志位
        record.preInfo = info; // 将预布局表项动画信息存储在 preInfo 字段中
    }
}
复制代码

```

`addToAppearedInPreLayoutHolders()`和`addToPreLayout()`的实现几乎一摸一样，唯一的不同是，标志位追加了`FLAG_APPEAR`，用于标记表项 3 是即将出现在屏幕中的表项。

分析至此，可以得出下面的结论：

> RecyclerView 经历了预布局、后布局及布局第三阶段后，`ViewInfoStore`中就记录了每一个参与动画表项的三重信息：预布局位置信息 + 后布局位置信息 + 经历过的布局阶段。

以 Demo 为例，表项 1、2、3 的预布局和后布局位置信息都被记录在`ViewInfoStore`中，其中表项 1 在预布局和后布局中均出现了，所以标志位中包含了`FLAG_PRE | FLAG_POST`，`InfoRecord`中用一个新的常量表示了这种状态`FLAG_PRE_AND_POST`：

```
class ViewInfoStore {
    static class InfoRecord {
        static final int FLAG_PRE = 1 << 2;
        static final int FLAG_POST = 1 << 3;
        static final int FLAG_PRE_AND_POST = FLAG_PRE | FLAG_POST;
    }
}
复制代码

```

而表项 2 只出现在预布局阶段，所以标志位仅包含了`FLAG_PRE`。表项 3 出现在预布局之后及后布局中，所以标志位中包含了`FLAG_APPEAR | FLAG_POST`。

应用动画属性值
-------

```
public class RecyclerView {
    private void dispatchLayoutStep3() {
            // 遍历后布局表项并构建动画信息再存储到 mViewInfoStore
            for (int i = mChildHelper.getChildCount() - 1; i >= 0; i--) {
                ViewHolder holder = getChildViewHolderInt(mChildHelper.getChildAt(i));
                long key = getChangedHolderKey(holder);
                final ItemHolderInfo animationInfo = mItemAnimator.recordPostLayoutInformation(mState, holder);
                ViewHolder oldChangeViewHolder = mViewInfoStore.getFromOldChangeHolders(key);
                 mViewInfoStore.addToPostLayout(holder, animationInfo);
            }
            // 触发表项执行动画
            mViewInfoStore.process(mViewInfoProcessCallback);
            ...
    }
}
复制代码

```

RecyclerView 布局的第三个阶段中，在遍历完后布局表项后，调用了`mViewInfoStore.process(mViewInfoProcessCallback)`来触发表项执行动画：

```
class ViewInfoStore {
    void process(ProcessCallback callback) {
        // 遍历所有参与动画表项的位置信息
        for (int index = mLayoutHolderMap.size() - 1; index >= 0; index--) {
            // 获取表项 ViewHolder
            final RecyclerView.ViewHolder viewHolder = mLayoutHolderMap.keyAt(index);
            // 获取与 ViewHolder 对应的动画信息
            final InfoRecord record = mLayoutHolderMap.removeAt(index);
            // 根据动画信息的标志位确定动画类型以执行对应的 ProcessCallback 回调
            if ((record.flags & FLAG_APPEAR_AND_DISAPPEAR) == FLAG_APPEAR_AND_DISAPPEAR) {
                callback.unused(viewHolder);
            } else if ((record.flags & FLAG_DISAPPEARED) != 0) {
                if (record.preInfo == null) {
                    callback.unused(viewHolder);
                } else {
                    callback.processDisappeared(viewHolder, record.preInfo, record.postInfo);
                }
            } else if ((record.flags & FLAG_APPEAR_PRE_AND_POST) == FLAG_APPEAR_PRE_AND_POST) {
                callback.processAppeared(viewHolder, record.preInfo, record.postInfo);
            } else if ((record.flags & FLAG_PRE_AND_POST) == FLAG_PRE_AND_POST) {
                callback.processPersistent(viewHolder, record.preInfo, record.postInfo);// 保持
            } else if ((record.flags & FLAG_PRE) != 0) {
                callback.processDisappeared(viewHolder, record.preInfo, null); // 消失动画
            } else if ((record.flags & FLAG_POST) != 0) {
                callback.processAppeared(viewHolder, record.preInfo, record.postInfo);// 出现动画
            } else if ((record.flags & FLAG_APPEAR) != 0) {
            }
            // 回收动画信息实例到池中
            InfoRecord.recycle(record);
        }
    }
}
复制代码

```

`ViewInfoStore.process()`中遍历了包含所有表项动画信息的`mLayoutHolderMap`结构，并根据每个表项的标志位来确定执行的动画类型：

*   表项 1 的标志位为`FLAG_PRE_AND_POST`所以会命中`callback.processPersistent()`。
    
*   表项 2 的标志位中只包含`FLAG_PRE`，所以`(record.flags & FLAG_PRE) != 0`成立，`callback.processDisappeared()`会命中。
    
*   表项 3 的标志位中只包含`FLAG_APPEAR | FLAG_POST`，所以`(record.flags & FLAG_APPEAR_PRE_AND_POST) == FLAG_APPEAR_PRE_AND_POST`不成立，而`(record.flags & FLAG_POST) != 0`成立，`callback.processAppeared()`会命中。
    

作为参数传入`ViewInfoStore.process()`的`ProcessCallback`是 RecyclerView 中预定义的动画回调：

```
class ViewInfoStore {
    // 动画回调
    interface ProcessCallback {
        // 消失动画
        void processDisappeared(RecyclerView.ViewHolder viewHolder, RecyclerView.ItemAnimator.ItemHolderInfo preInfo,RecyclerView.ItemAnimator.ItemHolderInfo postInfo);
        // 出现动画
        void processAppeared(RecyclerView.ViewHolder viewHolder, RecyclerView.ItemAnimator.ItemHolderInfo preInfo,RecyclerView.ItemAnimator.ItemHolderInfo postInfo);
        ...
    }
}

public class RecyclerView {
    // RecyclerView 动画回调默认实现
    private final ViewInfoStore.ProcessCallback mViewInfoProcessCallback =
            new ViewInfoStore.ProcessCallback() {
                @Override
                public void processDisappeared(ViewHolder viewHolder, ItemHolderInfo info, ItemHolderInfo postInfo) {
                    mRecycler.unscrapView(viewHolder);
                    animateDisappearance(viewHolder, info, postInfo);//消失动画
                }
                @Override
                public void processAppeared(ViewHolder viewHolder,ItemHolderInfo preInfo, ItemHolderInfo info) {
                    animateAppearance(viewHolder, preInfo, info);//出现动画
                }
                ...
            };
    // 表项动画执行器
    ItemAnimator mItemAnimator = new DefaultItemAnimator();
    // 出现动画
    void animateAppearance(@NonNull ViewHolder itemHolder,ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
        itemHolder.setIsRecyclable(false);
        if (mItemAnimator.animateAppearance(itemHolder, preLayoutInfo, postLayoutInfo)) {
            postAnimationRunner();
        }
    }
    // 消失动画
    void animateDisappearance(@NonNull ViewHolder holder,ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
        addAnimatingView(holder);
        holder.setIsRecyclable(false);
        if (mItemAnimator.animateDisappearance(holder, preLayoutInfo, postLayoutInfo)) {
            postAnimationRunner();
        }
    }
}
复制代码

```

RecyclerView 执行表项动画的代码结构如下：

```
if (mItemAnimator.animateXXX(holder, preLayoutInfo, postLayoutInfo)) {
    postAnimationRunner();
}
复制代码

```

根据`ItemAnimator.animateXXX()`的返回值来决定是否要在下一帧执行动画，以 Demo 中表项 3 的出现动画为例：

```
public abstract class SimpleItemAnimator extends RecyclerView.ItemAnimator {
    @Override
    public boolean animateAppearance(RecyclerView.ViewHolder viewHolder,ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
        // 如果预布局和后布局中表项左上角的坐标有变化 则执行位移动画
        if (preLayoutInfo != null 
            && (preLayoutInfo.left != postLayoutInfo.left || preLayoutInfo.top != postLayoutInfo.top)) {
            // 执行位移动画，并传入动画起点坐标（预布局表项左上角坐标）和终点坐标（后布局表项左上角坐标）
            return animateMove(viewHolder, 
                    preLayoutInfo.left, 
                    preLayoutInfo.top,
                    postLayoutInfo.left, 
                    postLayoutInfo.top);
        } else {
            return animateAdd(viewHolder);
        }
    }
}
复制代码

```

之前存储的表项位置信息，终于在这里被用上了，它作为参数传入`animateMove()`，这是一个定义在`SimpleItemAnimator`中的抽象方法，`DefaultItemAnimator`实现了它：

```
public class DefaultItemAnimator extends SimpleItemAnimator {
    @Override
    public boolean animateMove(final RecyclerView.ViewHolder holder, int fromX, int fromY,
            int toX, int toY) {
        final View view = holder.itemView;
        fromX += (int) holder.itemView.getTranslationX();
        fromY += (int) holder.itemView.getTranslationY();
        resetAnimation(holder);
        int deltaX = toX - fromX;
        int deltaY = toY - fromY;
        if (deltaX == 0 && deltaY == 0) {
            dispatchMoveFinished(holder);
            return false;
        }
        // 表项水平位移
        if (deltaX != 0) {
            view.setTranslationX(-deltaX);
        }
        // 表项垂直位移
        if (deltaY != 0) {
            view.setTranslationY(-deltaY);
        }
        // 将待移动的表项动画包装成 MoveInfo 并存入 mPendingMoves 列表
        mPendingMoves.add(new MoveInfo(holder, fromX, fromY, toX, toY));
        // 表示在下一帧执行动画
        return true;
    }
}
复制代码

```

如果水平或垂直方向的位移增量不为 0，则将待移动的表项动画包装成`MoveInfo`并存入`mPendingMoves`列表，然后返回 true，表示在下一帧执行动画：

```
public class RecyclerView {  
    // 出现动画
    void animateAppearance(ViewHolder itemHolder,ItemHolderInfo preLayoutInfo, ItemHolderInfo postLayoutInfo) {
        itemHolder.setIsRecyclable(false);
        if (mItemAnimator.animateAppearance(itemHolder, preLayoutInfo, postLayoutInfo)) {
            postAnimationRunner();// 触发动画执行
        }
    }
    
    // 将动画执行代码抛到 Choreographer 中的动画队列中
    void postAnimationRunner() {
        if (!mPostedAnimatorRunner && mIsAttached) {
            ViewCompat.postOnAnimation(this, mItemAnimatorRunner);
            mPostedAnimatorRunner = true;
        }
    }
    // 动画执行代码
    private Runnable mItemAnimatorRunner = new Runnable() {
        @Override
        public void run() {
            if (mItemAnimator != null) {
                // 在下一帧执行动画
                mItemAnimator.runPendingAnimations();
            }
            mPostedAnimatorRunner = false;
        }
    };
}
复制代码

```

通过将一个`Runnable`抛到`Choreographer`的动画队列中来触发动画执行，当下一个垂直同步信号到来时，`Choreographer`会从动画队列中获取待执行的`Runnable`实例，并将其抛到主线程执行（关于`Choreographer`的详细解析可以点击[读源码长知识 | Android 卡顿真的是因为” 掉帧 “？](https://juejin.cn/post/6864365886837686285 "https://juejin.cn/post/6864365886837686285")）。执行的内容定义在`ItemAnimator.runPendingAnimations()`中：

```
public class DefaultItemAnimator extends SimpleItemAnimator {
    @Override
    public void runPendingAnimations() {
        // 如果位移动画列表不空，则表示有待执行的位移动画
        boolean movesPending = !mPendingMoves.isEmpty();
        // 是否有待执行的删除动画
        boolean removalsPending = !mPendingRemovals.isEmpty();
        ...
        // 处理位移动画
        if (movesPending) {
            final ArrayList<MoveInfo> moves = new ArrayList<>();
            moves.addAll(mPendingMoves);
            mMovesList.add(moves);
            mPendingMoves.clear();
            Runnable mover = new Runnable() {
                @Override
                public void run() {
                    for (MoveInfo moveInfo : moves) {
                        // 位移动画具体实现
                        animateMoveImpl(moveInfo.holder, moveInfo.fromX, moveInfo.fromY,
                                moveInfo.toX, moveInfo.toY);
                    }
                    moves.clear();
                    mMovesList.remove(moves);
                }
            };
            // 若存在删除动画，则延迟执行位移动画，否则立刻执行
            if (removalsPending) {
                View view = moves.get(0).holder.itemView;
                ViewCompat.postOnAnimationDelayed(view, mover, getRemoveDuration());
            } else {
                mover.run();
            }
        }
        ...
    }
}
复制代码

```

遍历`mPendingMoves`列表，为每一个待执行的位移动画调用`animateMoveImpl()`构建动画：

```
public class DefaultItemAnimator extends SimpleItemAnimator {
    void animateMoveImpl(final RecyclerView.ViewHolder holder, int fromX, int fromY, int toX, int toY) {
        final View view = holder.itemView;
        final int deltaX = toX - fromX;
        final int deltaY = toY - fromY;
        if (deltaX != 0) {
            view.animate().translationX(0);
        }
        if (deltaY != 0) {
            view.animate().translationY(0);
        }

        // 获取动画实例
        final ViewPropertyAnimator animation = view.animate();
        mMoveAnimations.add(holder);
        // 设置动画参数并启动
        animation.setDuration(getMoveDuration()).setListener(new AnimatorListenerAdapter() {
            @Override
            public void onAnimationStart(Animator animator) {
                dispatchMoveStarting(holder);
            }

            @Override
            public void onAnimationCancel(Animator animator) {
                if (deltaX != 0) {
                    view.setTranslationX(0);
                }
                if (deltaY != 0) {
                    view.setTranslationY(0);
                }
            }

            @Override
            public void onAnimationEnd(Animator animator) {
                animation.setListener(null);
                dispatchMoveFinished(holder);
                mMoveAnimations.remove(holder);
                dispatchFinishedWhenDone();
            }
        }).start();
    }
}
复制代码

```

原来默认的表项动画是通过`ViewPropertyAnimator`实现的。

总结
--

1.  RecyclerView 将表项动画数据封装了两层，依次是`ItemHolderInfo`和`InfoRecord`，它们记录了列表预布局和后布局表项的位置信息，即表项矩形区域与列表左上角的相对位置，它还用一个`int`类型的标志位来记录表项经历了哪些布局阶段，以判断表项应该做的动画类型（出现，消失，保持）。
2.  `InfoRecord`被集中存放在一个商店类`ViewInfoStore`中。所有参与动画的表项的`ViewHolder`与`InfoRecord`都会以键值对的形式存储其中。
3.  RecyclerView 在布局的第三阶段会遍历商店类中所有的键值对，以`InfoRecord`中的标志位为依据，判断执行哪种动画。表项预布局和后布局的位置信息会一并传递给`RecyclerView.ItemAnimator`，以触发动画。
4.  `RecyclerView.ItemAnimator`收到动画指令和数据后，又将他们封装为`MoveInfo`，不同类型的动画被存储在不同的`MoveInfo`列表中。然后将执行动画的逻辑抛到 Choreographer 的动画队列中，当下一个垂直同步信号到来时，Choreographer 从动画队列中取出并执行表项动画，执行动画即遍历所有的`MoveInfo`列表，为每一个`MoveInfo`构建 ViewPropertyAnimator 实例并启动动画。

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
