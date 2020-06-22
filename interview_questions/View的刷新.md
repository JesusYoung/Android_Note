# View的刷新

## 1、View用invalidate和requestLayout方法的区别





## 2、屏幕中有ViewGroup1，里面嵌套View1，ViewGroup2，里面嵌套View2，当调用View1的requestLayout()方法时，ViewGroup2和View2会重新测量吗？

当调用View1的requestLayout()方法时，会将该View的mPrivateFlags赋值PFLAG_FORCE_LAYOUT和PFLAG_INVALIDATED两个标志位，然后调用该View的parent的requestLayout()方法，一直到最顶层View，即ViewRootImpl；

```java
public void requestLayout() {
  ...
	mPrivateFlags |= PFLAG_FORCE_LAYOUT;
  mPrivateFlags |= PFLAG_INVALIDATED;

	if (mParent != null && !mParent.isLayoutRequested()) {
    mParent.requestLayout();
  }
  ...
}
```

在ViewRootImpl类的requestLayout()方法中，会执行scheduleTraversals()方法去触发View的重新测量；

```java
public void requestLayout() {
  if (!mHandlingLayoutInLayoutRequest) {
    checkThread();
    mLayoutRequested = true;
    scheduleTraversals();
  }
}
```

在scheduleTraversals()方法中，会通过Choreographer对象发送一个同步屏障消息（关于Choreographer流程详见“”），将一个任务mTraversalRunnable发送到主线程的Handler，此时会监听屏幕下一帧信号，当屏幕下一帧信号到来时，会执行该Runnable任务，即mTraversalRunnable；

```java
void scheduleTraversals() {
  if (!mTraversalScheduled) {
    mTraversalScheduled = true;
    mTraversalBarrier = mHandler.getLooper().getQueue().postSyncBarrier();
    mChoreographer.postCallback(Choreographer.CALLBACK_TRAVERSAL, mTraversalRunnable, null);
    ...
  }
}

final class TraversalRunnable implements Runnable {
  @Override
  public void run() {
    doTraversal();
  }
}
final TraversalRunnable mTraversalRunnable = new TraversalRunnable();
```

在该Runnable任务中只是执行了doTraversal()方法，在该方法中又执行了performTraversals()方法，在performTraversals()方法中会调用performMeasure()方法，该方法主要用来调度View的measure()方法去执行测量；

```java
void doTraversal() {
  ..
  performTraversals();
  ...
}

private void performTraversals() {
	...
  performMeasure(childWidthMeasureSpec, childHeightMeasureSpec);
  ...
}

private void performMeasure(int childWidthMeasureSpec, int childHeightMeasureSpec) {
  ...
  try {
    mView.measure(childWidthMeasureSpec, childHeightMeasureSpec);
  } finally {
    Trace.traceEnd(Trace.TRACE_TAG_VIEW);
  }
}
```

而在View的measure()方法中，或先获取forceLayout变量的值，该值是判断mPrivateFlags是否被赋值PFLAG_FORCE_LAYOUT，在View的requestLayout()方法中会赋值该变量值，所以此处获取到的forceLayout的值为true，会去执行该View的onMeasure()方法，如果未被赋值，则不会执行，所以只有View1一直往上的视图会被赋值，会触发重新测量，而ViewGroup2和View2不会触发重新测量；

```java
public final void measure(int widthMeasureSpec, int heightMeasureSpec) {
  ...
  final boolean forceLayout = (mPrivateFlags & PFLAG_FORCE_LAYOUT) == PFLAG_FORCE_LAYOUT;
  if (forceLayout || needsLayout) {
    mPrivateFlags &= ~PFLAG_MEASURED_DIMENSION_SET;
  	...
    onMeasure(widthMeasureSpec, heightMeasureSpec);
    ...
    mPrivateFlags |= PFLAG_LAYOUT_REQUIRED;
  }
  ...
}
```

在measure()方法最后又执行mPrivateFlags |= PFLAG_LAYOUT_REQUIRED，为View添加一个PFLAG_LAYOUT_REQUIRED值，该值会在View的layout()方法中用到，判断该View是否需要重新布局，如果未被赋值则不会执行View的onLayout()方法，如果赋值了会执行，并且执行完后会将该标志位清除；

```java
public void layout(int l, int t, int r, int b) {
	...
  if (changed || (mPrivateFlags & PFLAG_LAYOUT_REQUIRED) == PFLAG_LAYOUT_REQUIRED) {
    onLayout(changed, l, t, r, b);
    ...
    mPrivateFlags &= ~PFLAG_LAYOUT_REQUIRED;
    ...
  }
  ...
}
```

所以，当一个View调用requestLayout()方法时，只有该View对应的视图树会执行重新测量和重新绘制，与其平级的View的视图树则不会执行；













## n、修改View的padding属性，需要调用哪个方法才可以使View更新

