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


```
    private void sendNewTaskResultRequestIfNeeded() {
        final ActivityStack sourceStack = mStartActivity.resultTo != null
                ? mStartActivity.resultTo.getStack() : null;
        if (sourceStack != null && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
            sourceStack.sendActivityResultLocked(-1 /* callingUid */, mStartActivity.resultTo,
                    mStartActivity.resultWho, mStartActivity.requestCode, RESULT_CANCELED,
                    null /* data */);
            mStartActivity.resultTo = null;
        }
    }
```
关于`sendNewTaskResultRequestIfNeeded`方法，当待启动Activity的flags包含`FLAG_ACTIVITY_NEW_TASK`，此时会直接回调给启动者（接收回调）的Activity一个空结果。
注释说明在一个标记`Intent.FLAG_ACTIVITY_NEW_TASK`的启动情况下**启动者Activity不应该请求待启动Activity返回结果回调**.此时会有一个奇怪的现象，启动者Activity被迅速回调生命周期的onResume -> onPause -> onResume -> onPause。而且此时`mStartActivity.resultTo`被置为null，可以看出只要设置了`Intent.FLAG_ACTIVITY_NEW_TASK`标志`mStartActivity.resultTo`就会被置为null;(根据先后顺序后面singleTask和singleInstance也会被加上`Intent.FLAG_ACTIVITY_NEW_TASK`但是mStartActivity.resultTo却为null，当然一般情况下非startActivityForResult启动resultTo都为null)


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

至此方法的**初始化**基本已经完成，接下来是开始判断待启动Activity所应该被放置的Task

## getReusableIntentActivity 确定新Activity的Task栈
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

（注：这里处理可能是在找到的Task中去启动一个新的Activity也有可能只是将Task移到前台。）

### 1.SingleInstance启动模式
#### 1.1 ActivityStackSupervisor.findActivityLocked
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

#### 1.2 ActivityStack.findActivityLocked
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

### 2.FLAG_ACTIVITY_LAUNCH_ADJACENT分屏
略

### 3.SingleTask启动模式 或 FLAG_ACTIVITY_NEW_TASK

#### 3.1 ActivityStackSupervisor.findTaskLocked
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
* taskIntent：标记了是哪个intent在启动Activity时创建的task,一般不会改变，即使这个Activity后面destroy了也不会变。
* affinityIntent：在Task栈发生reset时，Activity被分配到一个新的Task栈时会被设置上，和taskIntent基本一致。基本可以理解为创建Activity时被设置的intent
* rootAffinity：Task在setIntent过程中读取到affinity后第一次被设置(从null到有的过程,这里其实就是构造的时候)，以后被设置intent就不再会改动rootAffinity。

这里从Stack中的所有Task开始遍历，分三种情况

1. taskIntent.getComponent和待启动的Activit一致（包名和类名均一致），这个时候就直接返回`getTopActivity()`最顶部的Activity。可以理解成曾经这个Task就是由当前待启动的Activiy创建的
2. affinityIntent.getComponent和待启动的Activit一致（包名和类名均一致），这个时候就直接返回`getTopActivity()`最顶部的Activity。和1的情况差不多
3. 第一次被设置的intent的Affinity和当前待启动Activity在Manifest中配置的一致，则先缓存起来，在其它的Stack中继续遍历，看看有没有满足**情况1**和**情况2**的Task，如果没有就返回3中找到的顶部Activity。（继续遍历的逻辑在当前方法的上层执行方法，通过判断matchedByRootAffinity标志来确定是否继续遍历）可以看出匹配Manifest中配置的taskAffinity的优先级是最低的！

### getReusableIntentActivity 总结
* **SingleInstance：** 遍历所有Stack栈，在所有栈中寻找**同应用，同包名，同类名**的Activity返回，SingleInstance作用域覆盖整个App甚至包括多进程的App。

* **SingleTask 和 FLAG_ACTIVITY_NEW_TASK：** 在确定栈的时候**优先匹配Task的创建者（Activity），其次匹配Mainifest中配置的TaskAffinity**,该类Activity曾经创建了这个Task，则在启动同类型Activity时会优先考虑放置在该task中。

* 方法返回的一律都是Task.getTopActivity

