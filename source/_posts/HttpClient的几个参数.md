---
title: HttpClient的几个参数
date: 2020-05-09 14:05:35
tags: Java
---

# Apache HttpClient的几个重要参数

最近在业务中发现一些性能问题，主要是和HttpClient有关的，刚好在这里总结一下。

针对版本号如下：

```xml
<dependency>
    <groupId>org.apache.httpcomponents</groupId>
    <artifactId>httpclient</artifactId>
    <version>4.5.3</version>
</dependency>
```

## maxConnPerRoute 与 maxConnTotal

正如大家所知，HttpClient与目的主机进行HTTP通信，必须要进行TCP连接。而因为三次握手、四次挥手、滑动窗口机制，连接的建立和释放是颇为耗时的一件事情，因此，HttpClient实现了连接池。以便在请求同一个`host:port`时，会复用已经建立的连接。

那么显然，这个复用的连接针对同一个`host:port`才有意义，这就是`maxConnPerRoute`的含义，即允许同一个`host:port`最多有几个活跃连接。当设置为2时，意味着同时最多有两个到`host:port`的连接，如果有空闲连接，本次请求会使用已经建立的空闲连接，但如果没有，就会进入等待队列，等待到该`host:port`的连接可用。

而maxConnTotal更直接了，意思就是总共允许HttpClient建立多少个连接，超出以后所有的连接都会进入等待队列。

在`CloseableHttpClient`的实现类`InternalHttpClient`中，连接的管理是依靠`HttpClientConnectionManager`的实现类，默认为`PoolingHttpClientConnectionManager`。

具体的代码，从`httpclient.execute(...)`可以追溯到：

```java
@Override
protected CloseableHttpResponse doExecute(
        final HttpHost target,
        final HttpRequest request,
        final HttpContext context) throws IOException, ClientProtocolException{
  // 上半部分省略，try catch省略  
  final HttpRoute route = determineRoute(target, wrapper, localcontext);
  return this.execChain.execute(route, wrapper, localcontext, execAware);
}
```

继续追寻`execChain.execute(...)`，最终会进入`MainClientExec#exec(...)`中，有关建立连接的语句如下：

```java
final ConnectionRequest connRequest = connManager.requestConnection(route, userToken);
final HttpClientConnection managedConn = connRequest.get(timeout > 0 ? timeout : 0, TimeUnit.MILLISECONDS);
```

而这个`ConnectionRequest`，则是定义于`PoolingHttpClientConnectionManager`中的一个匿名类，如下：

```java
@Override
public ConnectionRequest requestConnection(final HttpRoute route,final Object state) {
    Args.notNull(route, "HTTP route");
    if (this.log.isDebugEnabled()) {
        this.log.debug("Connection request: " + format(route, state) + formatStats(route));
    }
    final Future<CPoolEntry> future = this.pool.lease(route, state, null);
    return new ConnectionRequest() {
        @Override
        public boolean cancel() {
            return future.cancel(true);
        }

        @Override
        public HttpClientConnection get(
                final long timeout,
                final TimeUnit tunit) throws InterruptedException, ExecutionException, ConnectionPoolTimeoutException {
            return leaseConnection(future, timeout, tunit);
        }
    };
}
```

我们可以看到，他的get方法重载了，实际上调用了`leaseConnection`方法，那这个方法：

```java
protected HttpClientConnection leaseConnection(
        final Future<CPoolEntry> future,
        final long timeout,
        final TimeUnit tunit) throws InterruptedException, ExecutionException, ConnectionPoolTimeoutException {
    final CPoolEntry entry;
    try {
        entry = future.get(timeout, tunit);
        // 以下省略
        return CPoolProxy.newProxy(entry);
    } catch (final TimeoutException ex) {
        throw new ConnectionPoolTimeoutException("Timeout waiting for connection from pool");
    }
}
```

这个proxy可以先不去理会，可见最重要的还是这个future。那这个future是什么呢？

```java
final Future<CPoolEntry> future = this.pool.lease(route, state, null);
```

进到lease方法里，会发现他是直接构造了一个Future实例，那我们关注他的`get()`方法的实现：

