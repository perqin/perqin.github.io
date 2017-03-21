---
title: 从源码探索事件分发 1
date: 2017-03-22 00:37:51
tags: [explore-event-dispatching-from-source, android]
---

以前一直认为身为Android开发者，一定要去读Android的源码，然而由于各种各样的原因迟迟没有开始，最近终于下定决心提高一下，买了《Android开发艺术探索》，并开始尝试从源码看懂触摸事件的分发。这个系列文章会记录我通过阅读源代码的方式一步步搞懂Android里的事件分发的过程。

作为系列的第一篇，我们先开开上帝视角，搬出一些结论来。从Android的布局xml结构我们就能很容易发现，Android中的View们是以一棵树的形式组成的，这样的设计是几乎所有有GUI相关概念的平台里都会使用的设计，在树状的视图结构中，我们可以把里面的一个子树当作是一个View，也就是单一的一个节点来处理，这样就可以通过递归的思想进行事件分发、测量布局和渲染等行为。

当我们的手指落在屏幕上的时候，驱动捕获到我们的行为，发送到Android系统上层框架，最终，一个包含这次屏幕操作的MotionEvent对象被传递给了对应的Activity对象。这个Activity会把这个对象分发给这个Activity中的根View，并通过递归的形式一直传递下去，直到这个触摸行为被某个View识别并消费掉。形象地说，就像一份通知从学校教务处发到了辅导员，辅导员把它分发给他管理的班级的班长，班长分发给班级的每个宿舍长，最终被发到每个学生手上。

前面说道，我们可以把一个子树看作一个View，因此我们只需要弄懂最简单的情形：一个ViewGroup包含一个View，事件从ViewGroup分发给View的情况，就可以很快理解事件在一棵巨大的View树里如何分发。

## 测试代码

为了测试，我自定义了一个ViewGroup和一个View，他们的唯一目的就是在和事件分发相关的方法里输出日志。其中`EdFrameLayout`的代码如下：

````java
public class EdFrameLayout extends FrameLayout {
    private static final String TAG = "EdFrameLayout";

    public EdFrameLayout(Context context) {
        super(context);
    }

    public EdFrameLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent ev) {
        Log.i(TAG, "dispatchTouchEvent: " + EventNameUtils.eventNameOf(ev.getAction()));
        return super.dispatchTouchEvent(ev);
    }

    @Override
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        Log.i(TAG, "onInterceptTouchEvent");
        return super.onInterceptTouchEvent(ev);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i(TAG, "onTouchEvent");
        return super.onTouchEvent(event);
    }
}
````

`EdView`的代码如下：

````java
public class EdView extends View {
    private static final String TAG = "EdView";

    public EdView(Context context) {
        super(context);
    }

    public EdView(Context context, @Nullable AttributeSet attrs) {
        super(context, attrs);
    }

    @Override
    public boolean dispatchTouchEvent(MotionEvent event) {
        Log.i(TAG, "dispatchTouchEvent: " + EventNameUtils.eventNameOf(event.getAction()));
        return super.dispatchTouchEvent(event);
    }

    @Override
    public boolean onTouchEvent(MotionEvent event) {
        Log.i(TAG, "onTouchEvent");
        return super.onTouchEvent(event);
    }
}
````

其中`EventNameUtils`类仅仅是为了输出事件的名称：

```java
public class EventNameUtils {
    public static String eventNameOf(int action) {
        switch (action) {
            case MotionEvent.ACTION_DOWN: return "ACTION_DOWN";
            case MotionEvent.ACTION_MOVE: return "ACTION_MOVE";
            case MotionEvent.ACTION_UP: return "ACTION_UP";
            case MotionEvent.ACTION_CANCEL: return "ACTION_CANCEL";
            default: return "UNKNOWN ACTION";
        }
    }
}
```

布局文件如下：

```xml
<com.perqin.playground.eventdispatching.EdFrameLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context="com.perqin.playground.activities.MainActivity">

    <com.perqin.playground.eventdispatching.EdView
        android:id="@+id/ed_view"
        android:layout_width="240dp"
        android:layout_height="240dp"
        android:background="@color/colorAccent"
        android:layout_gravity="center" />

</com.perqin.playground.eventdispatching.EdFrameLayout>
```

运行起来之后是这样的：

![Playground](playground.png)

