---
title: okhttp源码解析
date: 2022-06-28 22:00:00
tags: 
- 源码
---

以okhttp_3.14.x作为示例代码，是最后一个java版本，后续版本使用了kotlin封装了部分实现。

# 请求流程

## 同步请求

官网示例代码SyncPost.java

```java
public void run() throws Exception {
    String postBody = ""
        + "Releases\n"
        + "--------\n"
        + "\n"
        + " * _1.0_ May 6, 2013\n"
        + " * _1.1_ June 15, 2013\n"
        + " * _1.2_ August 11, 2013\n";

    Request request = new Request.Builder()
        .url("https://api.github.com/markdown/raw")
        .post(RequestBody.create(MEDIA_TYPE_MARKDOWN, postBody))
        .build();

    try (Response response = client.newCall(request).execute()) {
        if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

        System.out.println(response.body().string());
    }
}
```

同步请求调用了RealCall.execute()

```java
/**
 * 同步执行方法
 */
@Override public Response execute() throws IOException {
    //先加个锁， 防止一个call被执行两次
    synchronized (this) {
        if (executed) throw new IllegalStateException("Already Executed");
        executed = true;
    }
    // 开启scheduleTimeout，起一个watchDog线程，超时执行timeout()回调方法
    transmitter.timeoutEnter();
    // 给EventListener发布一个callStart的事件
    transmitter.callStart();
    try {
        client.dispatcher().executed(this);
        //核心调用逻辑在这里
        return getResponseWithInterceptorChain();
    } finally {
        //执行完成 1、runningSyncCalls中移除call 2、执行idleCallback
        client.dispatcher().finished(this);
    }
}
```

进入getResponseWithInterceptorChain()

```java
//核心方法，多个拦截器完成整个调用工作
Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    //这里可以看到，自定义的拦截器分两种， interceptors和networkInterceptors,执行顺序不一样
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
        interceptors.addAll(client.networkInterceptors());
    }
    //最后一个拦截器，最终执行http调用（前一个拦截器依赖于后一个拦截器返回，注意执行顺序）
    interceptors.add(new CallServerInterceptor(forWebSocket));
    //将拦截器数组封装为拦截器chain，递归调用，注意index
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
                                                       originalRequest, this, client.connectTimeoutMillis(),
                                                       client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
        Response response = chain.proceed(originalRequest);
        if (transmitter.isCanceled()) {
            closeQuietly(response);
            throw new IOException("Canceled");
        }
        return response;
    } catch (IOException e) {
        calledNoMoreExchanges = true;
        throw transmitter.noMoreExchanges(e);
    } finally {
        if (!calledNoMoreExchanges) {
            transmitter.noMoreExchanges(null);
        }
    }
}
```

这里把几个Interceptor封装为Interceptor.Chain，最后一个拦截器CallServerInterceptor最终执行了请求

