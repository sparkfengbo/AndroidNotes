摘自[Understanding the Java ClassLoader](http://www.ibm.com/developerworks/java/tutorials/j-classloader/j-classloader.html)，总结主要的关于ClassLoader的内容。


-------
### 什么是ClassLoader
&nbsp;&nbsp;&nbsp;&nbsp;Java是平台无关的语言，Java的代码是运行在JVM上的，这就意味着程序被编译成一种特殊的与平台无关的格式。Java的程序和用C或C++写的不同，它并不是单独一个可执行文件，而是很多独立的类文件组合成的，每一个文件和单独的Java类对应。并且，这些类文件不是同时全部加载到内存中，而是需要的时候再加载。ClassLoader是JVM中的一部分，并且Java ClassLoader使用Java语言写的，这意味着你可以自定义ClassLoader而不用去理解JVM内部的细节。

### 为什么自定义ClassLoader
&nbsp;&nbsp;&nbsp;&nbsp;JVM里面有一个ClassLoader了，为什么我们还要自定义一个ClassLoader？默认的ClassLoader只知道如何从本地的文件系统加载类文件，但你的Java程序完全在你的电脑上编译等待加载时默认的ClassLoader可以很好的完成任务,
但是Java语言比较创新的一个特点是可以使JVM很简便的从本地硬盘或网络之外的地方获得类。

**你可以通过自定义的ClassLoader完成：**
- 在执行不信任的代码前自动的完成签名校验
- 使用密码解密代码（Transparently）
- 根据用户特定的需要动态构建类

任何你能想象到的可以生成Java字节码的事情够可以集成到你的应用中。

#### ClassLoader结构概览
&nbsp;&nbsp;&nbsp;&nbsp;ClassLoader最基本的目标是对一个类进行请求，JVM需要一个类，JVM让ClassLoader通过名字去查找类返回对应的`Class`对象。
下面是ClassLoader中重要的方法：

**loadClass**

`Class loadClass( String name, boolean resolve );`
`name`是要加载的类的名称，需要加包的路径。`resolve`参数是否类需要被解析，类解析（class resolution）可以看成是执行的类的完整的准备任务。

**defineClass**

这个方法时ClassLoader的迷之方法，这个方法能够通过一组字节串（raw array of bytes）并将其转成`Class`对象。
`defineClass`维护了很多复杂、神秘、实现相关的JVM的方面。它将字节码格式转成运行时的数据结构，检查合法性等。final方法，不能重写。

**findSystemClass**

从本地文件系统加载文件。如果存在文件，它通过使用`defineClass`将文件原始的字节转换成`Class`对象。这就是当你运行Java程序时JVM如何正常的加载类的实现机制。

对于自定义的ClassLoader，我们只有在尝试使用所有其他加载类的方式后再使用`findSystemClass`。原因很简单，自定义的ClassLoader负责特殊记载类的步骤，而不是所有类。比如，即使我们从远程服务器加载一些类，但这些类可能使用大量在本地的基础的Java类库，这些类也要被加载。而这些类我们并不关心，所以让JVM用默认的方式加载，就是通过本地文件系统加载。

流程是这样的：
- 自定义的ClassLoader去加载类
- 验证远程服务器是否有那个类
- 有的话，拿到类加载
- 没有的话，我们假设这个类是基础Java类库里面的类，使用`findSystemClass`加载

**resolveClass**

As I mentioned previously, loading a class can be done partially (without resolution) or completely (with resolution). When we write our version of loadClass, we may need to call resolveClass, depending on the value of the resolve parameter to loadClass.

**findLoadedClass**

作为一个缓存使用，当loadClass加载类，使用这个方法查看是否已经被ClassLoader加载，解决重复加载已经被加载的类的问题。

我们事先loadClass的例子的流程：
- 调用`findLoadedClass`查看是否已经被加载
- 如果没有加载，通过某种途径得到原始的字节数据
- 如果得到了原始的字节数据，调用`defineClass`将其转成`Class`对象
- 如果没有原始的字节数据，调用`findSystemClass`查看是否可以从本地文件系统得到
- 如果`resovle`参数是true，调用`resolveClass`去链接class对象
- 如果仍然没有class对象，抛出`ClassNotFoundException`异常
- 否则，返回这个类给调用者。

------
#### ClassLoader在Java2的变化

**New method**：findClass

This new method is called by the default implementation of loadClass. The purpose of findClass is to contain all your specialized code for your ClassLoader, without having to duplicate the other code (such as calling the system ClassLoader when your specialized method has failed).


**New method**：getSystemClassLoader

无论你是否重写findClass或loadClass，getSystemClassLoader能够使你以ClassLoader对象的形式直接获得系统的ClassLoader，而不是通过findSystemClass调用。


**New method**：getParent

新方法允许ClassLoader获得它的父ClassLoader，这样可以将类代理给它。当你自定义的ClassLoader不能找到类时可能会用到这个方法


