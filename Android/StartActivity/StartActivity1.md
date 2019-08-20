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

## 1. StartActivity整体流程概括
Activity的启动流程涉及到IPC跨进程通讯，主要关联Application所在进程以及AMS进程。交互流程图如下

![StartActivit](./pic/start_activity1.png)


* StartActivity整个过程类似与一个C/S架构，Application担任Client端，ActivityManagerService担任Server端
* ActivityManagerService管理者所有应用的Activity的状态以及回退栈，而Application只是ActivityManagerService在其中一个进程中具体表现
* StartActivity的过程中永远是启动者Activity优先进入Pause阶段，而后再创建新Activity


## 2. StartActivity中的一些疑问
1. **Activity作为Client端表现，它在ActivityManagerService的表现上形式是什么？**
2. **如何确定一个Activity在Application端和AMS端的映射关系？**
3. **Stack和Task的概念，它们有什么区别？**
4. **整个启动过程中Activity的生命周期是如何发生变化的？是否有固定顺序？**

接下来根据上述问题以及流程图来分析整个Activity的流程

## 3. 源码分析
### 3.1 Activity.startActivity
```
    // [code]android.app.Activity
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
    // [code]android.app.Instrumentation
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
由Activity.startActivity方法最终调用ActivityManagerService跨进程传入了