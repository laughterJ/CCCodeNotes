# 初识Volley（二），高效的图片加载方案

标签（空格分隔）： Android

---

&emsp;&emsp;在上一篇 [初识Volley（一），Volley的基本用法](https://github.com/laughterJ/CCCodeNotes/blob/main/Android/Volley/%E5%88%9D%E8%AF%86Volley%EF%BC%88%E4%B8%80%EF%BC%89%EF%BC%8CVolley%E7%9A%84%E5%9F%BA%E6%9C%AC%E7%94%A8%E6%B3%95.md) 中我们了解了 ImageRequest 的用法，已经可以实现加载图片的功能了，不过我们不能止步于此，下面再来学习一种更高效的图片加载方法--- **ImageLoader** 的用法。

### ImageLoader 的基本用法

&emsp;&emsp;ImageLoader内部也是基于ImageRequest实现的，它对ImageRequest做了很好的封装，可以帮我们实现图片缓存的功能，而且还会自动过滤掉重复的连接，避免重复发送请求。ImageLoader的使用可以分四步，下面结合代码来看：
```Java
private void loader() {
    final String imgUrl = "https://cn.bing.com/th?id=OHR.CambronBridge_EN-CN3974503184_1920x1080.jpg";
    // 第一步：创建请求队列
    // 无比熟悉的套路，毕竟ImageLoader是基于ImageRequest封装的
    RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
    // 第二步：创建ImageLoader对象
    // ImageLoader只有这一个构造方法，接受两个参数：RequestQueue对象和 ImageCache 对象
    // ImageCache是我们实现缓存的关键，这里先new一个空实现即可
    ImageLoader loader = new ImageLoader(queue, new ImageLoader.ImageCache() {
        @Override
        public Bitmap getBitmap(String url) {
            return null;
        }

        @Override
        public void putBitmap(String url, Bitmap bitmap) {

        }
    });
    // 第三步：创建ImageListener对象
    // ImageLoader的getImageListener()方法会返回一个ImageListener对象
    // 它接收三个参数：第一个参数指定显示图片的控件，第二个参数指定加载过程中显示的图片，第三个参数指定加载失败时显示的图片。
    ImageLoader.ImageListener listener = ImageLoader.getImageListener(binding.imgLoader,
            R.drawable.loading, R.drawable.img_loading);
    // 第四步调用ImageLoader的 get() 方法加载图片
    loader.get(imgUrl, listener);
}
```

&emsp;&emsp;第四步中，ImageLoader的 get()方法有三个重载：
```Java
// 接收2个参数：图片Url 和 ImageListener
public ImageContainer get(String requestUrl, final ImageListener listener) {
    return get(requestUrl, listener, /* maxWidth= */ 0, /* maxHeight= */ 0);
}
// 接收4个参数，maxWidth、maxHeight可以指定图片允许的最大宽高
public ImageContainer get(String requestUrl, ImageListener imageListener, int maxWidth, int maxHeight) {
    return get(requestUrl, imageListener, maxWidth, maxHeight, ScaleType.CENTER_INSIDE);
}
// 接收5个参数：scaleType可以指定图片缩放比例，与最大宽高共同决定返回图片的最大宽高及缩放效果
public ImageContainer get(String requestUrl, ImageListener imageListener, int maxWidth, int maxHeight, ScaleType scaleType) {
    ···
}
```
&emsp;&emsp;从上面的源码得知，无论调用几个参数的 get() 方法，最终都会调用5个参数的 get() 方法，无非是通过指定 maxWidth、maxHeight、scaleType这3个参数，我们可以控制最终返回图片的最大宽高与缩放效果。

&emsp;&emsp;再来看第二步中的 ImageCache 参数，我们如果只new一个空的 ImageCache 对象，是无法实现缓存的作用的。也就是说，ImageLoader 只是提供了一个实现图片缓存的接口，具体如何对图片进行缓存，还需要我们自己来实现。

&emsp;&emsp;说到这里，你可能会无从下手，那么要如何实现图片缓存的功能呢？我们可以借助Android提供的LruCache来实现，建议看看**郭霖老师**的一篇博客 [Android高效加载大图、多图解决方案，有效避免程序OOM](https://blog.csdn.net/guolin_blog/article/details/9316683) ，讲解十分详实。我们目前只注重Volley的用法，因此先参照上面博客中的实现即可：
```Java
public class BitmapCache implements ImageLoader.ImageCache {

    private final LruCache<String, Bitmap> cache;

    // 这里我们自己指定缓存的大小
    public BitmapCache(int cacheSize) {
        cache = new LruCache<String, Bitmap>(cacheSize) {
            @Override
            protected int sizeOf(String key, Bitmap bitmap) {
                return bitmap.getRowBytes() * bitmap.getHeight();
            }
        };
    }

    @Override
    public Bitmap getBitmap(String url) {
        return cache.get(url);
    }

    @Override
    public void putBitmap(String url, Bitmap bitmap) {
        cache.put(url, bitmap);
    }
}
```
&emsp;&emsp;相应的，只需要将前面第2步中的代码修改一下，使用我们实现的 BitmapCache 来对图片进行缓存。
```Java
// 将应用可用内存最大值的1/8作为缓存大小
final int cacheSize = (int) Runtime.getRuntime().maxMemory() / 8;
ImageLoader loader = new ImageLoader(queue, new BitmapCache(cacheSize));
```
&emsp;&emsp;到这里，我们不仅实现了图片加载的功能，还充分利用了Volley中 ImageLoader 提供的图片缓存功能，赶紧去运行看看吧（虽然除了成功把图片加载出来，你什么也看不到emmmmm）。

### NetworkImageView 的基本用法

&emsp;&emsp;Volley还提供了一种加载图片的方式，NetworkImageView是一个基于ImageView封装的自定义View，它继承自ImageView，并且提供了加载网络图片的功能。

&emsp;&emsp;当然，既然是Volley提供的，自然包含我们熟悉的元素，NetworkImageView是基于 ImageLoader 封装的，因此在使用时也需要用到ImageLoader对象。不啰嗦，直接上代码：

&emsp;&emsp;既然是自定义View，首先自然是在布局文件中加入NetworkImageView控件。
```xml
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <com.android.volley.toolbox.NetworkImageView
        android:id="@+id/network_img"
        android:layout_width="match_parent"
        android:layout_height="match_parent"/>

</LinearLayout>
```
&emsp;&emsp;接下来看Java代码：
```Java
private void network() {
    final String imgUrl = "https://cn.bing.com/th?id=OHR.CambronBridge_EN-CN3974503184_1920x1080.jpg";
    RequestQueue queue = Volley.newRequestQueue(MyApp.getContext());
    final int cacheSize = (int) Runtime.getRuntime().maxMemory() / 8;
    ImageLoader loader = new ImageLoader(queue, new BitmapCache(cacheSize));
    // 设置加载过程中显示的图片
    binding.networkImg.setDefaultImageResId(R.drawable.loading);
    // 设置加载失败显示的图片
    binding.networkImg.setErrorImageResId(R.drawable.error);
    // 设置图片地址和ImageLoader
    binding.networkImg.setImageUrl(imgUrl, loader);
}
```
&emsp;&emsp;用法很简单，与ImageLoader异曲同工。唯一需要注意的是，NetworkImageView没有提供设置最大宽高的API，而是通过控件中设置的width和height等属性来对图片进行压缩及缩放。

--------

&emsp;&emsp;好了，到这里我们已经掌握了Volley的基本用法，接下来会有一些提高的内容，敬请期待吧！


&emsp;&emsp;如果对图片加载及缓存技术感兴趣，可以仔细研读郭霖老师的两篇博客（写的时间很早，但是架不住写得好啊）：

[Android高效加载大图、多图解决方案，有效避免程序OOM](https://blog.csdn.net/guolin_blog/article/details/9316683)

[Android照片墙应用实现，再多的图片也不怕崩溃](https://blog.csdn.net/guolin_blog/article/details/9526203)

