参考：

- [Android图片加载框架最全解析（四），玩转Glide的回调与监听](http://blog.csdn.net/guolin_blog/article/details/70215985)

加之自己屡着源码，写笔记。

- 1.Target接口
- 2.preload方法
- 3.downloadOnly
- 4.listener


------

#### 1.Target


```
Glide.with(mContext).load(url).into(holder.entry_list_ige);
```
调用into最终会走到GenericRequestBuilder的into方法中，可以看到GenericRequestBuilder的into方法有若干重载方法

```
    public <Y extends Target<TranscodeType>> Y into(Y target);
    public Target<TranscodeType> into(ImageView view);
    public FutureTarget<TranscodeType> into(int width, int height);
```

Target的继承关系如图

![](http://img.blog.csdn.net/20170614222823388?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

主要考虑SimpleTarget和ViewTarget

摘抄一下博客上的例子吧

**SimpleTarget**

```
SimpleTarget<GlideDrawable> simpleTarget = new SimpleTarget<GlideDrawable>() {
    @Override
    public void onResourceReady(GlideDrawable resource, GlideAnimation glideAnimation) {
        imageView.setImageDrawable(resource);
    }
};

public void loadImage(View view) {
    String url = "http://cn.bing.com/az/hprichbg/rb/TOAD_ZH-CN7336795473_1920x1080.jpg";
    Glide.with(this)
         .load(url)
         .into(simpleTarget);
}
```

```
SimpleTarget<Bitmap> simpleTarget = new SimpleTarget<Bitmap>() {
    @Override
    public void onResourceReady(Bitmap resource, GlideAnimation glideAnimation) {
        imageView.setImageBitmap(resource);
    }
};

public void loadImage(View view) {
    String url = "http://cn.bing.com/az/hprichbg/rb/TOAD_ZH-CN7336795473_1920x1080.jpg";
    Glide.with(this)
         .load(url)
         .asBitmap()
         .into(simpleTarget);
}
```

这里源码细节就不深究了，看第一篇的笔记就可以了，总体的思路是当图片资源拿到后会调用target的onResourceReady方法，即使在使用Glide时传递给into的是一个ImageView也会被包装成一个Target。当调用asBitmap或asGif方法时，会被包装成**BitmapImageViewTarget**类型，默认情况下会包装成**GlideDrawableImageViewTarget**

**ViewTarget**

```
public class MyLayout extends LinearLayout {

    private ViewTarget<MyLayout, GlideDrawable> viewTarget;

    public MyLayout(Context context, AttributeSet attrs) {
        super(context, attrs);
        viewTarget = new ViewTarget<MyLayout, GlideDrawable>(this) {
            @Override
            public void onResourceReady(GlideDrawable resource, GlideAnimation glideAnimation) {
                MyLayout myLayout = getView();
                myLayout.setImageAsBackground(resource);
            }
        };
    }

    public ViewTarget<MyLayout, GlideDrawable> getTarget() {
        return viewTarget;
    }

    public void setImageAsBackground(GlideDrawable resource) {
        setBackground(resource);
    }

}

public class MainActivity extends AppCompatActivity {

    MyLayout myLayout;

    @Override
    protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.activity_main);
        myLayout = (MyLayout) findViewById(R.id.background);
    }

    public void loadImage(View view) {
        String url = "http://cn.bing.com/az/hprichbg/rb/TOAD_ZH-CN7336795473_1920x1080.jpg";
        Glide.with(this)
             .load(url)
             .into(myLayout.getTarget());
    }

}
```

#### 2. preload

```
Glide.with(this)
     .load(url)
     .diskCacheStrategy(DiskCacheStrategy.SOURCE)
     .preload();
     
public class GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> implements Cloneable {
    ...

    public Target<TranscodeType> preload(int width, int height) {
        final PreloadTarget<TranscodeType> target = PreloadTarget.obtain(width, height);
        return into(target);
    }

    public Target<TranscodeType> preload() {
        return preload(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL);
    }

    ...
}
```

preload会构造PreloadTarget，

```

    public static <Z> PreloadTarget<Z> obtain(int width, int height) {
        return new PreloadTarget<Z>(width, height);
    }

    private PreloadTarget(int width, int height) {
        super(width, height);
    }

    @Override
    public void onResourceReady(Z resource, GlideAnimation<? super Z> glideAnimation) {
        Glide.clear(this);
    }
}
```

```
Glide.java
    public static void clear(Target<?> target) {
        Util.assertMainThread();
        Request request = target.getRequest();
        if (request != null) {
            request.clear();
            target.setRequest(null);
        }
    }
```

PreloadTarget在onResourceReady除了clear没有其他操作，只是清楚了上次的Http请求而已。

#### 3. downloadOnly

- downloadOnly(int width, int height) ： 用于在子线程中下载图片
- downloadOnly(Y target)：在主线程中下载图片

```
public void downloadImage(View view) {
    new Thread(new Runnable() {
        @Override
        public void run() {
            try {
                String url = "http://cn.bing.com/az/hprichbg/rb/TOAD_ZH-CN7336795473_1920x1080.jpg";
                final Context context = getApplicationContext();
                FutureTarget<File> target = Glide.with(context)
                                                 .load(url)
                                                 .downloadOnly(Target.SIZE_ORIGINAL, Target.SIZE_ORIGINAL);
                final File imageFile = target.get();
                runOnUiThread(new Runnable() {
                    @Override
                    public void run() {
                        Toast.makeText(context, imageFile.getPath(), Toast.LENGTH_LONG).show();
                    }
                });
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }).start();
}
```

>如果此时图片还没有下载完，那么get()方法就会阻塞住，一直等到图片下载完成才会有值返回。

**此部分会关联线程的同步内容，此处不再赘述，有时间可以看看源代码，实现的还是比较巧妙的。**

#### 4. listener

```
public void loadImage(View view) {
    String url = "http://cn.bing.com/az/hprichbg/rb/TOAD_ZH-CN7336795473_1920x1080.jpg";
    Glide.with(this)
            .load(url)
            .listener(new RequestListener<String, GlideDrawable>() {
                @Override
                public boolean onException(Exception e, String model, Target<GlideDrawable> target,
                    boolean isFirstResource) {
                    return false;
                }

                @Override
                public boolean onResourceReady(GlideDrawable resource, String model,
                    Target<GlideDrawable> target, boolean isFromMemoryCache, boolean isFirstResource) {
                    return false;
                }
            })
            .into(imageView);
}
```

```
public final class GenericRequest<A, T, Z, R> implements Request, SizeReadyCallback,
        ResourceCallback {

    private RequestListener<? super A, R> requestListener;
    ...

    private void onResourceReady(Resource<?> resource, R result) {
        boolean isFirstResource = isFirstReadyResource();
        status = Status.COMPLETE;
        this.resource = resource;
        //这行代码很关键
        if (requestListener == null || !requestListener.onResourceReady(result, model, target,
                loadedFromMemoryCache, isFirstResource)) {
            GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);
            target.onResourceReady(result, animation);
        }
        notifyLoadSuccess();
    }
    ...
}
```