```java
/**
   * 拦截器，实际进行了http请求
   */
@Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Exchange exchange = realChain.exchange();
    Request request = realChain.request();

    long sentRequestMillis = System.currentTimeMillis();

    exchange.writeRequestHeaders(request);

    boolean responseHeadersStarted = false;
    Response.Builder responseBuilder = null;
    /**
     * 这部分不好懂，理解大意
     * 如果有requestBody，并且请求头有Expect: 100-continue，在一次http连接内分两次传输
     * 第一次请求看服务器是否接受、第二次才发送大的requestBody。避免有的服务器不接受造成网络io浪费
     * 1、不带requestBody请求，服务器返回100说明允许requestBody
     * 2、获取httpcode 100后,将requestBody发送给服务器
     * 3、如果第一次发起请求不带Expect: 100-continue，服务器仍然返回了100。再读取一次httpcode（服务器默认按照1 2两部处理了）
     * 参考：https://blog.csdn.net/skh2015java/article/details/88723028
     */
    if (HttpMethod.permitsRequestBody(request.method()) && request.body() != null) {
        // If there's a "Expect: 100-continue" header on the request, wait for a "HTTP/1.1 100
        // Continue" response before transmitting the request body. If we don't get that, return
        // what we did get (such as a 4xx response) without ever transmitting the request body.
        if ("100-continue".equalsIgnoreCase(request.header("Expect"))) {
            exchange.flushRequest();
            responseHeadersStarted = true;
            exchange.responseHeadersStart();
            responseBuilder = exchange.readResponseHeaders(true);
        }

        if (responseBuilder == null) {
            if (request.body().isDuplex()) {
                // Prepare a duplex body so that the application can send a request body later.
                exchange.flushRequest();
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, true));
                request.body().writeTo(bufferedRequestBody);
            } else {
                // Write the request body if the "Expect: 100-continue" expectation was met.
                BufferedSink bufferedRequestBody = Okio.buffer(
                    exchange.createRequestBody(request, false));
                request.body().writeTo(bufferedRequestBody);
                bufferedRequestBody.close();
            }
        } else {
            exchange.noRequestBody();
            if (!exchange.connection().isMultiplexed()) {
                // If the "Expect: 100-continue" expectation wasn't met, prevent the HTTP/1 connection
                // from being reused. Otherwise we're still obligated to transmit the request body to
                // leave the connection in a consistent state.
                exchange.noNewExchangesOnConnection();
            }
        }
    } else {
        exchange.noRequestBody();
    }
    //requestBody已经写入exchange，将数据发送到服务器
    if (request.body() == null || !request.body().isDuplex()) {
        exchange.finishRequest();
    }
    if (!responseHeadersStarted) {
        exchange.responseHeadersStart();
    }
    /**
     * 进入这个if的条件是：
     * 1、requestHeader中有Expect: 100-continue并且服务器返回了100接受，再读取一次真正的返回
     * 2、requestHeader中无xpect: 100-continue（或者没有RequestBody），这里第一次读取返回header
     * 注意，这里只读取header，后面的exchange.openResponseBody(response)才读取responseBody
     */
    if (responseBuilder == null) {
        responseBuilder = exchange.readResponseHeaders(false);
    }

    Response response = responseBuilder
        .request(request)
        .handshake(exchange.connection().handshake())
        .sentRequestAtMillis(sentRequestMillis)
        .receivedResponseAtMillis(System.currentTimeMillis())
        .build();

    //没有主动尝试100-continue，服务器也返回了100，这时候我们再获取一次响应
    int code = response.code();
    if (code == 100) {
        // server sent a 100-continue even though we did not request one.
        // try again to read the actual response
        response = exchange.readResponseHeaders(false)
            .request(request)
            .handshake(exchange.connection().handshake())
            .sentRequestAtMillis(sentRequestMillis)
            .receivedResponseAtMillis(System.currentTimeMillis())
            .build();

        code = response.code();
    }

    exchange.responseHeadersEnd(response);

    if (forWebSocket && code == 101) {
        // Connection is upgrading, but we need to ensure interceptors see a non-null response body.
        response = response.newBuilder()
            .body(Util.EMPTY_RESPONSE)
            .build();
    } else {
        //读取返回body
        response = response.newBuilder()
            .body(exchange.openResponseBody(response))
            .build();
    }
    // 服务器返回Connection:close 将exchange对应的connection标记不可用
    if ("close".equalsIgnoreCase(response.request().header("Connection"))
        || "close".equalsIgnoreCase(response.header("Connection"))) {
        exchange.noNewExchangesOnConnection();
    }

    if ((code == 204 || code == 205) && response.body().contentLength() > 0) {
        throw new ProtocolException(
            "HTTP " + code + " had non-zero Content-Length: " + response.body().contentLength());
    }

    return response;
}
```

CallServerInterceptor中执行了http请求并读取返回。  可以看到对100-continue的处理，以及对Response分步读取。  

100-continue处理：