我们轻触中间的强调色的方块，看到日志里有这样的输出：

````
com.perqin.playground I/EdFrameLayout: dispatchTouchEvent: ACTION_DOWN
com.perqin.playground I/EdFrameLayout: onInterceptTouchEvent
com.perqin.playground I/EdView: dispatchTouchEvent: ACTION_DOWN
com.perqin.playground I/EdView: onTouchEvent
com.perqin.playground I/EdFrameLayout: onTouchEvent
````

## 再开上帝视角

直接从源码就把事件分发无中生有地摸清对我来说还是挺困难的，因此我之前也早已看过书、搜索过相关文章。从上面的日志也得以映证时间分发的流程：

*  某个View（不论是否是ViewGroup）的`dispatchTouchEvent`被其父节点调用，从中该View可以得到这次触摸事件的MotionEvent对象。
*  如果这个View是ViewGroup，它会先调用`onInterceptTouchEvent`方法，根据返回值判断它自己要不要将它拦截。如果不拦截，它会找到需要接受这个事件的子View，通过调用子View的`dispatchTouchEvent`方法将事件分发下去（正如它的父View将事件这样分发给它一样）。
*  如果这个View不是ViewGroup，它会直接调用自己的`onTouchEvent`，根据返回值判断是否自己想消费这个事件，并在`dispatchTouchEvent`中返回自己是否消费了这个事件。
*  将事件分发给子View的ViewGroup会从其`dispatchTouchEvent`的返回值判断子View是否消费了事件。如果没有View愿意消费，那么它自己消费或者也不消费并告知自己的父View（正如拒绝消费的子View们一样）。

以上描述不一定准确，但基本把事件分发的逻辑讲述了一遍。接下来，我们可以开始看源码了～

## ViewGroup的dispatchTouchEvent

不得不说源码实在太复杂了，光是ViewGroup的`dispatchTouchEvent`就有200多行。我把里面的一些语句（包括多点触摸相关代码、只用于Debug的语句、安全校验等）删掉，得到下面的代码：

```java
public class ViewGroup {
    // ...
    public boolean dispatchTouchEvent(MotionEvent ev) {
        boolean handled = false;

        final int action = ev.getAction();
        final int actionMasked = action & MotionEvent.ACTION_MASK;

        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }

        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }

        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;

        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
        if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
                    }
                    if (preorderedList != null) preorderedList.clear();
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Did not find a child to receive the event.
                    // Assign the pointer to the least recently added target.
                    newTouchTarget = mFirstTouchTarget;
                    while (newTouchTarget.next != null) {
                        newTouchTarget = newTouchTarget.next;
                    }
                    newTouchTarget.pointerIdBits |= idBitsToAssign;
                }
            }
        }

        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }

        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP) {
            resetTouchState();
        }

        return handled;
    }
}
```

仍然有147行，我能怎么办，我也很绝望啊！实话说这个方法我断断续续看了几天才勉强捋清楚……好的， 废话不多说，我们来看看ViewGroup是如何分发事件的。

这个方法里的`handled`局部变量就是这个ViewGroup（或其子View）最终是否消费了这个事件。`actionMasked`只是将高3个字节复位（因为action是整型），直接看作是这个事件的action就好。

首先，如果这是一个`ACTION_DOWN`动作，也就是按下屏幕的一瞬间，我们需要把之前的状态重置，为这个新的手势做准备（手势就是按下、一系列的移动和抬起动作序列）：

```java
        // Handle an initial down.
        if (actionMasked == MotionEvent.ACTION_DOWN) {
            // Throw away all previous state when starting a new touch gesture.
            // The framework may have dropped the up or cancel event for the previous gesture
            // due to an app switch, ANR, or some other state change.
            cancelAndClearTouchTargets(ev);
            resetTouchState();
        }
```

从方法名就可以看出他们的作用是取消、清除原来的触摸目标，并重置触摸状态。这里先剧透一下，TouchTarget就是对能够接受触摸事件的子View的简单封装。

然后，由于我们是一个ViewGroup，所以我们需要判断要不要拦截这次事件：

```java
        // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            // There are no touch targets and this action is not an initial down
            // so this view group continues to intercept touches.
            intercepted = true;
        }
```

