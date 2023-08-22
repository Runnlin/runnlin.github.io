# 

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903778307538958)

1.  [RecyclerView 缓存机制 | 如何复用表项？](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")
    
2.  [RecyclerView 缓存机制 | 回收些什么？](https://juejin.cn/post/6844903778303361038 "https://juejin.cn/post/6844903778303361038")
    
3.  [RecyclerView 缓存机制 | 回收到哪去？](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")
    
4.  [RecyclerView 缓存机制 | scrap view 的生命周期](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")
    
5.  [RecyclerView 动画原理 | pre-layout，post-layout 与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")
    
6.  [RecyclerView 面试题 | 哪些情况下表项会被回收到缓存池？](https://juejin.cn/post/6930412704578404360/ "https://juejin.cn/post/6930412704578404360/")
    

如果想直接看结论可以移步到[第四篇](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")末尾（你会后悔的，过程更加精彩）。

回收入口
----

上一篇以列表滑动事件为起点沿着调用链一直往下寻找，验证了 “滑出屏幕的表项” 会被回收。那它们被回收去哪里了？沿着上一篇的调用链继续往下探究：

```
public class LinearLayoutManager {
    ...
    // 回收滚出屏幕的表项
    private void recycleViewsFromStart(RecyclerView.Recycler recycler, int dt) {
        final int limit = dt;
        final int childCount = getChildCount();
            //遍历LinearLayoutManager的孩子找出其中应该被回收的
            for (int i = 0; i < childCount; i++) {
                View child = getChildAt(i);
                //直到表项底部纵坐标大于 limit 隐形线，回收该表项以上的所有表项
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

**limit 隐形线** 是 “列表滚动后，哪些表项被该被回收” 的依据，当列表向下滚动时，所有位于这条线上方的表项都会被回收。关于 limit 隐形线 的详细解释可以点击[这里](https://juejin.cn/post/6903290882095579143 "https://juejin.cn/post/6903290882095579143") 。

`recycleViewsFromStart()`通过遍历找到滑出屏幕的表项，然后调用了`recycleChildren()`回收他们：

```
public class LinearLayoutManager {
    // 回收子表项
    private void recycleChildren(RecyclerView.Recycler recycler, int startIndex, int endIndex) {
        if (startIndex == endIndex) {
            return;
        }
        if (endIndex > startIndex) {
            for (int i = endIndex - 1; i >= startIndex; i--) {
                removeAndRecycleViewAt(i, recycler);
            }
        } else {
            for (int i = startIndex; i > endIndex; i--) {
                removeAndRecycleViewAt(i, recycler);
            }
        }
    }
}
复制代码

```

最终调用了父类`LayoutManager.removeAndRecycleViewAt()`：

```
public abstract static class LayoutManager {
        public void removeAndRecycleViewAt(int index, Recycler recycler) {
            final View view = getChildAt(index);
            removeViewAt(index);
            recycler.recycleView(view);
        }
}
复制代码

```

先从`LayoutManager`中删除表项，然后调用`Recycler.recycleView()`回收表项：

```
public final class Recycler {
        public void recycleView(View view) {
            // 获取表项 ViewHolder
            ViewHolder holder = getChildViewHolderInt(view);
            if (holder.isTmpDetached()) {
                removeDetachedView(view, false);
            }
            if (holder.isScrap()) {
                holder.unScrap();
            } else if (holder.wasReturnedFromScrap()) {
                holder.clearReturnedFromScrapFlag();
            }
            recycleViewHolderInternal(holder);
        }
}
复制代码

```

先通过表项视图拿到了对应`ViewHolder`，然后把其传入`Recycler.recycleViewHolderInternal()`，现在就可以更准地回答上一篇的那个问题 “回收些啥？”：**回收的是滑出屏幕表项对应的`ViewHolder`** 。

```
public final class Recycler {
        ...
        int mViewCacheMax = DEFAULT_CACHE_SIZE;
        static final int DEFAULT_CACHE_SIZE = 2;
        final ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
        ...

        void recycleViewHolderInternal(ViewHolder holder) {
            ...
            if (forceRecycle || holder.isRecyclable()) {
                //先存在mCachedViews里面
                //这里的判断条件决定了复用mViewCacheMax中的ViewHolder时不需要重新绑定数据
                if (mViewCacheMax > 0
                        && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
                        | ViewHolder.FLAG_REMOVED
                        | ViewHolder.FLAG_UPDATE
                        | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
                    // Retire oldest cached view
                    //如果mCachedViews大小超限了，则删掉最老的被缓存的ViewHolder
                    int cachedViewSize = mCachedViews.size();
                    if (cachedViewSize >= mViewCacheMax && cachedViewSize > 0) {
                        recycleCachedViewAt(0);
                        cachedViewSize--;
                    }

                    int targetCacheIndex = cachedViewSize;
                    if (ALLOW_THREAD_GAP_WORK
                            && cachedViewSize > 0
                            && !mPrefetchRegistry.lastPrefetchIncludedPosition(holder.mPosition)) {
                        // when adding the view, skip past most recently prefetched views
                        int cacheIndex = cachedViewSize - 1;
                        while (cacheIndex >= 0) {
                            int cachedPos = mCachedViews.get(cacheIndex).mPosition;
                            if (!mPrefetchRegistry.lastPrefetchIncludedPosition(cachedPos)) {
                                break;
                            }
                            cacheIndex--;
                        }
                        targetCacheIndex = cacheIndex + 1;
                    }
                    //ViewHolder加到缓存中
                    mCachedViews.add(targetCacheIndex, holder);
                    cached = true;
                }
                //若ViewHolder没有入缓存则存入回收池
                if (!cached) {
                    addViewHolderToRecycledViewPool(holder, true);
                    recycled = true;
                }
            } else {
                ...
            }
            ...
}
复制代码

```

ViewHolder 最终的落脚点有两个：

1.  mCachedViews
2.  RecycledViewPool

落脚点通过`cached`这个布尔值，实现互斥，即`ViewHolder`要么存入`mCachedViews`，要么存入`pool`

`mCachedViews`有大小限制，默认只能存 2 个`ViewHolder`，当第三个`ViewHolder`存入时会把第一个移除掉：

```
public final class Recycler {
        // 讲 mCachedViews 中的 ViewHolder 移到 RecycledViewPool 中
        void recycleCachedViewAt(int cachedViewIndex) {
            ViewHolder viewHolder = mCachedViews.get(cachedViewIndex);
            //将ViewHolder加入到回收池
            addViewHolderToRecycledViewPool(viewHolder, true);
            //将ViewHolder从cache中移除
            mCachedViews.remove(cachedViewIndex);
        }
        ...
}
复制代码

```

从`mCachedViews`移除掉的`ViewHolder`会加入到回收池中。 **`mCachedViews`有点像 “回收池预备队列”，即总是先回收到`mCachedViews`，当它放不下的时候，按照先进先出原则将最先进入的`ViewHolder`存入回收池** ：

```
public final class Recycler {
        // 缓存池实例
        RecycledViewPool mRecyclerPool;
        // 将viewHolder存入缓存池
        void addViewHolderToRecycledViewPool(ViewHolder holder, boolean dispatchRecycled) {
            ...
            getRecycledViewPool().putRecycledView(holder);
        }
        // 获取 RecycledViewPool 实例
        RecycledViewPool getRecycledViewPool() {
            if (mRecyclerPool == null) {
                mRecyclerPool = new RecycledViewPool();
            }
            return mRecyclerPool;
        }
}

//缓存池
public static class RecycledViewPool {
        // 但类型 ViewHolder 列表
        static class ScrapData {
            // 最终存储 ViewHolder 实例的列表
            ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
            //每种类型的 ViewHolder 最多存 5 个
            int mMaxScrap = DEFAULT_MAX_SCRAP;
            ...
        }
        //键值对:以 viewType 为键，ScrapData 为值，用以存储不同类型的 ViewHolder 列表
        SparseArray<ScrapData> mScrap = new SparseArray<>();
        //ViewHolder 入池 按 viewType 分类入池，相同的 ViewType 存放在同一个列表中
        public void putRecycledView(ViewHolder scrap) {
            final int viewType = scrap.getItemViewType();
            final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
            //如果超限了，则放弃入池
            if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
                return;
            }
            // 入回收池之前重置 ViewHolder
            scrap.resetInternal();
            // 最终 ViewHolder 入池
            scrapHeap.add(scrap);
        }
}
复制代码

```

`ViewHolder`会按`viewType`分类存入回收池，最终存储在`ScrapData` 的`ArrayList`中，回收池数据结构分析详见 [RecyclerView 缓存机制（咋复用？）](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")。

缓存优先级
-----

还记得 [RecyclerView 缓存机制（咋复用？）](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")中得出的结论吗？这里再引用一下：

> 虽然为了获取`ViewHolder`做了 5 次尝试（共从 6 个地方获取），先排除 3 种特殊情况，即从`mChangedScrap`获取、通过 id 获取、从自定义缓存获取，正常流程中只剩下 3 种获取方式，优先级从高到低依次是：
> 
> 1.  从 mAttachedScrap 获取
> 2.  从 mCachedViews 获取
> 3.  从 mRecyclerPool 获取 这样的缓存优先级意味着，对应的复用性能也是从高到低（复用性能越好意味着所做的昂贵操作越少）
> 4.  最坏情况：重新创建 ViewHodler 并重新绑定数据
> 5.  次好情况：复用 ViewHolder 但重新绑定数据
> 6.  最好情况：复用 ViewHolder 且不重新绑定数据

当时分析了`mAttachedScrap`和`mRecyclerPool`的复用性能，即 **从`mRecyclerPool`中复用的`ViewHolder`需要重新绑定数据，从`mAttachedScrap` 中复用的`ViewHolder`不需要重新创建也不需要重新绑定数据** 。

把存入`mCachedViews`的代码和复用时绑定数据的代码结合起来看一下：

```
void recycleViewHolderInternal(ViewHolder holder) {
    ...
    //满足这个条件才能存入mCachedViews
    if (mViewCacheMax > 0
        && !holder.hasAnyOfTheFlags(ViewHolder.FLAG_INVALID
        | ViewHolder.FLAG_REMOVED
        | ViewHolder.FLAG_UPDATE
        | ViewHolder.FLAG_ADAPTER_POSITION_UNKNOWN)) {
    }
    ...
}

ViewHolder tryGetViewHolderForPositionByDeadline(int position,boolean dryRun, long deadlineNs) {
    ...
    //满足这个条件就需要重新绑定数据
    if (!holder.isBound() || holder.needsUpdate() || holder.isInvalid()){
    }
    ...
复制代码

```

重新绑定数据的三个条件中，`holder.needsUpdate()`和`holder.isInvalid()`都是`false`时才能存入`mCachedViews` ，而`!holder.isBound()`对于`mCachedViews` 中的`ViewHolder`来说必然为`false`，因为只有当调用`ViewHolder.resetInternal()`重置`ViewHolder`后，才会将其设置为未绑定状态，而只有存入回收池时才会重置`ViewHolder`。所以 **从`mCachedViews`中复用的`ViewHolder`不需要重新绑定数据**

总结
--

*   滑出屏幕表项对应的 ViewHolder 会被回收到`mCachedViews`+`mRecyclerPool` 结构中。
*   `mCachedViews`是 ArrayList ，默认存储最多 2 个 ViewHolder ，当它放不下的时候，按照先进先出原则将最先进入的 ViewHolder 存入回收池的方式来腾出空间。`mRecyclerPool` 是 SparseArray ，它会按`viewType`分类存储 ViewHolder ，默认每种类型最多存 5 个。
*   从`mRecyclerPool`中复用的 ViewHolder 需要重新绑定数据
*   从`mCachedViews`中复用的 ViewHolder 不需要重新绑定数据

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
