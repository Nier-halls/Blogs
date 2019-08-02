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

缩减版源码
```
    private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,
            IVoiceInteractionSession voiceSession, IVoiceInteractor voiceInteractor,
            int startFlags, boolean doResume, ActivityOptions options, TaskRecord inTask,
            ActivityRecord[] outActivity) {

        setInitialState(r, options, inTask, doResume, startFlags, sourceRecord, voiceSession,
                voiceInteractor);

        computeLaunchingTaskFlags();

        computeSourceStack();

        mIntent.setFlags(mLaunchFlags);

        ActivityRecord reusedActivity = getReusableIntentActivity();

        final int preferredLaunchStackId =
                (mOptions != null) ? mOptions.getLaunchStackId() : INVALID_STACK_ID;
        final int preferredLaunchDisplayId =
                (mOptions != null) ? mOptions.getLaunchDisplayId() : DEFAULT_DISPLAY;

        if (reusedActivity != null) {

            if (mStartActivity.getTask() == null) {
                mStartActivity.setTask(reusedActivity.getTask());
            }

            if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                    || isDocumentLaunchesIntoExisting(mLaunchFlags)
                    || mLaunchSingleInstance || mLaunchSingleTask) {
                final TaskRecord task = reusedActivity.getTask();

                final ActivityRecord top = task.performClearTaskForReuseLocked(mStartActivity,
                        mLaunchFlags);

                if (reusedActivity.getTask() == null) {
                    reusedActivity.setTask(task);
                }

                if (top != null) {
                    if (top.frontOfTask) {
                        // 新启动Activity在Task栈底，刷新intent
                        top.getTask().setIntent(mStartActivity);
                    }
                    top.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                            mStartActivity.launchedFromPackage);
                }
            }


            reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

            setTaskFromIntentActivity(reusedActivity);

            if (!mAddingToTask && mReuseTask == null) {
                resumeTargetStackIfNeeded();
                if (outActivity != null && outActivity.length > 0) {
                    outActivity[0] = reusedActivity;
                }
                return START_TASK_TO_FRONT;
            }
        }

        final ActivityStack topStack = mSupervisor.mFocusedStack;
        final ActivityRecord topFocused = topStack.topActivity();
        final ActivityRecord top = topStack.topRunningNonDelayedActivityLocked(mNotTop);
        final boolean dontStart = top != null && mStartActivity.resultTo == null
                && top.realActivity.equals(mStartActivity.realActivity)
                && top.userId == mStartActivity.userId
                && top.app != null && top.app.thread != null
                && ((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0
                || mLaunchSingleTop || mLaunchSingleTask);
        if (dontStart) {
            ActivityStack.logStartActivity(AM_NEW_INTENT, top, top.getTask());
            // For paranoia, make sure we have correctly resumed the top activity.
            topStack.mLastPausedActivity = null;
            if (mDoResume) {
                mSupervisor.resumeFocusedStackTopActivityLocked();
            }
            
            ...

            top.deliverNewIntentLocked(
                    mCallingUid, mStartActivity.intent, mStartActivity.launchedFromPackage);
            ...

            return START_DELIVERED_TO_TOP;
        }

        boolean newTask = false;
        final TaskRecord taskToAffiliate = (mLaunchTaskBehind && mSourceRecord != null)
                ? mSourceRecord.getTask() : null;

        int result = START_SUCCESS;
        if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
                && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            newTask = true;
            result = setTaskFromReuseOrCreateNewTask(
                    taskToAffiliate, preferredLaunchStackId, topStack);
        } else if (mSourceRecord != null) {
            result = setTaskFromSourceRecord();
        } else if (mInTask != null) {
            result = setTaskFromInTask();
        } else {
            setTaskToCurrentTopOrCreateNewTask();
        }
      
        if (mSourceRecord != null) {
            mStartActivity.getTask().setTaskToReturnTo(mSourceRecord);
        }

        mTargetStack.startActivityLocked(mStartActivity, topFocused, newTask, mKeepCurTransition,
                mOptions);
        ...
        return START_SUCCESS;
    }
```



