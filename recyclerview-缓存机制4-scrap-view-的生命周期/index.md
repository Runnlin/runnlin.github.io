# 

> 本文由 [简悦 SimpRead](http://ksria.com/simpread/) 转码， 原文地址 [juejin.cn](https://juejin.cn/post/6844903780006264845)

1.  [RecyclerView 缓存机制 | 如何复用表项？](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")
    
2.  [RecyclerView 缓存机制 | 回收些什么？](https://juejin.cn/post/6844903778303361038 "https://juejin.cn/post/6844903778303361038")
    
3.  [RecyclerView 缓存机制 | 回收到哪去？](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")
    
4.  [RecyclerView 缓存机制 | scrap view 的生命周期](https://juejin.cn/post/6844903780006264845 "https://juejin.cn/post/6844903780006264845")
    
5.  [RecyclerView 动画原理 | pre-layout，post-layout 与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")
    

`scrap view`对应的存储结构是`final ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();`。理解成员变量用途的最好办法是 **“搜索它在什么时候被访问”** 。对于列表结构来说就相当于 1. 在什么时候往列表添加内容？ 2. 在什么时候清空列表内容？

添加内容
----

全局搜索`mAttachedScrap`被访问的地方，其中只有一处调用了`mAttachedScrap.add()`:

```
public final class Recycler {
         // 回收 ViewHolder 到 scrap 集合（mAttachedScrap或mChangedScrap）,
        void scrapView(View view) {
            final ViewHolder holder = getChildViewHolderInt(view);
            if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
                    || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
                holder.setScrapContainer(this, false);
                //添加到 mAttachedScrap 集合中
                mAttachedScrap.add(holder);
            } else {
                if (mChangedScrap == null) {
                    mChangedScrap = new ArrayList<ViewHolder>();
                }
                holder.setScrapContainer(this, true);
                //添加到 mChangedScrap 集合中
                mChangedScrap.add(holder);
            }
        }
}
复制代码

```

沿着调用链继续往上：

```
public abstract static class LayoutManager {
        private void scrapOrRecycleView(Recycler recycler, int index, View view) {
            final ViewHolder viewHolder = getChildViewHolderInt(view);
            // 删除表项并入回收池
            if (viewHolder.isInvalid() && !viewHolder.isRemoved()
                    && !mRecyclerView.mAdapter.hasStableIds()) {
                removeViewAt(index);
                recycler.recycleViewHolderInternal(viewHolder);
            }
            // detach 表项并入 scrap 集合
            else {
                detachViewAt(index);
                recycler.scrapView(view);
                mRecyclerView.mViewInfoStore.onViewDetached(viewHolder);
            }
        }
}
复制代码

```

根据`viewHolder`的不同状态，要么将其添加到`mAttachedScrap`集合，要么将其存入回收池。其中`recycleViewHolderInternal()`在 [RecyclerView 缓存机制（回收去哪？）](https://juejin.cn/post/6844903778307538958 "https://juejin.cn/post/6844903778307538958")分析过。 沿着调用链继续向上：

```
public abstract static class LayoutManager {
         // 暂时将当可见表项进行分离并回收
        public void detachAndScrapAttachedViews(Recycler recycler) {
            final int childCount = getChildCount();
            // 遍历所有可见表项并回收他们
            for (int i = childCount - 1; i >= 0; i--) {
                final View v = getChildAt(i);
                scrapOrRecycleView(recycler, i, v);
            }
        }

        // 布局所有子表项
        public void onLayoutChildren(Recycler recycler, State state) {
            ...
            // 在填充表项之前回收所有表项
            detachAndScrapAttachedViews(recycler);
            ...
            // 填充表项
            fill(recycler, mLayoutState, state, false);
            ...
        }
}

public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild2 {
    // RecyclerView布局的第二步
    private void dispatchLayoutStep2() {
        ...
        mLayout.onLayoutChildren(mRecycler, mState);
        ...
    }
}
复制代码

```

*   在将表项一个个填充到列表之前会先将其先回收到`mAttachedScrap`中，回收数据的来源是`LayoutManager`的孩子，而`LayoutManager`的孩子都是屏幕上可见的或即将可见的表项。
*   注释中 “暂时将当当前可见表项进行分离并回收”，既然是 “暂时回收”，那待会必然会发生 “复用”。复用逻辑可移步 [RecyclerView 缓存机制（咋复用？）](https://juejin.cn/post/6844903778303344647 "https://juejin.cn/post/6844903778303344647")
*   至此可以得出结论：**`mAttachedScrap`用于屏幕中可见表项的回收和复用**

清空内容
----

全局搜索`mAttachedScrap`被访问的地方，其中只有一处调用了`mAttachedScrap.clear()`:

```
public class RecyclerView {
    public final class Recycler {
        // 清空 scrap 结构
        void clearScrap() {
            mAttachedScrap.clear();
            if (mChangedScrap != null) {
                mChangedScrap.clear();
            }
        }
    }
}
复制代码

```

`Recycler.clearScrap()`清空了 scrap 列表。而它会在`LayoutManager.removeAndRecycleScrapInt()`中被调用：

```
public abstract static class LayoutManager {
         // 回收所有 scrapped view
        void removeAndRecycleScrapInt(Recycler recycler) {
            final int scrapCount = recycler.getScrapCount();
            // Loop backward, recycler might be changed by removeDetachedView()
            // 遍历搜有 scrap view 重置 ViewHolder 状态，并将其回收到缓存池
            for (int i = scrapCount - 1; i >= 0; i--) {
                final View scrap = recycler.getScrapViewAt(i);
                final ViewHolder vh = getChildViewHolderInt(scrap);
                if (vh.shouldIgnore()) {
                    continue;
                }
                vh.setIsRecyclable(false);
                if (vh.isTmpDetached()) {
                    mRecyclerView.removeDetachedView(scrap, false);
                }
                if (mRecyclerView.mItemAnimator != null) {
                    mRecyclerView.mItemAnimator.endAnimation(vh);
                }
                vh.setIsRecyclable(true);
                recycler.quickRecycleScrapView(scrap);
            }
            // 清空 scrap view 集合
            recycler.clearScrap();
            if (scrapCount > 0) {
                mRecyclerView.invalidate();
            }
        }
}
复制代码

```

沿着调用链向上：

```
public class RecyclerView extends ViewGroup implements ScrollingView, NestedScrollingChild2 {
    // RecyclerView布局的最后一步
    private void dispatchLayoutStep3() {
        ...
        mLayout.removeAndRecycleScrapInt(mRecycler);
        ...
}
复制代码

```

至此可以得出结论：**`mAttachedScrap`生命周期起始于`RecyclerView`布局开始，终止于`RecyclerView`布局结束。**

分析完了 scrap 结构的生命周期和作用后，不免产生新的疑问：什么场景下需要回收并复用屏幕中可见的表项？限于篇幅原因，在[读原码长知识 | RecyclerView 预布局 ，后布局与 scrap 缓存的关系](https://juejin.cn/post/6892809944702124045 "https://juejin.cn/post/6892809944702124045")中做了详细分析。

总结
--

经过四篇文章的分析，`RecyclerVeiw`的四级缓存都分析完了，总结如下：

1.  **`Recycler`有 4 个层次用于缓存`ViewHolder`对象，优先级从高到底依次为`ArrayList<ViewHolder> mAttachedScrap`、`ArrayList<ViewHolder> mCachedViews`、`ViewCacheExtension mViewCacheExtension`、`RecycledViewPool mRecyclerPool`。如果四层缓存都未命中，则重新创建并绑定`ViewHolder`对象**
    
2.  **缓存性能：**
    
    <table><thead><tr><th>缓存</th><th>重新创建<code>ViewHolder</code></th><th>重新绑定数据</th></tr></thead><tbody><tr><td>mAttachedScrap</td><td>false</td><td>false</td></tr><tr><td>mCachedViews</td><td>false</td><td>false</td></tr><tr><td>mRecyclerPool</td><td>false</td><td>true</td></tr></tbody></table>
3.  **缓存容量：**
    
    *   `mAttachedScrap`：没有大小限制，但最多包含屏幕可见表项。
    *   `mCachedViews`：默认大小限制为 2，放不下时，按照先进先出原则将最先进入的`ViewHolder`存入回收池以腾出空间。
    *   `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储（通过`SparseArray`），同类`ViewHolder`存储在默认大小为 5 的`ArrayList`中。
4.  **缓存用途：**
    
    *   `mAttachedScrap`：用于布局过程中屏幕可见表项的回收和复用。
    *   `mCachedViews`：用于移出屏幕表项的回收和复用，且只能用于指定位置的表项，有点像 “回收池预备队列”，即总是先回收到`mCachedViews`，当它放不下的时候，按照先进先出原则将最先进入的`ViewHolder`存入回收池。
    *   `mRecyclerPool`：用于移出屏幕表项的回收和复用，且只能用于指定`viewType`的表项
5.  **缓存结构：**
    
    *   `mAttachedScrap`：`ArrayList<ViewHolder>`
    *   `mCachedViews`：`ArrayList<ViewHolder>`
    *   `mRecyclerPool`：对`ViewHolder`按`viewType`分类存储在`SparseArray<ScrapData>`中，同类`ViewHolder`存储在`ScrapData`中的`ArrayList`中

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
