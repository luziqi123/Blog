[TOC]

# 使用介绍

```java
    /**
     * GET请求
     * @throws IOException
     */
    private void get() throws IOException {
        OkHttpClient client = new OkHttpClient();
        String url = "https://api.github.com/repos/octokit/octokit.rb";
        Request request = new Request.Builder()
                .url(url)
                .build();
        // 异步
        client.newCall(request).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                // 请求失败
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // 请求成功
                if (response.code() == 0) {
                    GetJsonBean getJsonBean = 
                      JSON.parseObject(response.body().string(), GetJsonBean.class);
                }
            }
        });
        // 同步
        client.newCall(request).execute();
    }

    /**
     * POST请求
     * @throws IOException
     */
    private void post() throws IOException {
        String url = "https://api.github.com/repos/octokit/octokit.rb";
        OkHttpClient client = new OkHttpClient();
        // RequestBody的数据格式都要指定Content-Type，常见的有三种：
        // 普通表单
        RequestBody body = new FormBody.Builder()
                .add("键","值")
                .build();
        // JSON
        MediaType JSON = MediaType.parse("application/json; charset=utf-8");
        RequestBody jsonBody = RequestBody.create(JSON, "你的json");
        // 包含文件
        RequestBody fileBody = new MultipartBody.Builder()
                .setType(MultipartBody.FORM)
                .addFormDataPart("file", file.getName(), RequestBody.create(MediaType.parse("image/png"), file))
                .build();

        // 创建Request , 将Body设置进去
        // post
        Request postRequest = new Request.Builder()
                .url(url)
                .post(body)
                .addHeader("键" , "值")
                .build();
        // put
        Request putRequest = new Request.Builder()
                .url(url)
                .put(body)
                .addHeader("键" , "值")
                .build();
        // delete
        Request delete = new Request.Builder()
                .url(url)
                .delete(body)
                .addHeader("键" , "值")
                .build();

        // 同步
        client.newCall(postRequest).execute();
        // 异步
        client.newCall(postRequest).enqueue(new Callback() {
            @Override
            public void onFailure(Call call, IOException e) {
                // 请求成功
            }

            @Override
            public void onResponse(Call call, Response response) throws IOException {
                // 请求失败
            }
        });
    }
```

这里需要注意的是`response.body().string()` 只能调用一次 , 调用过之后IO流就会关闭 , 再次调用就会出现IO异常 . 

基本

# 挑帘窥春色----源码分析

## 创建方式

首先观摩一下他的创建方式 , 细心的你可能发现他使用了很多 `new XXX.Builder().build(); ` 这样的构造方式 , 这个厉害了, 看看`build()` 里面干了什么?

```java
    public Request build() {
      if (url == null) throw new IllegalStateException("url == null");
      return new Request(this);
    }
```

随便找了个`Request` 的build()方法 , 发现Build是Request的一个内部类 , 在Build.build()方法中 , Build创建了Request , 还顺带着检查了以下url是不是null , 并且将自己作为参数给了Request . 用肛门腺想想也知道了 , Build蕴含了Request工作时所需要的所有参数 , 反过来说Request的工作离不开Build提供的参数 . Build其实就是Request的一个构造器 , 通过对Build的一系列设置 , 最后调用build()方法 , 根据本次设置生成一个Request实例 ,如果我没有猜错 , 这个应该就是构造器模式了 (瞎猜的.... 我也不知道构造器模式是啥样 ).

想想你会怎么做?  创建一个Request实例 , 然后提供一个setBuild()方法 , 然后再创建一个Build实例 , 设置进去 , 这么做比你之前的做法有什么好处? 我认为这么做可以保证你所拿到的工作实例的设置项是没有什么问题的 , 也就是他检查了url , 将安全隐患在创造时就排除掉 . 小的修行尚浅就看这么远了.....

## 错综复杂的关系

因为C跟W有仇 , 所以找A贿赂了B , B找Z买了军火, 又找K雇了S作为杀手 , 跟X接头 , 然后合伙干死了W.....现实生活如此 ,代码的世界也是亦然 . 我们来屡屡他们是怎么作案的 .

涉及作案人员: 

- OkHttpClient	客户端
- Request   请求
- Callback   请求回调
- Response  请求结果

上面这些是主谋 , 明眼一看就知道的 , 还要下面这些人 :

- RealCall  每一个请求对应一个RealCall
- Dispatcher  维护了一个Deque双向队列的请求集合 
- Platform  字面意义是平台 , 目前不知道是干嘛的
- Interceptor  拦截器接口
- RetryAndFollowUpInterceptor  重试和跟踪拦截器
- BridgeInterceptor 网桥拦截器
- CacheInterceptor 缓存拦截器
- ConnectInterceptor  链接拦截器
- CallServerInterceptor  调用服务拦截器
- RealInterceptorChain  真正的拦截器链