这时我们看到了一个`mFirstTouchTarget`的属性，这个属性挺重要，但网上对这个属性的解释足够详细的实在太少。如果看完了整个`dispatchTouchEvent`方法，就会发现，ViewGroup会管理一群能够消费触摸事件的子View，ViewGroup把他们放在一个单向链表里，而这个`mFirstTouchTarget`就是这个链表的头。而为什么一个手势会被多个View同时消费我还没有深挖，从直觉来说，应该和多点触摸等有关。

从上面的代码片段可以看出，如果这是手势的开始，我们就会通过调用`onInterceptTouchEvent`，决定是否拦截。如果这个链表不为空，表示有子View想消费，所以也需要判断要不要拦截。只有在手势进行到中间又没有子View愿意背锅（消费事件）的情况下，才会拦截。

直接这么解释似乎很牵强，其实只要和拦截行为的表现联系起来就理解了。对于ViewGroup而言，事实上他会在一个手势的发生过程中不停地判断是否拦截，这就是为什么在子View会消费的情况下仍然要调用`onInterceptTouchEvent`。而另一方面，一旦这个ViewGroup已经决定啦，就由它自己来拦截这个事件，那么**这之后`onInterceptTouchEvent`都不会再调用（废话，都决定拦截了就别反悔嘛），且之后会一直拦截下去**，因为决定拦截之后，后面会给所有原来消费事件的子View发送取消事件（`ACTION_CANCEL`），并将他们从`mFirstTouchTarget`中去除，所以如果`mFirstTouchTarget`是空的，又不是初始的按下事件，那么就表示之前已经被拦截了，所以要继续拦截。

注意这里的`disallowIntercept`可以先不考虑。其实子View可以通过设置父View的这个属性来强制取消父View的拦截。

接下来是判断是否是取消事件：

```java
        // Check for cancelation.
        final boolean canceled = resetCancelNextUpFlag(this)
                || actionMasked == MotionEvent.ACTION_CANCEL;
```

这里`resetCancelNextUpFlag`方法的目的我也不甚清楚，如果你知道，希望你能把答案留在评论区！

`split`我认为和多点触摸有关，在多点触摸的时候会把事件分割，因此我们先忽略……

接下来，`newTouchTarget`是新的触摸对象，`alreadyDispatchedToNewTouchTarget`显然就是说明是否被分发给新的触摸对象了：

```java
        // Update list of touch targets for pointer down, if needed.
        final boolean split = (mGroupFlags & FLAG_SPLIT_MOTION_EVENTS) != 0;
        TouchTarget newTouchTarget = null;
        boolean alreadyDispatchedToNewTouchTarget = false;
```

如果没有被取消，也没有被拦截，那么就要尝试分发给子View了：

```java
        if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN) {
                final int actionIndex = ev.getActionIndex(); // always 0 for down
                final int idBitsToAssign = TouchTarget.ALL_POINTER_IDS;

                // Clean up earlier touch targets for this pointer id in case they
                // have become out of sync.
                removePointersFromTouchTargets(idBitsToAssign);

                final int childrenCount = mChildrenCount;
                if (newTouchTarget == null && childrenCount != 0) {
                    // Part A
                }

                if (newTouchTarget == null && mFirstTouchTarget != null) {
                    // Part B
                }
            }
        }
```

从上面的结构可以看出，事件分发主要是在手势开始，也就是按下事件发生的时候开始决定谁来消费的。此时`newTouchTarget`显然还是空的，因此如果此时子View的数量不为0，就要从中找出能够消费事件的子View了（Part A）。注意，我认为这里的Part B是用于多点触摸的：如果Part A之后`newTouchTarget`仍然为空，那么就是说当前没有子View愿意消费，但是`mFirstTouchTarget`可能包含原来已经有的手指的触摸手势对应的触摸对象，因此是有可能不为空的。但这些暂时不是这次的主要分析目标，所以我们不考虑Part B。

接下来看看Part A的代码：

```java
                if (newTouchTarget == null && childrenCount != 0) {
                    final float x = ev.getX(actionIndex);
                    final float y = ev.getY(actionIndex);
                    // Find a child that can receive the event.
                    // Scan children from front to back.
                    final ArrayList<View> preorderedList = buildTouchDispatchChildList();
                    final boolean customOrder = preorderedList == null
                            && isChildrenDrawingOrderEnabled();
                    final View[] children = mChildren;
                    for (int i = childrenCount - 1; i >= 0; i--) {
                        // ...
                    }
                    if (preorderedList != null) preorderedList.clear();
                }
```

