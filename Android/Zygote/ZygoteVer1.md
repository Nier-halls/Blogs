
# Zygote App进程启动流程 


### 概述

* App发起进程：当从桌面启动应用，则发起进程便是Launcher所在进程；当从某App内启动远程进程，则发送进程便是该App所在进程。发起进程先通过binder发送消息给system_server进程；
* system_server进程：调用Process.start()方法，通过socket向zygote进程发送创建新进程的请求；
* zygote进程：在执行ZygoteInit.main()后便进入runSelectLoop()循环体内，当有客户端连接时便会执行ZygoteConnection.runOnce()方法，再经过层层调用后fork出新的应用进程；
* 新进程：执行handleChildProc方法，最后调用ActivityThread.main()方法。


### 1 发起创建进程请求
发起创建请求一般发生在打开一个Activity的时候，如果发现Application不存在则会创建进程
```
    //ActivityStackSupervisor
    void startSpecificActivityLocked(ActivityRecord r,
            boolean andResume, boolean checkConfig) {

        ProcessRecord app = mService.getProcessRecordLocked(r.processName,
                r.info.applicationInfo.uid, true);

        if (app != null && app.thread != null) {
           ...
        }
        // 检查如果进程不存在则创建进程
        mService.startProcessLocked(r.processName, r.info.applicationInfo, true, 0,
                "activity", r.intent.getComponent(), false, false, true);
    }
```
```
    //ActivityManagerService
    private final void startProcessLocked(ProcessRecord app, String hostingType,
            String hostingNameStr, String abiOverride, String entryPoint, String[] entryPointArgs) {

                startResult = Process.start(entryPoint,
                        app.processName, uid, uid, gids, debugFlags, mountExternal,
                        app.info.targetSdkVersion, seInfo, requiredAbi, instructionSet,
                        app.info.dataDir, invokeWith, entryPointArgs);
    }
```

### 2 Process.start
```
    public final Process.ProcessStartResult start(final String processClass,
                                                  final String niceName,
                                                  int uid, int gid, int[] gids,
                                                  int debugFlags, int mountExternal,
                                                  int targetSdkVersion,
                                                  String seInfo,
                                                  String abi,
                                                  String instructionSet,
                                                  String appDataDir,
                                                  String invokeWith,
                                                  String[] zygoteArgs) {
            return startViaZygote(processClass, niceName, uid, gid, gids,
                    debugFlags, mountExternal, targetSdkVersion, seInfo,
                    abi, instructionSet, appDataDir, invokeWith, zygoteArgs);
    }
```
```
    private Process.ProcessStartResult startViaZygote(final String processClass,
                                                      final String niceName,
                                                      final int uid, final int gid,
                                                      final int[] gids,
                                                      int debugFlags, int mountExternal,
                                                      int targetSdkVersion,
                                                      String seInfo,
                                                      String abi,
                                                      String instructionSet,
                                                      String appDataDir,
                                                      String invokeWith,
                                                      String[] extraArgs)
                                                      throws ZygoteStartFailedEx {
        ArrayList<String> argsForZygote = new ArrayList<String>();
        ... //拼装请求Zygote进程创建新进程的参数
        synchronized(mLock) {
            return zygoteSendArgsAndGetResult(openZygoteSocketIfNeeded(abi), argsForZygote);
        }
    }
```
这一步主要工作就是拼装参数，具体这些参数的含义还是不太明白；


### 3 Process.zygoteSendArgsAndGetResult
#### openZygoteSocketIfNeeded