```java
@Override
// 这里的E实际类型就是CPoolEntry
public E get(final long timeout, final TimeUnit tunit) throws InterruptedException, ExecutionException, TimeoutException {
    final E entry = entryRef.get();
    if (entry != null) {
        return entry;
    }
    synchronized (this) {
        try {
            for (;;) {	// 无限循环是为了一直尝试获取
                final E leasedEntry = getPoolEntryBlocking(route, state, timeout, tunit, this);
                if (validateAfterInactivity > 0)  {
               			// 检测一下是不是超时了，超时了就重来一次 
                    if (leasedEntry.getUpdated() + validateAfterInactivity <= System.currentTimeMillis()) {
                        if (!validate(leasedEntry)) {
                            leasedEntry.close();
                            release(leasedEntry, false);
                            continue;
                        }
                    }
                }
                return leasedEntry;
            }
        } catch (final IOException ex) {
            // 省略
        }
    }
}
```

所以真正的获取的方法是`getPoolEntryBlocking(...)`，追踪之：

```java
private E getPoolEntryBlocking(
        final T route, final Object state,
        final long timeout, final TimeUnit tunit,
        final Future<E> future) throws IOException, InterruptedException, TimeoutException {

    Date deadline = null;
    if (timeout > 0) {
      	// 指获得连接要等待多久，这个timeout就是ConnectionTimeout。
      	// 要追溯这个timeout是ConnectionTimeToLive
        deadline = new Date (System.currentTimeMillis() + tunit.toMillis(timeout));
    }
    this.lock.lock();
    try {
      	// 获得一个特定路由的连接池，细节实际就是从可用队列里获得一个连接
        final RouteSpecificPool<T, C, E> pool = getPool(route);
        E entry;
        for (;;) {
            Asserts.check(!this.isShutDown, "Connection pool shut down");
            for (;;) {
              	// 尝试从其中获得可用的连接，这个操作在第一次执行时一定返回null
                entry = pool.getFree(state);
                if (entry == null) {
                    break;
                }
              	// 如果超时了，关闭这个入口。因为不可能无限保持TCP连接，至少这是个不好的操作
              	// 所以超时了以后就关闭连接比较合理。这里的超时是指ConnectionTimeToLive
                if (entry.isExpired(System.currentTimeMillis())) {
                    entry.close();
                }
              	// 如果已经被关闭，那就从可用队列里移除。
                if (entry.isClosed()) {
                    this.available.remove(entry);
                    pool.free(entry, false);
                } else {
                    break;
                }
            }
          	// 这里一连串的if还是很严谨的
            if (entry != null) {
              	// 被使用了，当然要移出可用队列
                this.available.remove(entry);
              	// 加入已分配队列
                this.leased.add(entry);
                onReuse(entry);
                return entry;
            }

            // New connection is needed。
          	// 因为第一次的时候free一定是空的，所以会新建。
          	// 根据默认设置，这里会得到2，一会会详细看一下这个方法
            final int maxPerRoute = getMax(route);
            // Shrink the pool prior to allocating a new connection
          	// 超出了最大允许的连接数，即maxPerRoute，就回收一下
            final int excess = Math.max(0, pool.getAllocatedCount() + 1 - maxPerRoute);
            if (excess > 0) {
                for (int i = 0; i < excess; i++) {
                    final E lastUsed = pool.getLastUsed();
                    if (lastUsed == null) {
                        break;
                    }
                    lastUsed.close();
                    this.available.remove(lastUsed);
                    pool.remove(lastUsed);
                }
            }
          
						// 如果这个route的已分配连接没有超过允许的最大连接，就分配。
            if (pool.getAllocatedCount() < maxPerRoute) {
                final int totalUsed = this.leased.size();
                final int freeCapacity = Math.max(this.maxTotal - totalUsed, 0);
                if (freeCapacity > 0) {
                    final int totalAvailable = this.available.size();
                    if (totalAvailable > freeCapacity - 1) {
                        if (!this.available.isEmpty()) {
                            final E lastUsed = this.available.removeLast();
                            lastUsed.close();
                            final RouteSpecificPool<T, C, E> otherpool = getPool(lastUsed.getRoute());
                            otherpool.remove(lastUsed);
                        }
                    }
                  	// 真正新建一个connection，在这一步，为entry设置了超时时间。
                  	// 这个超时时间在上面一个代码块中会用
                    final C conn = this.connFactory.create(route);
                    entry = pool.add(conn);
                    this.leased.add(entry);
                    return entry;
                }
            }
          
            boolean success = false;
            try {
                if (future.isCancelled()) {
                    throw new InterruptedException("Operation interrupted");
                }
              	// 否则就将这个申请的请求加到等待队列里
                pool.queue(future);
                this.pending.add(future);
                if (deadline != null) {
                    success = this.condition.awaitUntil(deadline);
                } else {
                    this.condition.await();
                    success = true;
                }
                if (future.isCancelled()) {
                    throw new InterruptedException("Operation interrupted");
                }
            } finally {
                // In case of 'success', we were woken up by the
                // connection pool and should now have a connection
                // waiting for us, or else we're shutting down.
                // Just continue in the loop, both cases are checked.
              	// 所以接下来就重试了，注意最上面那个for (;;)无限循环
                pool.unqueue(future);
                this.pending.remove(future);
            }
            // check for spurious wakeup vs. timeout
            if (!success && (deadline != null && deadline.getTime() <= System.currentTimeMillis())) {
                break;
            }
        }
        throw new TimeoutException("Timeout waiting for connection");
    } finally {
        this.lock.unlock();
    }
}
```