很简单，得到触摸的坐标，并开始遍历所有的子View。对于每个被遍历到的子View：

```java
                        final int childIndex = getAndVerifyPreorderedIndex(
                                childrenCount, i, customOrder);
                        final View child = getAndVerifyPreorderedView(
                                preorderedList, children, childIndex);

                        if (!canViewReceivePointerEvents(child)
                                || !isTransformedTouchPointInView(x, y, child, null)) {
                            continue;
                        }

                        newTouchTarget = getTouchTarget(child);
                        if (newTouchTarget != null) {
                            // Child is already receiving touch within its bounds.
                            // Give it the new pointer in addition to the ones it is handling.
                            newTouchTarget.pointerIdBits |= idBitsToAssign;
                            break;
                        }

                        resetCancelNextUpFlag(child);
                        if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                            // Child wants to receive touch within its bounds.
                            newTouchTarget = addTouchTarget(child, idBitsToAssign);
                            alreadyDispatchedToNewTouchTarget = true;
                            break;
                        }
```

从上面可见，我们先得到当前遍历到的子View的对象和index序号。然后通过`canViewReceivePointerEvents`和 `isTransformedTouchPointInView`两个方法，判断这个子View能否接受这个事件。如果任何一个条件不满足，就会跳过这个子View。`getTouchTarget`方法是从`mFirstTouchTarget`触摸对象链表中找到这个View对应的触摸对象，显然是找不到的，因此会向后继续执行，调用`dispatchTransformedTouchEvent`。这个方法里面就进行了对子View调用`dispatchTouchEvent`这个操作，并从返回值判断是否被消费了（回顾前文，`dispatchTouchEvent`的返回值表示该View是否会消费这个事件）。如果消费了，那么我们通过`addTouchTarget`方法把这个View封装到一个TouchTarget里，添加到链表里，同时赋值给`newTouchTarget`，`alreadyDispatchedToNewTouchTarget`也被置为真。

分发给子View的代码结束了，看下面的代码，我们判断了`mFirstTouchTarget`是否为空。那么什么时候这个链表是空的呢？

1. 当手势开始，而如前面所述，没有子View接锅的时候
2. 手势中间，但之前已经被拦截下来的时候

在这两种情况下，我们ViewGroup自己要去考虑消费这个事件了：

```java
        // Dispatch to touch targets.
        if (mFirstTouchTarget == null) {
            // No touch targets so treat this as an ordinary view.
            handled = dispatchTransformedTouchEvent(ev, canceled, null,
                    TouchTarget.ALL_POINTER_IDS);
        } else {
            // ...
        }
```

如上，注意虽然我们又一次调用了`dispatchTransformedTouchEvent`，但我们给View参数传递了`null`，我们后面会发现，这个方法不仅可以把事件分发给子View，还能分发给自己。

如果我们已经有子View消费了事件，理论上会执行上面片段里的`else`块：

```java
        if (mFirstTouchTarget == null) {
            // ...
        } else {
            // Dispatch to touch targets, excluding the new touch target if we already
            // dispatched to it.  Cancel touch targets if necessary.
            TouchTarget predecessor = null;
            TouchTarget target = mFirstTouchTarget;
            while (target != null) {
                final TouchTarget next = target.next;
                if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                    handled = true;
                } else {
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
                }
                predecessor = target;
                target = next;
            }
        }
```

初看之下，`else`块中开始遍历链表，此时链表里就只有一个新的子View，`alreadyDispatchedToNewTouchTarget && target == newTouchTarget`也为真，一切都很和谐。

等等，我们是不是忽略了一个情况？我们前面说过，如果我们在手势中途拦截了手势，`mFirstTouchTarget`在哪里被清空呢？正是在这个`else`块里！由于在这个情况下，`newTouchTarget`必然为空，那么原来链表里的子View都会被删除：

```java
                    final boolean cancelChild = resetCancelNextUpFlag(target.child)
                            || intercepted;
                    if (dispatchTransformedTouchEvent(ev, cancelChild,
                            target.child, target.pointerIdBits)) {
                        handled = true;
                    }
                    if (cancelChild) {
                        if (predecessor == null) {
                            mFirstTouchTarget = next;
                        } else {
                            predecessor.next = next;
                        }
                        target.recycle();
                        target = next;
                        continue;
                    }
```

