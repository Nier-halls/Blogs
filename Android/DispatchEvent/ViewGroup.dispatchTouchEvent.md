# ViewGroup事件分发

## 提纲
>* 0. ViewGroup事件分发概览
>* 1. 拦截
>       1. 判断
>       2. 处理
>       3. 总结
>* 2. 分发
>       1. 首次分发（ActionDown）
>       2. 非首次分发（TouchTarget）



## 0. ViewGroup事件分发伪代码
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    boolean handle;
    if (onInterceptTouchEvent(ev)){
        handle = this.onTouchEvent();
    } else {
        handle = child.dispatchTouchEvent(ev)
    }
    return handle;
}
```
以上是事件分发流程的伪代码。

## 1. 拦截

### 1.1 判断
```
  // Check for interception.
        final boolean intercepted;
        if (actionMasked == MotionEvent.ACTION_DOWN
                || mFirstTouchTarget != null) {
            final boolean disallowIntercept = (mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0;

            //子View在接收到ActionDown到时候才有资格禁止父View拦截
            if (!disallowIntercept) {
                intercepted = onInterceptTouchEvent(ev);
                ev.setAction(action); // restore action in case it was changed
            } else {
                intercepted = false;
            }
        } else {
            intercepted = true;
        }
```
事件分发的拦截相关代码逻辑，前置条件
* `ACTION_DOWN` 
* `mFirstTouchTarget != null`
* `mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0`

为什么设置两个这两个条件：
1. actionMasked == MotionEvent.ACTION_DOWN：代表分发流程到开始，初次分发
2. mFirstTouchTarget != null：代表非初次分发，且非ViewGroup自己处理到情况

上述两个条件都是为了保证在事件分发的过程你各种功能，因此可知拦截的关键就是**必须在分发到流程中**

#### 条件一：TouchTarget != null + MotionEvent.ACTION_DOWN
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...

    if (actionMasked == MotionEvent.ACTION_DOWN) {
            resetTouchState();
    }
    ...
}
```
```
private void cancelAndClearTouchTargets(MotionEvent event) {
    if (mFirstTouchTarget != null) {
        boolean syntheticEvent = false;
        ...
        clearTouchTargets();

        if (syntheticEvent) {
            event.recycle();
        }
    }
}
```
当ViewGroup在接收到ActionDown的时候会把TouchTarget清空，为了确保初次分发时也能考虑拦截所以加上ActionDown的判断条件。

因此只有在`TouchTarget != null`的时候才会尝试去拦截，因为此时代表正在分发的过程中，已经有子View在处理事件了。（非ActionDown的情况如果TouchTarget还是null则一定是ViewGroup自己处理事件，拦截和不拦截都一样）


#### 条件二：FLAG_DISALLOW_INTERCEPT
FLAG_DISALLOW_INTERCEPT标志是通过`requestDisallowInterceptTouchEvent`方法来设置的，而且只能发生在事件分发过程中，因为事件分发开始（ActionDown触发时）会重置这个标志位，如下：
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...

    if (actionMasked == MotionEvent.ACTION_DOWN) {
            resetTouchState();
    }
    ...
}
```
```
private void resetTouchState() {
    clearTouchTargets();
    resetCancelNextUpFlag(this);
    mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
    mNestedScrollAxes = SCROLL_AXIS_NONE;
}

```
这个标志是子View防止Parent拦截本可以传递给自己的Event的一种手段。

#### 1.2 处理
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    if (mFirstTouchTarget == null) {
        // No touch targets so treat this as an ordinary view.
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
    }
    ...
}
```
```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    }
}
```
决定拦截后直接回调`super.dispatchTouchEvent`最后会转而调用`View.onTouchEvent`

#### 1.3 总结
1. onInterceptTouchEvent（触发拦截判断）只有在**拦截的过程中**发生（**初次ActionDown** + **非初次TouchTarget != null**）
2. ViewGroup决定拦截（onInterceptTouchEvent返回true）后续则不会再回调onInterceptTouchEvent方法，因为TouchTarget会被主动设置成null
3. FLAG_DISALLOW_INTERCEPT是一种特殊手段，允许子View组织Parent拦截（ActionDown后调用`requestDisallowInterceptTouchEvent`）