## ReusedActivity
```
if (reusedActivity != null) {
            // 设置待启动Activity的task字段
            if (mStartActivity.getTask() == null) {
                mStartActivity.setTask(reusedActivity.getTask());
            }

            // 处理FLAG_ACTIVITY_CLEAR_TOP标志
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
                        top.getTask().setIntent(mStartActivity);
                    }
                    ActivityStack.logStartActivity(AM_NEW_INTENT, mStartActivity, top.getTask());
                    top.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                            mStartActivity.launchedFromPackage);
                }
            }

            // 将对应的task和stack移动到前台，top front
            reusedActivity = setTargetStackAndMoveToFrontIfNeeded(reusedActivity);

            // 判断是否需创建一个Activity来加入到Task中  (是否只移栈)
            setTaskFromIntentActivity(reusedActivity);

            // 判断不需要新加入Activity实例到栈中
            if (!mAddingToTask && mReuseTask == null) {
                resumeTargetStackIfNeeded();
                if (outActivity != null && outActivity.length > 0) {
                    outActivity[0] = reusedActivity;
                }
                return START_TASK_TO_FRONT;
            }
        }
```
在获取到ReusedActivity后总共做了以下几件事：
1. 为待启动Activity设置上ReusedActivity.task，意味着待启动Activity的目标栈就是ReusedActivity所在栈。因此`getReusableIntentActivity`方法可以理解为替新Activity寻找合适的栈；
2. 处理FLAG_ACTIVITY_CLEAR_TOP标志，如果新的待启动Activity在目标task中已经存在一个实例，这个时候就需要清除该实例顶部的所有Activity；
   1. 在清除时（performClearTaskForReuseLocked），如果是standard模式此时会将自己也清除
   2. 清除完成后非standard模式会回调一次`onNewIntent`
3. 将对应新Activity所在的Task栈移到Stack的最前面
4. 判断是否需要创建新Activity实例加入到目标Task栈中

### setTaskFromIntentActivity 确定是否新增Activity实例到Task中
```
    private void setTaskFromIntentActivity(ActivityRecord intentActivity) {
        if ((mLaunchFlags & (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK))
                == (FLAG_ACTIVITY_NEW_TASK | FLAG_ACTIVITY_CLEAR_TASK)) {
            final TaskRecord task = intentActivity.getTask();
            task.performClearTaskLocked();
            mReuseTask = task;
            mReuseTask.setIntent(mStartActivity);

            mMovedOtherTask = true;
        } 

        //检查是否具备ClearTop的效果
        else if ((mLaunchFlags & FLAG_ACTIVITY_CLEAR_TOP) != 0
                || mLaunchSingleInstance || mLaunchSingleTask) {
            ActivityRecord top = intentActivity.getTask().performClearTaskLocked(mStartActivity,
                    mLaunchFlags);

            //检查是否存在待启动Activity        
            if (top == null) {
                //如果不存在待启动Activity,则这个时候准备好在ReusedActivity中创建一个新的Activity
                mAddingToTask = true;
                mStartActivity.setTask(null);
                mSourceRecord = intentActivity;
                final TaskRecord task = mSourceRecord.getTask();
                if (task != null && task.getStack() == null) {
                    mTargetStack = computeStackFocus(mSourceRecord, false /* newTask */,
                            null /* bounds */, mLaunchFlags, mOptions);
                    mTargetStack.addTask(task,
                            !mLaunchTaskBehind /* toTop */, "startActivityUnchecked");
                }
            }
        } 

        // 在ReusedActivity和待启动Activity一致的情况，并且位于Task顶部（TopActivity）
        else if (mStartActivity.realActivity.equals(intentActivity.getTask().realActivity)) {

            // 如果intent标记了FLAG_ACTIVITY_SINGLE_TOP，此时回调onNewIntent给ReusedActivity
            if (((mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) != 0 || mLaunchSingleTop)
                    && intentActivity.realActivity.equals(mStartActivity.realActivity)) {

                if (intentActivity.frontOfTask) {
                    intentActivity.getTask().setIntent(mStartActivity);
                }
                intentActivity.deliverNewIntentLocked(mCallingUid, mStartActivity.intent,
                        mStartActivity.launchedFromPackage);
            }
           
            // 如果没有SingleTop标志位的情况下，两者虽然是同一个Activity但是intentFilter不同
            else if (!intentActivity.getTask().isSameIntentFilter(mStartActivity)) {
                // 这个时候会重新启动Activity，安全性考虑？
                mAddingToTask = true;
                mSourceRecord = intentActivity;
            }
        } else if ((mLaunchFlags & FLAG_ACTIVITY_RESET_TASK_IF_NEEDED) == 0) {
            mAddingToTask = true;
            mSourceRecord = intentActivity;
        } else if (!intentActivity.getTask().rootWasReset) {
            intentActivity.getTask().setIntent(mStartActivity);
        }
    }
```
此方法主要是用来确定是否需要新创建一个Activity实例加入到栈中；

