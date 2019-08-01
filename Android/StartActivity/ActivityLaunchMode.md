# 索引
> * startActivityUnchecked解析
>   1. setInitialState
>   2. computeLaunchingTaskFlags
>   3. computeSourceStack
>   4. getReusableIntentActivity
>   5. setTask & tryClearTop
>   5. setTargetStackAndMoveToFrontIfNeeded
>   6. setTaskFromIntentActivity
>
> * 时序图
>   1. 存在reusedActivity
>   2. 不存在reusedActivity
> 
> * launcheMode & flag
>   1. standard
>   2. singleTop
>   3. singleTask
>   4. singleInstance
>   5. FLAG_ACTIVITY_SINGLE_TOP
>   6. FLAG_ACTIVITY_NEW_TASK
>   7. FLAG_ACTIVITY_CLEAR_TOP
> 
> * 总结

# startActivityUnchecked
## params
```
ActivityRecord r //待启动Activity
ActivityRecord sourceRecord //启动者Activity的映射
IVoiceInteractionSession voiceSession
IVoiceInteractor voiceInteractor
int startFlags //启动标志 正常通过startActivity启动的都是0
boolean doResume //true
ActivityOptions options //bundle的映射,ActivityOptions.fromBundle
TaskRecord inTask //默认为null
ActivityRecord[] outActivity //缓存结果的ActivityRecord数组
```

## setInitialState
```
    private void setInitialState(
        ActivityRecord r, ActivityOptions options, TaskRecord inTask,
        boolean doResume, int startFlags, ActivityRecord sourceRecord,
        IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor) 
        {
            reset();
            mStartActivity = r; //待启动Activity
            mIntent = r.intent;
            mOptions = options;
            mSourceRecord = sourceRecord; //启动者Activity的映射
            mSourceDisplayId = getSourceDisplayId(mSourceRecord, mStartActivity);
            ...
            mLaunchSingleTop = r.launchMode == LAUNCH_SINGLE_TOP;
            mLaunchSingleInstance = r.launchMode == LAUNCH_SINGLE_INSTANCE;
            mLaunchSingleTask = r.launchMode == LAUNCH_SINGLE_TASK;
            
            //在回调FLAG_ACTIVITY_NEW_TASK的情况下尝试回调onActivityResult
            sendNewTaskResultRequestIfNeeded();
            ...
            mDoResume = doResume; //是否执行Resume，也可以理解是否显示
            ...
            mInTask = inTask; //null
            mStartFlags = startFlags; //0
        }
```

方法主要做了初始化参数，赋值了关键变量。

这里最需要注意的就是mLaunchxxx这几个启动模式的情况以及局部变量和参数的映射关系。

关于sendNewTaskResultRequestIfNeeded方法，当待启动Activity的flags包含FLAG_ACTIVITY_NEW_TASK，此时会直接回调给启动者（接收回调）的Activity一个空结果。
注释说明在一个标记FLAG_ACTIVITY_NEW_TASK的启动情况下，启动者Activity不应该请求待启动Activity返回结果回调.此时会有一个奇怪的现象，启动者Activity被迅速回调生命周期的onResume -> onPause -> onResume -> onPause。


## computeLaunchingTaskFlags
```
    private void computeLaunchingTaskFlags() {
      
        if (mSourceRecord == null && mInTask != null && mInTask.getStack() != null) {
            // 当startActivity调用者不是来自于另一个Activity时的处理情况，这里暂时不做考虑
            ...
        } else {
            mInTask = null;
            //这里是默认程序选择器，这里不做考虑
            if ((mStartActivity.isResolverActivity() || mStartActivity.noDisplay) && 
                mSourceRecord != null &&
                mSourceRecord.isFreeform())  {
                mAddingToTask = true;
            }
        }

        if (mInTask == null) {
            //当不指定Task的情况下，一下三种情形会补充加入 FLAG_ACTIVITY_NEW_TASK 标志
            // 1. 当启动者不是一个Activity，
            if (mSourceRecord == null) {
                if ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) == 0 && mInTask == null) {
                    mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
                }
            } 
            // 2. 当启动者是一个 LAUNCH_SINGLE_INSTANCE 的Activity
            else if (mSourceRecord.launchMode == LAUNCH_SINGLE_INSTANCE) {
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
            // 3. 待启动Activity的launchMode是singleInstance或者singleTask
            else if (mLaunchSingleInstance || mLaunchSingleTask) {
                mLaunchFlags |= FLAG_ACTIVITY_NEW_TASK;
            }
        }
    }

```
方法主要是为了补充 FLAG_ACTIVITY_NEW_TASK 标志，又三种情况下AMS会为我们补充FLAG_ACTIVITY_NEW_TASK的标志。

#### 启动者非Activity
此时非Activity是一定不会拥有Task任务栈的，待启动Activity必定会被放置在一个其它Task中，因此AMS会补充上 FLAG_ACTIVITY_NEW_TASK 标志

#### 启动者launchMode = LAUNCH_SINGLE_INSTANCE
启动模式为 LAUNCH_SINGLE_INSTANCE 的Activity所属的Task栈只允许有自己这一个Activity，因此新启动Activity自然会被加上 FLAG_ACTIVITY_NEW_TASK 标志被放置到其它Task中

#### 待启动Activity.launchMode=singleInstance或singleTask
singleInstance 和 singleTask两种启动模式启动的Activity会尝试在所有栈中寻找一个合适的栈来放置新Activity（或只是回调onNewIntent）,这里也涉及到遍历寻找栈，因此需要加上 FLAG_ACTIVITY_NEW_TASK 尝试去寻找一个栈

## computeSourceStack
```
 private void computeSourceStack() {
        if (mSourceRecord == null) {
            mSourceStack = null;
            return;
        }
        if (!mSourceRecord.finishing) {
            mSourceStack = mSourceRecord.getStack();
            return;
        }
        ...
    }
```
方法主要是寻找待启动Activity的目标stack，一般由Activity.startActivity启动的Activity默认情况下都会被放在启动者所在的stack中

## getReusableIntentActivity