从上可见，如果`intercepted`为真，则`cancelChild`也为真，会通过`dispatchTransformedTouchEvent`把取消事件分发给子View，并把他们都从链表中删除。注意，从上面我们也可以看出，在首次拦截的时候，如果这个事件之前是被其他子View消费的，那么我们只分发取消事件给子View，子View的`onTouchEvent`仍然会被调用（事件类型为`ACTION_CANCEL`），而这个ViewGroup的`onTouchEvent`还不会被调用，需要等待下一次的`ACTION_MOVE`。

事实上，上面的`else`块还承载了分发后续事件给子View的任务：在后续的MOVE和UP事件中，由于`mFirstTouchTarget`不为空，会直接运行到`else`这个块，并由于没有`newTarget`，因此会通过上面片段的第3-4行分发这个后续事件给子View。

最后，如果是取消事件或抬起手指事件，我们重置状态：

```java
        // Update list of touch targets for pointer up or cancel, if needed.
        if (canceled
                || actionMasked == MotionEvent.ACTION_UP) {
            resetTouchState();
        }
```

那么，到此为止，ViewGroup的`dispatchTouchEvent`就看完了，真累啊……

对于`onInterceptTouchEvent`，源码如下，可以看作就是不拦截：

```java
    public boolean onInterceptTouchEvent(MotionEvent ev) {
        if (ev.isFromSource(InputDevice.SOURCE_MOUSE)
                && ev.getAction() == MotionEvent.ACTION_DOWN
                && ev.isButtonPressed(MotionEvent.BUTTON_PRIMARY)
                && isOnScrollbarThumb(ev.getX(), ev.getY())) {
            return true;
        }
        return false;
    }
```

ViewGroup并没有重写`onTouchEvent`，我们如果看View的`onTouchEvent`，会发现：

```java
    public boolean onTouchEvent(MotionEvent event) {
        final float x = event.getX();
        final float y = event.getY();
        final int viewFlags = mViewFlags;
        final int action = event.getAction();

        if ((viewFlags & ENABLED_MASK) == DISABLED) {
            if (action == MotionEvent.ACTION_UP && (mPrivateFlags & PFLAG_PRESSED) != 0) {
                setPressed(false);
            }
            // A disabled view that is clickable still consumes the touch
            // events, it just doesn't respond to them.
            return (((viewFlags & CLICKABLE) == CLICKABLE
                    || (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE)
                    || (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE);
        }
        if (mTouchDelegate != null) {
            if (mTouchDelegate.onTouchEvent(event)) {
                return true;
            }
        }

        if (((viewFlags & CLICKABLE) == CLICKABLE ||
                (viewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) ||
                (viewFlags & CONTEXT_CLICKABLE) == CONTEXT_CLICKABLE) {
            // A lot of code...
            return true;
        }

        return false;
    }

```

从上面的节选可以看出，如果这个View不可点击和长按（默认即是如此），那么`onTouchEvent`是返回`false`的。从源码中还可以发现，即使这个View被禁用（disabled）了，只要它是可点击的，它仍然会消费触摸事件，这个和我们的直觉是有点违背的，所以算是看源码的额外收获。很多坑也许都能从源码中找出答案！

到这里，我们就弄懂了ViewGroup的事件分发逻辑：

*  手指按下的时候
   *  重置各种状态
   *  不拦截：遍历子View，分发事件给能够接受这个事件的子View，并将消费这个事件的子View加入到链表，如果没有找到消费的子View，则分发事件给自己
   *  拦截：分发事件给自己
*  手指滑动或抬起的时候
   *  不拦截：
      *  有子View在消费该事件：继续分发给该子View
      *  无子View消费：分发事件给自己
   *  刚好拦截：清除正在消费事件的子View（如果有）并分发取消事件
   *  已经拦截：分发事件给自己

上面的“刚好拦截”指的是当前手势过程中第一次拦截，“已经拦截”表示后续的拦截。

到此为止，我们就基本了解了View树中为什么事件会这样传递，知其然亦知其所以然。但是，这个关键的`dispatchTouchEvent`中调用了很多其他方法，我会在下一篇中一一探索，包括它如何判断一个View有资格接受这个事件？子View的`dispatchTouchEvent`到底在哪里被调用？