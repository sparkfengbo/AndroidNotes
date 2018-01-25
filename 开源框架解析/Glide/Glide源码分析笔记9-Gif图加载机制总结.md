这一部分 在博客中没有提及，我又比较好奇，为什么Glide可以加载Gif图而Piccasso不能。

```
            Glide.with(mContext)
                    .load(url)
                    .asGif()
                    .into(holder.entry_list_ige);
```

##### 1.提要

就以强制使用Gif的上述代码为例吧,调用asGif后返回的类型是GifTypeRequest，如下代码所示。

```
    public GifTypeRequest<ModelType> asGif() {
        return optionsApplier.apply(new GifTypeRequest<ModelType>(this, streamModelLoader, optionsApplier));
    }
```

GifTypeRequest<ModelType> 继承自 GifRequestBuilder<ModelType>，而GifRequestBuilder继承自GenericRequestBuilder。

```
  //GifRequestBuilder.java  
        GifRequestBuilder(LoadProvider<ModelType, InputStream, GifDrawable, GifDrawable> loadProvider,
            Class<GifDrawable> transcodeClass, GenericRequestBuilder<ModelType, ?, ?, ?> other) {
        super(loadProvider, transcodeClass, other);
    }
```

```
//GifTypeRequest.java
    GifTypeRequest(GenericRequestBuilder<ModelType, ?, ?, ?> other,
            ModelLoader<ModelType, InputStream> streamModelLoader, RequestManager.OptionsApplier optionsApplier) {
        super(buildProvider(other.glide, streamModelLoader, GifDrawable.class, null), GifDrawable.class, other);
        this.streamModelLoader = streamModelLoader;
        this.optionsApplier = optionsApplier;

        // Default to animating.
        crossFade();
    }
        private static <A, R> FixedLoadProvider<A, InputStream, GifDrawable, R> buildProvider(Glide glide,
            ModelLoader<A, InputStream> streamModelLoader, Class<R> transcodeClass,
            ResourceTranscoder<GifDrawable, R> transcoder) {
        if (streamModelLoader == null) {
            return null;
        }

        if (transcoder == null) {
            transcoder = glide.buildTranscoder(GifDrawable.class, transcodeClass);
        }
        DataLoadProvider<InputStream, GifDrawable> dataLoadProvider = glide.buildDataProvider(InputStream.class,
                GifDrawable.class);
        return new FixedLoadProvider<A, InputStream, GifDrawable, R>(streamModelLoader, transcoder, dataLoadProvider);
    }
```

很关键的一行代码是

```
 DataLoadProvider<InputStream, GifDrawable> dataLoadProvider = glide.buildDataProvider(InputStream.class,
                GifDrawable.class);
```

根据Glide的构造方法中注册的表可以发现dataLoadProvider实际上是GifDrawableLoadProvider类型。

```
public class GifDrawableLoadProvider implements DataLoadProvider<InputStream, GifDrawable> {
    private final GifResourceDecoder decoder;
    private final GifResourceEncoder encoder;
    private final StreamEncoder sourceEncoder;
    private final FileToStreamDecoder<GifDrawable> cacheDecoder;

    public GifDrawableLoadProvider(Context context, BitmapPool bitmapPool) {
        decoder = new GifResourceDecoder(context, bitmapPool);
        cacheDecoder = new FileToStreamDecoder<GifDrawable>(decoder);
        encoder = new GifResourceEncoder(bitmapPool);
        sourceEncoder = new StreamEncoder();
    }
    }
```
而调用into时候所创建的target的类型实际上是GlideDrawableImageViewTarget，除非强制使用asBitmap。

##### 2.Git图的播放

在Glide总体流程分析的那篇笔记中，最终的显示操作时调用
GenericRequest的onResourceReady方法中。而GenericRequest的onResourceReady会调用ImageViewTarget的onResourceReady方法。我们在通过asGif构建的ImageViewTarget实际上是GlideDrawableImageViewTarget。

