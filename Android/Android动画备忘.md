Android动画分为三种：
1.View动画
2.帧动画
3.属性动画

####1.View动画(View Animation)
[官方文档](https://developer.android.com/guide/topics/graphics/view-animation.html)

|--名称--|--标签--|--子类--|
|---|---|---|
|平移|<translate>|TranslateAnimation|
|缩放|<scale>|ScaleAnimation|
|旋转|<rotate>|RotateAnimation|
|透明度|<alpha>|AlphaAnimation|

Button btn = findViewById...
Animation anim = AnimationUtils.loadAnimation(this, R.anim.animation_test);
btn.startAnimation(anim);



```
/**保存在/res/anim/animation_test.xml**/
<set android:shareInterpolator="false">    
    <scale        
      android:interpolator="@android:anim/accelerate_decelerate_interpolator"        
      android:fromXScale="1.0"        
      android:toXScale="1.4"       
      android:fromYScale="1.0"        
      android:toYScale="0.6"        
      android:pivotX="50%"        
      android:pivotY="50%"        
      android:fillAfter="false"        
      android:duration="700" />    
    <set android:interpolator="@android:anim/decelerate_interpolator">              
        <scale           
            android:fromXScale="1.4"           
            android:toXScale="0.0"           
            android:pivotX="50%"           
            android:pivotY="50%"           
            android:startOffset="700"          
            android:duration="400"           
            android:fillBefore="false" />        
        <rotate           
            android:fromDegrees="0"           
            android:toDegrees="-45"                   
            android:startOffset="700"           
            android:duration="400" />    
     </set>
</set>
```

--或者，自定创建对应的Animation或者AnimationSet


####2.帧动画(Drawable Animation)
[官方文档](https://developer.android.com/guide/topics/graphics/drawable-animation.html)

容易引起OOM，避免使用较大的图片

```
/**保存在res/drawable/ rocket_thrust.xml**/
<animation-list xmlns:android="http://schemas.android.com/apk/res/android"    android:oneshot="true">    
    <item android:drawable="@drawable/rocket_thrust1" android:duration="200" />    
    <item android:drawable="@drawable/rocket_thrust2" android:duration="200" />    
    <item android:drawable="@drawable/rocket_thrust3" android:duration="200" />
</animation-list>
```

```
AnimationDrawable rocketAnimation;
public void onCreate(Bundle savedInstanceState) {      
    super.onCreate(savedInstanceState);      
    setContentView(R.layout.main);  
    ImageView rocketImage = (ImageView) findViewById(R.id.rocket_image);  
    rocketImage.setBackgroundResource(R.drawable.rocket_thrust);  
    rocketAnimation = (AnimationDrawable)rocketImage.getBackground();
}
public boolean onTouchEvent(MotionEvent event) { 
    if (event.getAction() == MotionEvent.ACTION_DOWN) {    
        rocketAnimation.start();    
        return true;  
    }  
    return super.onTouchEvent(event);
}
```
>`start()`不能在`onCreate()`方法调用， 因为AnimationDrawable还没有attach到window. 如果你想立即执行动画，你可以在`onWindowFocusChanged()`中调用`start()`

####3.属性动画（Property Animation）
[官方文档](https://developer.android.com/guide/topics/graphics/prop-animation.html)

**ValueAnimator**：继承自**Animator**，计算值。

```
ValueAnimator anim = ValueAnimator.ofFloat(0f, 1f);  
anim.setDuration(300);  
anim.addUpdateListener(new ValueAnimator.AnimatorUpdateListener() {  
    @Override  
    public void onAnimationUpdate(ValueAnimator animation) {  
        float currentValue = (float) animation.getAnimatedValue();  
        Log.d("TAG", "cuurent value is " + currentValue);  
    }  
});  
anim.start();  
```
**ObjectAnimator**：继承自**ValueAnimator**，计算值并绑定到对象。

```
ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f, 1f);  
animator.setDuration(5000);  
animator.start();  

ObjectAnimator animator = ObjectAnimator.ofFloat(textview, "scaleY", 1f, 3f, 1f);  
animator.setDuration(5000);  
animator.start();  
```

**AnimatorSet**：该类提供了多个属性动画“协调工作”的途径，不同属性动画之间可以“同步执行”，亦可以“依次执行”，或者“延时执行”。
```
ObjectAnimator moveIn = ObjectAnimator.ofFloat(textview, "translationX", -500f, 0f);  
ObjectAnimator rotate = ObjectAnimator.ofFloat(textview, "rotation", 0f, 360f);  
ObjectAnimator fadeInOut = ObjectAnimator.ofFloat(textview, "alpha", 1f, 0f, 1f);  
AnimatorSet animSet = new AnimatorSet();  
animSet.play(rotate).with(fadeInOut).after(moveIn);  
animSet.setDuration(5000);  
animSet.start();  
```

- after(Animator anim)   将现有动画插入到传入的动画之后执行
- after(long delay)   将现有动画延迟指定毫秒后执行
- before(Animator anim)   将现有动画插入到传入的动画之前执行
- with(Animator anim)   将现有动画和传入的动画同时执行

----

或：

```
<set android:ordering="sequentially">    
    <set>        
        <objectAnimator            
            android:propertyName="x"            
            android:duration="500"            
            android:valueTo="400"            
            android:valueType="intType"/>        
      <objectAnimator            
            android:propertyName="y"            
            android:duration="500"            
            android:valueTo="300"            
            android:valueType="intType"/>    
    </set>    
    <objectAnimator         
            android:propertyName="alpha"        
            android:duration="500"        
            android:valueTo="1f"/>
</set>
```
```
AnimatorSet set = (AnimatorSet) AnimatorInflater.loadAnimator(this, R.anim.property_animator);
set.setTarget(mButton);
set.start();
```

**另外：属性动画要求动画作用的对象必须提供该属性的get和set方法**

解决：

- 修改代码，增加set和get
- 用一个类去包装对象
- 用ValueAnimator去监听动画执行的过程


------

**估值器**
- IntEvaluator
- FloatEvaluator
- ArgbEvaluator

----

**插值器**
- AccelerateDecelerateInterpolator
![AccelerateDecelerateInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-506e9f3b1c7975ee.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
- AccelerateInterpolator
![AccelerateInterpolator](http://upload-images.jianshu.io/upload_images/952890-d7bec0940375b5a3.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- AnticipateInterpolator

![AnticipateInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-068ab7741b87b5c1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- AnticipateInterpolator
- AnticipateOvershootInterpolator

![AnticipateOvershootInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-f8e99a0f2c4b81b0.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- BounceInterpolator


![BounceInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-8bf94ed3bb3cd9ec.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- CycleInterpolator


![CycleInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-b093baf6832f3b6c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- DecelerateInterpolator


![DecelerateInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-1d43c4c7e7a9a264.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- LinearInterpolator


![LinearInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-776023dead32791e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

- OvershootInterpolator
![OvershootInterpolator.png](http://upload-images.jianshu.io/upload_images/952890-e0f003db8e14806f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

----

####其他
**1.LayoutAnimation**
作用于ViewGroup，子元素都会有这个动画效果。具体看APIDemo，里面的比较有代表性。
```
<layoutAnimation 
    xmlns:android="http://schemas.android.com/apk/res/android" 
    android:animation="@anim/anim_item" 
    android:animationOrder="normal" 
    android:delay="0.5">
</layoutAnimation>
```
在xml布局文件中指定android:layoutAnimation属性：
```

<ListView android:layout_width="match_parent"   
    android:layout_height="wrap_content"     
    android:background="#fff4f7f9" 
    android:cacheColorHint="#00000000" 
    android:divider="#dddbdb" 
    android:dividerHeight="1.0px" 
    android:layoutAnimation="@anim/layout_animation" 
    android:listSelector="@android:color/transparent"/>
```
```
ListView listview = (ListView) findViewById(R.id.listview);
Animation animation = AnimationUtils.loadAnimation(TestAnimActivity.this, R.anim.anim_item);
LayoutAnimationController controller = new LayoutAnimationController(animation);
controller.setDelay(0.5f);
controller.setOrder(LayoutAnimationController.ORDER_NORMAL);
listview.setLayoutAnimation(controller);
```

**2.Activity**
overridePendingTransition(int enterAnim, int exitAnim);
**在startActivity或finish之后调用才有效。**