#Halls
## 提纲
> * Halls架构相关
>   1. 请求声明类（装饰模式）
>   2. 拦截器interceptor（责任链模式）
>   3. 请求器Adapter（适配器模式）
> 
> * 请求声明类
> * 拦截器interceptor
> * 请求Adapter
>
> * 总结

## 1. 架构介绍
封装Halls的需求是业务要求，当时的请求过程中需要统一对请求做多道处理；并且因为之前Android一直都是在用Retrofit来实现http的请求，因此为了快速让团队小伙伴上手所以封装了一个类似Retrofit的JavaScript网络请求库——Halls；Halls主要包括了请求声明类，拦截器interceptor，请求器Adapter三个大模块;

### 1.1 请求声明类
Halls和Retrofit（声明在接口中）类似需要在一个类来定义请求的方法，Halls使用了装饰者设计模式采用JavaScript的装饰器来实现对请求参数封装；

涉及知识：JavaScript的属性描述对象，Decorator装饰器，装饰者模式

### 1.2 拦截器interceptor
Halls在请求处理层借鉴OkHttp拦截器设计，通过一系列的interceptor来实现请求的拼装以及发送；这里采用责任链设计模式实现不同的业务解耦处理，在提高了请求的灵活度的同时也使的请求的逻辑层级分明；

涉及知识：OkHttp的Interceptor和chain，责任链模式

### 1.3 请求器Adapter
考虑到JavaScript在不同的平台最后发送请求的API可能不同，如目前项目在用的Taro，在React平台可能用的就是fetch，这里参考了著名的Python请求库Request的方案，封装了一个适配不同API的请求器Adapter，确保上层在不同平台能够有很好的兼容性；

涉及知识：Python的Request库，适配器模式

## 2. 请求声明类
### 2.1 设计考虑
```
class HexinApi {
    @unlogin
    @passthrough(URL_PATH.login)
    wxLogin(param) {
    }
}
```
类的表现大致如上，实现思想：

* 通过装饰器来为请求方法来添加自定义属性，如上方`@unlogin`是为了标记该请求使用的加密方式，`@passthrough(URL_PATH.login)`则标记本次请求是通过透传来间接请求的；
* 通过方法参数来传递请求需要的参数；
* 通过装饰器替换原方法`wxLogin`的属性描述对象的`value`值为一个预先配置好的请求方法

这么做的优点在于：

* 请求的属性非常直观，通过类似注解的形式标注在上面；
* 调用时就和普通方法一样，对上层完全是透明的；
* 方便测试，Halls在检测到方法体有返回时将不会发出请求直接返回方法的内容；

### 2.2 实现原理

而请求声明类装饰的实现代码如下
```
export const unlogin = buildProxyFunc(
  {
    user_category: UNLOGIN,
    encrypt_type: ENCRYPT_AES_AND_RSA
  }
)

function buildProxyFunc(config) {
  return function proxy(target, name, desc) {
    if (desc.request == null) {
      desc.request = {}
    }
    desc.request = {
      ...desc.request,
      ...config
    }

    /**
     * 记录一下最初的原始方法
     */
    if (desc.source == null) {
      desc.source = desc.value
    }

    /**
     * 方法是静态的？不会每次都创建一个function？
     */
    function sendRequest(param,
                         url = "",
                         request = desc.request,) {

      /**
       * 可以添加测试用例
       * 当方法返回数据时将不再进行代理，而是直接执行方法
       * 但是需要注意this指向的并不是当前对象了
       */
      let sourceResult = desc.source(param)
      if (sourceResult != null && sourceResult != undefined) {
        return sourceResult
      }

      request.rawParam = param
      if (url != null && url != "") {
        request.url = url
      }
      return halls.send(request)
    }

    desc.value = sendRequest
  }
}
```

### 2.3 装饰器函数 proxy
装饰器的前提是方法或者类上的表达式运行的结果是一个方法引用，这里所有的装饰器都会通过`buildProxyFunc`函数返回一个`proxy`方法引用；当执行被装饰方法时会调用proxy方法对被调用方法进行一次装饰；

```
@decorator
 A() {
 }

// 等同于
 A() {
 }

A = decorator(A, A.name, A.descriptor) || A;
```

而这里装饰器方法decorator则是`proxy`方法（注意并不是`buildProxyFunc`）,proxy方法主要任务有两个

* 拼装请求参数配置
* 替换被装饰方法

#### 2.3.1 拼装参数

```
function proxy(target, name, desc) {
    if (desc.request == null) {
      desc.request = {}
    }
    desc.request = {
      ...desc.request,
      ...config
    }
    ...
}
```
每次装饰都会将配置的信息写入到方法的属性描述对象`descriptor.request`对象中，这个对象是我们自己创建的，为了保证在层层装饰器进行装饰（执行）过程中能够将配置信息都记录下来；

```
export function method(method = "POST") {
  return buildProxyFunc({method: method})
}
```
如装饰器`method`就是为了将请求方法配置到`request`对象中去，当然这里也是用到了闭包来保证proxy方法能够获取到外部作用域的config也就是`{method: method}`的内容