## setInitialState 初始化配置参数
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

关于`sendNewTaskResultRequestIfNeeded`方法，当待启动Activity的flags包含`FLAG_ACTIVITY_NEW_TASK`，此时会直接回调给启动者（接收回调）的Activity一个空结果。
注释说明在一个标记`FLAG_ACTIVITY_NEW_TASK`的启动情况下，启动者Activity不应该请求待启动Activity返回结果回调.此时会有一个奇怪的现象，启动者Activity被迅速回调生命周期的onResume -> onPause -> onResume -> onPause。


## computeLaunchingTaskFlags 预配置FLAG_ACTIVITY_NEW_TASK
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

#### 1.启动者非Activity
此时非Activity是一定不会拥有Task任务栈的，待启动Activity必定会被放置在一个其它Task中，因此AMS会补充上`FLAG_ACTIVITY_NEW_TASK`标志

#### 2.启动者launchMode = LAUNCH_SINGLE_INSTANCE
启动模式为`LAUNCH_SINGLE_INSTANCE`的Activity所属的Task栈只允许有自己这一个Activity，因此新启动Activity自然会被加上 `FLAG_ACTIVITY_NEW_TASK`标志被放置到其它Task中

#### 3.待启动Activity.launchMode=singleInstance或singleTask
singleInstance 和 singleTask两种启动模式启动的Activity会尝试在所有栈中寻找一个合适的栈来放置新Activity（或只是回调onNewIntent）,这里也涉及到遍历寻找栈，因此需要加上`FLAG_ACTIVITY_NEW_TASK`尝试去寻找一个栈

## computeSourceStack 确定Activity所在栈
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
方法主要是寻找待启动Activity的目标stack，一般由`Activity.startActivity()`启动的Activity默认情况下都会被放在启动者所在的stack中。

至此方法的初始化工作基本已经完成，接下来是开始判断待启动Activity所应该被放置的Task

## getReusableIntentActivity 
```
    private ActivityRecord getReusableIntentActivity() {

        // putIntoExitingTask用来标记是否需要去寻找一个合适的Task来处理新的Activity启动
        boolean putIntoExistingTask = ((mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0 &&
                (mLaunchFlags & FLAG_ACTIVITY_MULTIPLE_TASK) == 0)
                || mLaunchSingleInstance || mLaunchSingleTask;
    
        putIntoExistingTask &= mInTask == null && mStartActivity.resultTo == null;
        ActivityRecord intentActivity = null;

        //指定TaskId的情况，一般用不到
        if (mOptions != null && mOptions.getLaunchTaskId() != -1) {
            final TaskRecord task = mSupervisor.anyTaskForIdLocked(mOptions.getLaunchTaskId());
            intentActivity = task != null ? task.getTopActivity() : null;
        } 
        // 不指定Task的情况
        else if (putIntoExistingTask) {
            // 1.SingleInstance启动模式
            if (mLaunchSingleInstance) {
               intentActivity = mSupervisor.findActivityLocked(mIntent, mStartActivity.info, false);
            } 
            // 2.分屏的情况，暂不考虑
            else if ((mLaunchFlags & FLAG_ACTIVITY_LAUNCH_ADJACENT) != 0) {
                intentActivity = mSupervisor.findActivityLocked(mIntent, mStartActivity.info,
                        !mLaunchSingleTask);
            }
            // 3.SingleTask启动模式或者标记Intent.FLAG_ACTIVITY_NEW_TASK
             else {
                intentActivity = mSupervisor.findTaskLocked(mStartActivity, mSourceDisplayId);
            }
        }
        return intentActivity;
    }
```
`getReusableIntentActivity`负责在Activity不确定Task栈的情况，也就是代码中提到的三种情况:

1. SingleInstance启动模式
2. FLAG_ACTIVITY_LAUNCH_ADJACENT分屏（暂不考虑）
3. SingleTask启动模式 或 FLAG_ACTIVITY_NEW_TASK

在上述情况下AMS会去尝试寻找一个合适的Task来处理它。

