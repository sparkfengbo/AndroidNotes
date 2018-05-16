[TOC]


**参考**

周志明. 深入理解Java虚拟机：JVM高级特性与最佳实践（第2版） (原创精品系列) . 机械工业出版社. Kindle 版本. 

----

>Java语言的“编译期”其实是一段“不确定”的操作过程，因为它可能是指一个前端编译器（其实叫“编译器的前端”更准确一些）把`*.java`文件转变成`*.class`文件的过程；也可能是指虚拟机的后端运行期编译器（JIT编译器，JustInTimeCompiler）把字节码转变成机器码的过程；还可能是指使用静态提前编译器（AOT编译器，AheadOfTimeCompiler）直接把`*.java`文件编译成本地机器代码的过程。
>
>下面摘取的内容中编译期间仅为Javac编译，不会有直接优化的内容。


## 1.早期（编译期）优化

### 1.1 Javac编译器

![Javac编译过程](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/JVM/Javac%E7%BC%96%E8%AF%91%E8%BF%87%E7%A8%8B.png?raw=true)

- 解析与填充符号表过程
 
 - 词法、语法分析
 - 填充符号表
- 插入式注解处理器的注解处理过程
- 分析与字节码生成过程

  - 标注检查
  - 数据及控制流分析
  - 解语法糖
  - 字节码生成

### 1.2 Java语法糖

### 1.2.1 泛型与类型擦除
  
>本质是参数化类型泛型技术
>
>在C#和Java之中的使用方式看似相同，但实现上却有着根本性的分歧
>
>**类型膨胀** : C#里面泛型无论在程序源码中、编译后的IL中（IntermediateLanguage，中间语言，这时候泛型是一个占位符），或是运行期的CLR中，都是切实存在的，List＜int＞与List＜String＞就是两个不同的类型，它们在系统运行期生成，有自己的虚方法表和类型数据，这种实现称为类型膨胀，基于这种方法实现的泛型称为真实泛型。
>
>**类型擦除** : Java语言中的泛型则不一样，它只在程序源码中存在，在编译后的字节码文件中，就已经替换为原来的原生类型（RawType，也称为裸类型）了，并且在相应的地方插入了强制转型代码，因此，对于运行期的Java语言来说，ArrayList＜int＞与ArrayList＜String＞就是同一个类，所以泛型技术实际上是Java语言的一颗语法糖，Java语言中的泛型实现方法称为类型擦除，基于这种方法实现的泛型称为伪泛型。