```
GenericRequest.java
    private void onResourceReady(Resource<?> resource, R result) {
        // We must call isFirstReadyResource before setting status.
        boolean isFirstResource = isFirstReadyResource();
        status = Status.COMPLETE;
        this.resource = resource;

        if (requestListener == null || !requestListener.onResourceReady(result, model, target, loadedFromMemoryCache,
                isFirstResource)) {
            GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);
            target.onResourceReady(result, animation);
        }

        notifyLoadSuccess();

        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logV("Resource ready in " + LogTime.getElapsedMillis(startTime) + " size: "
                    + (resource.getSize() * TO_MEGABYTE) + " fromCache: " + loadedFromMemoryCache);
        }
    }

```
可以看到这行很关键的代码 

```
GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);
            target.onResourceReady(result, animation);
```

构建了一个GlideAnimation类型传递给了target。

上面我们分析到了 构建的GenericRequest实际上是GifTypeRequest类型。在GifTypeRequest的构造方法中会调用crossFade方法。

```
GifTypeRequest.java

    GifTypeRequest(GenericRequestBuilder<ModelType, ?, ?, ?> other,
            ModelLoader<ModelType, InputStream> streamModelLoader, RequestManager.OptionsApplier optionsApplier) {
        super(buildProvider(other.glide, streamModelLoader, GifDrawable.class, null), GifDrawable.class, other);
        this.streamModelLoader = streamModelLoader;
        this.optionsApplier = optionsApplier;

        // Default to animating.
        crossFade();
    }
    
        public GifRequestBuilder<ModelType> crossFade() {
        super.animate(new DrawableCrossFadeFactory<GifDrawable>());
        return this;
    }
```

GifTypeRequest继承自GenericRequestBuilder，super方法会调用GenericRequestBuilder的animate方法，实际上是将DrawableCrossFadeFactory保存在了animationFactory变量中，而在调用obtainRequest方法时会将animationFactory传递给GenericRequest.

所以，很重要的一点

```
GenericRequest.java
    private void onResourceReady(Resource<?> resource, R result) {
        // We must call isFirstReadyResource before setting status.
        boolean isFirstResource = isFirstReadyResource();
        status = Status.COMPLETE;
        this.resource = resource;

        if (requestListener == null || !requestListener.onResourceReady(result, model, target, loadedFromMemoryCache,
                isFirstResource)) {
            GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);
            target.onResourceReady(result, animation);
        }

        notifyLoadSuccess();

        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logV("Resource ready in " + LogTime.getElapsedMillis(startTime) + " size: "
                    + (resource.getSize() * TO_MEGABYTE) + " fromCache: " + loadedFromMemoryCache);
        }
    }

```

我们在`GlideAnimation<R> animation = animationFactory.build(loadedFromMemoryCache, isFirstResource);`这行代码构建的animation和DrawableCrossFadeFactory有关。


```

DrawableCrossFadeFactory.java

    @Override
    public GlideAnimation<T> build(boolean isFromMemoryCache, boolean isFirstResource) {
        if (isFromMemoryCache) {
            return NoAnimation.get();
        } else if (isFirstResource) {
            return getFirstResourceAnimation();
        } else {
            return getSecondResourceAnimation();
        }
    }

    private GlideAnimation<T> getFirstResourceAnimation() {
        if (firstResourceAnimation == null) {
            GlideAnimation<T> defaultAnimation = animationFactory.build(false /*isFromMemoryCache*/,
                true /*isFirstResource*/);
            firstResourceAnimation = new DrawableCrossFadeViewAnimation<T>(defaultAnimation, duration);
        }
        return firstResourceAnimation;
    }

    private GlideAnimation<T> getSecondResourceAnimation() {
        if (secondResourceAnimation == null) {
            GlideAnimation<T> defaultAnimation = animationFactory.build(false /*isFromMemoryCache*/,
                false /*isFirstResource*/);
            secondResourceAnimation = new DrawableCrossFadeViewAnimation<T>(defaultAnimation, duration);
        }
        return secondResourceAnimation;
    }
```

可见，返回的是DrawableCrossFadeViewAnimation。

那么我们继续看GlideDrawableImageViewTarget

