# StartActivity

### 1. ActivityStarter.startActivityMayWait
1. 解析了intent.Component(Activity)的Manifest配置并且生成ResolveInfo，ActivityInfo;
   * ResolveInfo: 对应Manifest的intentFilter标签解析，包含了IntentFilter、resolvePackageName、activityInfo等信息
   * ActivityInfo: 对应Manifest文件中Activity标签的信息解析，包含了launchMode、permission、taskAffinity、targetActivity等信息
2. **加锁**继续调用startActivityLocked()方法转而调用ActivityStarter.startActivity()方法


### 2. ActivityStarter.startActivity
1. 检查状态及参数
   * 检查调用startActivity的pid是否合法
   * 检查intent中解析的数据是否合法，Activity是否存在
   * 找到sourceActivity(调用startActivity的Activity)在AMS中对应的ActivityRecord
   * 检查调用者的权限是否合法
2. 创建一个ActivityRecord对象


### 3. ActivityStarter.startActivityUnchecked
1. computeLaunchingTaskFlags：统一添加FLAG_ACTIVITY_NEW_TASK标志
   * mSourceActivity的启动模式为SingleInstance 
   * mStartActivity的启动模式为SingleTask和SingleInstance

2. getReusableIntentActivity：寻找mStartActivity的目标Task（返回目标Task的topActivity）
   * **SingleInstance：** 遍历所有Stack栈，在所有栈中寻找**同应用，同包名，同类名**的Activity返回
   * **SingleTask 和 FLAG_ACTIVITY_NEW_TASK：** 优先匹配Task的创建者（Activity），其次匹配Mainifest中配置的TaskAffinity

3. 处理ClearTop的情况
4. 移动目标Task到Stack顶部
5. 判断是否新建Activity实例，不需要新建则结束




## 关键方法参数表
### 1. ActivityManagerService.startActivity
```
## ActivityManagerService.startActivity

*IApplicationThread     caller:             ActivityThread.mAppThread //暂时不明确是什么
 String                 callingPackage:     sourceActivity.getBasePackageName() //应用包名
 Intent                 intent:             Intent(sourceActivity, targetActivity.class) //调用startActivity传入的intent，
 String                 resolveType:        Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
*IBinder                resultTo:           sourceActivity.mToken //暂时不明确是什么
*String                 resultWho:          sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
 int                    requestCode:        -1 //是原样返回的吗
 int                    startFlags:         0
 ProfilerInfo           profilerInfo:       null //暂时不明白是什么
 Bundle                 bOptions:           null //需要传递的参数

("*"标记参数后续从源码明确作用)
```

### 2. ActivityStarter.startActivityMayWait
```
## ActivityStarter.startActivityMayWait

*IApplicationThread         caller                  ActivityThread.mAppThread //暂时不明确是什么
 String                     callingUid              -1
 String                     callingPackage          sourceActivity.getBasePackageName() //应用包名
 Intent                     intent                  Intent(sourceActivity, targetActivity.class) //调用startActivity传入的intent，
 String                     resolveType             Intent.resolveTypeIfNeeded(activity.getContentResolver()) //暂时不明白有什么用
 IVoiceInteractionSession   voiceSession            null
 IVoiceInteractor           voiceInteractor         null
*IBinder                    resultTo:               sourceActivity.mToken //暂时不明确是什么
*String                     resultWho:              sourceActivity.mEmbeddedID //暂时不明确是什么 唯一id？
 int                        requestCode:            -1 //是原样返回的吗
 int                        startFlags:             0
 ProfilerInfo               profilerInfo:           null //暂时不明白是什么
 WaitResult                 outResult               null
 Configuration              globalConfig            null
 Bundle                     bOptions:               null //需要传递的参数
 boolean                    ignoreTargetSecurity    false
 int                        userId                  UserHandle.getCallingUserId() //暂时不确定是什么，可能是进程uid
 IActivityContainer         iContainer              null
 TaskRecord                 inTask                  null
 String                     reason                  "startActivityAsUser"
 
("*"标记参数后续从源码明确作用)
```

### 3.ActivityRecord构造:
```
## ActivityRecord.java

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


#### 数据结构:
##### ActivityRecord
##### ProcessRecord
##### TaskRecord
```
    intent.component:包含包名和启动项的类名
    
    task.rootAffinity:创建task的Activity的affinity，不会改变
    task.intent:创建task的Activity对应的intent
    task.rootAffinity:启动Task的activity的affinity,一般不指定的情况下默认是包名
    task.mTaskHistory:task中保存的所有ActivityRecord，新启Activity排在后面
    task.realActivity:启动task的activity.cmp

    activityRecord.launchMode.LAUNCH_MULTIPLE:就是Manifest中的普通启动模式standard
    
```

    
####TODO
1. 后续是怎么利用返回的Activity来处理的，总结此方法的作用
2. 整理之前的启动逻辑，归纳之前做了什么工作(ActivityRecord的创建, IntentResolve, ActivityInfo的创建，fix flags)
3. 整理启动模式的细节，launchMode
4. 整理整个ActivityThread和ActivityManagerService直接的交互逻辑
5. 思考如何整理一个文档，能够高效的回忆起startActivity的所有逻辑，以什么样的形式来整理归纳
6. ActivityTask和ActivityStack的含义以及区别
7. Activity.task是什么时候被置为null的，onDestroy整个流程AMS是否是同步执行的
8. 并不是singleTask或NEW_TASK就一定会新启动栈，singleInstance一定会新启动栈
