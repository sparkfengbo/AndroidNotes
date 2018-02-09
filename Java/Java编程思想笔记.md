[TOC]

##9 接口
接口也可以包含域，但这些域隐式的是static和final的

##10 内部类
在拥有外部类对象前不可能创建内部类对象，除了静态内部类
可以在方法里面或者任意作用于内定义内部类
匿名内部列相当于创建一个继承自xxx的匿名类的对象
普通内部类不能包含static数据和字段，也不嫩肤包含嵌套类，但是嵌套类可以包含这些关系
##12 Exception
1.可以在catch中重新将异常抛出，那么在上一层捕获的异常的printStackTrace信息还是原来抛出点的异常信息，如果想更新这个信息，重新抛出新的异常（new）或者调用fillInStackTrace方法
2.异常链
通过`initCause`方法添加引起异常的原因

3.分类
 Throwable -》1.Error（编译时和系统错误，一般不用关心）  2.Exception（可以被抛出的类型）
 
 Exception -》 RuntimeException（不受检查异常）
  1.RuntimeException 运行时异常，不用主动抛出，属于Java的标准运行时检测的一部分，RuntimeException可以不需要被捕获也可以被系统主动调用printStackTrace）
  

  ![](/Users/leegend/Desktop/Exception.jpg)
  >通常，Java的异常(包括Exception和Error)分为可查的异常（checked exceptions）和不可查的异常（unchecked exceptions）。
      可查异常（编译器要求必须处置的异常）：正确的程序在运行中，很容易出现的、情理可容的异常状况。可查异常虽然是异常状况，但在一定程度上它的发生是可以预计的，而且一旦发生这种异常状况，就必须采取某种方式进行处理。
 除了RuntimeException及其子类以外，其他的Exception类及其子类都属于可查异常。这种异常的特点是Java编译器会检查它，也就是说，当程序中可能出现这类异常，要么用try-catch语句捕获它，要么用throws子句声明抛出它，否则编译不会通过。
不可查异常(编译器不要求强制处置的异常):包括运行时异常（RuntimeException与其子类）和错误（Error）。
  
4.异常缺失
  在finally中直接return不会看见异常
  
5.异常限制
子类的构造方法必须抛出父类构造方法的异常，可以额外添加抛出的异常，但派生类构造器不能捕获基类抛出的异常
普通覆盖的方法必须遵循父类的方法，如果父类方法没有抛出异常，子类也不能，父类抛出了异常，子类可以不抛出
重写的方法可以抛出父类抛出异常的子类
接口不能向现有的方法上抛出异常
 
 不能通过异常说明来重载方法

##13字符串