然后再来稍微看一下这个`final int maxPerRoute = getMax(route);`方法：

```java
private int getMax(final T route) {
    final Integer v = this.maxPerRoute.get(route);
    if (v != null) {
        return v.intValue();
    } else {
        return this.defaultMaxPerRoute;
    }
}
```

可见，先是从一个map里面查一下数据，如果没有的话就使用`defaultMaxPerRoute`。这两个值都可以由我们自行设置。

自此，我们理清楚了一个connection是怎么获得的。然后再来看这两个参数怎么设置。当一个请求到来之后，先会去解析他的`host:port`，之后尝试根据这个`host:port`去尝试获取连接，如果当前到`host:port`的连接数量少于`maxConnPerRoute`，并且该`HttpClient`中发出的所有连接数量小于`maxConnTotal`，那么连接可以分配，如果两个条件有一个不满足，请求就会进入等待队列。不断轮询。关于这两个值的大小设置，我们来看一下官方的解读：

```java
/**
 * {@code ClientConnectionPoolManager} maintains a pool of
 * {@link HttpClientConnection}s and is able to service connection requests
 * from multiple execution threads. Connections are pooled on a per route
 * basis. A request for a route which already the manager has persistent
 * connections for available in the pool will be services by leasing
 * a connection from the pool rather than creating a brand new connection.
 * <p>
 * {@code ClientConnectionPoolManager} maintains a maximum limit of connection
 * on a per route basis and in total. Per default this implementation will
 * create no more than than 2 concurrent connections per given route
 * and no more 20 connections in total. For many real-world applications
 * these limits may prove too constraining, especially if they use HTTP
 * as a transport protocol for their services. Connection limits, however,
 * can be adjusted using {@link ConnPoolControl} methods.
 * 上面这一段划重点，意思是默认值对于生产环境可能太小了，特别是当我们把HTTP当RPC用的时候
 * </p>
 * <p>
 * Total time to live (TTL) set at construction time defines maximum life span
 * of persistent connections regardless of their expiration setting. No persistent
 * connection will be re-used past its TTL value.
 * </p>
 * <p>
 * The handling of stale connections was changed in version 4.4.
 * Previously, the code would check every connection by default before re-using it.
 * The code now only checks the connection if the elapsed time since
 * the last use of the connection exceeds the timeout that has been set.
 * The default timeout is set to 2000ms
 * </p>
 *
 * @since 4.3
 */
@Contract(threading = ThreadingBehavior.SAFE_CONDITIONAL)
public class PoolingHttpClientConnectionManager
    implements HttpClientConnectionManager, ConnPoolControl<HttpRoute>, Closeable {
  	// 省略
}
```

既然太小了，那就有调整的必要，调整主要有两种办法：

```java
CloseableHttpClient httpClient = HttpClientBuilder.create()
            .setMaxConnTotal(400).setMaxConnPerRoute(200).build();
```

不解释了。还有一种更详细的设置方式，就是既然connection的管理是依靠`PoolingHttpClientConnectionManager`，那我们给httpClient设置一个`PoolingHttpClientConnectionManager`，就可以完全掌握连接分配了，而`PoolingHttpClientConnectionManager`的设置项要更多一些。如下：

```java
PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager();
manager.setDefaultMaxPerRoute(100);
manager.setMaxTotal(500);
manager.setMaxPerRoute(new HttpRoute(
  new HttpHost("https://www.baidu.com", 443)), 200);
CloseableHttpClient httpClient = HttpClientBuilder.create()
  .setConnectionManager(manager).build();
```

