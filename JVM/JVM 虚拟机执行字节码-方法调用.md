[TOC]


**参考**

周志明. 深入理解Java虚拟机：JVM高级特性与最佳实践（第2版） (原创精品系列) (Kindle位置980). 机械工业出版社. Kindle 版本. 

----

>方法调用并不等同于方法执行，方法调用阶段唯一的任务就是确定被调用方法的版本（即调用哪一个方法），暂时还不涉及方法内部的具体运行过程。

包括

- 解析
- 分派

### 1.解析

方法在程序真正运行之前就有一个可确定的调用版本，并且这个方法的调用版本在运行期是不可改变的。换句话说，调用目标在程序代码写好、编译器进行编译时就必须确定下来。这类方法的调用称为解析（Resolution）。

解析调用一定是个静态的过程，在编译期间就完全确定，在类装载的解析阶段就会把涉及的符号引用全部转变为可确定的直接引用.

所有方法调用中的目标目标方法在Class文件里面都是一个常量池中的符号引用，在类加载的解析阶段，会将其中的一部分符号引用转化为直接引用。在Java语言中符合“编译期可知，运行期不可变”这个要求的方法，主要包括静态方法和私有方法两大类，前者与类型直接关联，后者在外部不可被访问，这两种方法各自的特点决定了它们都不可能通过继承或别的方式重写其他版本，因此它们都适合在类加载阶段进行解析。



>与之相对应的是，在Java虚拟机里面提供了5条方法调用字节码指令，分别如下。 
> 
> - invokestatic：调用静态方法。> - invokespecial：调用实例构造器＜init＞方法、私有方法和父类方法。invokevirtual：调用所有的虚方法。
> - invokeinterface：调用接口方法，会在运行时再确定一个实现此接口的对象。
> - invokedynamic：先在运行时动态解析出调用点限定符所引用的方法，然后再执行该方法，在此之前的4条调用指令，分派逻辑是固化在Java虚拟机内部的，而invokedynamic指令的分派逻辑是由用户所设定的引导方法决定的。
> 
> 只要能被invokestatic和invokespecial指令调用的方法，都可以在解析阶段中确定唯一的调用版本，符合这个条件的有静态方法、私有方法、实例构造器、父类方法4类### 2.分派

分派（Dispatch）调用则可能是静态的也可能是动态的。



根据分派依据的宗量数可分为

 - 单分派
 - 多分派

这两类分派方式的两两组合就构成了

 - 静态单分派
 - 静态多分派
 - 动态单分派
 - 动态多分派

4种分派组合情况

#### 1.静态分派


>静态分派:所有依赖静态类型来定位方法执行版本的分派动作称为静态分派。静态分派的典型应用是方法重载(Overload)。(后面会看到)

```

    package org.fenixsoft.polymorphic;

    /** 方法 静态 分派 演示 *@author zzm */
    public class StaticDispatch {
        static abstract class Human {

        }

        static class Man extends Human {

        }

        static class Woman extends Human {

        }

        public void sayHello(Human guy) {
            System.out.println(" hello, guy！");
        }

        public void sayHello(Man guy) {
            System.out.println(" hello, gentleman！");
        }

        public void sayHello(Woman guy) {
            System.out.println(" hello, lady！");
        }

        public static void main(String[] args)

        {
            Human man = new Man();
            Human woman = new Woman();
            StaticDispatch sr = new StaticDispatch();
            sr.sayHello(man);
            sr.sayHello(woman);
        }
    }
```输出 

```
hello, guy！ 
hello, guy！

```

详情参考 8.3.2 很好的例子

上述例子中 Human为静态类型，或者叫做外观类型，Man为实际类型，变量的实际类型只有在运行期才能确定，比如

```
Human a = new Man();
a = new Women();
```

**在编译阶段，Javac编译器会根据参数的静态类型决定使用哪个重载版本**

#### 2.动态分派

>把这种在运行期根据实际类型确定方法执行版本的分派过程称为动态分派。

针对方法的重写（Override）

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

>方法的接收者与方法的参数统称为方法的宗量，这个定义最早应该来源于《Java与模式》一书。根据分派基于多少种宗量，可以将分派划分为单分派和多分派两种。单分派是根据一个宗量对目标方法进行选择，多分派则是根据多于一个宗量对目标方法进行选择。>我们可以总结一句：今天（直至还未发布的Java1.8）的Java语言是一门静态多分派、动态单分派的语言。