- 1.String对象是不可变对象
- 2.StringBuilder可变，如果调用StringBuilder.append方法，eg append(a + ":" + c),那括号内部还会创建一个StringBuilder处理拼接的字符串操作
- 3.无意识的递归(在toString方法中再次调用toString)
- 4.格式化说明符
%b, 对于boolean 会得到 true或false，但是对于类型（基本类型和对象），只要不是null，就会得到true，即使是0，也会得到true
- 5.正则表达式
- 6.String.intern 
  
  [Java技术——你真的了解String类的intern()方法吗](http://blog.csdn.net/seu_calvin/article/details/52291082)

##16数组
数组是一种效率最高的存储和随机访问对象引用序列方式，为这种速度所付出的代价是数组的大小被固定，并且其生命周期不可改变。


##19枚举类型
编译器为你创建的enum类都继承自Enum类
values()是由编译器添加的static方法，Enum类中不包含values()方法
枚举类无法继承

```
enum Explore {HERE, THRERE}

执行 javap Explore ，得到

final class Explore extends java.lang.Enum {
	public static final Explore HERE;
	public static final Explore THERE;
	public static final Explore[] values();
	public static final Explore valueOf(java.lang.String);
	
	static {};

}

```

##20注解
也被称为元数据，

一般来说Annotation有如下三种使用情形：

- Information for the compiler — Annotations can be used by the compiler to detect errors or suppress * warnings.
- Compile-time and deployment-time processing — Software tools can process annotation information to generate code, XML files, and so forth.
- Runtime processing — Some annotations are available to be examined at runtime.
- 为编译器提供辅助信息 — Annotations可以为编译器提供而外信息，以便于检测错误，抑制警告等.
- 编译源代码时进行而外操作 — 软件工具可以通过处理Annotation信息来生成原代码，xml文件等等.
- 运行时处理 — 有一些annotation甚至可以在程序运行时被检测，使用.


####1.基本语法
#####1.1定义注解
很像接口，注解也会被编译成class文件
#####1.2定义注解元注解
#####1.2.1.@Target
 用来定义注解将应用在什么地方。
 
 - ElementType.CONSTRUCTOR 构造器声明
 - ElementType.FIELD 域声明（包括enum实例）
 - ElementType.LOCAL_VARIABLE 局部变量声明
 - ElementType.METHOD 方法声明
 - ElementType.PACKAGE 包声明
 - ElementType.PARAMETER 参数声明
 - ElementType.TYPE 类、接口（包括注解）或enum声明
 
 
#####1.2.2.@Retention
定义注解在哪一个级别可用，

- RetentionPolicy.SOURCE 注解在编译器丢弃
- RetentionPolicy.CLASS 注解在class文件可用，但会被VM丢弃
- RetentionPolicy.RUNTIME VM将在运行期也保留注解，可以通过反射机制读取注解信息

#####1.2.3.@Documented
此注解包含在JavaDoc中

#####1.2.4.@Inherited
允许子类继承父类中的注解
但是这并不是真的继承。通过使用@Inherited，只可以让子类类对象使用getAnnotations（）反射获取父类被@Inherited修饰的注解。

```
@Inherited
@Retention(RetentionPolicy.Runtime)
@interface Inheritable{
}
@interface UnInheritable{
}
public class Test{
    @UnInheritable
    @Inheritable
    public static class Super{
    }
    public static class Sub extends  Super {
    }
    public static void main(String... args){
        Super instance=new Sub();
        System.out.println(Arrays.toString(instance.getClass().getAnnotations()));
    }
}
```
因为这种伪继承机制，被@Documented与@Inherited同时修饰的注解来注解某个类，其子类Javadoc并不会出现这个注解的提示

>没有元素的注解称为标记注解

#####1.3.注解元素
基本类型、String、Class、enum、Annotation、以上类型数组
默认值限制：不能有不确定的值，要不有默认值要不在使用注解时赋值。非基本类型不能是null。

####2.编写注解处理器

使用apt生成注解处理器时，无法利用Java的反射机制。可以使用mirror API解决此类问题
。继承AnnotationProcessor，使用AnnotationProcessorFactory

处理循环
注解处理过程可能会多于一次。官方javadoc定义处理过程如下：

注解处理过程是一个有序的循环过程。在每次循环中，一个处理器可能被要求去处理那些在上一次循环中产生的源文件和类文件中的注解。第一次循环的输入是运行此工具的初始输入。这些初始输入，可以看成是虚拟的第0此的循环的输出。

![](/Users/leegend/Desktop/annotation3.png)
参考文章

- [Java注解（1）-基础](http://blog.csdn.net/duo2005duo/article/details/50505884)
- [Java注解（2）-运行时框架](http://blog.csdn.net/duo2005duo/article/details/50511476)
- [Java注解（3）-源码级框架](http://blog.csdn.net/duo2005duo/article/details/50541281)
- [还在用枚举？我早就抛弃了！（Android 注解详解）](http://www.jianshu.com/p/1fb27f46622c)
- [Android注解快速入门和实用解析](http://www.jianshu.com/p/9ca78aa4ab4d)
- [深入理解Java：注解（Annotation）自定义注解入门](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)
- [Java注解处理器](https://race604.com/annotation-processing/)
- [ Android 打造编译时注解解析框架 这只是一个开始](http://blog.csdn.net/lmj623565791/article/details/43452969#reply)
- [Annotation实战【自定义AbstractProcessor](http://www.cnblogs.com/avenwu/p/4173899.html)

####3.替代注解

```
public class Colors {
  @IntDef({RED, GREEN, YELLOW})
  @Retention(RetentionPolicy.SOURCE)
  public @interface LightColors{}

  public static final int RED = 0;
  public static final int GREEN = 1;
  public static final int YELLOW = 2;
}

```

##

