1. 不带requestBody请求，服务器返回100说明允许requestBody
2. 获取httpcode 100后,将requestBody发送给服务器
3. 如果第一次发起请求不带Expect: 100-continue，服务器仍然返回了100。再读取一次httpcode（服务器默认按照1 2两部处理了）

> 100-continue不影响主流程，不理解不影响看代码

对请求结果的读取分两步：  

1. 读取ResponseHeader
2. 读取ResponseBody

## 异步请求

官网示例代码AsyncGet.java

```java
public void run() throws Exception {
    Request request = new Request.Builder()
        .url("http://publicobject.com/helloworld.txt")
        .build();

    client.newCall(request).enqueue(new Callback() {
      @Override public void onFailure(Call call, IOException e) {
        e.printStackTrace();
      }

      @Override public void onResponse(Call call, Response response) throws IOException {
        try (ResponseBody responseBody = response.body()) {
          if (!response.isSuccessful()) throw new IOException("Unexpected code " + response);

          Headers responseHeaders = response.headers();
          for (int i = 0, size = responseHeaders.size(); i < size; i++) {
            System.out.println(responseHeaders.name(i) + ": " + responseHeaders.value(i));
          }

          System.out.println(responseBody.string());
        }
      }
    });
  }
```

调用了RealCall.enqueue()方法异步执行请求，请求完成后通过传入Callback的onResponse、onFailure处理成功、失败回调

```java
/**
   * 异步请求方法，传入一个callback回调处理onFailure、onResponse
   * @param responseCallback
   */
  @Override public void enqueue(Callback responseCallback) {
    synchronized (this) {
      if (executed) throw new IllegalStateException("Already Executed");
      executed = true;
    }
    // 发布一个callstart事件
    transmitter.callStart();
    //执行
    client.dispatcher().enqueue(new AsyncCall(responseCallback));
  }
```

```java
/**
   * 异步执行方法
   * @param call
   */
  void enqueue(AsyncCall call) {
    synchronized (this) {
      //加入到readyAsyncCalls队列（另一个runningAsyncCalls表示在调用中的call）
      readyAsyncCalls.add(call);

      // Mutate the AsyncCall so that it shares the AtomicInteger of an existing running call to
      // the same host.
      //相同host的call共享callsPerHost，以便根据maxRequestsPerHost限制每个host并发请求数
      if (!call.get().forWebSocket) {
        AsyncCall existingCall = findExistingCallWithHost(call.host());
        if (existingCall != null) call.reuseCallsPerHostFrom(existingCall);
      }
    }
    promoteAndExecute();
  }
```

两个enqueue很简单，将要调用Http请求的Call加入readyAsyncCalls队列，并调用promoteAndExecute();  

promoteAndExecute()方法的作用是将已提交的请求转入runningAsyncCalls，使用线程池执行，返回当前线程是否运行isRunning

```java
/**
   * 把符合条件的call，由readyAsyncCalls => runningAsyncCalls，并塞入线程池执行
   * @return 是否有运行中的call
   *
   * 调用此方法的四个地方
   * 1、修改maxRequests  修改参数后，可能有新的readyAsyncCalls符合条件可以执行了（例如从5修改为10）
   * 2、修改maxRequestsPerHost 同上
   * 3、enqueue()执行一个call
   * 4、finished() 一个runningCalls执行完成后，调用这个方法将执行下一个
   */
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }
    //执行
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
```

到这里，请求已经提交到线程池了，接下来看线程池执行的逻辑

AsyncCall继承了Runnable，线程池执行的是AsyncCall.run()方法,调用了execute()

