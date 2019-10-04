# OkHttp

## 提纲
> * OkHttp的组成部分
> * OkHttpClient
> * 异步管理器 Dispatcher
> * 拦截器 RealInterceptorChain
> * 连接池 ConnectionPool

## OkHttp的组成部分
OkHttp的组成具体包含四个部分：
* OkHttpClient：管理配置类，包含类OkHttp的所有资源的管理，尽量单例避免多实例；
* Dispatcher：异步管理器，管理请求发起以及关闭状态，包含同步请求的管理异步请求的管理；
* Interceptor & RealInterceptorChain：串联Interceptorl拦截器的串联类，组合一次请求所需要的所有步骤；
* ConnectionPool：连接池，提供socket连接的复用

## OkHttpClient
管理着OkHttp中的所有资源以及配置，下面是比较常用的配置项
```
callTimeout：整个请求的超时，是一个异步超时机制，在请求（RealCall）调用execute时开始计时，如果发生超时会异步cancel，默认是0也就是不设置超时；

connectTimeout：socket连接时的超时（socket.connect方法的参数），默认是10秒

readTimeout：流读取时的超时，用于配置source.timeout() 和 socket.setSoTimeout()，默认是10秒，可以理解成网络请求的超时

writeTimeout：流写入时的超时，用于配置sink.timeout().timeout(），默认是10秒

connectionPool：连接池，用于缓存Socket请求的，可以自己来创建来监控Sokcet连接的缓存复用情况，或者多OkHttpClient之间共用

dispatcher：请求任务管理器，用于执行异步请求的策略，可以用自己创建的，或者多OkHttpClient之间共用

interceptors：用户添加的拦截器，根据顺序是最先执行的拦截器，用于做预处理操作

networkInterceptors：网络请求的的操作，在确定来请求用的Socket连接以及参数准备发送前执行的拦截器，这部分拦截器被放在请求前（CallServerInterceptor之前）执行
```

## Dispatcher
管理请求任务用的类，当上层创建类RealCall准备发起请求时会有两种情况

* 同步请求，RealCall.execute()
* 异步请求，RealCall.enqueue()

Dispatcher管理着所有请求的发起操作，是请求的必经环节，所有请求都会被存放在这三个“区域”中
* runningSyncCalls：同步请求区
* readyAsyncCalls：异步请求准备区，存放着还未发送的异步请求，用于控制并发请求以及相同Host的并发请求量
* runningAsyncCalls：异步请求区，存放正在发送的异步请求
存放在这三个区域的目的是方便统一管理，统一Cancel等；

Dispatcher有个缺陷，在cancel请求时只能选择通过RealCall作为key来关闭或者cancelAll关闭所有请求，没有一种Tag机制；

具体两种请求的发起流程如下

### 同步请求
在发起同步请求时会进行下面操作

1. 设置callTimeout时开始记录超时时间
2. dispatcher记录下一次同步请求发起，请求被存放在**同步请求区runningSyncCalls**
3. 发起请求（getResponseWithInterceptorChain）
4. dispatcher记录下一次同步请求结束，请求从**同步请求区runningSyncCalls**中移除

### 异步请求
在发起请求时会进行下面操作

1. 将请求存放到**准备区readyAsyncCalls**
2. 检查一次当前**请求区runningAsyncCalls**，符合下面情况则将请求从**准备区**移入**请求区**
    1. **请求区runningAsyncCalls**的请求数未超过最大请求数maxRequests（默认64）
    2. **请求区runningAsyncCalls**中和当前相同Host请求数未超过maxRequestsPerHost时（默认5）
3. 将请求移入内部线程池去发起请求(getResponseWithInterceptorChain)

异步请求用到的线程池
```
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<Runnable>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
```
核心线程数为0，没有线程上线的的线程池；不是很明白？

### 配图
同步和异步请求的流程图说明

### Interceptor & RealInterceptorChain

OkHttp中的请求是由多个Interceptor组成的，其中包含

* [interceptors]：OkHttpClient中配置的拦截器
* RetryAndFollowUpInterceptor：责失败重试以及重定向
* BridgeInterceptor：负责把用户构造的请求转换为发送到服务器的请求、把服务器返回的响应转换为用户友好的响应结果
* CacheInterceptor：负责读取缓存直接返回、更新缓存
* ConnectInterceptor：负责寻找可用的Socket并且和服务器建立连接的
* [networkInterceptors]：OkHttpClient中配置的网络层拦截器
* CallServerInterceptor：负责向服务器发送请求数据、从服务器读取响应数据