行了.....不能继续跟了....涉案人员太多了....我们先搞清楚这些关系网....先从简单的入手 , 然后再看看拦截器是何方神圣 , 貌似这么帮派在本案中有着重大作用.

## 挖坟掘墓(第一夜)

```java
client.newCall(request).execute();
```

一个同步请求引发的血案 , 我先大致讲解一下案情经过 :

newCall()方法返回一个RealCall实例 , 他代表了一次请求 :

```java
/**
 * Prepares the {@code request} to be executed at some point in the future.
 */
@Override public Call newCall(Request request) {
  return new RealCall(this, request, false /* for web socket */);
}
```

### Quest 1 : execute()方法 

紧接着就调用了execute()方法执行这次请求 , 在execute()中 , 先使用一个boolean值判断自己是不是已经执行了 , 如果已经执行了再次执行就会抛出异常, 然后调用了一个captureCallStackTrace();方法 , 暂且不知道是干嘛的 , 随后 , 就调用了OkHttpClient的despatcher()方法拿到Dispatcher , 就是维护了一个请求集合的家伙 , 调用他的executed()方法将这次请求也就是本次RealCall添加进去 , 最后 , 就是getResponseWithInterceptorChain()方法了,他返回了一个Response , 在点进去的时候看到了一大堆拦截器.

```java
  @Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

### Quest 2 : captureCallStackTrace()方法

就此打住, 我们先把上面的那个captureCallStackTrace()方法搞清楚 , 再来检查拦截器帮派的结构体系及工作方式 , 从而揭开所有真相 .

先来点进去看一下captureCallStackTrace这个方法的实现, 从名字上来看他叫"捕获调用堆栈跟踪" :

```java
  private void captureCallStackTrace() {
    Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
    retryAndFollowUpInterceptor.setCallStackTrace(callStackTrace);
  }
```

### Quest 3 : Platform类

发现了一个Platform类 , 调用了他的get方法 , get方法中返回了一个Platform , 而这个Platform则是通过调用静态方法findPlatform()得到的 , 看看这个方法的实现:

```java
/** Attempt to match the host runtime to a capable Platform implementation. */
  private static Platform findPlatform() {
    Platform android = AndroidPlatform.buildIfSupported();

    if (android != null) {
      return android;
    }

    Platform jdk9 = Jdk9Platform.buildIfSupported();

    if (jdk9 != null) {
      return jdk9;
    }

    Platform jdkWithJettyBoot = JdkWithJettyBootPlatform.buildIfSupported();

    if (jdkWithJettyBoot != null) {
      return jdkWithJettyBoot;
    }

    // Probably an Oracle JDK like OpenJDK.
    return new Platform();
  }
```

### Quest 4 : buildIfSupported()方法

大致意思是:如果支持Android平台,就返回AndroidPlatform , 如果支持Jdk9Platform就返回Jdk9Platform.....那这个类型的东西又是什么? 我们随便点一个 , AndroidPlatform吧:

```java
public static Platform buildIfSupported() {
    // Attempt to find Android 2.3+ APIs.
    try {
      Class<?> sslParametersClass;
      try {
        sslParametersClass = Class.forName("com.android.org.conscrypt.SSLParametersImpl");
      } catch (ClassNotFoundException e) {
        // Older platform before being unbundled.
        sslParametersClass = Class.forName(
            "org.apache.harmony.xnet.provider.jsse.SSLParametersImpl");
      }

      OptionalMethod<Socket> setUseSessionTickets = new OptionalMethod<>(
          null, "setUseSessionTickets", boolean.class);
      OptionalMethod<Socket> setHostname = new OptionalMethod<>(
          null, "setHostname", String.class);
      OptionalMethod<Socket> getAlpnSelectedProtocol = null;
      OptionalMethod<Socket> setAlpnProtocols = null;

      // Attempt to find Android 5.0+ APIs.
      try {
        Class.forName("android.net.Network"); // Arbitrary class added in Android 5.0.
        getAlpnSelectedProtocol = new OptionalMethod<>(byte[].class, "getAlpnSelectedProtocol");
        setAlpnProtocols = new OptionalMethod<>(null, "setAlpnProtocols", byte[].class);
      } catch (ClassNotFoundException ignored) {
      }

      return new AndroidPlatform(sslParametersClass, setUseSessionTickets, setHostname,
          getAlpnSelectedProtocol, setAlpnProtocols);
    } catch (ClassNotFoundException ignored) {
      // This isn't an Android runtime.
    }

    return null;
  }