```
    private ZygoteState openZygoteSocketIfNeeded(String abi) throws ZygoteStartFailedEx {
        Preconditions.checkState(Thread.holdsLock(mLock), "ZygoteProcess lock not held");

        if (primaryZygoteState == null || primaryZygoteState.isClosed()) {
            try {
                primaryZygoteState = ZygoteState.connect(mSocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to primary zygote", ioe);
            }
        }

        if (primaryZygoteState.matches(abi)) {
            return primaryZygoteState;
        }

        // 这里有两个socket，主的连接不上就连接第二个的，不知道为什么要有两个？
        if (secondaryZygoteState == null || secondaryZygoteState.isClosed()) {
            try {
                secondaryZygoteState = ZygoteState.connect(mSecondarySocket);
            } catch (IOException ioe) {
                throw new ZygoteStartFailedEx("Error connecting to secondary zygote", ioe);
            }
        }

        if (secondaryZygoteState.matches(abi)) {
            return secondaryZygoteState;
        }

    }
```
创建用于和Zygote通讯的Socket，并且把socket请求封装成一个ZygoteState，这里有点像OkHttp中的HttpCodec类；

#### 4 zygoteSendArgsAndGetResult
```
    private static Process.ProcessStartResult zygoteSendArgsAndGetResult(
            ZygoteState zygoteState, ArrayList<String> args)
            throws ZygoteStartFailedEx {
        try {
            final BufferedWriter writer = zygoteState.writer;
            final DataInputStream inputStream = zygoteState.inputStream;

            //写入参数之前拼装好的参数数组
            writer.write(Integer.toString(args.size()));
            writer.newLine();

            for (int i = 0; i < sz; i++) {
                String arg = args.get(i);
                writer.write(arg);
                writer.newLine();
            }
            //发送flush到Server
            writer.flush();

            Process.ProcessStartResult result = new Process.ProcessStartResult();
            //读取结果
            result.pid = inputStream.readInt();
            result.usingWrapper = inputStream.readBoolean();

            return result;
        } catch (IOException ex) {
            zygoteState.close();
            throw new ZygoteStartFailedEx(ex);
        }
    }
```
两个方法主要工作是确保Socket的连接正常，并且将数据发送给Zygote进程去处理；


### 5 ZygoteServer.runSelectLoop
Zygote进程创建完成第一个执行的方法是ZygoteInit.main方法，main方法会去调用`ZygoteServer.runSelectLoop`开启Socket并且读取请求;

```
    //ZygoteServer
    Runnable runSelectLoop(String abiList) {
        ArrayList<FileDescriptor> fds = new ArrayList<FileDescriptor>();
        ArrayList<ZygoteConnection> peers = new ArrayList<ZygoteConnection>();

        fds.add(mServerSocket.getFileDescriptor());
        peers.add(null);

        while (true) {
            StructPollfd[] pollFds = new StructPollfd[fds.size()];
            for (int i = 0; i < pollFds.length; ++i) {
                pollFds[i] = new StructPollfd();
                pollFds[i].fd = fds.get(i);
                pollFds[i].events = (short) POLLIN;
            }
           
            for (int i = pollFds.length - 1; i >= 0; --i) {
                if ((pollFds[i].revents & POLLIN) == 0) {
                    continue;
                }

                if (i == 0) {
                    //为Socket创建连接，当有新的Socket接入类这个时候会创建一个peer的连接来处理任务（在第二次循环中）
                    ZygoteConnection newPeer = acceptCommandPeer(abiList);
                    peers.add(newPeer);
                    fds.add(newPeer.getFileDesciptor());
                } else {
                    ZygoteConnection connection = peers.get(i);
                    //处理发送过来的请求
                    final Runnable command = connection.processOneCommand(this);

                    if (mIsForkChild) {
                        //当前是子进程，退出循环
                        return command;
                    } else {
                        //表示当前不是子进程，没明白里面做了什么事？
                    }
                }
            }
        }
    }
```
主要工作是不停的循环来读取Socket请求，在读取到Socket请求会通过创建ZygoteConnection的对象来进行处理；处理的一般都是fork请求，当处理成功后子进程也会从`connection.processOneCommand()`方法返回；`connection.processOneCommand()`方法返回的是ActivityThread的main方法；


