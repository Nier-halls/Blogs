#StartActivity
## 提纲
> * StartActivity整体流程概括
> 
>   1. ActivityThread模块与AMS交互，图文
>   2. 关键问题提出
>      
>       1. ActivityThread和AMS之间Activity的映射关系
>       2. SourceActivity和StartingActivity之间生命周期的变化
>       3. Stack & Task 何时入栈 setTaskFromSourceRecord
> 
> * StartActivity源码细节
> 
>   0. 整体调用流程图
>   1. Client端Activity发起startActivity
>   2. AMS解析intent
>   3. ActivityRecord
>   4. 启动模式的判断（忽略）
>   5. Stack & Task (何时入栈)
>   6. Pause SourceActivity
>   7. 创建启动StartingActivity
>   8. Stop SourceActivity
> 
> * 总结
> 
> * TODO：分析过程一定要**精简**，控制篇幅，整个流程只要能足够自己回忆起全部内容就可以了，**抓住主线**

## 0. 前言
整理这篇文章的目的是在回顾时可以通过文章提到的主干流程回忆扩展完整的只是结构，分析过程会比较**精简**，只要能够帮助自己复盘即可。

## 1. StartActivity流程图
Activity的启动流程涉及到IPC跨进程通讯，主要关联Application所在进程以及AMS进程。交互流程图如下

![StartActivit](./pic/start_activity1.png)


* StartActivity整个过程类似与一个C/S架构，Application担任Client端，ActivityManagerService担任Server端
* ActivityManagerService管理者所有应用的Activity的状态以及回退栈，而Application只是ActivityManagerService在其中一个进程中具体表现
* StartActivity的过程中永远是启动者Activity优先进入Pause阶段，而后再创建新Activity


## 2. StartActivity中的一些疑问
1. **Activity作为Client端表现，那么它在ActivityManagerService的表现上形式是什么？**
2. **如何确定一个Activity在Application端和AMS端的映射关系？**
3. **Stack和Task的概念，它们有什么区别？**
4. **整个启动过程中Activity的生命周期是如何发生变化的，是否有序？**
5. **为什么StartActivity的过程需要以C/S的形式来实现**

根据上述问题以及流程图来分析StartActivity的源码

## 3. 源码分析(API-27)
StartActivity的源码流程可以总结为5步：
* 发起StartActivity请求
* 创建ActivityRecord
* 寻找栈 & 入栈
* 准备启动 —— call pause
* 启动Activity


### 3.1 Application发起startActivity请求   
```
    // [CODE]android.app.Activity
     public void startActivity(Intent intent, @Nullable Bundle options) {
        if (options != null) {
            startActivityForResult(intent, -1, options);
        } else {
            startActivityForResult(intent, -1);
        }
    }

    public void startActivityForResult(@RequiresPermission Intent intent, int requestCode,
            @Nullable Bundle options) {
            ...
                mInstrumentation.execStartActivity(
                    this, mMainThread.getApplicationThread(), mToken, this,
                    intent, requestCode, options);
            ...
    }
```
```
    // [CODE]android.app.Instrumentation
    public ActivityResult execStartActivity(
            Context who, IBinder contextThread, IBinder token, Activity target,
            Intent intent, int requestCode, Bundle options) {
        ...
            int result = ActivityManager.getService()
                .startActivity(whoThread, who.getBasePackageName(), intent,
                        intent.resolveTypeIfNeeded(who.getContentResolver()),
                        token, target != null ? target.mEmbeddedID : null,
                        requestCode, 0, null, options);
        ...                
        return null;
    }
```
Activity执行startActivity方法最终会跨进程调用ActivityManagerService的startActivity，作为参数的intent中包含着Application需要启动的所有Activity。此过程Application的任务就是组装参数，并且跨进程发起请求。

### 3.2 AMS创建ActivityRecord
#### 3.2.1 解析Intent中的参数
```
    // [CODE]com.android.server.am.ActivityStarter
    final int startActivityMayWait(IApplicationThread caller, int callingUid,
            String callingPackage, Intent intent, IBinder resultTo...) {
                ...
                //调用PackageManagerService来解析Intent
                ResolveInfo rInfo = mSupervisor.resolveIntent(intent, resolvedType, userId); 

                //从Intent解析获取的信息中解析处待启动的Activity信息
                ActivityInfo aInfo = mSupervisor.resolveActivity(intent, rInfo, startFlags, profilerInfo);

                int res = startActivityLocked(caller, intent, aInfo, 
                                                rInfo, resultTo, resultWho, requestCode...);
                ...
    }
```
AMS会调用PackageManagerService来解析Intent中的数据，其中包括带启动Activity的信息以及Manifes中对应的配置，解析完Itent并且获得对应Activity信息后就开始准备创建ActivitRecord；

#### 3.2.2 创建ActivityRecord
```
    // [CODE]com.android.server.am.ActivityStarter
    private int startActivity(IApplicationThread caller, Intent intent, 
                                ActivityInfo aInfo, ResolveInfo rInfo, 
                                IBinder resultTo, String resultWho...) {
        ...
                                    
        ActivityRecord r = new ActivityRecord(mService, callerApp, callingPid, callingUid,
                callingPackage, intent, resolvedType, aInfo, mService.getGlobalConfiguration(),
                resultRecord, resultWho, requestCode, componentSpecified, voiceSession != null,
                mSupervisor, options, sourceRecord);       

        ...

        return startActivity(r, sourceRecord, voiceSession, voiceInteractor, startFlags, true,
                options, inTask, outActivity);
    }
```