这个可以对单独的route设置最大连接数量，并且`setConnectionManager(manager)`会覆盖`setMaxConnTotal(400).setMaxConnPerRoute(200)`。

上面的这个`connectionTimeToLive`可以用以下两种方法设置，所有的超时连接都不会被复用：

```java
// 1
PoolingHttpClientConnectionManager manager = new PoolingHttpClientConnectionManager(100，TimeUnit.SECONDS);
// 2
CloseableHttpClient httpClient = HttpClientBuilder.create()
            .setConnectionTimeToLive(100，TimeUnit.SECONDS).build();
```

## RequestConfig 的 三个TimeOut

每一个请求都可以带一个RequestConfig参数，对此次请求的一些参数进行设置。这里面比较重要的是三个timeout参数，如下：

```java
RequestConfig requestConfig = RequestConfig.custom()
            .setConnectionRequestTimeout(30000).setConnectTimeout(30000)
            .setSocketTimeout(30000).build();
HttpRequest request = new HttpGet();	// 或者new HttpPost()等
request.setConfig(requestConfig);
httpclient.execute(request);
```

这三个timeout到底是干什么的？

###ConnectionRequestTimeout

看了上一章关于Connection的解读，这个参数的概念就十分清晰了。但我们还是从源码里追溯一下：

其调用栈大概是：

`Httpclient#execute(HttpRequest)`  --> `InternalHttpClient#doExecute(HttpHost, HttpRequest, HttpContext)`  --> `MainClientExec#execute(HttpRoute, HttpRequestWrapper, HttpClientContext, HttpExecutionAware)`  

这个connectionTimeOut真正起作用的地方在这里：

```java
public class MainClientExec implements ClientExecChain {
		@Override
    public CloseableHttpResponse execute(
            final HttpRoute route,
            final HttpRequestWrapper request,
            final HttpClientContext context,
            final HttpExecutionAware execAware) throws IOException, HttpException {
      // 省略前后的无关语句
     	// 在doExecute中，requestConfig参数被包装到context里面了
      final RequestConfig config = context.getRequestConfig();
      // 下面这个语句我们是熟悉的
      final ConnectionRequest connRequest = connManager.requestConnection(route, userToken);
      final int timeout = config.getConnectionRequestTimeout();
      managedConn = connRequest.get(timeout > 0 ? timeout : 0, TimeUnit.MILLISECONDS);
      // 之后的省略
    }
}
```

之后去追溯`connRequest.get`方法，最终还是追溯到`getPoolEntryBlocking`方法里面去了。就是timeout和deadline的那个关系，可以关注一下上面的代码块。意思就是在等待队列里等到deadline还没法获取connection，就会抛`TimeoutException`异常。

### ConnectTimeout

还是从`MainClientExec#execute(HttpRoute, HttpRequestWrapper, HttpClientContext, HttpExecutionAware)`去追溯，会发现有这么一个方法：

```java
if (!managedConn.isOpen()) {
    this.log.debug("Opening connection " + route);
    try {
      	// 当连接未打开时，首先建立连接
        establishRoute(proxyAuthState, managedConn, route, request, context);
    } catch (final TunnelRefusedException ex) {
        if (this.log.isDebugEnabled()) {
            this.log.debug(ex.getMessage());
        }
        response = ex.getResponse();
        break;
    }
}
```

调用栈是：`MainClientExec#establishRoute`  -->  `PoolingHttpClientConnectionManager#connect(HttpClientConnection, HttpRoute, int, HttpContext context)`  -->  `DefaultHttpClientConnectionOperator#connect(ManagedHttpClientConnection, HttpHost, InetSocketAddress, int, SocketConfig, HttpContext) `

最后调用了`ConnectionSocketFactory#connectSocket(...)`方法，其实就是说这次建立连接的超时时间……好吧，是因为我懒得继续追溯源代码了。就这样敷衍了事吧。

### SocketTimeout

这个调用栈太深了，转来转去。我也懒得追踪了……总之意思就是连接多久不断开。当我们连接的是一个需要很长时间才能返回的Http接口时，将这个时间调的长一些以避免连接断开。但是这个也有副作用，就是假如设置了一个不合理的长时间，而被连接的服务出错或者挂掉了，那就会等待太久。