```

`Attempt to find Android 2.3+ APIs.` 尝试寻找Android 2.3 APIs , 啥意思...接着就是一堆反射 , 虽然不知道他在干嘛 , 但很牛逼的样子, 最后我只找到一个之前在监控里面见过的面孔 , Socket , 下面就是我的理解了: 通过try包裹起反射过程 , 如果反射过程中出错了 , 也就是报出ClassNotFoundException , 就返回null(This isn't an Android runtime.) , 就回去接着反射别的平台支持 , 如果反射成功 , 就拿到了一个AndroidPlatform实例 , 这个实例里面包含了sslParametersClass , setUseSessionTickets , setHostname , getAlpnSelectedProtocol , setAlpnProtocols , 这些是刚才通过反射得到的引用 . 那就看看这些变量都是干嘛的吧 , AndroidPlatform只是个酒馆儿 , 里面的英雄才是真正有战斗力的 . (默默打开百度...)

- sslParametersClass SSL(Secure Socket Layer)安全套接层 , 是网络传输协议中的是为网络通信提供安全及数据完整性的一种安全协议。在传输层对网络连接进行加密.sslParameters则是Java SE 8 中的一个API , 用于封装SSL/TLS连接的参数 . 
- setUseSessionTickets 字面意思是`设置回话许可证` 
- setHostname 字面意思是 `设置主机名称`
- getAlpnSelectedProtocol 字面意思是 `获取ALPN选择协议` ALPN则是HTTP2的相关技术了,没有深究...
- setAlpnProtocols 同上了 , 这个是设置

之所以后4个参数我只是寥寥带过 , 是因为我发现他们都同属于一个OptionalMethod类 , 注释上写着 `Duck-typing for methods: Represents a method that may or may not be present on an object.`  动态类型的方法 , 代表一个方法可能或不可能出现在当前这个对象中. 妈呀....这是啥呀.....

### Quest 5 : OptionalMethod类

 做好了锚点, 我们看看OptionalMethod是干嘛的 :

```java
  /**
   * Invokes the method on {@code target} with {@code args}. If the method does not exist or is not
   * public then {@code null} is returned. See also {@link #invokeOptionalWithoutCheckedException}.
   *
   * @throws IllegalArgumentException if the arguments are invalid
   * @throws InvocationTargetException if the invocation throws an exception
   */
  public Object invokeOptional(T target, Object... args) throws InvocationTargetException {
    Method m = getMethod(target.getClass());
    if (m == null) {
      return null;
    }
    try {
      return m.invoke(target, args);
    } catch (IllegalAccessException e) {
      return null;
    }
  }
  /**
   * Invokes the method on {@code target} with {@code args}. Throws an error if the method is not
   * supported. See also {@link #invokeWithoutCheckedException(Object, Object...)}.
   *
   * @throws IllegalArgumentException if the arguments are invalid
   * @throws InvocationTargetException if the invocation throws an exception
   */
  public Object invoke(T target, Object... args) throws InvocationTargetException {
    Method m = getMethod(target.getClass());
    if (m == null) {
      throw new AssertionError("Method " + methodName + " not supported for object " + target);
    }
    try {
      return m.invoke(target, args);
    } catch (IllegalAccessException e) {
      // Method should be public: we checked.
      AssertionError error = new AssertionError("Unexpectedly could not call: " + m);
      error.initCause(e);
      throw error;
    }
  }

```

看了一圈,  发现主要的就是一堆invokeXXX()方法 , 而其他两个有调用了上面两个 , So ....

- goto Quest 5 : OptionalMethod类的主要职责就是封装了一个反射功能 , 反射调用一个不一定有的方法 . 一个OptionalMethod对象就代表了一个方法 . 
- goto Quest 4 : AndroidPlatform.buildIfSupported()方法呢,就是通过反射 , 获取到了一个SSLParameters对象 , 这个对象是用于封装SSL/TLS连接的参数 . 并且还创建了几个可能在这个对象中会出现的方法 , 然后将这个对象和一些不靠谱的方法作为参数创建了AndroidPlatform对象.
- goto Quest 3 : 这样看来 , 在Android上使用okhttp返回的Platform肯定是AndroidPlatform了.
- goto Quest 2 : 那么在锚点2的方法中 , 则是调用了AndroidPlatform的getStackTraceForCloseable方法 , 而getStackTraceForCloseable()方法做了什么? 继续做锚点..... 

### Quest 6 : getStackTraceForCloseable()方法

```java
// 前文回顾....Quest 2 
private void captureCallStackTrace() {
    Object callStackTrace = Platform.get().getStackTraceForCloseable("response.body().close()");
	...
  }
  // AndroidPlatform
  @Override public Object getStackTraceForCloseable(String closer) {
    return closeGuard.createAndOpen(closer);
  }
