参考：

- [Android图片加载框架最全解析（五），Glide强大的图片变换功能](http://blog.csdn.net/guolin_blog/article/details/71524668)

加之自己屡着源码，写笔记。

------

#### 1.寻找图片变换的实现类

```
Glide.with(this)
     .load(url)
     .centerCrop()
     .into(imageView);

Glide.with(this)
     .load(url)
     .fitCenter()
     .into(imageView);
```

在调用centerCrop和fitCenter时，根据构建的RequestBuilder类型不同，进入不同的方法。

```
//DrawableRequestBuilder.java
    public DrawableRequestBuilder<ModelType> centerCrop() {
        return transform(glide.getDrawableCenterCrop());
    }
    public DrawableRequestBuilder<ModelType> fitCenter() {
        return transform(glide.getDrawableFitCenter());
    }
        public DrawableRequestBuilder<ModelType> transform(Transformation<GifBitmapWrapper>... transformation) {
        super.transform(transformation);
        return this;
    }
```

```
//GifRequestBuilder.java

    public GifRequestBuilder<ModelType> centerCrop() {
        return transformFrame(glide.getBitmapCenterCrop());
    }
    
    public GifRequestBuilder<ModelType> fitCenter() {
        return transformFrame(glide.getBitmapFitCenter());
    }
    
        public GifRequestBuilder<ModelType> transformFrame(BitmapTransformation... bitmapTransformations) {
        return transform(toGifTransformations(bitmapTransformations));
    }
    
        @Override
    public GifRequestBuilder<ModelType> transform(Transformation<GifDrawable>... transformations) {
        super.transform(transformations);
        return this;
    }
    
    
```
可以看到虽然最后根据加载图像类型的不同，返回不同的RequestBuilder，但是都会走到父类GenericRequestBuilder的transform方法。


```
GenericRequestBuilder.java
    public GenericRequestBuilder<ModelType, DataType, ResourceType, TranscodeType> transform(
            Transformation<ResourceType>... transformations) {
        isTransformationSet = true;
        if (transformations.length == 1) {
            transformation = transformations[0];
        } else {
            transformation = new MultiTransformation<ResourceType>(transformations);
        }

        return this;
    }
```

可以看到transform方法只是把我们传入的transformations保存在自身的变量中。

我们看看transform传入参数时所调用的Glide的getDrawableCenterCrop方法 

```
Glide.java

    private final CenterCrop bitmapCenterCrop;
    private final GifBitmapWrapperTransformation drawableCenterCrop;

	Glide(...) {
	
	...
	bitmapCenterCrop = new CenterCrop(bitmapPool);
        drawableCenterCrop = new GifBitmapWrapperTransformation(bitmapPool, bitmapCenterCrop);
	
	}
    GifBitmapWrapperTransformation getDrawableCenterCrop() {
        return drawableCenterCrop;
    }
```

可以看到在初始化Glide时创建了类型是CenterCrop的对象，并将Bitmap池放在了CenterCrop中，将位图池和CenterCrop类都放在了GifBitmapWrapperTransformation中进行了包装。

我们先看GifBitmapWrapperTransformation类

```
public class GifBitmapWrapperTransformation implements Transformation<GifBitmapWrapper> {
    private final Transformation<Bitmap> bitmapTransformation;
    private final Transformation<GifDrawable> gifDataTransformation;

    public GifBitmapWrapperTransformation(BitmapPool bitmapPool, Transformation<Bitmap> bitmapTransformation) {
        this(bitmapTransformation, new GifDrawableTransformation(bitmapTransformation, bitmapPool));
    }

    GifBitmapWrapperTransformation(Transformation<Bitmap> bitmapTransformation,
            Transformation<GifDrawable> gifDataTransformation) {
        this.bitmapTransformation = bitmapTransformation;
        this.gifDataTransformation = gifDataTransformation;
    }

    @Override
    public Resource<GifBitmapWrapper> transform(Resource<GifBitmapWrapper> resource, int outWidth, int outHeight) {
        Resource<Bitmap> bitmapResource = resource.get().getBitmapResource();
        Resource<GifDrawable> gifResource = resource.get().getGifResource();
        if (bitmapResource != null && bitmapTransformation != null) {
            Resource<Bitmap> transformed = bitmapTransformation.transform(bitmapResource, outWidth, outHeight);
            if (!bitmapResource.equals(transformed)) {
                GifBitmapWrapper gifBitmap = new GifBitmapWrapper(transformed, resource.get().getGifResource());
                return new GifBitmapWrapperResource(gifBitmap);
            }
        } else if (gifResource != null && gifDataTransformation != null) {
            Resource<GifDrawable> transformed = gifDataTransformation.transform(gifResource, outWidth, outHeight);
            if (!gifResource.equals(transformed)) {
                GifBitmapWrapper gifBitmap = new GifBitmapWrapper(resource.get().getBitmapResource(), transformed);
                return new GifBitmapWrapperResource(gifBitmap);
            }
        }
        return resource;
    }

    @Override
    public String getId() {
        return bitmapTransformation.getId();
    }
}
```

我们只要关注transform方法就好了（源码就不想分析了，实际在调用transform的地方在DecodeJob.java的transformEncodeAndTranscode方法，使用的就是实现了Transformation接口的GifBitmapWrapperTransformation或者其他的类）。

可以看到transform方法中先处理Bitmap类型的图片再处理GifDrawable类型图片。
很关键的一句代码是bitmapTransformation.transform。bitmapTransformation的类型是CenterCrop。

```
public class CenterCrop extends BitmapTransformation {

    public CenterCrop(Context context) {
        super(context);
    }

    public CenterCrop(BitmapPool bitmapPool) {
        super(bitmapPool);
    }

    // Bitmap doesn't implement equals, so == and .equals are equivalent here.
    @SuppressWarnings("PMD.CompareObjectsWithEquals")
    @Override
    protected Bitmap transform(BitmapPool pool, Bitmap toTransform, int outWidth, int outHeight) {
        final Bitmap toReuse = pool.get(outWidth, outHeight, toTransform.getConfig() != null
                ? toTransform.getConfig() : Bitmap.Config.ARGB_8888);
        Bitmap transformed = TransformationUtils.centerCrop(toReuse, toTransform, outWidth, outHeight);
        if (toReuse != null && toReuse != transformed && !pool.put(toReuse)) {
            toReuse.recycle();
        }
        return transformed;
    }

    @Override
    public String getId() {
        return "CenterCrop.com.bumptech.glide.load.resource.bitmap";
    }
}
```

可以看到实际上进行特殊变换的实现类实际上是TransformationUtils，请看代码

```
TransformationUtils.java
    public static Bitmap centerCrop(Bitmap recycled, Bitmap toCrop, int width, int height) {
        if (toCrop == null) {
            return null;
        } else if (toCrop.getWidth() == width && toCrop.getHeight() == height) {
            return toCrop;
        }
        // From ImageView/Bitmap.createScaledBitmap.
        final float scale;
        float dx = 0, dy = 0;
        Matrix m = new Matrix();
        if (toCrop.getWidth() * height > width * toCrop.getHeight()) {
            scale = (float) height / (float) toCrop.getHeight();
            dx = (width - toCrop.getWidth() * scale) * 0.5f;
        } else {
            scale = (float) width / (float) toCrop.getWidth();
            dy = (height - toCrop.getHeight() * scale) * 0.5f;
        }

        m.setScale(scale, scale);
        m.postTranslate((int) (dx + 0.5f), (int) (dy + 0.5f));
        final Bitmap result;
        if (recycled != null) {
            result = recycled;
        } else {
            result = Bitmap.createBitmap(width, height, getSafeConfig(toCrop));
        }

        // We don't add or remove alpha, so keep the alpha setting of the Bitmap we were given.
        TransformationUtils.setAlpha(toCrop, result);

        Canvas canvas = new Canvas(result);
        Paint paint = new Paint(PAINT_FLAGS);
        canvas.drawBitmap(toCrop, m, paint);
        return result;
    }
```

这个算法这里就不赘述了。

#### 2.自定义图片变换


仿照CenterCrop类实现BitmapTransformation接口就可以。


glide-transformations的项目主页地址是 ![https://github.com/wasabeef/glide-transformations](https://github.com/wasabeef/glide-transformations)。

```
Glide.with(this)
     .load(url)
     .bitmapTransform(new BlurTransformation(this))
     .into(imageView);
```

![](http://img.blog.csdn.net/20170823205319689?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

```
Glide.with(this)
     .load(url)
     .bitmapTransform(new GrayscaleTransformation(this))
     .into(imageView);
```

![](http://img.blog.csdn.net/20170823205616112?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvZ3VvbGluX2Jsb2c=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)