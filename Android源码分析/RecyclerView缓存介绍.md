>转载请以链接形式标明出处： 
本文出自:[**103style的博客**](http://blog.csdn.net/lxk_1993) 

源码 `base on androidx.recyclerview:recyclerview:1.1.0-alpha05`

代码说明中已省略了暂不需要关注的代码.

建议先看下最后的小结。

参考 [让你彻底掌握RecyclerView的缓存机制](https://www.jianshu.com/p/3e9aa4bdaefd) 的测试代码待添加...


---

### 目录
* 获取 `RecyclerView` 子项的流程
* 缓存相关的数据变量的介绍
* 获取缓存的逻辑介绍
* 小结

---

### 获取 RecyclerView 子项的流程
获取的流程大致为： `RecyclerView.onLayout(...)` →  `LayoutManager.onLayoutChildren(...)` → `LayoutState.next(...)`。

源代码调用如下：

`RecyclerView` 的 `onLayout(...)`方法：
```
protected void onLayout(boolean changed, int l, int t, int r, int b) {
    dispatchLayout();
}
void dispatchLayout() {
        dispatchLayoutStep2();
}
private void dispatchLayoutStep2() {
    mLayout.onLayoutChildren(mRecycler, mState);
}
```

`RecyclerView` 设置的 `LayoutManager` 的 `onLayoutChildren(...)`方法：
以 `LinearLayoutManager` 为例：
```
public void onLayoutChildren(...) {
        fill(recycler, mLayoutState, state, false);
}
int fill(...) {
        layoutChunk(recycler, state, layoutState, layoutChunkResult);
}
void layoutChunk(...) {
    View view = layoutState.next(recycler);
}
List<RecyclerView.ViewHolder> mScrapList;
static class LayoutState {
    View next(RecyclerView.Recycler recycler) {
        if (mScrapList != null) {
            return nextViewFromScrapList();
        }
        final View view = recycler.getViewForPosition(mCurrentPosition);
        mCurrentPosition += mItemDirection;
        return view;
    }
}
```
这里主要看 `recycler.getViewForPosition(mCurrentPosition)`，`mScrapList` 是有动画效果时预布局保存的`ViewHolder`。

---

### 缓存相关的数据变量的介绍
**Recycler中的 mAttachedScrap、mChangedScrap、mCachedViews**
```
ArrayList<ViewHolder> mAttachedScrap = new ArrayList<>();
ArrayList<ViewHolder> mChangedScrap = null;
ArrayList<ViewHolder> mCachedViews = new ArrayList<ViewHolder>();
static final int DEFAULT_CACHE_SIZE = 2;
```
**mAttachedScrap**：用于缓存显示在屏幕上并符合相应规则的 `ViewHolder`。
**mChangedScrap**：用于缓存显示在屏幕上不符合 `mAttachedScrap` 规则的 `ViewHolder`。
`Recycler.scrapView(View view)`， 在 调用 `attchView` 和 `detachView`的时候调用。
```
void scrapView(View view) {
    final ViewHolder holder = getChildViewHolderInt(view);
    if (holder.hasAnyOfTheFlags(ViewHolder.FLAG_REMOVED | ViewHolder.FLAG_INVALID)
            || !holder.isUpdated() || canReuseUpdatedViewHolder(holder)) {
        mAttachedScrap.add(holder);
    } else {
        if (mChangedScrap == null) {
            mChangedScrap = new ArrayList<ViewHolder>();
        }
        mChangedScrap.add(holder);
    }
}
```

**mCachedViews**：保存滑动过程中从可见到不可见的`ViewHolder`，大小为`DEFAULT_CACHE_SIZE = 2 `，并且原本数据信息都在，所以可以直接添加到 `RecyclerView` 中显示，不需要再次重新 `onBindViewHolder()`。
```
private final ViewInfoStore.ProcessCallback mViewInfoProcessCallback =
        new ViewInfoStore.ProcessCallback() {
            @Override
            public void unused(ViewHolder viewHolder) {
                mLayout.removeAndRecycleView(viewHolder.itemView, mRecycler);
            }
        };
void process(ProcessCallback callback) {
    for (int index = mLayoutHolderMap.size() - 1; index >= 0; index--) {
        final RecyclerView.ViewHolder viewHolder = mLayoutHolderMap.keyAt(index);
        final InfoRecord record = mLayoutHolderMap.removeAt(index);
        if ((record.flags & FLAG_APPEAR_AND_DISAPPEAR) == FLAG_APPEAR_AND_DISAPPEAR) {
            //从可见变成不可见
            callback.unused(viewHolder);
        } else if ((record.flags & FLAG_DISAPPEARED) != 0) {
            if (record.preInfo == null) {
                // 类似出现消失但发生在不同的布局通道之间。 
                // 当布局管理器使用自动测量时，可能会发生这种情况
                callback.unused(viewHolder);
            }
        }
    }
}
```

---

**ChildHelper.mHiddenViews**

`mHiddenViews` 中保留 **仅出于动画目的子视图**。
```
List<View> mHiddenViews = new ArrayList<View>();
```
`mHiddenViews` 是在 `RecycleView` 中 `addAnimatingView(ViewHolder viewHolder)`的时候，根据特定情况添加的。
```
private void addAnimatingView(ViewHolder viewHolder) {
    final boolean alreadyParented = view.getParent() == this;
    if (viewHolder.isTmpDetached()) {
        mChildHelper.attachViewToParent(view, -1, view.getLayoutParams(), true);
    } else if (!alreadyParented) {
        mChildHelper.addView(view, true);
    } else {
        mChildHelper.hide(view);
    }
}
```

---

**RecycledViewPool.mScrap**

每个 `viewType` 的缓存最多是 `DEFAULT_MAX_SCRAP = 5` 个，`viewType`即 `RecyclerView` 中 **子view** 的样式，即 `adapter` 中的 `getItemViewType`获取的值。
```
private static final int DEFAULT_MAX_SCRAP = 5;
static class ScrapData {
    final ArrayList<ViewHolder> mScrapHeap = new ArrayList<>();
    int mMaxScrap = DEFAULT_MAX_SCRAP;
}
SparseArray<ScrapData> mScrap = new SparseArray<>();
```
添加和获取的方法： 
* 我们可以看到每获取一个值，则会从缓存中删掉。取数据即从缓存中取数据时调用。
* 添加的时候，当数量超过`DEFAULT_MAX_SCRAP = 5`时，则丢弃。
  存数据即为 释放 `mCachedViews` 的数据则会放入缓存池。
```
public ViewHolder getRecycledView(int viewType) {
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        for (int i = scrapHeap.size() - 1; i >= 0; i--) {
            if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                return scrapHeap.remove(i);
            }
        }
    }
    return null;
}
public void putRecycledView(ViewHolder scrap) {
    final int viewType = scrap.getItemViewType();
    final ArrayList<ViewHolder> scrapHeap = getScrapDataForType(viewType).mScrapHeap;
    if (mScrap.get(viewType).mMaxScrap <= scrapHeap.size()) {
        return;
    }
    scrapHeap.add(scrap);
}
```

---

###  获取缓存的逻辑介绍

`RecyclerView.Recycler`的 `getViewForPosition(int position)`方法：
```
public View getViewForPosition(int position) {
    return getViewForPosition(position, false);
}
View getViewForPosition(int position, boolean dryRun) {
    return tryGetViewHolderForPositionByDeadline(position, dryRun, FOREVER_NS).itemView;
}
```

---

`RecyclerView.Recycler`的`tryGetViewHolderForPositionByDeadline(...)` 方法获取 `holder`。
**也就是对应的缓存机制的主要逻辑，找到即返回**。
* 首先判断`mState.isPreLayout()`，是不是执行动画的预布局，是的话则尝试从 `mChangedScrap` 中获取。
* 然后尝试从`mAttachedScrap`、`mHiddenViews`、`mCachedViews`获取。
* 然后根据 `mAdapter.hasStableIds()` 以及 `mViewCacheExtension!=null` 判断是否从用户设置的相关逻辑中获取。
* 然后再尝试从缓存池 `RecycledViewPool` 中的 `mScrap` 的 `ScrapData` 中获取。
* 还没找到则通过 `mAdapter.createViewHolder` 重新创建。
```
ViewHolder tryGetViewHolderForPositionByDeadline(...) {
    ...
    if (mState.isPreLayout()) {
        //如果是预布局，即执行简单的动画 就会尝试这里
        holder = getChangedScrapViewForPosition(position);
    }
    // 从 scrap/hidden list/cache 尝试获取
    if (holder == null) {
        holder = getScrapOrHiddenOrCachedHolderForPosition(position, dryRun);
    }
    if (holder == null) {
        final int offsetPosition = mAdapterHelper.findPositionOffset(position);
        ...
        final int type = mAdapter.getItemViewType(offsetPosition);
        //通过 adapter.setHasStableIds设置 默认为false
        if (mAdapter.hasStableIds()) {}
        //mViewCacheExtension 通过 RecyclerView.setViewCacheExtension设置 默认为null
        if (holder == null && mViewCacheExtension != null) {
            //通过自定义的缓存机制区获取
        }
        if (holder == null) {
            // 尝试从RecycledViewPool中去获取holder
            holder = getRecycledViewPool().getRecycledView(type);
        }
        if (holder == null) {
            //调用Adapter.createViewHolder创建holder
            holder = mAdapter.createViewHolder(RecyclerView.this, type);
        }
    }
    ...
    return holder;
}
```

---

**如果是预布局，则尝试从 mChangedScrap 中获取**：
```
ViewHolder getChangedScrapViewForPosition(int position) {
    // 如果是预布局，请检查 mChangedScrap 是否完全匹配。
    for (int i = 0; i < mChangedScrap.size(); i++) {
        final ViewHolder holder = mChangedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    // 默认 false
    if (mAdapter.hasStableIds()) {
        ...
    }
    return null;
}
```

---

**尝试从缓存中 mAttachedScrap、mHiddenViews、mCachedViews获取**：
`RecyclerView.Recycler`的`getScrapOrHiddenOrCachedHolderForPosition(...)` 获取 `holder`.
```
ViewHolder getScrapOrHiddenOrCachedHolderForPosition(int position, boolean dryRun) {
    final int scrapCount = mAttachedScrap.size();

    // 尝试从mAttachedScrap获取
    for (int i = 0; i < scrapCount; i++) {
        final ViewHolder holder = mAttachedScrap.get(i);
        if (!holder.wasReturnedFromScrap() && holder.getLayoutPosition() == position
                && !holder.isInvalid() && (mState.mInPreLayout || !holder.isRemoved())) {
            holder.addFlags(ViewHolder.FLAG_RETURNED_FROM_SCRAP);
            return holder;
        }
    }
    //dryRun: getViewForPosition(position, false)传入的是false
    if (!dryRun) {
        View view = mChildHelper.findHiddenNonRemovedView(position);
        if (view != null) {
        ...
            return vh;
        }
    }
    // 从一级缓存中尝试获取
    final int cacheSize = mCachedViews.size();
    for (int i = 0; i < cacheSize; i++) {
        final ViewHolder holder = mCachedViews.get(i);
        // 如果适配器具有稳定的ID，则无效的视图持有者可能在缓存中，
        // 因为可以通过getScrapOrCachedViewForId检索它们
        if (!holder.isInvalid() && holder.getLayoutPosition() == position
                && !holder.isAttachedToTransitionOverlay()) {
            if (!dryRun) {
                mCachedViews.remove(i);
            }
            return holder;
        }
    }
    return null;
}
```

---

**从缓存池RecycledViewPool中的 mScrap 的 ScrapData 中获取ViewHolder，并从缓存池中移除**：
`RecycledViewPool`的 `getRecycledView(int viewType)`:
```
SparseArray<ScrapData> mScrap = new SparseArray<>();

public ViewHolder getRecycledView(int viewType) {
    final ScrapData scrapData = mScrap.get(viewType);
    if (scrapData != null && !scrapData.mScrapHeap.isEmpty()) {
        final ArrayList<ViewHolder> scrapHeap = scrapData.mScrapHeap;
        for (int i = scrapHeap.size() - 1; i >= 0; i--) {
            if (!scrapHeap.get(i).isAttachedToTransitionOverlay()) {
                return scrapHeap.remove(i);
            }
        }
    }
    return null;
}
```

---

### 小结
通过上文我们可以了解到：

获取 子View 最终是调用了 `LayoutManager` 中 `LayoutState` 的  `next(RecyclerView.Recycler recycler)`。
然后判断是否是显示动画效果，是的话则直接从  `mScrapList` 获取，否则通过缓存机制去获取。

---

缓存数据相关的变量分别代表的意思：
**mAttachedScrap**：用于缓存显示在屏幕上并符合对应规则的 `ViewHolder`。
**mChangedScrap**：用于缓存显示在屏幕上不符合 `mAttachedScrap` 规则的 `ViewHolder`。
**mCachedViews**：保存滑动过程中从可见到不可见的`ViewHolder`，大小为`DEFAULT_CACHE_SIZE = 2 `，并且原本数据信息都在，所以可以直接添加到 `RecyclerView` 中显示，不需要再次重新 `onBindViewHolder()`。
**mHiddenViews** 中保留 **仅出于动画目的子视图**。

---

**RecyclerView** 的缓存机制大致是：
* 首先判断是不是执行动画的预布局，是的话则尝试从 `mChangedScrap` 中获取。
* 然后尝试从`mAttachedScrap`、`mHiddenViews`、`mCachedViews`获取。
* 然后根据 `mAdapter.hasStableIds()` 以及 `mViewCacheExtension!=null` 判断是否从用户自定义设置的相关逻辑中获取。
* 然后再尝试从缓存池 `RecycledViewPool` 中 `mScrap` 的 `ScrapData` 中获取，每种样式最多缓存 **5个**。
* 还没找到则通过 `mAdapter.createViewHolder` 创建新的。


---

如果觉得不错的话，请帮忙点个赞呗。

以上

---

扫描下面的二维码，关注我的公众号 **Android1024**， 点关注，不迷路。
![Android1024](https://upload-images.jianshu.io/upload_images/1709375-84aaffe67e21a7e9.jpg?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