```java
//AsyncCall.run()调用这个方法
@Override protected void execute() {
    //
    boolean signalledCallback = false;
    transmitter.timeoutEnter();
    try {
        //和同步方法一样，调用一些列拦截器执行请求
        Response response = getResponseWithInterceptorChain();
        signalledCallback = true;
        //调用成功回调，这里可能会抛出IOException
        responseCallback.onResponse(RealCall.this, response);
    } catch (IOException e) {
        if (signalledCallback) {
            //调用成功，onResponse失败
            // Do not signal the callback twice!
            Platform.get().log(INFO, "Callback failure for " + toLoggableString(), e);
        } else {
            //调用失败
            responseCallback.onFailure(RealCall.this, e);
        }
    } catch (Throwable t) {
        cancel();
        if (!signalledCallback) {
            //调用失败
            IOException canceledException = new IOException("canceled due to " + t);
            canceledException.addSuppressed(t);
            responseCallback.onFailure(RealCall.this, canceledException);
        }
        throw t;
    } finally {
        //完成调用，移除call和同步请求基本一致
        client.dispatcher().finished(this);
    }
}
```

可以看到本质还是调用了getResponseWithInterceptorChain()， 和同步方法一致，通过拦截器链完成http请求



# 组件解析

## 拦截器链

再看前面的getResponseWithInterceptorChain()方法，把自定义的拦截器，和固定的拦截器组装为RealInterceptorChain

```java
/**
   * 核心方法，多个拦截器完成整个调用工作
   * @return
   * @throws IOException
   */
  Response getResponseWithInterceptorChain() throws IOException {
    // Build a full stack of interceptors.
    //这里可以看到，自定义的拦截器分两种， interceptors和networkInterceptors,执行顺序不一样
    List<Interceptor> interceptors = new ArrayList<>();
    interceptors.addAll(client.interceptors());
    interceptors.add(new RetryAndFollowUpInterceptor(client));
    interceptors.add(new BridgeInterceptor(client.cookieJar()));
    interceptors.add(new CacheInterceptor(client.internalCache()));
    interceptors.add(new ConnectInterceptor(client));
    if (!forWebSocket) {
      interceptors.addAll(client.networkInterceptors());
    }
    //最后一个拦截器，最终执行http调用（前一个拦截器依赖于后一个拦截器返回，注意执行顺序）
    interceptors.add(new CallServerInterceptor(forWebSocket));
    //将拦截器数组封装为拦截器chain，递归调用，注意index
    Interceptor.Chain chain = new RealInterceptorChain(interceptors, transmitter, null, 0,
        originalRequest, this, client.connectTimeoutMillis(),
        client.readTimeoutMillis(), client.writeTimeoutMillis());

    boolean calledNoMoreExchanges = false;
    try {
      Response response = chain.proceed(originalRequest);
      if (transmitter.isCanceled()) {
        closeQuietly(response);
        throw new IOException("Canceled");
      }
      return response;
    } catch (IOException e) {
      calledNoMoreExchanges = true;
      throw transmitter.noMoreExchanges(e);
    } finally {
      if (!calledNoMoreExchanges) {
        transmitter.noMoreExchanges(null);
      }
    }
  }
```

{% asset_img  interceptor.png interceptor image %}

代码很简单，直接看就ok



### 两种自定义拦截器