![类型擦除](https://github.com/sparkfengbo/AndroidNotes/blob/master/PictureRes/JVM/%E7%B1%BB%E5%9E%8B%E6%93%A6%E9%99%A4.png?raw=true)
  
### 1.2.2 自动装箱、自动拆箱与遍历循环

代码清单10-6　自动装箱、拆箱与遍历循环

```
    public static void main(String[] args){
        List＜Integer＞list = Arrays.asList(1,2,3,4);
        //如果 在 JDK 1. 7 中, 还有 另外 一颗 语法 糖 
        // 能 让 上面 这句 代码 进一步 简写 成 List ＜ Integer ＞ list=[ 1, 2, 3, 4]; 
        int sum= 0; 
        for( int i: list) {
            sum+= i; 
        }
        System. out. println( sum); 
    }
```

代码清单10-7　自动装箱、拆箱与遍历循环编译之后



```
 public static void main( String[] args){
        List list= Arrays. asList( new Integer[]{
                Integer. valueOf( 1),
                Integer. valueOf( 2),
                Integer. valueOf( 3),
                Integer. valueOf( 4)
        });
        int sum= 0;
        for( Iterator localIterator= list. iterator(); localIterator. hasNext();){
            int i=(( Integer) localIterator. next()). intValue();
            sum+= i;
        }
        System. out. println( sum);
    }
```



>代码清单10-6中一共包含了泛型、自动装箱、自动拆箱、遍历循环与变长参数5种语法糖，代码清单10-7则展示了它们在编译后的变化。泛型就不必说了，自动装箱、拆箱在编译之后被转化成了对应的包装和还原方法，如本例中的`Integer.valueOf()`与`Integer.intValue()`方法，而遍历循环则把代码还原成了迭代器的实现，这也是为何遍历循环需要被遍历的类实现Iterable接口的原因。最后再看看变长参数，它在调用的时候变成了一个数组类型的参数，在变长参数出现之前，程序员就是使用数组来完成类似功能的。
>
>而且 自动装箱拆箱是发生在编译器，不是在运行期


参考：

- [Java自动装箱的陷阱](https://www.cnblogs.com/liujinhong/p/6288160.html)
- [java 语法糖-自动装箱的陷阱](http://lizhensan.iteye.com/blog/1689176)

```
 1 		public static void main(String args[]) {
 2         Integer a = 1;
 3         Integer b = 2;
 4         Integer c = 3;
 5         Integer d = 3;
 6         Integer e = 321;
 7         Integer f = 321;
 8         Long g = 3L;
 9         int x = 3;
10         long y = 3L;
11         
12         //x,y虽然类型不同但是可以直接进行数值比较
13         System.out.println(x == y);
14         //System.out.println(c == g); 提示出错，不可比较的类型。说明此时没有自动拆箱
15         System.out.println(c == d);
16         System.out.println(e == f);
17         System.out.println(c == (a+b));
18         System.out.println(c.equals(a+b));
19         //此时进行了自动的拆箱
20         System.out.println(g == (a+b));
21         System.out.println(g.equals(a+b));
22     }
```

输出

```
True
True
False
True
True
True
False
```


>在自动装箱时对于值从–128到127之间的值，它们被装箱为Integer对象后，会存在内存中被重用。如果超过了从–128到127之间的值，被装箱后的Integer对象并不会被重用，即相当于每次装箱时都新建一个Integer对象.
>
>
>包装类进行 +、-  等运算操作时会 自动拆箱
>
>鉴于包装类的“==”运算在不遇到算术运算的情况下不会自动拆箱，以及它们equals()方法不处理数据转型的关系，笔者建议在实际编码中尽量避免这样使用自动装箱与拆箱。


### 1.2.3 条件编译

>Java语言当然也可以进行条件编译，方法就是使用条件为常量的if语句。


>除了本节中介绍的泛型、自动装箱、自动拆箱、遍历循环、变长参数和条件编译之外，Java语言还有不少其他的语法糖，如内部类、枚举类、断言语句、对枚举和字符串（在JDK1.7中支持）的switch支持、try语句中定义和关闭资源（在JDK1.7中支持）等


## 2.晚期（运行期）优化

### 2.1 HosSpot即时编译器

#### 2.1.1 解释器与编译器

>Java程序最初是通过解释器（Interpreter）进行解释执行的，当虚拟机发现某个方法或代码块的运行特别频繁时，就会把这些代码认定为“热点代码”（HotSpotCode）。为了提高热点代码的执行效率，在运行时，虚拟机将会把这些代码编译成与本地平台相关的机器码，并进行各种层次的优化，完成这个任务的编译器称为即时编译器（JustInTimeCompiler，下文中简称JIT编译器）。


>解释器与编译器两者各有优势：当程序需要迅速启动和执行的时候，解释器可以首先发挥作用，省去编译的时间，立即执行。在程序运行后，随着时间的推移，编译器逐渐发挥作用，把越来越多的代码编译成本地代码之后，可以获取更高的执行效率。当程序运行环境中内存资源限制较大（如部分嵌入式系统中），可以使用解释执行节约内存，反之可以使用编译执行来提升效率。同时，解释器还可以作为编译器激进优化时的一个“逃生门”，让编译器根据概率选择一些大多数时候都能提升运行速度的优化手段，当激进优化的假设不成立，如加载了新类后类型继承结构出现变化、出现“罕见陷阱”（UncommonTrap）时可以通过逆优化（Deoptimization）退回到解释状态继续执行（部分没有解释器的虚拟机中也会采用不进行激进优化的C1编译器[2]担任“逃生门”的角色），因此，在整个虚拟机执行架构中，解释器与编译器经常配合工作，

#### 2.1.2 编译对象与触发条件

#### 2.1.3 编译过程


#### 2.1.4 查看及分析即时编译结果

### 2.2 编译优化技术

参考11.3.1 罗列了很多优化技术

下面笔记记录 ：

- 语言无关的经典优化技术之一：公共子表达式消除
- 语言相关的经典优化技术之一：数组范围检查消除
- 最重要的优化技术之一：方法内联
- 最前沿的优化技术之一：逃逸分析

#### 2.2.1 公共子表达式消除

>公共子表达式消除是一个普遍应用于各种编译器的经典优化技术，它的含义是：如果一个表达式E已经计算过了，并且从先前的计算到现在E中所有变量的值都没有发生变化，那么E的这次出现就成为了公共子表达式。
>

```
int d=（c*b）*12+a+（a+b*c）；
```

在JIT之后可能变成

```
int d=E*12+a+（a+E）；
```

如果再进行代数化简，

```
int d=E*13+a*2；
```

#### 2.2.2 数组范围检查消除

>数组边界检查消除（ArrayBoundsCheckingElimination）是即时编译器中的一项语言相关的经典优化技术。我们知道Java语言是一门动态安全的语言，对数组的读写访问也不像C、C++那样在本质上是裸指针操作。如果有一个数组foo[]，在Java语言中访问数组元素foo[i]的时候系统将会自动进行上下界的范围检查，即检查i必须满足i＞=0＆＆i＜foo.length这个条件，否则将抛出一个运行时异常：java.lang.ArrayIndexOutOfBoundsException。这对软件开发者来说是一件很好的事情，即使程序员没有专门编写防御代码，也可以避免大部分的溢出攻击。但是对于虚拟机的执行子系统来说，每次数组元素的读写都带有一次隐含的条件判定操作，对于拥有大量数组访问的程序代码，这无疑也是一种性能负担。


>那以Java伪代码来表示虚拟机访问foo.value的过程如下。
>

```
	if（foo!=null）{
		returnfoo.value；
	} else {
		thrownewNullPointException()；
		}
```

>在使用隐式异常优化之后，虚拟机会把上面伪代码所表示的访问过程变为如下伪代码。
>

```
	try {
		returnfoo.value；
	} catch（segment_fault）{
		uncommon_trap()；
	}
```

>虚拟机会注册一个SegmentFault信号的异常处理器（伪代码中的uncommon_trap()），这样当foo不为空的时候，对value的访问是不会额外消耗一次对foo判空的开销的。代价就是当foo真的为空时，必须转入到异常处理器中恢复并抛出NullPointException异常，这个过程必须从用户态转到内核态中处理，结束后再回到用户态，速度远比一次判空检查慢。当foo极少为空的时候，隐式异常优化是值得的，但假如foo经常为空的话，这样的优化反而会让程序更慢，还好HotSpot虚拟机足够“聪明”，它会根据运行期收集到的Profile信息自动选择最优方案。


#### 2.2.3 方法内联

#### 2.2.4 逃逸分析

>逃逸分析的基本行为就是分析对象动态作用域：
>
>- 方法逃逸: 当一个对象在方法中被定义后，它可能被外部方法所引用，例如作为调用参数传递到其他方法中，称为方法逃逸。
>- 线程逃逸: 甚至还有可能被外部线程访问到，譬如赋值给类变量或可以在其他线程中访问的实例变量，称为线程逃逸。
>

通过逃逸分析可做的 优化有：

> - 栈上分配（StackAllocation）
> - 同步消除（SynchronizationElimination）
> - 标量替换（ScalarReplacement）：

