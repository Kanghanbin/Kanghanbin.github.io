
title: Recycleview的SnapHelper的理解
date: 2018-06-06 17:05:52
tags: 源码

---
 SnapHelper是RecycleView在24.2.0版本中新增的辅助类，用于在RecycleView滚动结束时，将Item对其到某个位置。SnapHelper是个抽象类，继承了RecycleView.OnFlingListener类

## 1.recycleview使用SnapHelper非常简单，只需要调用一行代码：attachToRecyclerView(mRecyclerView)，SnapHelper里方法如下：
```java
 public void attachToRecyclerView(@Nullable RecyclerView recyclerView)

            throws IllegalStateException {

        //如果之前已经附着过该recyclerView，则不作任何处理

        if (mRecyclerView == recyclerView) {

            return; // nothing to do

        }

        //如果不一致，则清理监听

        if (mRecyclerView != null) {

            destroyCallbacks();

        }

        mRecyclerView = recyclerView;

        if (mRecyclerView != null) {

            setupCallbacks();

            mGravityScroller = new Scroller(mRecyclerView.getContext(),

                    new DecelerateInterpolator());

            snapToTargetExistingView();

        }

    }

```


将recycleView赋值给mRecycleview，重新建立监听，创建Scroller，然后调用snapToTargetExistingView方法用于首次对齐。



## 2.snapToTargetExistingView()
```java
void snapToTargetExistingView() {
        if (mRecyclerView == null) {
            return;
        }
        LayoutManager layoutManager = mRecyclerView.getLayoutManager();
        if (layoutManager == null) {
            return;
        }
        View snapView = findSnapView(layoutManager);
        if (snapView == null) {
            return;
        }
        int[] snapDistance = calculateDistanceToFinalSnap(layoutManager, snapView);
        if (snapDistance[0] != 0 || snapDistance[1] != 0) {
            mRecyclerView.smoothScrollBy(snapDistance[0], snapDistance[1]);
        }
    }
```
该方法是通过findSnapView获得snapView，然后通过calcuteDistanceToFinalSnap（）方法计算出snapView当前坐标到目的坐标之间的举例，然后调用smoothScrollBy（）方法滑动。

## 3.setUpCallbacks（） 和 destroyCallbacks（）

```java
    private void setupCallbacks() throws IllegalStateException {
        if (mRecyclerView.getOnFlingListener() != null) {
            throw new IllegalStateException("An instance of OnFlingListener already set.");
        }
        mRecyclerView.addOnScrollListener(mScrollListener);
        mRecyclerView.setOnFlingListener(this);
    }

    /**
     * Called when the instance of a {@link RecyclerView} is detached.
     */
    private void destroyCallbacks() {
        mRecyclerView.removeOnScrollListener(mScrollListener);
        mRecyclerView.setOnFlingListener(null);
    }
```
这两个方法很简单，分别是建立和移除两个回调：

## 4.OnScrollListener和OnFlingListener，OnFlingListener已经实现了，所以直接用，OnScrollListener
```java
  private final RecyclerView.OnScrollListener mScrollListener =
            new RecyclerView.OnScrollListener() {
                boolean mScrolled = false;

                @Override
                public void onScrollStateChanged(RecyclerView recyclerView, int newState) {
                    super.onScrollStateChanged(recyclerView, newState);
                    if (newState == RecyclerView.SCROLL_STATE_IDLE && mScrolled) {
                        mScrolled = false;
                        snapToTargetExistingView();
                    }
                }
    
                @Override
                public void onScrolled(RecyclerView recyclerView, int dx, int dy) {
                    if (dx != 0 || dy != 0) {
                        mScrolled = true;
                    }
                }
            }
```
## 5.在滚动停止的时候，调用snapToTargetExistingView方法将snapView滚到指定位置。SnapHelper实现onFling方法：

```java
 @Override

    public boolean onFling(int velocityX, int velocityY) {

        LayoutManager layoutManager = mRecyclerView.getLayoutManager();

        if (layoutManager == null) {

            return false;

        }

        RecyclerView.Adapter adapter = mRecyclerView.getAdapter();

        if (adapter == null) {

            return false;

        }

        int minFlingVelocity = mRecyclerView.getMinFlingVelocity();

        return (Math.abs(velocityY) > minFlingVelocity || Math.abs(velocityX) > minFlingVelocity)

                && snapFromFling(layoutManager, velocityX, velocityY);

    }

```