```
GlideDrawableImageViewTarget.java

    @Override
    public void onResourceReady(GlideDrawable resource, GlideAnimation<? super GlideDrawable> animation) {
        if (!resource.isAnimated()) {
            //TODO: Try to generalize this to other sizes/shapes.
            // This is a dirty hack that tries to make loading square thumbnails and then square full images less costly
            // by forcing both the smaller thumb and the larger version to have exactly the same intrinsic dimensions.
            // If a drawable is replaced in an ImageView by another drawable with different intrinsic dimensions,
            // the ImageView requests a layout. Scrolling rapidly while replacing thumbs with larger images triggers
            // lots of these calls and causes significant amounts of jank.
            float viewRatio = view.getWidth() / (float) view.getHeight();
            float drawableRatio = resource.getIntrinsicWidth() / (float) resource.getIntrinsicHeight();
            if (Math.abs(viewRatio - 1f) <= SQUARE_RATIO_MARGIN
                    && Math.abs(drawableRatio - 1f) <= SQUARE_RATIO_MARGIN) {
                resource = new SquaringDrawable(resource, view.getWidth());
            }
        }
        super.onResourceReady(resource, animation);
        this.resource = resource;
        resource.setLoopCount(maxLoopCount);
        resource.start();
    }
    
    
```

```
public abstract class ImageViewTarget<Z> extends ViewTarget<ImageView, Z> implements GlideAnimation.ViewAdapter {

    public void onResourceReady(Z resource, GlideAnimation<? super Z> glideAnimation) {
        if (glideAnimation == null || !glideAnimation.animate(resource, this)) {
            setResource(resource);
        }
    }

    protected abstract void setResource(Z resource);

}

```

很重要的一行代码`glideAnimation.animate(resource, this)`,这个glideAnimation实际上是DrawableCrossFadeViewAnimation，并将实现了GlideAnimation .ViewAdapter接口的自己传递进去，实际上是想拿回调。

```
public interface GlideAnimation<R> {

    /**
     * An interface wrapping a view that exposes the necessary methods to run the various types of android animations
     * ({@link com.bumptech.glide.request.animation.ViewAnimation},
     * {@link com.bumptech.glide.request.animation.ViewPropertyAnimation} and animated
     * {@link android.graphics.drawable.Drawable}s).
     */
    interface ViewAdapter {
        /**
         * Returns the wrapped {@link android.view.View}.
         */
        View getView();

        /**
         * Returns the current drawable being displayed in the view, or null if no such drawable exists (or one cannot
         * be retrieved).
         */
        Drawable getCurrentDrawable();

        /**
         * Sets the current drawable (usually an animated drawable) to display in the wrapped view.
         *
         * @param drawable The drawable to display in the wrapped view.
         */
        void setDrawable(Drawable drawable);
    }
    }
```

我们再看DrawableCrossFadeViewAnimation的animate方法。

```
    @Override
    public boolean animate(T current, ViewAdapter adapter) {
        Drawable previous = adapter.getCurrentDrawable();
        if (previous != null) {
            TransitionDrawable transitionDrawable = new TransitionDrawable(new Drawable[] { previous, current });
            transitionDrawable.setCrossFadeEnabled(true);
            transitionDrawable.startTransition(duration);
            adapter.setDrawable(transitionDrawable);
            return true;
        } else {
            defaultAnimation.animate(current, adapter);
            return false;
        }
    }
```

**原理就是利用TransitionDrawable，将目前ImageView正在展示的Drawble替换成将要展示的Drawble，而duration正是通过解码Gif图得到的数据，这样不停的轮换，形成了Gif图播放的效果**

##### 3.Gif图的解码

根据第一节讲述，在DecodeJob在请求会Gif图时，会调用

`decodeFromSource->decodeSource->decodeFromSourceData`方法

```
DecodeJob.java
    private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        if (diskCacheStrategy.cacheSource()) {
            decoded = cacheAndDecodeSourceData(data);
        } else {
            long startTime = LogTime.getLogTime();
            decoded = loadProvider.getSourceDecoder().decode(data, width, height);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Decoded from source", startTime);
            }
        }
        return decoded;
    }
   
```

这里 的loadProvider.getSourceDecoder()实际上调用的正是GifDrawableLoadProvider的GifResourceDecoder类。