### 6 ZygoteConnection.processOneCommand
```
    //ZygoteConnection
    Runnable processOneCommand(ZygoteServer zygoteServer) {
        String args[];
        Arguments parsedArgs = null;
        FileDescriptor[] descriptors;

        //读取socket中的参数
        args = readArgumentList();
        descriptors = mSocket.getAncillaryFileDescriptors();

        int pid = -1;
        FileDescriptor childPipeFd = null;
        FileDescriptor serverPipeFd = null;

        parsedArgs = new Arguments(args);

        //调用native层fork方法来创建进程
        pid = Zygote.forkAndSpecialize(parsedArgs.uid, parsedArgs.gid, parsedArgs.gids,
                parsedArgs.debugFlags, rlimits, parsedArgs.mountExternal, parsedArgs.seInfo,
                parsedArgs.niceName, fdsToClose, fdsToIgnore, parsedArgs.instructionSet,
                parsedArgs.appDataDir);

        try {
            if (pid == 0) {
                // 代表子进程
                zygoteServer.setForkChild();

                // 关闭之前循环读取请求的ServerSocket
                zygoteServer.closeServerSocket();
                IoUtils.closeQuietly(serverPipeFd);
                serverPipeFd = null;

                //对子进程做一些初始化工作
                return handleChildProc(parsedArgs, descriptors, childPipeFd);
            } else {
                // 代表Zygote进程
            }
        } finally {
            IoUtils.closeQuietly(childPipeFd);
            IoUtils.closeQuietly(serverPipeFd);
        }
    }
```
从Socket中读取请求的参数，解析参数调用native层的方法来进程fork进程的操作；在进程fork完成后会对紫禁城做一些初始化操作


### 7 Zygote.forkAndSpecialize
#### forkAndSpecialize
```
public static int forkAndSpecialize(int uid, int gid, int[] gids, int debugFlags,
      int[][] rlimits, int mountExternal, String seInfo, String niceName, int[] fdsToClose,
      String instructionSet, String appDataDir) {
    VM_HOOKS.preFork();
    int pid = nativeForkAndSpecialize(
              uid, gid, gids, debugFlags, rlimits, mountExternal, seInfo, niceName, fdsToClose,
              instructionSet, appDataDir);
    ...
    VM_HOOKS.postForkCommon(); 
    return pid;
}
```
调用native层的fork方法`nativeForkAndSpecialize`，这个方法对应的则是`com_android_internal_os_Zygote.cpp`中的`com_android_internal_os_Zygote_nativeForkAndSpecialize()`方法；

#### 8 com_android_internal_os_Zygote_nativeForkAndSpecialize
```

```
fork()采用copy on write技术，这是linux创建进程的标准方法，调用一次，返回两次，返回值有3种类型。

* 父进程中，fork返回新创建的子进程的pid;
* 子进程中，fork返回0；
* 出现错误，fork返回负数。（当进程数超过上限或者系统内存不足时会出错）
fork()的主要工作是寻找空闲的进程号pid，然后从父进程拷贝进程信息，例如数据段和代码段空间等，当然也包含拷贝fork()代码之后的要执行的代码到新的进程。

Zygote进程是所有Android进程的母体，包括system_server进程以及App进程都是由Zygote进程孵化而来。zygote利用fork()方法生成新进程，对于新进程A复用Zygote进程本身的资源，再加上新进程A相关的资源，构成新的应用进程A。何为copy on write(写时复制)？当进程A执行修改某个内存数据时（这便是on write时机），才发生缺页中断，从而分配新的内存地址空间（这便是copy操作），对于copy on write是基于内存页，而不是基于进程的。

到此App进程已完成了创建的所有工作，接下来开始新创建的App进程的工作。在前面`ZygoteConnection.processOneCommand`方法中，zygote进程执行完`forkAndSpecialize()`后，新创建的App进程便进入handleChildProc()方法，下面的操作运行在App进程。

### 9 handleChildProc
```
    private Runnable handleChildProc(Arguments parsedArgs, FileDescriptor[] descriptors,
            FileDescriptor pipeFd) {
        
        //关闭ServerSocket accept返回的Socket连接，不是很明白这里为什么还要再关闭一次
        closeSocket();

        //设置进程名
        if (parsedArgs.niceName != null) {
            Process.setArgV0(parsedArgs.niceName);
        }

        //初始化
        return ZygoteInit.zygoteInit(parsedArgs.targetSdkVersion, parsedArgs.remainingArgs,
                null /* classLoader */);
    }
```
```
    public static final Runnable zygoteInit(int targetSdkVersion, String[] argv, ClassLoader classLoader) {
        //重定向log输出，不是很明白？
        RuntimeInit.redirectLogStreams();

        //初始化工作1
        RuntimeInit.commonInit();
        //初始化工作2
        ZygoteInit.nativeZygoteInit();
        //初始化工作3
        return RuntimeInit.applicationInit(targetSdkVersion, argv, classLoader);
    }
```

