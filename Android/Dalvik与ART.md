-[ Android开发——JVM、Dalvik以及ART的区别](http://blog.csdn.net/seu_calvin/article/details/52354964)

> 
- 1）在Dalvik下，应用每次运行都需要通过即时编译器（JIT）将字节码转换为机器码，即每次都要编译加运行，这虽然会使安装过程比较快，但是会拖慢应用以后每次启动的效率。而在ART 环境中，应用在第一次安装的时候，字节码就会预编译（AOT）成机器码，这样的话，虽然设备和应用的首次启动（安装慢了）会变慢，但是以后每次启动执行的时候，都可以直接运行，因此运行效率会提高。
- 2）ART占用空间比Dalvik大（字节码变为机器码之后，可能会增加10%-20%），这也是著名的“空间换时间大法"。
- 3）预编译也可以明显改善电池续航，因为应用程序每次运行时不用重复编译了，从而减少了 CPU 的使用频率，降低了能耗。 

- [Android 中的Dalvik和ART是什么，有啥区别？](http://www.jianshu.com/p/58f817d176b7)

>
Dalvik和JVM有啥关系？主要区别：
>
- Dalvik是基于寄存器的，而JVM是基于栈的。
- Dalvik运行dex文件，而JVM运行java字节码
自Android 2.2开始，Dalvik支持JIT（just-in-time，即时编译技术）。
优化后的Dalvik较其他标准虚拟机存在一些不同特性:　
- 1.占用更少空间　
- 2.为简化翻译，常量池只使用32位索引　　
- 3.标准Java字节码实行8位堆栈指令,Dalvik使用16位指令集直接作用于局部变量。局部变量通常来自4位的“虚拟寄存器”区。这样减少了Dalvik的指令计数，提高了翻译速度。　
　当Android启动时，Dalvik VM 监视所有的程序（APK），并且创建依存关系树，为每个程序优化代码并存储在Dalvik缓存中。Dalvik第一次加载后会生成Cache文件，以提供下次快速加载，所以第一次会很慢。
　Dalvik解释器采用预先算好的Goto地址，每个指令对内存的访问都在64字节边界上对齐。这样可以节省一个指令后进行查表的时间。为了强化功能, Dalvik还提供了快速翻译器（Fast Interpreter）。

-[ART 和 Dalvik](https://source.android.com/devices/tech/dalvik/)