Activity类是作为Application端也就是Client端展现的形式，而对应在AMS中Activity则是以ActivityRecord的形式展现，ActivityRecord可以理解为Application中Activity的映射。

Application中的Activity和AMS中的ActivityRecord是一一对应的关系。

以下是Activity的构造方法参数供以后查阅：
```
[type]                      [name]                      [description]
ActivityManagerService      _service                    AMS对应的实例
ProcessRecord               _caller                     根据Activity.startActivity传入的IApplicationThread映射找到的
int                         _launchedFromPid            调用方sourceActivity所在的进程id
int                         _launchedFromUid            调用方sourceActivity所在进程的uid
Intent                      _intent                     intent
String                      _resolveType                Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
ActivityInfo                aInfo                       解析Manifest获得的信息描述对象
Configuration               _configuration              AMS.getGlobalConfiguration()
ActivityRecord              _resultTo                   null //有什么意义吗？和resultWho有什么区别
String                      _resultWho                  sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
int                         _reqCode                    requestCode -1
boolean                     _componentSpecified         intent.getComponent() != null, true
boolean                     _rootVoiceInteraction       false
ActivityStackSupervisor     supervisor                  调用基础方法的执行对象
ActivityContainer           container                   null
ActivityOption              options                     ActivityOptions.fromBundle(bOptions)
ActivityRecord              sourceRecord                发起startActivity的Activity对应的ActivityRecord
```

### 3.3 入栈
#### 3.3.1 Stack & Task
Android中所有Activity都归Stack(间接)管理，主要是为了实现Activity的新加入后回退。在Android中它的实现形式是ActivityStack类。

##### Stack ActivityStack
ActivityStack保证Activity先进先出（FIFO）的顺序的基础上以Task为单位来管理Activity。
因此ActivityStack并非是直接管理Activity的。

```com.android.server.am.ActivityStack
// [CODE]com.android.server.am.ActivityStarter
ActivityRecord findActivityLocked(Intent intent, ActivityInfo info,
                                    boolean compareIntentFilters) {
    ...
    for (int taskNdx = mTaskHistory.size() - 1; taskNdx >= 0; --taskNdx) {
        //
        final TaskRecord task = mTaskHistory.get(taskNdx);
        final ArrayList<ActivityRecord> activities = task.mActivities;
        for (int activityNdx = activities.size() - 1; activityNdx >= 0; --activityNdx){
            ...
        }
    }
    return null;
}
```
上面是一个ActivityStack类中的其中一个查找Activity的方法，从方法中可以看出ActivityStack想要找到指定的Activity需要先遍历Stack中的所有Task（ActivityStack.mTaskHistory）,然后再从栈中获取所有Activity（TaskRecord.mActivities）。从上述方法可以看出ActivityStack在管理Activity的形式并非是直接管理的。

##### Task TaskRecor
Task在Android中是直接管理Activity的容器，Task秉承Activity先进先出的顺序，是ActivityStack的具体细节体现。

Activity在被压入Task时虽然遵循FIFO原则但并非只是一个一个压入一个一个弹出，根据Activity不同的启动模式（LaunchMode）可以定制Activity在Task中的行为，如单例（SingTask），单例单栈（SingInstance）等。

任何一个Activity都会被放置在一个Task中，默认放置在启动者相同的栈中。

##### Task & Stack & Activity 关系
![StartActivit](./pic/start_activity2.png)

关系：
* Stack管理一系列Task
* Task管理一系列Activity
* Task遵循FIFO，是Stack栈形式的体现

 ActivityStack利用Task这一个中间层管理Acitivty带来了跟多的灵活性，Activity并非只能死板的先进先出，而是可以以任务组Task的形式来切换任意一组的Task（以及Task包含的Activity）到前台或者后台。

#### 3.3.2 寻找新Activity的位置
单独整理在LaunchMode篇

#### 3.3.3 压入目标栈
```
// [CODE]com.android.server.am.ActivityStarter
private int startActivityUnchecked(final ActivityRecord r, ActivityRecord sourceRecord,  boolean doResume, TaskRecord inTask ...) {
    ...
    if (mStartActivity.resultTo == null && mInTask == null && !mAddingToTask
            && (mLaunchFlags & FLAG_ACTIVITY_NEW_TASK) != 0) {
        // 1.需要创建并加入到一个Task中     
        newTask = true;
        result = setTaskFromReuseOrCreateNewTask(
                taskToAffiliate, preferredLaunchStackId, topStack);
    } else if (mSourceRecord != null) {
        // 2.加入到启动者相同的task（mSourceRecord严格上说并非是启动者，可能被替换）
        result = setTaskFromSourceRecord();
    } else if (mInTask != null) {
        // 3.需要加入一个指定Task的情况    
        result = setTaskFromInTask();
    }
}

private int setTaskFromSourceRecord() {
    ...
    addOrReparentStartingActivity(sourceTask, "setTaskFromSourceRecord");
    return START_SUCCESS;
}

private void addOrReparentStartingActivity(TaskRecord parent, String reason) {
    ...
    parent.addActivityToTop(mStartActivity);
}
```
Activity在入栈时一般会有三种可能：
1. 创建一个新的栈并且放入，如Luancher启动的Activity
2. 加入到一个已有的Task栈(mSourceActivity.getTask())中，如普通启动时加入到当前启动者对应栈中，FLAG_ACTIVITY_NEW_TASK或者SingleTask等时寻找一个已经存在的合适的栈加入。
3. 加入到一个指定的栈中，这种情况比较少

最后都会将新启动的Activity压入到Task的顶部（特定情况除外）



### 3.4 启动前准备
#### Pause当前显示的Activity


### 3.5 启动新Activity