#### RuntimeInit.commonInit 初始化工作1

```
   protected static final void commonInit() {

        //设置异常处理器
        Thread.setUncaughtExceptionPreHandler(new LoggingHandler());
        Thread.setDefaultUncaughtExceptionHandler(new KillApplicationHandler());

        //设置时区
        TimezoneGetter.setInstance(new TimezoneGetter() {
            @Override
            public String getId() {
                return SystemProperties.get("persist.sys.timezone");
            }
        });

       
        TimeZone.setDefault(null);

        LogManager.getLogManager().reset();
        new AndroidConfig();

        String userAgent = getDefaultUserAgent();
        System.setProperty("http.agent", userAgent);

        NetworkManagementSocketTagger.install();

        String trace = SystemProperties.get("ro.kernel.android.tracing");

        initialized = true;
    }
```
初始化工作中主要初始化来以下异常处理器，这里除了`setDefaultUncaughtExceptionHandler`还有一个`setUncaughtExceptionPreHandler`，不知道有什么用；

#### ZygoteInit.nativeZygoteInit 初始化工作2

```
static void com_android_internal_os_RuntimeInit_nativeZygoteInit(JNIEnv* env, jobject clazz)
{
    gCurRuntime->onZygoteInit(); //此处的gCurRuntime为AppRuntime，是在AndroidRuntime.cpp中定义的
}
```
```
virtual void onZygoteInit()
{
    sp proc = ProcessState::self();
    proc->startThreadPool(); //启动新binder线程
}
```
ProcessState::self()是单例模式，主要工作是调用open()打开/dev/binder驱动设备，再利用mmap()映射内核的地址空间，将Binder驱动的fd赋值ProcessState对象中的变量mDriverFD，用于交互操作。startThreadPool()是创建一个新的binder线程，不断进行`talkWithDriver`

#### RuntimeInit.applicationInit 初始化工作3

```
    protected static Runnable applicationInit(int targetSdkVersion, String[] argv,
            ClassLoader classLoader) {
      
        //不清楚有什么作用，在调用System.exit相关的配置
        nativeSetExitWithoutCleanup(true);

        // 设置gc相关的
        VMRuntime.getRuntime().setTargetHeapUtilization(0.75f);
        // 设置targetSdkVersion
        VMRuntime.getRuntime().setTargetSdkVersion(targetSdkVersion);

        final Arguments args = new Arguments(argv);


        // 通过反射找到ActivityThread.main方法
        return findStaticMain(args.startClass, args.startArgs, classLoader);
    }
```
```
    private static Runnable findStaticMain(String className, String[] argv,
            ClassLoader classLoader) {
        //反射获取类
        Class<?> cl = Class.forName(className, true, classLoader);
        //反射找到main方法
        Method m = cl.getMethod("main", new Class[] { String[].class });
        //返回一个执行main方法的Runnable
        return new MethodAndArgsCaller(m, argv);
    }
```

#### 总结
当进程创建结束以后会进行三次初始化：

* 初始化App的通用配置，如时区、崩溃处理器等
* 初始化Binder创建Binder线程来接收请求
* 初始化vm相关的虚拟机配置，寻找main方法并且返回执行main的Runnable

### 10 ZygoteInit.main
```
    public static void main(String argv[]) {
        ...
        caller = zygoteServer.runSelectLoop(abiList);

        if (caller != null) {
            caller.run();
        }
    }
```
在fork结束以后`zygoteServer.runSelectLoop`会退出循环返回main方法执行的Runnable，最后执行main方法，这里也就是ActivityThread的main方法；