```
GifResourceDecoder
    private final GifHeaderParserPool parserPool;
    private final GifDecoderPool decoderPool;
    
    @Override
    public GifDrawableResource decode(InputStream source, int width, int height) {
        byte[] data = inputStreamToBytes(source);
        final GifHeaderParser parser = parserPool.obtain(data);
        final GifDecoder decoder = decoderPool.obtain(provider);
        try {
            return decode(data, width, height, parser, decoder);
        } finally {
            parserPool.release(parser);
            decoderPool.release(decoder);
        }
    }
    
        private GifDrawableResource decode(byte[] data, int width, int height, GifHeaderParser parser, GifDecoder decoder) {
        final GifHeader header = parser.parseHeader();
        if (header.getNumFrames() <= 0 || header.getStatus() != GifDecoder.STATUS_OK) {
            // If we couldn't decode the GIF, we will end up with a frame count of 0.
            return null;
        }

        Bitmap firstFrame = decodeFirstFrame(decoder, header, data);
        if (firstFrame == null) {
            return null;
        }

        Transformation<Bitmap> unitTransformation = UnitTransformation.get();

        GifDrawable gifDrawable = new GifDrawable(context, provider, bitmapPool, unitTransformation, width, height,
                header, data, firstFrame);

        return new GifDrawableResource(gifDrawable);
    }
```

从上述代码中可以看到在decode时调用了GifHeaderParserPool中的GifHeaderParser和GifDecoderPool的GifDecoder。深入代码的话会发现Glide的解析和编码操作都是Glide自己写的。

```
public class GifHeaderParser {

private ByteBuffer rawData;
private GifHeader header;

}
```

```
public class GifHeader {

    int[] gct = null;
    int status = GifDecoder.STATUS_OK;
    int frameCount = 0;

    GifFrame currentFrame;
    List<GifFrame> frames = new ArrayList<GifFrame>();
    
    }
```

```
class GifFrame {
    int ix, iy, iw, ih;
    /** Control Flag. */
    boolean interlace;
    /** Control Flag. */
    boolean transparency;
    /** Disposal Method. */
    int dispose;
    /** Transparency Index. */
    int transIndex;
    /** Delay, in ms, to next frame. */
    int delay;
    /** Index in the raw buffer where we need to start reading to decode. */
    int bufferFrameStart;
    /** Local Color Table. */
    int[] lct;
}

```

GifHeader、GifFrame都是和Gif格式有关的Data类型，而GifHeaderParser、GifDecoder才是具体的编码的实现。

##### 4.Gif图的编码

```
    private Resource<T> decodeFromSourceData(A data) throws IOException {
        final Resource<T> decoded;
        if (diskCacheStrategy.cacheSource()) {
            decoded = cacheAndDecodeSourceData(data);
        } else {
            long startTime = LogTime.getLogTime();
            decoded = loadProvider.getSourceDecoder().decode(data, width, height);
            if (Log.isLoggable(TAG, Log.VERBOSE)) {
                logWithTimeAndKey("Decoded from source", startTime);
            }
        }
        return decoded;
    }
    
        private Resource<T> cacheAndDecodeSourceData(A data) throws IOException {
        long startTime = LogTime.getLogTime();
        SourceWriter<A> writer = new SourceWriter<A>(loadProvider.getSourceEncoder(), data);
        diskCacheProvider.getDiskCache().put(resultKey.getOriginalKey(), writer);
        if (Log.isLoggable(TAG, Log.VERBOSE)) {
            logWithTimeAndKey("Wrote source to cache", startTime);
        }

        startTime = LogTime.getLogTime();
        Resource<T> result = loadFromCache(resultKey.getOriginalKey());
        if (Log.isLoggable(TAG, Log.VERBOSE) && result != null) {
            logWithTimeAndKey("Decoded source from cache", startTime);
        }
        return result;
    }
```

同样的DecodeJob.java的decodeFromSourceData，在从网络请求数据后会执行cacheAndDecodeSourceData方法，在cacheAndDecodeSourceData会调用SourceWriter将Gif图写在磁盘里，那么使用的具体的编码是实际上是`GifDrawableLoadProvider `的`GifResourceEncoder`类。这里不再细究编码的具体实现了。