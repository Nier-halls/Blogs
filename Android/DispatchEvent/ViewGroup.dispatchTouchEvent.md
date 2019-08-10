## ViewGroup事件分发

### 提纲
>* 1. 拦截
>* 2. 分发
>       1. 首次分发（ActionDown）
>       2. 非首次分发（TouchTarget）



### 1. 拦截 onInterceptTouchEvent
```
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
事件分发的拦截相关代码逻辑，前置条件
* `ACTION_DOWN` 
* `mFirstTouchTarget != null`
* `mGroupFlags & FLAG_DISALLOW_INTERCEPT) != 0`
  
1. ViewGroup能够拦截的前提是TouchTarget不能为null，除非Event是ActionDown
2. ViewGroup没设置FLAG_DISALLOW_INTERCEPT标志

#### 条件一：TouchTarget不能为null，除非Event是ActionDown
当ViewGroup在接收到ActionDown的时候会把TouchTarget清空
```
    if (actionMasked == MotionEvent.ACTION_DOWN) {
            resetTouchState();
    }

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
因此只有在TouchTarget不为Null的时候才会尝试取拦截，并且只有在不为Null的时候拦截才有意义。（非ActionDown的情况TouchTarget还是Null则一定是ViewGroup自己处理事件，拦截和不拦截都一样）

#### 条件二：没设置FLAG_DISALLOW_INTERCEPT
FLAG_DISALLOW_INTERCEPT标志是通过`requestDisallowInterceptTouchEvent`方法来设置的，而且只能发生在事件分发过程中，因为事件分发开始（ActionDown触发时）会重置这个标志位，如下：
```
    if (actionMasked == MotionEvent.ACTION_DOWN) {
            resetTouchState();
    }

    private void resetTouchState() {
        clearTouchTargets();
        resetCancelNextUpFlag(this);
        mGroupFlags &= ~FLAG_DISALLOW_INTERCEPT;
        mNestedScrollAxes = SCROLL_AXIS_NONE;
    }
```
这个标志是子View防止Parent拦截本可以传递给自己的Event的一种手段

#### 总结记忆
拦截的两个条件都是和TouchTarget相关的，前提都是TouchTarget的存在（ActionDown不知道会不会有TouchTarget所以也一并判断）且TouchTarget允许拦截（未设置FLAG_DISALLOW_INTERCEPT）。