// 接着调用了这个方法
Object createAndOpen(String closer) {
  if (getMethod != null) {
    try {
      Object closeGuardInstance = getMethod.invoke(null);
      openMethod.invoke(closeGuardInstance, closer);
      return closeGuardInstance;
    } catch (Exception ignored) {
    }
  }
  return null;
}
// 上面closeGuard对象就是调用这个方法赋值的
static CloseGuard get() {
  Method getMethod;
  Method openMethod;
  Method warnIfOpenMethod;

  try {
    Class<?> closeGuardClass = Class.forName("dalvik.system.CloseGuard");
    getMethod = closeGuardClass.getMethod("get");
    openMethod = closeGuardClass.getMethod("open", String.class);
    warnIfOpenMethod = closeGuardClass.getMethod("warnIfOpen");
  } catch (Exception ignored) {
    getMethod = null;
    openMethod = null;
    warnIfOpenMethod = null;
  }
  return new CloseGuard(getMethod, openMethod, warnIfOpenMethod);
}
```

让人兴奋的是查看接口实现情况时 , 只出来了AndroidPlatform和Platform两个地方有这个方法, 所以我们的判断是对的 . 

> 旁白: 再看这厮 , 传了一个`response.body().close()` 的字符串进去 , 而字符串的意思就是请求结构的body关闭 , 唉哟~我之前打log调用了一次request.body().string();紧接着解析Json又调用了一次居然报错了 , 查了之后发现这个方法只能调用一次 ,调用过之后IO流就关闭了 , 是不是跟这里有这千丝万缕的关系?? (看个源码连蒙带猜的也没谁了...)

上面代码列出了整个调用流程 , 可以看到核心是调用了closeGuard对象的createAndOpen()方法 , closeGuard是一个CloseGuard类型的对象 , 跟Platform一样 , 也是一个酒馆 , 就是调用上面的get()方法 , 通过反射拿到了一个"dalvik.system.CloseGuard"对象以及一些方法 , 然后作为参数创建了这个对象 . dalvik.system.CloseGuard是GC相关的类 , 经常在操作数据库或使用流的时候 , 如果不关闭 , 报错信息中可能就会出现他的身影了 , 他的注释上说明 , 他是一个应该用明确方法清理资源的机制 , 所以他通过反射拿到这个对象 , 显式的调用他的close和open方法 . 

那么我们屡一下 , 我们知道Quest 2中的方法captureCallStackTrace是在你发送请求的时候调用的 , 为什么在这个开业大吉 皆大欢喜的日子里 , 他却传了一个`response.body().close()` ?? 这不是关闭么~ 如果你够细心会发现这里 `closeGuardClass.getMethod("open", String.class);` 其实他是调用了open方法 , 但是调用打开的同时你需要将其对应的关闭的方法也传进去 , 这样是不是就抑制了资源访问关闭不及时的情况??(还是瞎猜的.....但感觉八九不离十) . 

我们来看一下这里调用到的dalvik.system.CloseGuard的三个方法的源码:

```java
    /**
     * Returns a CloseGuard instance. If CloseGuard is enabled, {@code
     * #open(String)} can be used to set up the instance to warn on
     * failure to close. If CloseGuard is disabled, a non-null no-op
     * instance is returned.
     */
    public static CloseGuard get() {
        if (!ENABLED) {
            return NOOP;
        }
        return new CloseGuard();
    }



    /**
     * If CloseGuard is enabled, {@code open} initializes the instance
     * with a warning that the caller should have explicitly called the
     * {@code closer} method instead of relying on finalization.
     *
     * @param closer non-null name of explicit termination method
     * @throws NullPointerException if closer is null, regardless of
     * whether or not CloseGuard is enabled
     */
    public void open(String closer) {
        // always perform the check for valid API usage...
        if (closer == null) {
            throw new NullPointerException("closer == null");
        }
        // ...but avoid allocating an allocationSite if disabled
        if (this == NOOP || !ENABLED) {
            return;
        }
        String message = "Explicit termination method '" + closer + "' not called";
        allocationSite = new Throwable(message);
    }



    /**
     * Marks this CloseGuard instance as closed to avoid warnings on
     * finalization.
     */
    public void close() {
        allocationSite = null;
    }