![](https://square.github.io/okhttp/assets/images/interceptors%402x.png)

两者区别，参考官网介绍。 简单说就是Application interceptors偏重功能，重试、重定向、缓存这些网络请求都不会影响。 Network Interceptors侧重实际执行，例如如果命中缓存没有发生网络请求，则不会调用。

**Application interceptors**

- Don’t need to worry about intermediate responses like redirects and retries.
- Are always invoked once, even if the HTTP response is served from the cache.
- Observe the application’s original intent. Unconcerned with OkHttp-injected headers like `If-None-Match`.
- Permitted to short-circuit and not call `Chain.proceed()`.
- Permitted to retry and make multiple calls to `Chain.proceed()`.
- Can adjust Call timeouts using withConnectTimeout, withReadTimeout, withWriteTimeout.

**Network Interceptors**

- Able to operate on intermediate responses like redirects and retries.
- Not invoked for cached responses that short-circuit the network.
- Observe the data just as it will be transmitted over the network.
- Access to the `Connection` that carries the request.



## 连接池

#### http1.x/http2

客户端发起http请求到服务端，要经历DNS解析、TCP三次握手等一些列复杂过程才与服务器建立连接, 建立连接的过程开销很大。访问个普通网页，请求几十个甚至更多资源是很正常的。为了性能，大部分客户端对http请求做了连接复用。

**http1.x**

连接复用是通过请求头中的Connection:keep-alive实现的。服务端收到此请求头，响应请求后不会立即关闭连接，等待客户端发送下一个请求

**http2**

引入了多路复用机制，在一个http连接（或者说一个tcp连接）中可以并发发送多个http请求（对于http1.x，一个连接中多个http请求只能串行执行）

对于不同域名，一个ip:端口，也可以只维护一个http连接

>  对于http1.x chrome限制一个域名最大6个连接，多次http请求可能复用一个连接。也就是并发http请求数最大6个
>
> 对于http2，chrome在一个ip:端口下只维护1个连接，在1个连接内处理所有http请求

http2下，http2.epean.com.cn可以使用http.epean.com.cn的connection

{% asset_img  http2.png http2 image %}



> java8发布时还没有http2，java9及以上版本原生支持，java8需要引入第三方jar，并在启动参数中指定
>
> -Xbootclasspath/p:/development/repository/org/mortbay/jetty/alpn/alpn-boot/8.1.12.v20180117/alpn-boot-8.1.12.v20180117.jar

#### 获取连接

拦截器ConnectInterceptor中实现了获取连接的逻辑，很简单

```java
public final class ConnectInterceptor implements Interceptor {
  @Override public Response intercept(Chain chain) throws IOException {
    RealInterceptorChain realChain = (RealInterceptorChain) chain;
    Request request = realChain.request();
    Transmitter transmitter = realChain.transmitter();
    //get不用判断!source.exhausted()
    // We need the network to satisfy this request. Possibly for validating a conditional GET.
    boolean doExtensiveHealthChecks = !request.method().equals("GET");
    //这里获取连接
    Exchange exchange = transmitter.newExchange(chain, doExtensiveHealthChecks);
    return realChain.proceed(request, transmitter, exchange);
  }
}
```

最终调用exchangeFinder.findConnection()方法从线程池内获取到一个RealConnection。 看代码

```java
/**
   * 1、尝试使用transmitter.connection已有的连接
   * 2、尝试根据host匹配连接
   * 3、尝试匹配所有routes，匹配ip、端口，找到可用的连接（http2）
   * 4、新创建一个连接，并把连接放入connectionPool
   */
  private RealConnection findConnection(int connectTimeout, int readTimeout, int writeTimeout,
      int pingIntervalMillis, boolean connectionRetryEnabled) throws IOException {
    boolean foundPooledConnection = false;
    RealConnection result = null;
    Route selectedRoute = null;
    RealConnection releasedConnection;
    Socket toClose;
    // 第一次尝试 获取已有连接
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");
      hasStreamFailure = false; // This is a fresh attempt.

      // Attempt to use an already-allocated connection. We need to be careful here because our
      // already-allocated connection may have been restricted from creating new exchanges.
      releasedConnection = transmitter.connection;
      toClose = transmitter.connection != null && transmitter.connection.noNewExchanges
          ? transmitter.releaseConnectionNoEvents()
          : null;

      if (transmitter.connection != null) {
        // We had an already-allocated connection and it's good.
        result = transmitter.connection;
        releasedConnection = null;
      }
      //第二次尝试，获取host匹配的连接
      if (result == null) {
        // Attempt to get a connection from the pool.
        if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, null, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        } else if (nextRouteToTry != null) {
          selectedRoute = nextRouteToTry;
          nextRouteToTry = null;
        } else if (retryCurrentRoute()) {
          selectedRoute = transmitter.connection.route();
        }
      }
    }
    closeQuietly(toClose);

    if (releasedConnection != null) {
      eventListener.connectionReleased(call, releasedConnection);
    }
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
    }
    if (result != null) {
      // If we found an already-allocated or pooled connection, we're done.
      return result;
    }

    // If we need a route selection, make one. This is a blocking operation.
    boolean newRouteSelection = false;
    if (selectedRoute == null && (routeSelection == null || !routeSelection.hasNext())) {
      newRouteSelection = true;
      routeSelection = routeSelector.next();
    }

    List<Route> routes = null;
    //第三次尝试 获取routes，匹配(http2不同域名相同ip、端口)
    synchronized (connectionPool) {
      if (transmitter.isCanceled()) throw new IOException("Canceled");

      if (newRouteSelection) {
        // Now that we have a set of IP addresses, make another attempt at getting a connection from
        // the pool. This could match due to connection coalescing.
        routes = routeSelection.getAll();
        if (connectionPool.transmitterAcquirePooledConnection(
            address, transmitter, routes, false)) {
          foundPooledConnection = true;
          result = transmitter.connection;
        }
      }

      if (!foundPooledConnection) {
        if (selectedRoute == null) {
          selectedRoute = routeSelection.next();
        }
        //创建一个连接
        // Create a connection and assign it to this allocation immediately. This makes it possible
        // for an asynchronous cancel() to interrupt the handshake we're about to do.
        result = new RealConnection(connectionPool, selectedRoute);
        connectingConnection = result;
      }
    }

    // If we found a pooled connection on the 2nd time around, we're done.
    if (foundPooledConnection) {
      eventListener.connectionAcquired(call, result);
      return result;
    }

    // Do TCP + TLS handshakes. This is a blocking operation.
    result.connect(connectTimeout, readTimeout, writeTimeout, pingIntervalMillis,
        connectionRetryEnabled, call, eventListener);
    connectionPool.routeDatabase.connected(result.route());
    Socket socket = null;
    // 第四次尝试， 获取routes，再次匹配(http2不同域名相同ip、端口)
    // 这里是为了方式第三步并发获取连接时，创建了两个连接。
    // 注意这里的requireMultiplexed=true，只获取http2的多路复用连接。http1连接可以有多个，http2连接1个就够了
    synchronized (connectionPool) {
      connectingConnection = null;
      // Last attempt at connection coalescing, which only occurs if we attempted multiple
      // concurrent connections to the same host.
      //如果能获取到，说明其他线程已经创建了http2多路复用的连接，将自己创建的连接释放掉
      //获取不到，将连接put到connectionPool
      if (connectionPool.transmitterAcquirePooledConnection(address, transmitter, routes, true)) {
        // We lost the race! Close the connection we created and return the pooled connection.
        result.noNewExchanges = true;
        socket = result.socket();
        result = transmitter.connection;

        // It's possible for us to obtain a coalesced connection that is immediately unhealthy. In
        // that case we will retry the route we just successfully connected with.
        nextRouteToTry = selectedRoute;
      } else {
        connectionPool.put(result);
        transmitter.acquireConnectionNoEvents(result);
      }
    }
    closeQuietly(socket);

    eventListener.connectionAcquired(call, result);
    return result;
  }
```

代码很长，但是逻辑很清晰，分4步

1.  尝试使用transmitter.connection已有的连接
2. 尝试从连接池获取连接，根据连接的host匹配
3. 尝试从routeSelector获取routes，匹配相同ip、端口（http2）
4. 创建一个新的连接，put到线程池

注意第四步时的并发处理,http2多路复用的情况下，只创建1个连接

#### 连接清理

在新创建连接，塞入连接池时执行清理

```java
void put(RealConnection connection) {
    assert (Thread.holdsLock(this));
    //如果当前没有清理，执行
    if (!cleanupRunning) {
        cleanupRunning = true;
        executor.execute(cleanupRunnable);
    }
    //新建的连接添加到连接池中
    connections.add(connection);
}
```

cleanupRunnable，配合cleanup方法一起看

```java
private final Runnable cleanupRunnable = () -> {
    //waitNanos -1 退出清理 >0 wait(waitNanos)后继续  0 立即继续
    while (true) {
      long waitNanos = cleanup(System.nanoTime());
      if (waitNanos == -1) return;
      if (waitNanos > 0) {
        long waitMillis = waitNanos / 1000000L;
        waitNanos -= (waitMillis * 1000000L);
        synchronized (RealConnectionPool.this) {
          try {
            RealConnectionPool.this.wait(waitMillis, (int) waitNanos);
          } catch (InterruptedException ignored) {
          }
        }
      }
    }
  };
```

重点

```java
/**
   * 清除闲置的connection
   * return 下一次执行清除线程毫秒数
   * 0 清除线程执行了一次，立即执行下一次cleanup
   * >0 case1 有闲置线程，wait(keepAliveDurationNs - longestIdleDurationNs)到闲置时间满后执行
   * >0 case2 无闲置线程，有线程在使用，wait(keepAliveDurationNs)后继续执行
   * -1 没有任何线程，cleanupRunning=false 退出线程清理循环
   */
  long cleanup(long now) {
    int inUseConnectionCount = 0;
    int idleConnectionCount = 0;
    RealConnection longestIdleConnection = null;
    long longestIdleDurationNs = Long.MIN_VALUE;

    // Find either a connection to evict, or the time that the next eviction is due.
    synchronized (this) {
      for (Iterator<RealConnection> i = connections.iterator(); i.hasNext(); ) {
        RealConnection connection = i.next();

        // If the connection is in use, keep searching.
        // 连接在用, continue下一个
        if (pruneAndGetAllocationCount(connection, now) > 0) {
          inUseConnectionCount++;
          continue;
        }
        //空置连接
        idleConnectionCount++;
        //计算空闲时间，找到闲置最久的connection
        // If the connection is ready to be evicted, we're done.
        long idleDurationNs = now - connection.idleAtNanos;
        if (idleDurationNs > longestIdleDurationNs) {
          longestIdleDurationNs = idleDurationNs;
          longestIdleConnection = connection;
        }
      }
      //如果超出了最大闲置时间 or 最大闲置连接数，移除
      if (longestIdleDurationNs >= this.keepAliveDurationNs
          || idleConnectionCount > this.maxIdleConnections) {
        // We've found a connection to evict. Remove it from the list, then close it below (outside
        // of the synchronized block).
        connections.remove(longestIdleConnection);
      } else if (idleConnectionCount > 0) {
        // A connection will be ready to evict soon.
        return keepAliveDurationNs - longestIdleDurationNs;
      } else if (inUseConnectionCount > 0) {
        // All connections are in use. It'll be at least the keep alive duration 'til we run again.
        return keepAliveDurationNs;
      } else {
        // No connections, idle or in use.
        cleanupRunning = false;
        return -1;
      }
    }

    closeQuietly(longestIdleConnection.socket());

    // Cleanup again immediately.
    return 0;
  }
```

循环执行cleanup()清理超出keepAliveDurationNs,maxIdleConnections的线程，如果当前没有可以清理的闲置线程，计算出下个预计有线程空闲的时间，wait()后继续循环。直到线程池内无任何线程结束循环

{% asset_img  liucheng.png elk image %}

## Dispatcher

主要是一些变量的定义与调用方法，直接看注释很简单

```java
/**
 * 调度器，异步请求用
 */
public final class Dispatcher {
  //调度器最大并发请求数
  private int maxRequests = 64;
  //每个host最大并发数量
  private int maxRequestsPerHost = 5;
  //线程池空闲时回调（isRunning 1=>0时触发）
  private @Nullable Runnable idleCallback;
  //异步执行线程池
  private @Nullable ExecutorService executorService;
  // 已加入调度器的Call
  private final Deque<AsyncCall> readyAsyncCalls = new ArrayDeque<>();
  // 线程池内正在运行的Call
  private final Deque<AsyncCall> runningAsyncCalls = new ArrayDeque<>();
  //同步的call，统计用，例如计算线程池是否空闲isRunning
  private final Deque<RealCall> runningSyncCalls = new ArrayDeque<>();

  public Dispatcher(ExecutorService executorService) {
    this.executorService = executorService;
  }

  public Dispatcher() {
  }
  //默认初始化一个核心0，无上限的线程池（实际上限取决于maxRequests）
  public synchronized ExecutorService executorService() {
    if (executorService == null) {
      //SynchronousQueue 是无缓冲队列，不缓存Call，直接创建线程
      executorService = new ThreadPoolExecutor(0, Integer.MAX_VALUE, 60, TimeUnit.SECONDS,
          new SynchronousQueue<>(), Util.threadFactory("OkHttp Dispatcher", false));
    }
    return executorService;
  }
  //异步执行方法  省略
  void enqueue(AsyncCall call)
  /**
   * 把符合条件的call，由readyAsyncCalls => runningAsyncCalls，并塞入线程池执行
   * @return 是否有运行中的call
   *
   * 调用此方法的四个地方
   * 1、修改maxRequests  修改参数后，可能有新的readyAsyncCalls符合条件可以执行了（例如从5修改为10）
   * 2、修改maxRequestsPerHost 同上
   * 3、enqueue()执行一个call
   * 4、finished() 一个runningCalls执行完成后，调用这个方法将执行下一个
   */
  private boolean promoteAndExecute() {
    assert (!Thread.holdsLock(this));

    List<AsyncCall> executableCalls = new ArrayList<>();
    boolean isRunning;
    synchronized (this) {
      for (Iterator<AsyncCall> i = readyAsyncCalls.iterator(); i.hasNext(); ) {
        AsyncCall asyncCall = i.next();

        if (runningAsyncCalls.size() >= maxRequests) break; // Max capacity.
        if (asyncCall.callsPerHost().get() >= maxRequestsPerHost) continue; // Host max capacity.

        i.remove();
        asyncCall.callsPerHost().incrementAndGet();
        executableCalls.add(asyncCall);
        runningAsyncCalls.add(asyncCall);
      }
      isRunning = runningCallsCount() > 0;
    }
    //执行
    for (int i = 0, size = executableCalls.size(); i < size; i++) {
      AsyncCall asyncCall = executableCalls.get(i);
      asyncCall.executeOn(executorService());
    }

    return isRunning;
  }
  /**
   * call执行完毕
   * @param <T>
   */
  private <T> void finished(Deque<T> calls, T call) {
    Runnable idleCallback;
    //将call从runningAsyncCalls/runningSyncCalls队列中移除
    synchronized (this) {
      if (!calls.remove(call)) throw new AssertionError("Call wasn't in-flight!");
      idleCallback = this.idleCallback;
    }
    //把符合条件的call，由readyAsyncCalls => runningAsyncCalls，并塞入线程池执行。
    boolean isRunning = promoteAndExecute();
    //线程池空闲，执行idleCallback回调
    if (!isRunning && idleCallback != null) {
      idleCallback.run();
    }
  }
}
```

注意maxRequests和executorService()这两个参数。 默认初始化maxRequests=64，线程池数量无上限，实际请求数/线程数取两者最小值。两个参数哪个大对性能无影响。



# okhttp结构

![](https://img-blog.csdnimg.cn/20210630103500293.png)

懒得画了，网上找了个，很简单。

# 总结

## 几个主要组件

1.  RealInterceptorChain 一系列拦截器实现http请求
2. ExchangeFinder.findConnection() 从线程池内找出可用连接，http1连接和http2可复用连接处理
3. Dispatcher 异步http请求调度器

## 用到的设计模式

1. 责任链模式，RealInterceptorChain 
2. 建造者模式 OkHttpClient.build()
3. 工厂模式 EventListener.Factory.create
4. 代理模式(RealCall/Call),委派模式(RealConnectionPool/ConnectionPool)等



# 参考资料

[okhttp官网](https://square.github.io/okhttp/)

[okhttp连接池建立与复用](https://blog.csdn.net/Giagor/article/details/122019377)

[http2简介](https://juejin.cn/post/6844903577006129159)