在找到一个合适的Task后会以`ActivityRecord`的形式来返回。这里寻找Task却返回`ActivityRecord`是因为后面的操作光靠`TaskRecord`还是无法确定的，还需要`ActivityRecord`中的部分信息配合才能决定新Activity的启动行为；

（注：这里处理可能是在找到的Task中去启动一个新的Activity也有可能只是将Task移到前台后什么都不处理。）

#### 1.SingleInstance启动模式
##### 1.1 ActivityStackSupervisor.findActivityLocked
```
    ActivityRecord findActivityLocked(Intent intent, ActivityInfo info,
                                      boolean compareIntentFilters) {
        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                // 遍历所有Stack，调用Stack.findActivityLocked
                // compareIntentFilters参数是false
                final ActivityRecord ar = stacks.get(stackNdx)
                        .findActivityLocked(intent, info, compareIntentFilters);
                if (ar != null) {
                    return ar;
                }
            }
        }
        return null;
    }
```
遍历所有的Stack栈并且调用`ActivityStack.findActivityLocked`来寻找合适的

##### 1.2 ActivityStack.findActivityLocked
```
    ActivityRecord findActivityLocked(Intent intent, ActivityInfo info, boolean compareIntentFilters) {
        
        ComponentName cls = intent.getComponent();
        if (info.targetActivity != null) {
            cls = new ComponentName(info.packageName, info.targetActivity);
        }

        //应用对应的userId
        final int userId = UserHandle.getUserId(info.applicationInfo.uid);

        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            final TaskRecord task = mTaskHistory.get(taskNdx);
            final ArrayList<ActivityRecord> activities = task.mActivities;

            for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx) {
                ActivityRecord r = activities.get(activityNdx);
                if (!r.okToShowLocked()) {
                    continue;
                }
                //通过uid判断是否是相同的应用
                if (!r.finishing && r.userId == userId) {
                    // 参数传递的compareIntentFilters = false，进入else模块
                    if (compareIntentFilters) {
                        if (r.intent.filterEquals(intent)) {
                            return r;
                        }
                    } else {
                        // 判断intent.getComponent()一致就返回
                        if (r.intent.getComponent().equals(cls)) {
                            return r;
                        }
                    }
                }
            }
        }

        return null;
    }
```
代码逻辑：找到和待启动Activity相同应用且包名类名相同的Task
这里可以得到一个结论：SingleInstance启动模式启动的Activity作用域范围是在同一个应用（可以是不同进程）中只允许有一个

（注：`r.intent.getComponent().equals(cls) `比较的是包名和类名 ` mPackage.equals(other.mPackage) && mClass.equals(other.mClass);`）

#### 2.FLAG_ACTIVITY_LAUNCH_ADJACENT分屏
略

#### 3.SingleTask启动模式 或 FLAG_ACTIVITY_NEW_TASK

##### 3.1 ActivityStackSupervisor.findTaskLocked
```
    ActivityRecord findTaskLocked(ActivityRecord r, int displayId) {
        mTmpFindTaskResult.r = null;
        mTmpFindTaskResult.matchedByRootAffinity = false;
        ActivityRecord affinityMatch = null;

        for (int displayNdx = mActivityDisplays.size() - 1; displayNdx >= 0; --displayNdx) {
            final ArrayList<ActivityStack> stacks = mActivityDisplays.valueAt(displayNdx).mStacks;
            for (int stackNdx = stacks.size() - 1; stackNdx >= 0; --stackNdx) {
                final ActivityStack stack = stacks.get(stackNdx);
                
                ...

                stack.findTaskLocked(r, mTmpFindTaskResult);
              
                // 匹配最优选项
                if (mTmpFindTaskResult.r != null) {
                    if (!mTmpFindTaskResult.matchedByRootAffinity) {
                        return mTmpFindTaskResult.r;
                    } else if (mTmpFindTaskResult.r.getDisplayId() == displayId) {
                        affinityMatch = mTmpFindTaskResult.r;
                    }
                }
            }
        }

        return affinityMatch;
    }
```
遍历所有Stack，然后调用`Stack.findTaskLocked()`方法，结果会保存在`mTmpFindTaskResult`参数中返回

