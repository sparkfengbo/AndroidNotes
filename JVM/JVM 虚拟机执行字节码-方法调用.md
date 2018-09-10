**参考**

- 周志明. 深入理解Java虚拟机：JVM高级特性与最佳实践（第2版） (原创精品系列) (Kindle位置980). 机械工业出版社. Kindle 版本. 

- [多态性实现机制——静态分派与动态分派](http://wiki.jikexueyuan.com/project/java-vm/polymorphism.html)
- [JVM--详解虚拟机字节码执行引擎之静态链接、动态链接与分派](https://blog.csdn.net/championhengyi/article/details/78760590)

----

**方法调用并不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还不涉及方法内部的具体运行过程。**

**包括**

- 解析
- 分派

### 1.解析

方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。**换句话说，调用目标在程序代码写好、编译器进行编译时就必须确定下来。这类方法的调用称为解析（Resolution）**。

>Class 文件的编译过程中不包含传统编译中的连接步骤，一切方法调用在 Class 文件里面存储的都只是符号引用，而不是方法在实际运行时内存布局中的入口地址。这个特性给 Java 带来了更强大的动态扩展能力，使得可以在类运行期间才能确定某些目标方法的直接引用，称为动态连接，也有一部分方法的符号引用在类加载阶段或第一次使用时转化为直接引用，这种转化称为静态解析。

解析调用一定是个静态的过程，在编译期间就完全确定，在类装载的解析阶段就会把涉及的符号引用全部转变为可确定的直接引用。

所有方法调用中的目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用。

在Java语言中符合“编译期可知，运行期不可变”这个要求的方法，主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自的特点决定了它们都不可能通过继承或别的方式重写其他版本，因此它们都适合在类加载阶段进行解析。


- [java -- JVM的符号引用和直接引用](https://www.cnblogs.com/shinubi/articles/6116993.html)

>与之相对应的是，在Java虚拟机里面提供了5条方法调用字节码指令，分别如下。 
> 
> - invokestatic：调用静态方法。
> - invokespecial：调用实例构造器＜init＞方法、私有方法和父类方法。invokevirtual：调用所有的虚方法。
> - invokeinterface：调用接口方法，会在运行时再确定一个实现此接口的对象。
> - invokedynamic：先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法，在此之前的4条调用指令，分派逻辑是固化在Java虚拟机内部的，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。
> 
> 只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版本，符合这个条件的有**静态方法、私有方法、实例构造器、父类方法4类**


### 2.分派

**推荐文章**

- [JVM--详解虚拟机字节码执行引擎之静态链接、动态链接与分派](https://blog.csdn.net/championhengyi/article/details/78760590)

分派（Dispatch）调用则可能是静态的也可能是动态的。


根据分派依据的宗量数(方法的接收者与方法的参数统称为方法的宗量)可分为

 - 单分派
 - 多分派

这两类分派方式的两两组合就构成了

 - 静态单分派
 - 静态多分派
 - 动态单分派
 - 动态多分派

4种分派组合情况


#### 1.静态分派

静态分派:所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派发生在编译阶段，因此确定静态分配的动作实际上不是由虚拟机来执行的。静态分派的典型应用是**方法重载(Overload)**。

```
class Human{  
}    
class Man extends Human{  
}  
class Woman extends Human{  
}  

public class StaticPai{  

    public void say(Human hum){  
        System.out.println("I am human");  
    }  
    public void say(Man hum){  
        System.out.println("I am man");  
    }  
    public void say(Woman hum){  
        System.out.println("I am woman");  
    }  

    public static void main(String[] args){  
        Human man = new Man();  
        Human woman = new Woman();  
        StaticPai sp = new StaticPai();  
        sp.say(man);  
        sp.say(woman);  
    }  
}  
```

输出 

```
 I am human
 I am human
```

详情参考 8.3.2 很好的例子

以上结果的得出应该不难分析。在分析为什么会选择参数类型为 Human 的重载方法去执行之前，先看如下代码：

```
Human man = new Man（）;
```

我们把上面代码中的“Human”称为变量的静态类型，或者叫做外观类型，后面的“Man”称为变量的实际类型。静态类型和实际类型在程序中都可以发生一些变化，区别是静态类型的变化仅仅在使用时发生，变量本身的静态类型不会被改变，并且最终的静态类型是在编译期可知的，而实际类型变化的结果在运行期才可确定。

回到上面的代码分析中，在调用 say()方法时，方法的调用者（回忆上面关于宗量的定义，方法的调用者属于宗量）都为 sp 的前提下，使用哪个重载版本，完全取决于传入参数的数量和数据类型（方法的参数也是数据宗量）。代码中刻意定义了两个静态类型相同、实际类型不同的变量，可见编译器（不是虚拟机，因为如果是根据静态类型做出的判断，那么在编译期就确定了）在重载时是通过参数的静态类型而不是实际类型作为判定依据的。并且静态类型是编译期可知的，所以在编译阶段，javac 编译器就根据参数的静态类型决定使用哪个重载版本。这就是静态分派最典型的应用。



**在编译阶段，Javac编译器会根据参数的静态类型决定使用哪个重载版本**

方法重载要求方法具备不同的特征签名，返回值并不包含在方法的特征签名之中，所以返回值不参与重载选择，但是在Class文件格式之中，只要描述符不是完全一致的两个方法就可以共存。也就是说，两个方法如果有相同的名称和特征签名，但返回值不同，那它们也是可以合法地共存于一个Class文件中的。

这里需要参考10.3.1 类型擦书的例子去学习

#### 2.动态分派

把这种在运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

针对**方法的重写（Override）**

```
        package org.fenixsoft.polymorphic;

        /** 方法 动态 分派 演示 *@author zzm */
        public class DynamicDispatch {
            static abstract class Human {
                protected abstract void sayHello();
            }

            static class Man extends Human {
                @Override
                protected void sayHello() {
                    System.out.println(" man say hello");
                }
            }

            static class Woman extends Human {
                @Override
                protected void sayHello() {
                    System.out.println(" woman say hello");
                }
            }

            public static void main(String[] args) {
                Human man = new Man();
                Human woman = new Woman();

                man.sayHello();
                woman.sayHello();
                man = new Woman();
                man.sayHello();
            }
        }
```

```
man say hello 
woman say hello 
woman say hello
```

#### 3.单分派和多分派

方法的接收者与方法的参数统称为方法的宗量，这个定义最早应该来源于《Java与模式》一书。根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。

我们可以总结一句：今天（直至还未发布的Java1.8）的Java语言是一门**静态多分派、动态单分派**的语言。

```
class Eat{  
}  
class Drink{  
}  

class Father{  
    public void doSomething(Eat arg){  
        System.out.println("爸爸在吃饭");  
    }  
    public void doSomething(Drink arg){  
        System.out.println("爸爸在喝水");  
    }  
}  

class Child extends Father{  
    public void doSomething(Eat arg){  
        System.out.println("儿子在吃饭");  
    }  
    public void doSomething(Drink arg){  
        System.out.println("儿子在喝水");  
    }  
}  

public class SingleDoublePai{  
    public static void main(String[] args){  
        Father father = new Father();  
        Father child = new Child();  
        father.doSomething(new Eat());  
        child.doSomething(new Drink());  
    }  
}  
```

我们首先来看编译阶段编译器的选择过程，即静态分派过程。这时候选择目标方法的依据有两点：一是方法的接受者（即调用者）的静态类型是 Father 还是 Child，二是方法参数类型是 Eat 还是 Drink。因为是根据两个宗量进行选择，所以 Java 语言的静态分派属于多分派类型。

再来看运行阶段虚拟机的选择，即动态分派过程。由于编译期已经了确定了目标方法的参数类型（编译期根据参数的静态类型进行静态分派），因此唯一可以影响到虚拟机选择的因素只有此方法的接受者的实际类型是 Father 还是 Child。因为只有一个宗量作为选择依据，所以 Java 语言的动态分派属于单分派类型。
