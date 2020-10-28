# 初识Volley（一），Volley的基本用法

标签（空格分隔）： Android

---

[Google官方文档](https://developer.android.google.cn/training/volley)
[Github仓库地址](https://github.com/google/volley)

&emsp;&emsp;官方镇楼，下面我们正式开始介绍Volley的使用吧。
&emsp;&emsp;首先当然是添加依赖(建议使用官方文档中的最新版本)：
```Java
    dependencies {
        ...
        implementation 'com.android.volley:volley:1.1.1'
    }
```
### Volley 使用简介
>大体上讲，您可以通过创建 RequestQueue 并向其传递 Request 对象以使用 Volley。RequestQueue 管理用于运行网络操作、向缓存读写数据以及解析响应的工作器线程。请求（Request）负责解析原始响应，而 Volley 负责将已解析的响应调度回主线程以供传送。

&emsp;&emsp;这是官方文档中对Volley用法的描述，从使用的角度讲，我们只需要知道 Volley 的网络请求是基于请求队列（RequestQueue）的，我们将网络请求（Request）添加到请求队列中，Volley就会依次帮我们完成请求并返回数据。因此我们首先要关注的就是 Volley 提供的几种 Request ：

- StringRequest
- JsonRequest
- ImageRequest

### StringRequest 的基本用法
&emsp;&emsp;StringRequest提供了两个构造方法：
```Java
public StringRequest(String url, Listener<String> listener, @Nullable ErrorListener errorListener) {
    this(Method.GET, url, listener, errorListener);
}

/**
 * Creates a new request with the given method.
 *
 * @param method Http请求方法
 * @param url 请求地址
 * @param listener 请求成功回调接口
 * @param errorListener 请求出错回调接口
 */
public StringRequest(int method, String url, Listener<String> listener, @Nullable ErrorListener errorListener) {
    super(method, url, errorListener);
    mListener = listener;
}
```
&emsp;&emsp;从构造方法可以看出，不指定请求方法则默认GET请求。下面来看一个示例：
```Java
private void post() {
    // API 来自 https://www.wanandroid.com/blog/show/2
    final String url = "https://www.wanandroid.com/user/login";
    // 首先创建 请求队列，传入一个Context参数
    RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
    // 然后创建请求，这里我们发起一个POST请求
    StringRequest request = new StringRequest(Request.Method.POST, url, new Response.Listener<String>() {
        @Override
        public void onResponse(String response) {
            // 返回数据为 String 类型
            Log.e(TAG, "------------- onSucceed -------------\n" + response);
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            Log.e(TAG, "------------- onError -------------\n" + error.getMessage());
        }
    }) {
        // POST 参数通过重写 getParams() 方法传递，GET 请求则不需要重写
        @Override
        protected Map<String, String> getParams() throws AuthFailureError {
            Map<String, String> params = new HashMap<>();
            params.put("username", "CCCode");
            params.put("password", "123456");
            return params;
        }
    };
    queue.add(request);
}
```
&emsp;&emsp;调用上面的方法，可以看到日志打印如下：
![请求返回](http://static.zybuluo.com/CCCode/y81o5mspd3ya59fyngl558y6/image_1elmuei5r1945ri49jri9g179mp.png)

### JsonRequest 的基本用法
&emsp;&emsp;JsonRequest是一个抽象类，其实现类有两个：JsonObjectRequest 和 JsonArrayRequest，除了返回类型不同以外，其实现和用法基本相同，下面我们具体来看 JsonObjectRequest 即可。
&emsp;&emsp;JsonObjectRequest也提供了两个构造方法：
```Java
public JsonObjectRequest(int method, String url, @Nullable JSONObject jsonRequest, Listener<JSONObject> listener, @Nullable ErrorListener errorListener) {
    super(method, url, (jsonRequest == null) ? null : jsonRequest.toString(),listener,errorListener);
}

public JsonObjectRequest(String url, @Nullable JSONObject jsonRequest, Listener<JSONObject> listener, @Nullable ErrorListener errorListener) {
    this(jsonRequest == null ? Method.GET : Method.POST, url, jsonRequest, listener, errorListener);
}
```
&emsp;&emsp;如果只是处理比较常见的GET和POST请求，用第二个构造方法即可，请求参数传null即为GET请求。我们来看一个GET请求示例：
```Java
private void get() {
    // API 来自 https://www.wanandroid.com/blog/show/2
    final String url = "https://wanandroid.com/wxarticle/chapters/json";
    // 同样的套路，Volley上手很简单有没有！！！
    RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
    JsonObjectRequest request = new JsonObjectRequest(url, null, new Response.Listener<JSONObject>() {
        @Override
        public void onResponse(JSONObject response) {
            Log.e(TAG, "------------- onSucceed -------------\n");
            try {
                JSONArray array = response.getJSONArray("data");
                for (int i = 0; i < array.length(); i++) {
                    Log.e(TAG, array.get(i).toString());
                }
            } catch (JSONException e) {
                e.printStackTrace();
            }
        }
    }, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            Log.e(TAG, "------------- onError -------------\n" + error.getMessage());
        }
    });
    queue.add(request);
}
```
&emsp;&emsp;调用上面的方法，可以看到日志打印如下：
![请求返回](http://static.zybuluo.com/CCCode/hdfgb21jhkrmd8ylbz6nss2l/image_1eln0fb5g1rn91c5h14u7o4b1r6f1j.png)

### ImageRequest 的基本用法
&emsp;&emsp;看名字就知道和图片相关，没错，ImageRequest就是用来加载图片的。依旧先来看一下构造方法：
```Java
/**
 * @param url 图片地址
 * @param listener 请求成功回调接口
 * @param maxWidth 最大不压缩宽度，图片实际尺寸大于这个值就对图片进行压缩，设为0则不压缩
 * @param maxHeight 最大不压缩高度，图片实际尺寸大于这个值就对图片进行压缩，设为0则不压缩
 * @param scaleType 指定图片缩放类型
 * @param decodeConfig 指定图片颜色属性
 * @param errorListener 请求失败回调接口
 */
public ImageRequest(String url, Response.Listener<Bitmap> listener, int maxWidth, int maxHeight, ScaleType scaleType, Config decodeConfig, @Nullable Response.ErrorListener errorListener) {
    super(Method.GET, url, errorListener);
    setRetryPolicy(new DefaultRetryPolicy(DEFAULT_IMAGE_TIMEOUT_MS, DEFAULT_IMAGE_MAX_RETRIES, DEFAULT_IMAGE_BACKOFF_MULT));
    mListener = listener;
    mDecodeConfig = decodeConfig;
    mMaxWidth = maxWidth;
    mMaxHeight = maxHeight;
    mScaleType = scaleType;
}
// @Deprecated注解 表示此方法已废弃、暂时可用，但以后此类或方法都不会再更新、后期可能会删除，建议后来人不要调用此方法。
@Deprecated
public ImageRequest(String url, Response.Listener<Bitmap> listener, int maxWidth, int maxHeight, Config decodeConfig, @Nullable Response.ErrorListener errorListener) {
    this(url, listener, maxWidth, maxHeight, ScaleType.CENTER_INSIDE, decodeConfig, errorListener);
}
```
&emsp;&emsp;由于第二个构造方法已废弃，我们忽略即可。第一个构造方法接收7个参数：
  
- maxWidth 和 maxHeight：这两个参数用于设置返回图片的最大宽高，若大于设置值，则会对图片进行压缩；若指定为0，则无论图片多大，都不会压缩。
- scaleType：只有 **ScaleType.FIT_XY** 和 **ScaleType.CENTER_CROP** 会生效，与maxWidth 和 maxHeight共同决定返回图片的尺寸。
- decodeConfig：用于指定图片的颜色属性，其中ARGB_8888可以展示最好的颜色属性，每个图片像素占据4个字节的大小，而RGB_565则表示每个图片像素占据2个字节大小。

&emsp;&emsp;下面来看一个简单的示例：
```Java
private void getImage() {
    final String url = "https://cn.bing.com/th?id=OHR.CambronBridge_EN-CN3974503184_1920x1080.jpg";
    RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
    ImageRequest request = new ImageRequest(url, new Response.Listener<Bitmap>() {
        @Override
        public void onResponse(Bitmap response) {
            Log.e(TAG, "------------- onSucceed -------------");
            Log.e(TAG, response.getWidth() + " * " + response.getHeight());
        }
        // 在 ScaleType.FIT_XY 模式下，只要 maxWidth 和 maxHeight 不为0，则返回图片宽高与指定最大宽高一致
    }, 800, 800, ImageView.ScaleType.FIT_XY, Bitmap.Config.ARGB_8888, new Response.ErrorListener() {
        @Override
        public void onErrorResponse(VolleyError error) {
            Log.e(TAG, "------------- onError -------------\n" + error.getMessage());
        }
    });
    queue.add(request);
}
```

&emsp;&emsp;调用上面的方法，可以看到日志打印如下：
![日志打印](http://static.zybuluo.com/CCCode/19r8nbr61g0n507rnx6qvkwc/image_1elniinhgkk3151n5m3gkhhkh23.png)

&emsp;&emsp;相信看到这里，大家都已经掌握了Volley的基本用法，并且可以举一反三，毕竟Volley的用法十分简单，也很好理解。