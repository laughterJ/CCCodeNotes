# 初识Volley（三），Volley源码解析

标签（空格分隔）： Android

---

&emsp;&emsp;了解了Volley的基本用法只是知其然，要想知其所以然，自然少不了阅读源码，好在Volley的源码并不复杂。



&emsp;&emsp;在 [初识Volley（一），Volley的基本用法](https://github.com/laughterJ/CCCodeNotes/blob/main/Android/Volley/%E5%88%9D%E8%AF%86Volley%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%8CVolley%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%94%A8%E6%B3%95.md) 这篇中，我们知道使用Volley的第一步是创建一个RequestQueue对象：

```java
RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
```

&emsp;&emsp;既然如此，我们就从这个 newRequestQueue(Context context) 方法入手吧。

```Java
public static RequestQueue newRequestQueue(Context context) {
  // 这里什么也没做，只是调用了另一个重载方法，并给第二个参数传入 null
	return newRequestQueue(context, (BaseHttpStack) null);
}

public static RequestQueue newRequestQueue(Context context, BaseHttpStack stack) {
	BasicNetwork network;
  // 已知 stack 为 null
	if (stack == null) {
    // 在Android2.3及以上系统，会创建一个 HurlStack 对象
    // HurlStack 内部使用 HttpURLConnection 进行网络通讯
		if (Build.VERSION.SDK_INT >= 9) {
			network = new BasicNetwork(new HurlStack());
		} else {
			String userAgent = "volley/0";
			try {
				String packageName = context.getPackageName();
				PackageInfo info = context.getPackageManager().getPackageInfo(packageName, /* flags= */ 0);
				userAgent = packageName + "/" + info.versionCode;
			} catch (NameNotFoundException e) {}
      // 而在Android2.3以下系统，则创建一个 HttpClientStack 对象
      // HttpClientStack 内部使用 HttpClient 进行网络通讯
			network = new BasicNetwork(new HttpClientStack(AndroidHttpClient.newInstance(userAgent)));
		}
	} else {
		network = new BasicNetwork(stack);
	}
  // 最终，又会调用另一个方法重载
	return newRequestQueue(context, network);
}

private static RequestQueue newRequestQueue(Context context, Network network) {
	File cacheDir = new File(context.getCacheDir(), DEFAULT_CACHE_DIR);
  // 根据前面构建好的 Network 对象创建一个 RequestQueue 对象，然后调用它的 start() 方法。
	RequestQueue queue = new RequestQueue(new DiskBasedCache(cacheDir), network);
	queue.start();
  // 最后将这个 RequestQueue 对象返回。
	return queue;
}
```

&emsp;&emsp;接下来，我们来看看 RequestQueue 的 start() 方法做了些什么。

```java
public void start() {
    stop(); // Make sure any currently running dispatchers are stopped.
    // Create the cache dispatcher and start it.
    mCacheDispatcher = new CacheDispatcher(mCacheQueue, mNetworkQueue, mCache, mDelivery);
    mCacheDispatcher.start();

    // Create network dispatchers (and corresponding threads) up to the pool size.
    // 查看 RequestQueue 的构造方法，我们可以知道 mDispatchers.length 默认为4
    for (int i = 0; i < mDispatchers.length; i++) {
        NetworkDispatcher networkDispatcher =
                new NetworkDispatcher(mNetworkQueue, mNetwork, mCache, mDelivery);
        mDispatchers[i] = networkDispatcher;
        networkDispatcher.start();
    }
}

/** Stops the cache and network dispatchers. */
public void stop() {
    if (mCacheDispatcher != null) {
        mCacheDispatcher.quit();
    }
    for (final NetworkDispatcher mDispatcher : mDispatchers) {
        if (mDispatcher != null) {
            mDispatcher.quit();
        }
    }
}
```

&emsp;&emsp;首先，我们要知道这里的 NetworkDispatcher 和 CacheDispatcher 是什么，点进去一看我们会发现，它们都继承自Therad类。也就是说，这里其实创建了五个线程，一个缓存线程（CacheDispatcher），四个网络请求线程（NetworkDispatcher）。当我们创建一个 RequestQueue 对象时，其内部默认会创建五个线程在后台运行，其用意如何我们继续往下看。



&emsp;&emsp;我们知道，RequestQueue 对象创建完成后，我们就要创建 Request 对象，然后调用 RequestQueue 的 add() 方法，将 Request对象添加到 RequestQueue 当中完成网络请求。那么，我们再来看一下 add() 方法到底做了什么：

```Java
public <T> Request<T> add(Request<T> request) {
    // Tag the request as belonging to this queue and add it to the set of current requests.
    request.setRequestQueue(this);
    synchronized (mCurrentRequests) {
        mCurrentRequests.add(request);
    }

    // Process requests in the order they are added.
    request.setSequence(getSequenceNumber());
    request.addMarker("add-to-queue");

    // If the request is uncacheable, skip the cache queue and go straight to the network.
    // shouldCache()方法判断请求是否可以缓存
    if (!request.shouldCache()) {
        mNetworkQueue.add(request);
        return request;
    }
    mCacheQueue.add(request);
    return request;
}
```

&emsp;&emsp;这里主要就是根据 Request 是否可以缓存，将 Request 分别添加到 缓存队列 和 网络请求队列，不过默认情况下，每条请求都是可以缓存的。到这里，我们基本理清了Volley的工作流程，再来看一下Google官方文档中给出的 请求生命周期图，相信你的思路会更加清晰。

![请求生命周期](https://developer.android.google.cn/images/training/volley-request.png?hl=zh_cn)

&emsp;&emsp;到目前为止，我们已经了解了Volley基本的工作流程，不过具体每个Request是如何完成请求并返回结果的呢？缓存线程与网络请求线程又是如何调度的呢？要想知道这两个问题的答案，恐怕我们还得看看CacheDispatcher和NetworkDispatcher内部是如何工作的。首先来看一下CacheDispatcher的run()方法：

```java
@Override
public void run() {
    if (DEBUG) VolleyLog.v("start new dispatcher");
    Process.setThreadPriority(Process.THREAD_PRIORITY_BACKGROUND);
    // Make a blocking call to initialize the cache.
    mCache.initialize();
    // 开始轮询
    while (true) {
        try {
            processRequest();
        } catch (InterruptedException e) {
            // We may have been interrupted because it was time to quit.
            if (mQuit) {
                Thread.currentThread().interrupt();
                return;
            }
            VolleyLog.e("Ignoring spurious interrupt of CacheDispatcher thread; " + "use quit() to terminate it");
        }
    }
}

private void processRequest() throws InterruptedException {
    // Get a request from the cache triage queue, blocking until
    // at least one is available.
    // 取出缓存队列中的 Request
    final Request<?> request = mCacheQueue.take();
    processRequest(request);
}

/**
 * Retrieves and removes the head of this queue, waiting if necessary
 * until an element becomes available.
 */
E take() throws InterruptedException;
```

&emsp;&emsp;我们看到，缓存线程一旦开始执行，就会不停轮询，取出缓存队列中的Request，具体处理逻辑在 processRequest() 方法的另一个重载方法中，我们继续往下看：

```java
void processRequest(final Request<?> request) throws InterruptedException {
    request.addMarker("cache-queue-take");

    // If the request has been canceled, don't bother dispatching it.
    if (request.isCanceled()) {
        request.finish("cache-discard-canceled");
        return;
    }

    // Attempt to retrieve this item from cache.
    // 尝试从缓存中取出Request对应的响应结果
    Cache.Entry entry = mCache.get(request.getCacheKey());
    if (entry == null) {
        request.addMarker("cache-miss");
        // Cache miss; send off to the network dispatcher.
        // 这里有一个等待响应队列的概念，重复的请求会被添加到同一个等待响应队列，等待请求响应（过滤重复请求）
        // maybeAddToWaitingRequests() 方法的源码在下面，可以先看一下
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            // 返回 false，则说明当前请求是第一次发起，需要添加到网络请求队列，进行请求
            mNetworkQueue.put(request);
        }
        return;
    }

    // If it is completely expired, just send it to the network.
    // 判断缓存中的响应结果是否已经过期，对应 ttl 这个值
    if (entry.isExpired()) {
        // 如果过期了，再判断是否有等待响应的相同请求，没有则添加到网络请求队列，进行请求
        request.addMarker("cache-hit-expired");
        request.setCacheEntry(entry);
        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            mNetworkQueue.put(request);
        }
        return;
    }

    // We have a cache hit; parse its data for delivery back to the request.
    // 到这里，则说明缓存中的响应结果有效，直接使用缓存中的响应结果
    request.addMarker("cache-hit");
    // 解析数据由 Request 自行实现，不同的 Request 解析方式不同
    Response<?> response = request.parseNetworkResponse(new NetworkResponse(entry.data, entry.responseHeaders));
    request.addMarker("cache-hit-parsed");
    // refreshNeeded() 方法会判断缓存中的响应结果是否 “即将过期”，对应 softTtl 这个值
    // 无论是否即将过期，都会先把缓存中的响应数据返回给主线程
    if (!entry.refreshNeeded()) {
        // Completely unexpired cache hit. Just deliver the response.
        mDelivery.postResponse(request, response);
    } else {
        // 即将过期的话，则会重新判断是否有等待响应的相同请求，没有则添加到网络请求队列，进行请求
        request.addMarker("cache-hit-refresh-needed");
        request.setCacheEntry(entry);
        // Mark the response as intermediate.
        response.intermediate = true;

        if (!mWaitingRequestManager.maybeAddToWaitingRequests(request)) {
            // Post the intermediate response back to the user and have
            // the delivery then forward the request along to the network.
            mDelivery.postResponse(request, response, new Runnable() {
            	  @Override
                public void run() {
                    try {
                        mNetworkQueue.put(request);
                    } catch (InterruptedException e) {
                        // Restore the interrupted status
                        Thread.currentThread().interrupt();
                    }
                }
            });
        } else {
            // request has been added to list of waiting requests
            // to receive the network response from the first request once it returns.
            mDelivery.postResponse(request, response);
        }
    }
}

private synchronized boolean maybeAddToWaitingRequests(Request<?> request) {
    String cacheKey = request.getCacheKey();
    // mWaitingRequests 是一个Map对象，以 cacheKey 作为键与 等待响应队列 关联
    if (mWaitingRequests.containsKey(cacheKey)) {
        // mWaitingRequests中有相应的cacheKey，表示当前请求已经在进行中
        // 于是将当前请求添加到 等待响应队列 中。
        List<Request<?>> stagedRequests = mWaitingRequests.get(cacheKey);
        if (stagedRequests == null) {
            stagedRequests = new ArrayList<>();
        }
        request.addMarker("waiting-for-response");
        stagedRequests.add(request);
        mWaitingRequests.put(cacheKey, stagedRequests);
        if (VolleyLog.DEBUG) {
            VolleyLog.d("Request for cacheKey=%s is in flight, putting on hold.", cacheKey);
        }
        return true;
    } else {
        // 第一次发起请求时，只将 cacheKey 添加到Map中作为标识，不创建相应的 stagedRequests ，节省开销
        mWaitingRequests.put(cacheKey, null);
        request.setNetworkRequestCompleteListener(this);
        if (VolleyLog.DEBUG) {
            VolleyLog.d("new request, sending to network %s", cacheKey);
        }
        return false;
    }
}
```

&emsp;&emsp;看到这里，你可能会觉得有些眼花缭乱了，不过我们只需要把握主线，缓存线程的作用主要是过滤掉重复请求，并且在可以使用缓存中的响应数据时，尽可能使用缓存，以节省开销。



&emsp;&emsp;下面 NetworkDispatcher 就比较轻松了，网络请求相关的细节我们没必要关心太多，只需要理清网络请求线程内部做了哪些事情即可，run() 方法与CacheDispatcher大致相同，我们重点看 processRequest(Request<?> request) 方法：

```java
void processRequest(Request<?> request) {
    long startTimeMs = SystemClock.elapsedRealtime();
    try {
        request.addMarker("network-queue-take");

        // If the request was cancelled already, do not perform the
        // network request.
        if (request.isCanceled()) {
            request.finish("network-discard-cancelled");
            request.notifyListenerResponseNotUsable();
            return;
        }
        addTrafficStatsTag(request);
        // 发起网络请求
        NetworkResponse networkResponse = mNetwork.performRequest(request);
        request.addMarker("network-http-complete");

        // 服务端返回304错误码，表示客户端已经有缓存，并且我们已经给主线程发送过数据
        if (networkResponse.notModified && request.hasHadResponseDelivered()) {
            request.finish("not-modified");
            request.notifyListenerResponseNotUsable();
            return;
        }
        // 数据解析由 Request 自己实现
        Response<?> response = request.parseNetworkResponse(networkResponse);
        request.addMarker("network-parse-complete");

        // Write to cache if applicable.
        // TODO: Only update cache metadata instead of entire record for 304s.
        if (request.shouldCache() && response.cacheEntry != null) {
            // 将响应数据写入缓存
            mCache.put(request.getCacheKey(), response.cacheEntry);
            request.addMarker("network-cache-written");
        }
        // Post the response back.
        request.markDelivered();
        // 回调解析后的数据
        mDelivery.postResponse(request, response);
        request.notifyListenerResponseReceived(response);
    } catch (VolleyError volleyError) {
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        parseAndDeliverNetworkError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    } catch (Exception e) {
        VolleyLog.e(e, "Unhandled exception %s", e.toString());
        VolleyError volleyError = new VolleyError(e);
        volleyError.setNetworkTimeMs(SystemClock.elapsedRealtime() - startTimeMs);
        mDelivery.postError(request, volleyError);
        request.notifyListenerResponseNotUsable();
    }
}
```

&emsp;&emsp;网络请求线程主要是发起网络请求，将返回结果回调给主线程并写入缓存，以及处理异常。到这里，我们基本分析完了Volley的主要源码，再回过头看看 Google 给出的 请求生命周期图 ，豁然开朗有没有！

------

&emsp;&emsp;既然理清了Volley的工作流程，知其所以然了，下面我们就可以做一些定制化的事情了，比如说自己实现一个Request，虽然StringRequest、JsonRequest、ImageRequest基本已经够用了，但是我们完全可以结合具体使用场景定制使用更方便的Request。另外，我们也可以对Volley进一步封装，满足自己的使用需求。