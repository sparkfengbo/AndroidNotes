### 保存与恢复过程

Activity onSaveInstanceState

参考 《Andriod源码分析设计模式解析与实战》 P250 ￥13.6


>如果View没有ID，是不会保存这个View的状态的


### onSaveInstanceState 调用时机

当某个Activity变得容易被系统销毁时，该Activity的onSaveInstanceState方法会被执行，除非该Activity是被用户主动销毁的，如当用户按Back键时。

意思是 该Activity还没有被销毁，而仅仅是一种可能性

- 当用户按下Home键时
- 长按Home键，选择运行其他的程序时
- 按下电源键关闭屏幕时
- 从Activity A 中启动新的Activity时
- 屏幕方向切换时
- 电话打入等情况

一句话概括就是 不是用户主动退出某个Activity，或者跳转到其他Activity时会触发