#### 2.3.2 替换原方法

当执行之前被装饰的`HexinApi.wxLogin`函数时将会被装饰方法`proxy`替换为`sendRequest`方法来提供发请求的服务
```
proxy(target, name, desc) {
    ...
    
    if (desc.source == null) {
      desc.source = desc.value
    }
    function sendRequest(param, //请求的参数
                         url = "", //请求url
                         request = desc.request, // 利用默认参数保存之前装饰拼装好的配置
                         ) {

      /**
       * 可以添加测试用例
       * 当方法返回数据时将不再进行代理，而是直接执行方法
       * 但是需要注意this指向的并不是当前对象了
       */
      let sourceResult = desc.source(param)
      if (sourceResult != null && sourceResult != undefined) {
        return sourceResult
      }

      // 当原方法被执行时参数会被记录下来保存到request对象中
      request.rawParam = param
      if (url != null && url != "") {
        request.url = url
      }
      return halls.send(request)
    }
    desc.value = sendRequest
  }
```
`proxy`方法会将原方法的`desc.value`赋值成指定的`sendRequest`，这样在执行原方法时就会直接执行`sendRequest`，`sendRequest`方法负责在被执行时将

1. Http的请求参数param
2. 请求的url
3. 之前利用其它装饰器记录的配置信息request

汇总到requst对象中，最后通过调用`halls.send(request)`来发送请求并且返回一个`Promise`；

这里考虑到前端模拟接口返回数据，因此在`sendRequest`方法执行时会尝试执行一次原方法，如果有返回结果就不会在发送请求；

## 3. 拦截器 interceptor
当`proxy`用于替换被装饰方法的`sendRequest`方法最后调用的`halls.send(request)`来发起请求

```
class Halls {
  send(request) {
    this.send_times = this.send_times + 1
    let interceptors = [
      this.keepTokenAliveInterceptor,
      new PrepareInterceptor(),
      new PassthroughInterceptor(),
      new UserInterceptor(),
      new EncryptInterceptor(),
      new RequestInterceptor()
    ]
    return new InterceptorChain(interceptors, 0, request, this).proceed(request)
  }
}
```
方法会构造一系列interceptor来组成一次请求流程，每个interceptor都负责一块任务

* KeepTokenAliveInterceptor：保证token正常
* PrepareInterceptor：参数二次处理，附加一些通用配置或默认配置
* PassthroughInterceptor：将请求格式化透传形式的请求
* UserInterceptor：附加请求时的用户信息
* EncryptInterceptor：请求加密
* RequestInterceptor：调用平台API发送请求
  
interceptor构造完成后会被传入到一个chain中，然后依次执行；伪代码如下：
```
export default class InterceptorChain {
  proceed(request) {
    let nextInterceptorChain = new InterceptorChain(nextInterceptor, request)
    let response = currentInterceptor.intercept(nextInterceptorChain)
    return response
  }
}

export default class PrepareInterceptor {
  intercept(nextInterceptorChain) {
    ...
    return nextInterceptorChain.proceed(request)
  }
```
具体逻辑可以总结为：

1. chain1.proceed执行interceptor1.intercept并传入chain2
2. interceptor1执行完成后调用chain2.proceed
3. chain2.proceed执行interceptor2.intercept并传入chain3
4. interceptor2执行完成后调用chain3.proceed

按照上述逻辑循环产生一个调用链，另外在chain调用interceptor中可以加一些监控或自检；

![interceptor](./pic/pic_interceptor.png)

## 4. 请求Adapter
为了兼容不同平台的请求方式，Halls封装了一个Adapter，用于适配不同平台的HTTP请求API
```
    REQUEST_ADAPTERS_REPO = {'taro': TaroRequestAdapter}

    this.default_requesting = {
      adapter: "taro",
      method: "POST",
      header: {
        'content-type': 'application/x-www-form-urlencoded',
      }
    }
```
adapter的配置会存放在默认的请求配置中，目前默认设置是运用Taro平台的请求方式，Adapter统一要求有一个`promise`方法来返回一个`Promise`对象来异步返回请求结果；以Taro平台为例，具体代码如下：

```
export default class TaroRequestAdapter {
  promise(request) {
    if (!request || !request.url) {
      throw new Error("request url can not be undinfed or null")
    }

    let preparedRequest = {
      url: request.url,
      data: request.param,
      method: request.method,
      header: request.header || {
        'content-type': 'application/x-www-form-urlencoded',
      }
    }

    let requestPromise = Taro.request(preparedRequest)
    return new Promise(
      (onResponse, onError) => {
        requestPromise.then(
          (response) => {
            onResponse(
              {
                code: response.statusCode,
                data: response.data,
                header: response.header,
                request: request
              }
            )
          },
          (error) => {
            onError(error)
          }
        )
      }
    );
  }
}
```
Adapter主要做了以下几件事：

1. 将之前halls请求的参数拼装成平台API需要的参数
2. 通过平台API发起请求
3. 解析平台API返回的结果转换成协议好的请求格式返回给上层