```

很简单了 , get()是一个单例的实例获取  , open()我们还真的猜对了 , 注释上说明了参数的意义`显式终止方法的非空名字` , 至于close() , 就是简单的置空 , 具体方法内容我们不在细究了 , 不然没完没了了. 

> 不过真想吐槽一句这个open()方法...作为一个机制级别的类 , 还真是TM简陋....就是把你传进来的那个关闭用的方法名用字符串拼接成了一句异常信息......等你没有关闭这个资源的时候抛这个异常提醒你 , 喂~你么有调用这个方法关闭~ 醉了....(这也是我猜的)

好了 , 至此 , 我们再来总结一下锚点(含上集回顾) : 

- goto Quest 5 : OptionalMethod类的主要职责就是封装了一个反射功能 , 反射调用一个不一定有的方法 . 一个OptionalMethod对象就代表了一个方法 . 虽然这里反射用的挺好 , 但明显是运行时反射 , 为了性能考虑还是慎用.
- goto Quest 4 : 所以AndroidPlatform.buildIfSupported()方法呢,就是通过反射 , 获取到了一个SSLParameters对象 , 这个对象是用于封装SSL/TLS连接的参数 . 并且还创建了几个可能在这个对象中会出现的方法 , 然后将这个对象和一些不靠谱的方法作为参数创建了AndroidPlatform对象.
- goto Quest 3 : 这样看来 , 在Android上使用okhttp返回的Platform肯定是AndroidPlatform了.
- goto Quest 2 : 那么在锚点2的方法中 , 则是调用了AndroidPlatform的getStackTraceForCloseable方法 .

- goto Quest 6 : 起初不知道getStackTraceForCloseable()方法中的一个变量`callStackTrace` 调用堆栈跟踪是个什么意思 , 现在感觉他应该就是调用了一下`dalvik.system.CloseGuard` 获取单例 , CloseGuard就是一个管理资源使用开闭的机制 , 我们给他起个外号儿叫"有头有尾" , 然后添加到了一个拦截器的某个子类中 , 拦截器我们后面再说 . 然后顺便还告诉了这个"有头有尾"的"尾"怎么调用 . 
- goto Quest 1 : 好了 , 这么多篇幅才搞懂了一个captureCallStackTrace();方法......总而言之就是一句话 , 标记了对资源访问的开始 , 并且声明了怎么结束这次访问 ,还真是费劲...

## 挖坟掘墓(第二夜)

好了 , 继续看execute()方法 . 

```java
@Override public Response execute() throws IOException {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    captureCallStackTrace();
    try {
      client.dispatcher().executed(this);
      Response result = getResponseWithInterceptorChain();
      if (result == null) throw new IOException("Canceled");
      return result;
    } finally {
      client.dispatcher().finished(this);
    }
  }
```

`client.dispatcher().executed(this);` 就是将这次请求添加到了Dispatcher中 , Dispatcher中可热闹了 , 简直就是个物流分拨中心 , "快递公司已揽件~" 就是被他拿走的 , 我们看看他长什么样:

### Quest 2-0 : Dispatcher是什么?

```java
private int maxRequests = 64;
  private int maxRequestsPerHost = 5;
  private Runnable idleCallback;

  /** Executes calls. Created lazily. */
  private ExecutorService executorService;

  /** Ready async calls in the order they'll be run. */
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();

  /** Running asynchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();

  /** Running synchronous calls. Includes canceled calls that haven't finished yet. */
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();
```

只看参数吧 :

- maxRequests  最多只接收64个件.
- maxRequestsPerHost  我觉得这个是给socket使用的 , 话说他不是支持socket链接吗?
- executorService 执行calls的服务.
- readyAsyncCalls 准备好的异步请求队列 ,如果是异步请求 , 会先判断当前请求数量和socket链接数是否大于最大值 , 如果大于则先添加到这个队列中.
- runningAsyncCalls 正在运行的异步请求队列 , 如果是异步请求 , 则将请求添加到该队列.
- runningSyncCalls 正在运行的同步请求队列 , 如果是同步请求 , 则将该请求添加到该队列.

### Quest 2-1 : getResponseWithInterceptorChain()方法干了什么?

好了 , 终于重头戏了:`Response result = getResponseWithInterceptorChain();` 这个方法最终返回了一个请求响应,看看他做了什么操作:

```java
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(retryAndFollowUpInterceptor);
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    interceptors.add(new CallServerInterceptor(forWebSocket));

    Interceptor.Chain chain = new RealInterceptorChain(
        interceptors, null, null, null, 0, originalRequest);
    return chain.proceed(originalRequest);
  }
```

有一个`interceptors` 集合存放了各种拦截器 , 在添加完稀奇古怪的拦截器后 , 由`Interceptor.Chain` 做了`proceed()` 操作 , 并将刚才的集合传了进去 .拦截器是什么?

### Quest 2-2 : Interceptor拦截器是什么?

```java
/**
 * Observes, modifies, and potentially short-circuits requests going out and the corresponding
 * responses coming back in. Typically interceptors add, remove, or transform headers on the request
 * or response.
 */
public interface Interceptor {
  Response intercept(Chain chain) throws IOException;

  interface Chain {
    Request request();

    Response proceed(Request request) throws IOException;