具体有以下几种情况需要新启动：
1. 在NEW_TASK + CLEAR_TOP的情况下，或者 singleTask、singleInstance的启动模式
   1. 如果目标栈中本身没有待启动Activity，则**新加入Activity实例**。（此时执行`performClearTaskLocked`其实并不是为了真正去清除顶部其它Activity，只是为了确定待启动Activity是否存在于目标栈中）*
   2. 如果目标栈中存在待启动Activity，则什么都不做（因为之前外层已经回调给已经存的Activity实例一次onNewIntent了）

2. ReusedActivity（目标Task的topActivity）**目标Task的realActivity也就是创建Tasks时的Activity和待启动Activity一致**（这里比较的不是reusedActivity 和 mStartActivity 需要注意！）
   1. 待启动Activity是SingleTop的启动模式，这个时候只是回调一次onNewIntent
   2. 如果不是SingleTop，则判断如果IntentFilter相同则什么都不做，如果**不同则新加入Activity实例** *
   
3. 其它情况大部分是需要新加入Activity实例的（暂时忽略）

**重点，此方法所作的逻辑都是为了确定`mAddingToTask`的值。`mAddingToTask = true`代表要新加入Activity实例**


```
if (reusedActivity != null) {
        
        ...

        // 判断是否需创建一个Activity来加入到Task中  (是否只移栈)
        setTaskFromIntentActivity(reusedActivity);

        // 判断不需要新加入Activity实例到栈中
        if (!mAddingToTask && mReuseTask == null) {
            resumeTargetStackIfNeeded();
            if (outActivity != null && outActivity.length > 0) {
                outActivity[0] = reusedActivity;
            }
            return START_TASK_TO_FRONT;
        }
    }
```
从上述代码可以发现，要发生只移栈不新添加Activity只要保证**mAddingToTask**变量为false就可以了（当然前提是`reusedActivity != null`）。之前分析的setTaskFromIntentActivity就是来确定**mAddingToTask**变量的；

可以总结出几种**只移栈**的情况：
1. rootActivity(创建Task的Activity) 和 mStartActivity(待启动Activity)一致 && ( Task.topActivity和mStartActivity不一致 || `(mLaunchFlags & FLAG_ACTIVITY_SINGLE_TOP) == 0 `)
2. FLAG_ACTIVITY_NEW_TASK + FLAG_ACTIVITY_CLEAR_TOP 或 SingleTask 或 SingleInstance 并且reusedActivity.task中存在startActivity实例

情况1比较难理解，一般现象是栈被拉起来了，但是目标Activity却没有展示在前台
情况2则是基本的ClearTop操作执行以后会将顶部Activity出栈，将startActivity展示在前台

```
**思考, 此时的stack堆栈信息以及顶部的Activity (此时mAddingToTask是false)

[ActivityA] startActivity [ActivityB + Intent.FLAG_ACTIVITY_NEW_TASK + affinity]
[ActivityB] startActivity [ActivityC]
[ActivityB] finish
[ActivityC] startActivity [ActivityA + Intent.FLAG_ACTIVITY_NEW_TASK]
[ActivityA] startActivity [ActivityB + Intent.FLAG_ACTIVITY_NEW_TASK + affinity]

```

## 阶段总结
1. computeLaunchingTaskFlags：统一添加FLAG_ACTIVITY_NEW_TASK标志
   * mSourceActivity的启动模式为SingleInstance 
   * mStartActivity的启动模式为SingleTask和SingleInstance

2. getReusableIntentActivity：寻找mStartActivity的目标Task（返回目标Task的topActivity）
   * **SingleInstance：** 遍历所有Stack栈，在所有栈中寻找**同应用，同包名，同类名**的Activity返回
   * **SingleTask 和 FLAG_ACTIVITY_NEW_TASK：** 优先匹配Task的创建者（Activity），其次匹配Mainifest中配置的TaskAffinity

3. 处理ClearTop的情况
4. 移动目标Task到Stack顶部
5. setTaskFromIntentActivity：判断是否新建Activity实例，不需要新建则结束

注意点：SingleInstance作用域为应用而非进程，特殊情况会导致只移栈而看不到启动的Activity，NEW_TASK的含义是尝试去寻找一个合适的栈（可能是新栈也可能和启动者同栈）