##### 3.2 Stack.findTaskLocked
```
    void findTaskLocked(ActivityRecord target, FindTaskResult result) {
        Intent intent = target.intent;
        ActivityInfo info = target.info;
        ComponentName cls = intent.getComponent();
        if (info.targetActivity != null) {
            cls = new ComponentName(info.packageName, info.targetActivity);
        }
        final int userId = UserHandle.getUserId(info.applicationInfo.uid);

        for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
            final TaskRecord task = mTaskHistory.get(taskNdx);
            final Intent affinityIntent = task.affinityIntent;
            final Intent taskIntent = task.intent;
            final ActivityRecord r = task.getTopActivity();

            // 下面分析暂时不考虑document相关的情况
            ...

            // taskIntent.getComponent() 和 待启动Activity一致的情况
            if (taskIntent != null && taskIntent.getComponent() != null &&
                    taskIntent.getComponent().compareTo(cls) == 0 &&
                    Objects.equals(documentData, taskDocumentData)) {
                result.r = r;
                result.matchedByRootAffinity = false;
                break;
            }
            // Task.affinityIntent和待启动Activity.taskAffinity一致
             else if (affinityIntent != null && affinityIntent.getComponent() != null &&
                    affinityIntent.getComponent().compareTo(cls) == 0 &&
                    Objects.equals(documentData, taskDocumentData)) {
                result.r = r;
                result.matchedByRootAffinity = false;
                break;
            } 
            // 如果待启动Activity.taskAffinity和task.rootAffinity一致
            else if (!isDocument && !taskIsDocument
                    && result.r == null && task.rootAffinity != null) {
                if (task.rootAffinity.equals(target.taskAffinity)) {
                    result.r = r;
                    // 此时会设置true返回给上层，上层会继续遍历
                    result.matchedByRootAffinity = true;
                }
            }
        }
    }
```
三个概念
* taskIntent：标记了是哪个intent在启动Activity时创建的task,一般不会发生改变，即使这个Activity后面destroy了也不会。
* affinityIntent：在Task栈发生reset时，Activity被分配到一个新的Task栈时会被设置上，和taskIntent基本一致。基本可以理解为创建Activity时被设置的intent
* rootAffinity：Task在setIntent过程中读取到affinity后第一次被设置(从null到有的过程)，以后被设置就不再会改动。最初的affinity，可以说这个Task的affinity曾经被设置过rootAffinity过

这里从Stack中的所有Task开始遍历，分三种情况

1. taskIntent.getComponent和待启动的Activit一致（包名和类名均一致），这个时候就直接返回`getTopActivity()`最顶部的Activity。可以理解成曾经这个Task就是由当前待启动的Activiy创建的
2. affinityIntent.getComponent和待启动的Activit一致（包名和类名均一致），这个时候就直接返回`getTopActivity()`最顶部的Activity。和1的情况差不多
3. taskAffinity在某些情况下会被设置intent，第一次被设置的intent的Affinity和当前待启动的一致，则先缓存起来，在其它的Stack中继续遍历，看看有没有满足情况1和情况2的Task，如果没有就返回3中找到的顶部Activity。（继续遍历的逻辑在上层调用方法，判断matchedByRootAffinity标志）

#### 总结
`getReusableIntentActivity`获取一下几种Activity
1. SingleInstance的情况下，应用中已启动且和待启动Activity相同类型的Activity
2. SingleTask或者NEW_TASK的情况下，Task.topActivity, 且此Task的启动时Activity和待启动Acitvity相同

其余情况都会返回null


## 思考
```
[ActivityA] startActivity [ActivityB + Intent.FLAG_ACTIVITY_NEW_TASK + affinity]
[ActivityB] startActivity [ActivityC]
[ActivityB] finish
[ActivityC] startActivity [ActivityA + Intent.FLAG_ACTIVITY_NEW_TASK]
[ActivityA] startActivity [ActivityB + Intent.FLAG_ACTIVITY_NEW_TASK + affinity]
此时的stack堆栈信息以及顶部的Activity
```