    Connection connection();
  }
}
```

随便点开了一个实体类, 也是不知从何入手 , 那就看这个方法的核心吧 , 该方法最后将拦截器集合作为参数 , 创造了`RealInterceptorChain` 对象 , 并调用了该对象的`proceed()` 方法 .

### Quest 2-3 : RealInterceptorChain是什么 ? proceed()方法又做了什么?

下面就是`RealInterceptorChain` 的构造方法和 `proceed()` 方法的代码 : 

```java
// 构造函数
public RealInterceptorChain(List<Interceptor> interceptors, 
                            StreamAllocation streamAllocation,
                            HttpCodec httpCodec, 
                            Connection connection, 
                            int index, 
                            Request request) {
        this.interceptors = interceptors;
        this.connection = connection;
        this.streamAllocation = streamAllocation;
        this.httpCodec = httpCodec;
        this.index = index;
        this.request = request;
  }  

  // 主要方法
  @Override 
  public Response proceed(Request request) throws IOException {
    return proceed(request, streamAllocation, httpCodec, connection);
  }

  public Response proceed(Request request, StreamAllocation streamAllocation, HttpCodec httpCodec,
      Connection connection) throws IOException {
    if (index >= interceptors.size()) throw new AssertionError();

    calls++;

    // If we already have a stream, confirm that the incoming request will use it.
    if (this.httpCodec != null && !sameConnection(request.url())) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must retain the same host and port");
    }

    // If we already have a stream, confirm that this is the only call to chain.proceed().
    if (this.httpCodec != null && calls > 1) {
      throw new IllegalStateException("network interceptor " + interceptors.get(index - 1)
          + " must call proceed() exactly once");
    }

    // Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);

    // Confirm that the next interceptor made its required call to chain.proceed().
    if (httpCodec != null && index + 1 < interceptors.size() && next.calls != 1) {
      throw new IllegalStateException("network interceptor " + interceptor
          + " must call proceed() exactly once");
    }

    // Confirm that the intercepted response isn't null.
    if (response == null) {
      throw new NullPointerException("interceptor " + interceptor + " returned null");
    }

    return response;
  }
```

重点在这里:

首先这个类实现了`Interceptor.Chain` 接口 , 构造函数为一个赋值的过程 , 除了`StreamAllocation` , `HttpCodec` , `Connection` 其他几个我们都认识 , 分别是拦截器集合 , 请求体 , 和一个index . 结合`proceed()` 方法 , index代表了使用集合中的第几个拦截器 ,  从这里来看这个类应该又是一个载体了 , 联系着他的类名 : 真正的拦截器链 , 他应该是一个拦截器链条的管理器 . 我们接着看 `proceed()` 方法 .

```java
	// Call the next interceptor in the chain.
    RealInterceptorChain next = new RealInterceptorChain(
        interceptors, streamAllocation, httpCodec, connection, index + 1, request);
    Interceptor interceptor = interceptors.get(index);
    Response response = interceptor.intercept(next);
```

从上面这段开始应该才是该方法的主体 , 其他的都是在处理特殊情况的异常 .这里有点饶了.... `RealInterceptorChain` 又创建了一个新的实例next , 参数跟自己一样 , 唯有index + 1 , 然后通过index拿到第一个拦截器 , 然后调用了第一个拦截器的 `intercept(next)` 方法 , 并将next传进去 . 最终获得了一个Response , 也就是说Response是拦截器返回的?

### Quest 2-4 : Interceptor.intercept()方法做了什么?

我们已经知道了他当前拿到的是第一个拦截器 , 所以这个拦截器应该是 `BridgeInterceptor` 看看  `BridgeInterceptor.intercept(next)` 里面具体干了什么 :

```java
  @Override public Response intercept(Chain chain) throws IOException {
    Request userRequest = chain.request();
    Request.Builder requestBuilder = userRequest.newBuilder();

    RequestBody body = userRequest.body();
    if (body != null) {
      MediaType contentType = body.contentType();
      if (contentType != null) {
        requestBuilder.header("Content-Type", contentType.toString());
      }

      long contentLength = body.contentLength();
      if (contentLength != -1) {
        requestBuilder.header("Content-Length", Long.toString(contentLength));
        requestBuilder.removeHeader("Transfer-Encoding");
      } else {
        requestBuilder.header("Transfer-Encoding", "chunked");
        requestBuilder.removeHeader("Content-Length");
      }
    }

    if (userRequest.header("Host") == null) {
      requestBuilder.header("Host", hostHeader(userRequest.url(), false));
    }

    if (userRequest.header("Connection") == null) {
      requestBuilder.header("Connection", "Keep-Alive");
    }

    // If we add an "Accept-Encoding: gzip" header field we're responsible for also decompressing
    // the transfer stream.
    boolean transparentGzip = false;
    if (userRequest.header("Accept-Encoding") == null && userRequest.header("Range") == null) {
      transparentGzip = true;
      requestBuilder.header("Accept-Encoding", "gzip");
    }

    List<Cookie> cookies = cookieJar.loadForRequest(userRequest.url());
    if (!cookies.isEmpty()) {
      requestBuilder.header("Cookie", cookieHeader(cookies));
    }

    if (userRequest.header("User-Agent") == null) {
      requestBuilder.header("User-Agent", Version.userAgent());
    }

    Response networkResponse = chain.proceed(requestBuilder.build());

    HttpHeaders.receiveHeaders(cookieJar, userRequest.url(), networkResponse.headers());

    Response.Builder responseBuilder = networkResponse.newBuilder()
        .request(userRequest);

    if (transparentGzip
        && "gzip".equalsIgnoreCase(networkResponse.header("Content-Encoding"))
        && HttpHeaders.hasBody(networkResponse)) {
      GzipSource responseBody = new GzipSource(networkResponse.body().source());
      Headers strippedHeaders = networkResponse.headers().newBuilder()
          .removeAll("Content-Encoding")
          .removeAll("Content-Length")
          .build();
      responseBuilder.headers(strippedHeaders);
      responseBuilder.body(new RealResponseBody(strippedHeaders, Okio.buffer(responseBody)));
    }

    return responseBuilder.build();
  }
