- 1.[安卓自定义View基础-坐标系](http://www.gcssloop.com/customview/CoordinateSystem)
 - 1	在数学中常见的坐标系与屏幕默认坐标系的差别
 - 2	View的坐标系是相对于父控件而言的
 - 3	MotionEvent中get和getRaw的区别

 ----
 
- 2.[安卓自定义View基础-颜色](http://www.gcssloop.com/customview/Color)
 - 颜色种类和使用
 - Xfermode
 - ![](http://ww4.sinaimg.cn/large/005Xtdi2gw1f1wa0f0mzjj30hh0fsjt8.jpg)

 
---

- 3.[安卓自定义View进阶-分类与流程](http://www.gcssloop.com/customview/CustomViewProcess)
	- 包括自定义View的流程
	- 构造方法、onMeasure、onSizeChanged、onLayout方法的介绍
	- ![自定义View绘制流程函数调用链(简化版)](http://ww4.sinaimg.cn/large/005Xtdi2jw1f638wreu74j30fc0heaay.jpg)

----

- 4.关于Canvas的使用

- [安卓自定义View进阶-Canvas之绘制图形](http://www.gcssloop.com/customview/Canvas_BasicGraphics)
	- Canvas的常用画图API
	- Canvas的详解	（各API的使用）
	- Canvas的save、restore、translate、rotate等
	- **自定义饼状图 PieView的小例子**
- [安卓自定义View进阶-Canvas之画布操作](http://www.gcssloop.com/customview/Canvas_Convert)
	- Canvas的位移(translate)、缩放(scale)、旋转(rotate)、错切(skew)、快照(save)和回滚(restore)
- [安卓自定义View进阶-Canvas之图片文字](http://www.gcssloop.com/customview/Canvas_PictureText)



----

- 5.关于Path的使用

	- [安卓自定义View进阶-Path之基本操作](http://www.gcssloop.com/customview/Path_Basic/)
	- [安卓自定义View进阶-Path之贝塞尔曲线](http://www.gcssloop.com/customview/Path_Bezier)
		- [三次贝塞尔曲线练习之弹性的圆
](https://www.jianshu.com/p/791d3a791ec2)
	- [安卓自定义View进阶-Path之完结篇](http://www.gcssloop.com/customview/Path_Over)
	- [安卓自定义View进阶-PathMeasure](http://www.gcssloop.com/customview/Path_PathMeasure)

	
	
----

- 6.关于Matrix的使用	
	- [安卓自定义View进阶-Matrix原理](http://www.gcssloop.com/customview/Matrix_Basic)
	- [安卓自定义View进阶-Matrix详解](http://www.gcssloop.com/customview/Matrix_Method)
	- [安卓自定义View进阶-Matrix Camera](http://www.gcssloop.com/customview/matrix-3d-camera)

----

- 7.触控相关

	- [安卓自定义View进阶-MotionEvent详解](http://www.gcssloop.com/customview/motionevent)	
	- [安卓自定义View进阶-特殊控件的事件处理方案](http://www.gcssloop.com/customview/touch-matrix-region)
	- [安卓自定义View进阶-多点触控详解](http://www.gcssloop.com/customview/multi-touch)
	- [安卓自定义View进阶-手势检测(GestureDecetor)](http://www.gcssloop.com/customview/gestruedector)

----

- 8.View的事件分发
	- [安卓自定义View进阶-事件分发机制原理](http://www.gcssloop.com/customview/dispatch-touchevent-theory)
		- 传递顺序：`Activity －> PhoneWindow －> DecorView －> ViewGroup －> ... －> View`
		- View的处理顺序 : `事件的调度顺序应该是 onTouchListener > onTouchEvent > onLongClickListener > onClickListener`
	- [安卓自定义View进阶-事件分发机制详解](http://www.gcssloop.com/customview/dispatch-touchevent-source)	
		- 有一些要点还是需要看看的，比如重叠区域点击、子view的xml设置了clickable为true，拦截了父view的onClick事件、View 只有消费了 ACTION_DOWN 事件，才能接收到后续的事件等。
	- [图解 Android 事件分发机制](https://www.jianshu.com/p/e99b5e8bd67b)

----


- 9.自定义View与自定义ViewGroup

 	- [教你步步为营掌握自定义View
](https://www.jianshu.com/p/d507e3514b65)
	- [手把手教你写一个完整的自定义View](https://www.jianshu.com/p/e9d8420b1b9c)
	- [自定义View Measure过程](https://www.jianshu.com/p/1dab927b2f36)
	- [自定义View Layout过程](https://www.jianshu.com/p/158736a2549d)
	- [自定义View Draw过程](https://www.jianshu.com/p/95afeb7c8335)
	- [教你步步为营掌握自定义ViewGroup](https://www.jianshu.com/p/5e61b6af4e4c)

	