这些拦截器统一通过RealInterceptorChain串联起来；它的工作原理伪代码如下：

```
    chain[0].process(){
        interceptor[0].intercept(chain[1]){
            chain[1].process(){
                interceptor[1].intercept(chain[2]){
                     chain[2].process(){
                        interceptor[2].intercept(chain[3])
                     }
                }
            }
        }
    }
```
chian[i].process -> interceptor[i].intercept(chain[i+1])

当前index的chain执行process方法调用当前index的Interceptor执行intercept方法并且传入index+1的chain；

index的Interceptor执行interpt方法结束后调用index+1的chain的process方法调用index+1的Interceptor执行intercept方法并且传入index+2的chain；

当前chain调用当前的interceptor处理拦截并传入下一个chain给interceptor用于处理完后调用；

### ConnectionPool
OkHttp中的请求在请求完成后在读取完对应Socket通道的结果并且close时会进行回收存放到ConnectionPool连接池中，在下次请求发生时会尝试先从连接池中查找是否有符合条件的Socket连接

注：StreamAllocation代表一次请求的封装对象

```
    //ConnectInterceptor阶段
    private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
        boolean foundPooledConnection = false;
        RealConnection result = null;
        Route selectedRoute = null;
        Connection releasedConnection;
        Socket toClose;
        synchronized (connectionPool) {
                if (result == null) {
                    // 尝试从连接池中找到一个符合请求地址的Socket连接
                    Internal.instance.get(connectionPool, address, this, null);
                    if (connection != null) {
                    foundPooledConnection = true;
                    result = connection;
                } else {
                    selectedRoute = route;
                }
            }
        }
        if (result != null) {
        // 说明已经找到来，则直接返回
        return result;
        }
        //没有找到可服用的连接，新创建一个长连接
        result = new RealConnection(connectionPool, selectedRoute);
        //放入连接池
        Internal.instance.put(connectionPool, result);
      }
```
如果能找到合适的Socket连接则会直接返回，否则会创建一个新的长连接；

当请求结束后，一般是在调用ResponseBody.close方法后会层层调用最后执行`StreamAllocation.deallocate`方法；

```
  private Socket deallocate(boolean noNewStreams, boolean released, boolean streamFinished) {

    if (streamFinished) {
      this.codec = null;
    }
    if (released) {
      this.released = true;
    }
    Socket socket = null;
    if (connection != null) {
      if (noNewStreams) {
        //标记这个连接不再接收新的请求
        connection.noNewStreams = true;
      }
      if (this.codec == null && (this.released || connection.noNewStreams)) {
        //删除connection关于streamAllocation自己的引用
        release(connection);
        //当前长连接没有正在处理的请求时
        if (connection.allocations.isEmpty()) {
          connection.idleAtNanos = System.nanoTime();
          //标记长连接空闲
          if (Internal.instance.connectionBecameIdle(connectionPool, connection)) {
            socket = connection.socket();
          }
        }
        connection = null;
      }
    }
    return socket;
  }
```
在请求结束后会调用`deallocate`方法，方法参数

* noNewStreams：标记当前长连接不再接收新的请求，当标记为true时长连接则不可复用
* released：标记当前StreamAllocation是否完成请求，完成请求则会释放StreamAllocation和Connection之间的引用关系
* streamFinished：标记当前请求是否结束，如果结束则释放当前持有的流

当标记当前请求已经结束并且长连接没有处理请求或不再处理后面的请求时会对长连接进行回收，回收时connectionPool连接池也会检查长连接的`noNewStreams`，如果是true则踢出缓存池；

```
  boolean connectionBecameIdle(RealConnection connection) {
    assert (Thread.holdsLock(this));
    if (connection.noNewStreams || maxIdleConnections == 0) {
      // 清理不再可用的长连接
      connections.remove(connection);
      return true;
    } else {
      // 唤醒检查线程确认等待的Socket长连接是否超出阈值
      notifyAll(); 
      return false;
    }
  }
```

ConnectionPool内部有一个循环的线程，会一直检查长链接，当ConnectionPool在标记有新的空闲长连接时都会去唤醒检查是否超过最大的空闲长连接缓存数`maxIdleConnections`,默认是5；如果没有超过，线程会阻塞到最近一个将要超时的空闲长链接，在超时的时候唤醒来清理超时的空闲长连接；空闲长连接最大空闲时间默认是5分钟；