```

可以看到他又拿到了Request请求体的Builder一通造 , 主要就是设置一些请求必要的字段 , 创建他的时候还传进去一个cookie相关的东西,  这里就不看他了 . 

待一切处理完毕 , 执行了这么一句 ,  ` Response networkResponse = chain.proceed(requestBuilder.build());` 看到这里 , 已经可以确定他使用了拦截器模式 .我们通过一段伪代码 , 看看他的工作流程 :

```java
// 拦截器接口
interface Interceptor{
  Response intercept(Chain chain);
  interface Chain{
    proceed();
    ...
  }
}

// 拦截器链
class RealInterceptorChain implements Interceptor.Chain{
  Interceptor[] interceptors = 
    InterceptorA , 
    InterceptorB , 
    InterceptorC;
  
  int index;
  
  RealInterceptorChain(int index){
    this.index = index;
  }
  
  @Override
  public Response proceed(){
    Interceptor.Chain chain = new RealInterceptorChain(index + 1);
    interceptors[index].intercept(chain);
  }
}

//拦截器实现类A ~ D
class InterceptorA{
  Response intercept(Interceptor.Chain chain){
    ...
    Response r = chain.proceed();
  }
}
.....
class InterceptorC{
  Response intercept(Interceptor.Chain chain){
    ...
    Response r = new Response();
    return r;
  }
}
```

还记得前面传进来的那个拦截器集合么 ? 他在依次执行元素的同一个方法 , 我们已经知道第一个拦截器负责处理请求的常规设置 , 那么接下来我们就要分析一下其他拦截器负责什么工作了 , 但在此之前 , 我们需要整理一下前面的东西 :

- goto : Quest 2-0 : 首先将请求添加到了请求管理类中 , 根据是否同步添加到不同的队列 . 
- goto : Quest 2-1 : 看下面 , 和下面的一起说.
- goto : Quest 2-2 : 看下面 , 和下面的一起说.
- goto : Quest 2-3 : 看下面 , 和下面的一起说.
- goto : Quest 2-4 : 其实上面所产生的问题 , 都是关于拦截器模式的一些操作 , 知道了拦截器模式 , 这些问题也就不是问题了 , getResponseWithInterceptorChain()方法创建了一堆拦截器 , 每个拦截器都对这次请求做了相应的操作 , 这样做的目的还要等看完其他拦截器的具体代码才知道.

### Quest 2-5 : 不同拦截器的具体工作

上面已经看了 `BridgeInterceptor` 接着我们来看后面几个拦截器.

- CacheInterceptor 从名字也知道是对缓存的处理了. 这里先不看.


- ConnectInterceptor 连接拦截器 , 设置请求时候会用到的 : 请求编码和响应解码对象 /  流分配对象 / 真正使用到的链接. 这里的真正链接类是 `RealConnection` 里面有一个Socket链接 .

  ```java
    @Override public Response intercept(Chain chain) throws IOException {
      RealInterceptorChain realChain = (RealInterceptorChain) chain;
      Request request = realChain.request();
      StreamAllocation streamAllocation = realChain.streamAllocation();

      // We need the network to satisfy this request. Possibly for validating a conditional GET.
      boolean doExtensiveHealthChecks = !request.method().equals("GET");
      HttpCodec httpCodec = streamAllocation.newStream(client, doExtensiveHealthChecks);
      RealConnection connection = streamAllocation.connection();

      return realChain.proceed(request, streamAllocation, httpCodec, connection);
    }
  ```

  ​

- CallServerInterceptor 这是链表中的最后一个拦截器 , 如果说前面几个连接器只是对发送一次请求做出准备的话 , 那么他就是真正开始发送请求的拦截器了.

  到了这里 , 已经开始真正的去发送一个网络请求了 , 我们附上 `CallServerInterceptor ` 的代码吧 : 

  ```java
  @Override public Response intercept(Chain chain) throws IOException {
      HttpCodec httpCodec = ((RealInterceptorChain) chain).httpStream();
      StreamAllocation streamAllocation = ((RealInterceptorChain) chain).streamAllocation();
      Request request = chain.request();

      long sentRequestMillis = System.currentTimeMillis();
      httpCodec.writeRequestHeaders(request);

      Response.Builder responseBuilder = null;
      if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
        // Continue" response before transmitting the request body. If we don't get that, return what
        // we did get (such as a 4xx response) without ever transmitting the request body.
        if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
          httpCodec.flushRequest();
          responseBuilder = httpCodec.readResponseHeaders(true);
        }

        // Write the request body, unless an "Expect: 100-continue" expectation failed.
        if (responseBuilder == null) {
          Sink requestBodyOut = httpCodec.createRequestBody(request, request.body().contentLength());
          BufferedSink bufferedRequestBody = Okio.buffer(requestBodyOut);
          request.body().writeTo(bufferedRequestBody);
          bufferedRequestBody.close();
        }
      }

      httpCodec.finishRequest();

      if (responseBuilder == null) {
        responseBuilder = httpCodec.readResponseHeaders(false);
      }

      Response response = responseBuilder
          .request(request)
          .handshake(streamAllocation.connection().handshake())
          .sentRequestAtMillis(sentRequestMillis)
          .receivedResponseAtMillis(System.currentTimeMillis())
          .build();

      int code = response.code();
      if (forWebSocket && code == 101) {
        // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
        response = response.newBuilder()
            .body(Util.EMPTY_RESPONSE)
            .build();
      } else {
        response = response.newBuilder()
            .body(httpCodec.openResponseBody(response))
            .build();
      }

      if ("close".equalsIgnoreCase(response.request().header("Connection"))
          || "close".equalsIgnoreCase(response.header("Connection"))) {
        streamAllocation.noNewStreams();
      }

      if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
        throw new ProtocolException(
            "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
      }

      return response;
    }
  ```

  ​

代码步骤:

- 拿到HttpCodec , 子类为Http1Codec / Http2Codec , 他是真正发起请求工作的类 . 由StreamAllocation创建.
- 拿到StreamAllocation对象 , 他是流分配器 , 主要负责链接服务器相关的工作. 在这个类中, 会创建一个HttpCodec ,并进行请求链接分配等一些操作 . 
- 拿到Request请求体.
- 通过Request对象的设置 , 设置HttpCodec对象的请求头, 包括读/写超时时间 , 是否有body体 . 
- 将Request请求写入socket流中 , 如果有请求体 , 则将请求刷新到socket流中 . 如果没有请求体 , 则创建一个请求体 , 刷新到socket流中.
- 调用 `httpCodec.finishRequest();` 完成请求.
- 至此 , 一个请求已经完成了 , 后续则是根据返回的一些参数创建Response对象了.

# 填坑埋土----分析总结

总的来说有不完善的地方 , 比如:

- 只分析了同步请求 , 对异步请求没有分析.
- 没有分析缓存机制.

对此后续会慢慢补上 . 前面的分析比较凌乱 , 这里再以概述的方式从头到尾的屡一遍:

1. 从调用同步请求 `RealCall.execute()` 开始发起一个同步请求.
2. `captureCallStackTrace();` 通过一系列反射 , 根据不同平台获取资源管理器 , 标记一个资源访问的开始.
3. 将请求体添加到 `Dispatcher` 请求调度器中 , 这个调度器根据不同的请求类型 , 将请求添加至相应队列(同步队列和异步队列) .
4. 最后调用 `getResponseWithInterceptorChain();` 方法 , 创建一个拦截器链 , 并开始对链中的拦截器进行链式调用.
5. 首先是 `BridgeInterceptor` 对请求体进行常规设置 , 比如Cookie / "Content-Type" / "Content-Length" 之类参数.
6. 然后是 `CacheInterceptor` 开始处理缓存.
7. 之后为 `ConnectInterceptor` 开始对请求时所需要的链接进行准备. 包括: HttpCodec  /  StreamAllocation / RealConnection .  `RealConnection` 就是真正链接服务器的类了 , 里面有一个Socket链接 .
8. 最后是 `CallServerInterceptor` 真正开始这次请求. 后面是 `CallServerInterceptor.intercept()` 方法中的步骤.
9. 拿到HttpCodec , 子类为Http1Codec / Http2Codec , 他是发起请求的类 . 在ConnectInterceptor拦截器中 , 由StreamAllocation创建并配置.
10. 拿到StreamAllocation对象 , 他是流分配器 , 主要负责链接服务器相关的工作. 
11. 拿到Request请求体.
12. 通过Request对象的设置 , 设置HttpCodec对象的请求头, 包括读/写超时时间 , 是否有body体 . 
13. 将Request请求写入socket流中 , 如果有请求体 , 则将请求刷新到socket流中 . 如果没有请求体 , 则创建一个请求体 , 刷新到socket流中.
14. 调用 `httpCodec.finishRequest();` 完成请求.
15. 至此 , 一个请求已经完成了 , 后续则是根据返回的一些参数创建Response对象了.