## 6.获取最小滑动速率，判断超过速率，执行snapFromFling方法：
```java
 private boolean snapFromFling(@NonNull LayoutManager layoutManager, int velocityX, int velocityY) {
        if (!(layoutManager instanceof ScrollVectorProvider)) {
            return false;
        }

        RecyclerView.SmoothScroller smoothScroller = createSnapScroller(layoutManager);
        if (smoothScroller == null) {
            return false;
        }
    
        int targetPosition = findTargetSnapPosition(layoutManager, velocityX, velocityY);
        if (targetPosition == RecyclerView.NO_POSITION) {
            return false;
        }
    
        smoothScroller.setTargetPosition(targetPosition);
        layoutManager.startSmoothScroll(smoothScroller);
        return true;
}
```
创建滚动器SmoothScroller ，通过findTargetSnapPosition计算速率targetPosition并设置给smoothScroller，具体创建SmoothScroller如下：
```java
    @Nullable
    protected LinearSmoothScroller createSnapScroller(LayoutManager layoutManager) {
        if (!(layoutManager instanceof ScrollVectorProvider)) {
            return null;
        }
        return new LinearSmoothScroller(mRecyclerView.getContext()) {
            @Override
            protected void onTargetFound(View targetView, RecyclerView.State state, Action action) {
                int[] snapDistances = calculateDistanceToFinalSnap(mRecyclerView.getLayoutManager(),
                        targetView);
                final int dx = snapDistances[0];
                final int dy = snapDistances[1];
                final int time = calculateTimeForDeceleration(Math.max(Math.abs(dx), Math.abs(dy)));
                if (time > 0) {
                    action.update(dx, dy, time, mDecelerateInterpolator);
                }
            }
            @Override
            protected float calculateSpeedPerPixel(DisplayMetrics displayMetrics) {
                return MILLISECONDS_PER_INCH / displayMetrics.densityDpi;
            }
        };
    }
```

创建了一个LinearSmoothScroller滚动器，onTargetFound方法具体实现计算当前坐标与目的坐标之间距离，计算做减速时所需要的时间，然后调用update方法根据滚动事件，距离和插值器更新SmoothScroller状态。





## 最后重点方法：

### 1.findTargetSnapPosition

```java
public abstract int findTargetSnapPosition(LayoutManager layoutManager, int velocityX,
        int velocityY);
```
该方法会根据触发Fling操作的速率（参数velocityX和参数velocityY）来找到RecyclerView需要滚动到哪个位置，该位置对应的ItemView就是那个需要进行对齐的列表项。我们把这个位置称为targetSnapPosition，对应的View称为targetSnapView。如果找不到targetSnapPosition，就返回RecyclerView.NO_POSITION。

### 2.findSnapView

```java
public abstract View findSnapView(LayoutManager layoutManager);
```
该方法会找到当前layoutManager上最接近对齐位置的那个view，该view称为SanpView，对应的position称为SnapPosition。如果返回null，就表示没有需要对齐的View，也就不会做滚动对齐调整。

### 3.calculateDistanceToFinalSnap

```java
public abstract int[] calculateDistanceToFinalSnap(@NonNull LayoutManager layoutManager,
        @NonNull View targetView);
```
这个方法会计算第二个参数对应的ItemView当前的坐标与需要对齐的坐标之间的距离。该方法返回一个大小为2的int数组，分别对应x轴和y轴方向上的距离。




## 补充：

SnapHelper是个抽象类，可以自己实现一个自定义SnapHelper类，安卓也提供了该类的两个继承类。LininearSnapHelper和PagerSnapHelper。这两个类都可以直接拿来用。LininearSnapHelper实现了recycleview滑动后itemview居中对齐，类似一个支持快速滑动viewpager；PagerSnapHelper一次只能滑动一项，不支持快速滑动的viewpager。

