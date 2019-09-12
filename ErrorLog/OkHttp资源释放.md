# 记录OkHttp资源释放导致的崩溃

## 崩溃现象

崩溃现象是项目运行一段时间后会出现随机崩溃，没有固定的复现条件，崩溃Logcat是报在native层的。

```
A/DEBUG: ABI: 'arm64'
A/DEBUG: pid: 30784, tid: 30919, name: RxCachedThreadS  >>> pacage name <<<
A/DEBUG: signal 6 (SIGABRT), code -6 (SI_TKILL), fault addr --------
A/DEBUG: Abort message: 'FORTIFY: FD_SET: file descriptor >= FD_SETSIZE'
A/DEBUG:     x0   0000000000000000  x1   00000000000078c7  x2   0000000000000006
```

从崩溃信息看主要内容是`FORTIFY: FD_SET: file descriptor >= FD_SETSIZE`，在网上大致查询了一下是因为fd文件描述符没有释放导致的，一般都是File或者Socket相关的资源文件没有及时close导致的。

## 猜测原因

OkHttp（v3.8.0）底层是使用Socket进行Http请求的，因此猜测是OkHttp的链接池中的长连接没有释放导致的问题。

于是验证，写了一个循环发送100次请求，来看是否是OkHttp请求导致的。

### Step1 记录发送请求前的fd
在发送请求前查看自己项目进程下的fd数量（需要root的手机或者模拟器）
```
mido:/ # ps -ef | grep you.package.name
u0_a85       16130  1542 0 11:13:01 ?     00:00:45 you.package.name
```
找到进程号是`16130`，到对应目录查看发送请求前的fd数量
```
mido:/ # cd /proc/16130/fd

mido:/proc/16130/fd # ls -l | wc -l
181
```
发现当前项目对应进程的fd持有数量大约是181个

### Step2 记录发送请求后的fd

```
mido:/proc/16130/fd # ls -l | wc -l
430
```
发现确实fd数量骤增，而且会存在很长一段时间，而此时短时间重复发送1000次请求左右(不同手机不一样)，应用就会发生崩溃

## 修改
查看了一下是因为项目中封装了的自定义的Converter中读取数据时没有对ResponseBody进行释放（close）导致的。

原代码如下
```
class FormObjectResponseConverter<T>(
        val gson: Gson,
        val responseType: Type,
        val adapter: TypeAdapter<T>,
        val annotations: Array<Annotation>
) : Converter<ResponseBody, T> {
    override fun convert(responseBody: ResponseBody): T? {
        if (isSpecilRequest(annotations)) {
            ...
        } else {
            return if (String::class.java == responseType) {
                // 调用string方法时responseBody会帮我们close
                responseBody.string() as T
            } else {
                val jsonReader = gson.newJsonReader(responseBody.charStream())
                // 读取完后没有调用 responseBody.close()
                adapter.read(jsonReader)
            }
        }
    }
}
```
这里就是我将responseBody的数据读取并且反序列化成对应对象时没有及时调用`responseBody.close()`导致的；而上述请求`responseBody.string()`方法内部就会帮我们自动close因此不会有问题。

修改方法，最简单的就是全局套一个`try finally`块，在finally块中加上`responseBody.close()`,当然也可以用Kotlin封装的`use`方法

## 其它情况

另外需要注意的是这种情况，我们会在自定义的Interceptor中嵌套封装原ResponseBody做一些附加操作（如加解密）

```
class EncryptResponseBody(val sourceResponseBody: ResponseBody) : ResponseBody() {
    private val sourceResponseContent = Buffer()
    private var isReadData = false
    ...
    override fun source(): BufferedSource {
        if (!isReadData) {
            isReadData = true
            val responseContent = sourceResponseBody.string()
            ... //读取解密数据然后做返回
            sourceResponseContent.write(responseJsonObj.toString().toByteArray())
        }
        return sourceResponseContent
    }
}
```
如上，也是项目中封装的一个统一对请求内容解密的ResponseBody，在调用string()或其他方式读取数据时会对加密的内容先进行一次解密然后再返回，此时也很容易忽略对sourceResponseBody的释放。解决方案是重写自定义ResponseBody类的close方法，在方法内去关闭源ResponseBody;

```
class EncryptResponseBody(val sourceResponseBody: ResponseBody) : ResponseBody() {
    private val sourceResponseContent = Buffer()
    private var isReadData = false
    ...
    override fun source(): BufferedSource {
        if (!isReadData) {
            isReadData = true
            val responseContent = sourceResponseBody.string()
            ... //读取解密数据然后返回
            sourceResponseContent.write(responseJsonObj.toString().toByteArray())
        }
        return sourceResponseContent
    }

    //重写close方法并且回收资源
    override fun close() {
        super.close()
        sourceResponseBody.close()
    }
}
```

## OkHttp(3.8.0)连接资源释放机制




