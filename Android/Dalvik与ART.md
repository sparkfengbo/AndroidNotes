**参考文章**

- [Android开发——JVM、Dalvik以及ART的区别](http://blog.csdn.net/seu_calvin/article/details/52354964)
- [ART 和 Dalvik](https://source.android.com/devices/tech/dalvik/)
- [Android 中的Dalvik和ART是什么，有啥区别？](http://www.jianshu.com/p/58f817d176b7)
- [Dalvik虚拟机详解（上）](https://blog.csdn.net/u012481172/article/details/50890216)



**Dalvik和ART区别摘要**

- 1）在Dalvik下，应用每次运行都需要通过即时编译器（JIT）将字节码转换为机器码，即每次都要编译加运行，这虽然会使安装过程比较快，但是会拖慢应用以后每次启动的效率。而在ART 环境中，应用在第一次安装的时候，字节码就会预编译（AOT）成机器码，这样的话，虽然设备和应用的首次启动（安装慢了）会变慢，但是以后每次启动执行的时候，都可以直接运行，因此运行效率会提高。
- 2）ART占用空间比Dalvik大（字节码变为机器码之后，可能会增加10%-20%），这也是著名的“空间换时间大法"。
- 3）预编译也可以明显改善电池续航，因为应用程序每次运行时不用重复编译了，从而减少了 CPU 的使用频率，降低了能耗。 



**Dalvik和JVM有啥关系？主要区别：**

- Dalvik是基于寄存器的，而JVM是基于栈的。
- Dalvik运行dex文件，而JVM运行java字节码

自Android 2.2开始，Dalvik支持JIT（just-in-time，即时编译技术）。



**如何理解基于寄存器和基于栈的VM？**

参考[基于栈虚拟机和基于寄存器虚拟机的比较](https://blog.csdn.net/u012481172/article/details/50904574)。

基于栈的VM，其操作数都是保存在Stack数据结构中，从栈中取出数据、计算然后再将结果存入栈中（LIFO，Last in first out）。

如下就是一个典型的计算20+7在栈中的计算过程：

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/davik1.png?raw=true)

```
   1、POP 20
   2、POP 7
   3、ADD 20, 7, result
   4、PUSH result
```



基于寄存器的VM，它们的操作数是存放在CPU的寄存器的。没有入栈和出栈的操作和概念。但是执行的指令就需要包含操作数的地址了，也就是说，指令必须明确的包含操作数的地址，这不像栈可以用栈指针去操作。

![](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/Android/davik2.png?raw=true)

`ADD R1, R2, R3 ;`就一条指令搞定了。正如前面所说，基于寄存器的VM没有入栈和出栈的操作。所以加法指令只需要一行就够了，但是不像Stack-Based一样，我们需要明确的制定操作数R1、R2、R3（这些都是寄存器）的地址。这种设计的有点就是去掉了入栈和出栈的操作，并且指令在寄存器虚拟机执行得更快。

基于寄存器得虚拟机还有一个优点就是一些在基于Stack的虚拟机中无法实现的优化，比如，在代码中有一些相同的减法表达式，那么寄存器只需要计算一次，然后将结果保存，如果之后还有这种表达式需要计算就直接返回结果。这样就减少了重复计算所造成的开销。

当然，寄存器虚拟机也有一些问题，比如虚拟机的指令比Stack vm指令要长（因为Register指令包含了操作数地址）。

-----

**优化后的Dalvik较其他标准虚拟机存在一些不同特性:**

- 1.占用更少空间　
- 2.为简化翻译，常量池只使用32位索引　　
- 3.标准Java字节码实行8位堆栈指令,Dalvik使用16位指令集直接作用于局部变量。局部变量通常来自4位的“虚拟寄存器”区。这样减少了Dalvik的指令计数，提高了翻译速度。

当Android启动时，Dalvik VM 监视所有的程序（APK），并且创建依存关系树，为每个程序优化代码并存储在Dalvik缓存中。Dalvik第一次加载后会生成Cache文件，以提供下次快速加载，所以第一次会很慢。

Dalvik解释器采用预先算好的Goto地址，每个指令对内存的访问都在64字节边界上对齐。这样可以节省一个指令后进行查表的时间。为了强化功能, Dalvik还提供了快速翻译器（Fast Interpreter）。