## 2. 分发

### 2.1 首次分发
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...

    if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                for (int i = childrenCount - 1; i >= 0; i--) {
                    final int childIndex = getAndVerifyPreorderedIndex(
                            childrenCount, i, customOrder);
                    final View child = getAndVerifyPreorderedView(
                            preorderedList, children, childIndex);

                    // 检查之前View是否已经在处理事件来，这个是多点触控的情况下
                    // 如果这个View已经处理来则还是交给这个View来处理
                    newTouchTarget = getTouchTarget(child);
                    if (newTouchTarget != null) {
                        newTouchTarget.pointerIdBits |= idBitsToAssign;
                        break;
                    }

                    if (dispatchTransformedTouchEvent(ev, false, child, idBitsToAssign)) {
                        //事件被child处理了
                        //保存本次处理事件的child到mTouchTarget变量中
                        newTouchTarget = addTouchTarget(child, idBitsToAssign);
                        alreadyDispatchedToNewTouchTarget = true;
                        break;
                    }

                    ev.setTargetAccessibilityFocus(false);
                }
    
            }
    }
    ...
}
```

```
private boolean dispatchTransformedTouchEvent(MotionEvent event, boolean cancel,
        View child, int desiredPointerIdBits) {
    final boolean handled;

    if (child == null) {
        handled = super.dispatchTouchEvent(transformedEvent);
    } else {
        final float offsetX = mScrollX - child.mLeft;
        final float offsetY = mScrollY - child.mTop;
        transformedEvent.offsetLocation(offsetX, offsetY);
   
        handled = child.dispatchTouchEvent(transformedEvent);
    }

    return handled;
}
```
流程：

0. 前置条件：为非ActionCancel + 非拦截。这两种情况都是ViewGroup自己（作为一个View）来处理；
1. 遍历所有的child，找出合适的调用`dispatchTransformedTouchEvent`
2. `dispatchTransformedTouchEvent`调用child.dispatchTouchEvent


### 2.2 非首次分发 递归分发
```
public boolean dispatchTouchEvent(MotionEvent ev) {
    ...
    // 不会触发寻找child的逻辑（不考虑多点情况）
    if (!canceled && !intercepted) {
            if (actionMasked == MotionEvent.ACTION_DOWN
                        || (split && actionMasked == MotionEvent.ACTION_POINTER_DOWN)
                        || actionMasked == MotionEvent.ACTION_HOVER_MOVE) {

                ...
            }
    }

    if (mFirstTouchTarget == null) {
        // 没有子View处理事件，ViewGroup自己处理
        handled = dispatchTransformedTouchEvent(ev, canceled, null,
                TouchTarget.ALL_POINTER_IDS);
    } else {
        // 之前有child正在接收事件
        TouchTarget target = mFirstTouchTarget;
        while (target != null) {
            final TouchTarget next = target.next;
            if (alreadyDispatchedToNewTouchTarget && target == newTouchTarget) {
                handled = true;
            } else {
                //回调child.dispathTouhEvent
                if (dispatchTransformedTouchEvent(ev, cancelChild,
                        target.child, target.pointerIdBits)) {
                    handled = true;
                }
            }
            predecessor = target;
            target = next;
        }
    }

    return handled;
}
```
主流程：

0. 分发过程中处理事件的child被保存在局部变量`mFirstTouchTarget`中
1. 不需要重新去寻找child（多点触控除外），直接从`mFirstTouchTarget`获取之前处理事件的child；
2. 调用`dispatchTransformedTouchEvent`递归分发事件

结论：
1. child只有处理了ActionDown才有可能接收后续event
2. 一个事件只能被一个child消耗，即使在拦截的时候
3. 当View处理了ActionDown就能持续获取后续事件，哪怕后续事件它不处理，mTouchTarget是会一致保存着
