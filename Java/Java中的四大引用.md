[TOC]


按引用强度递减的顺序

- 1.强引用
- 2.软引用
- 3.弱引用
- 4.虚引用

### 1.强引用（Strong Reference）

`Object o = new Object()`

只要强引用还存在，垃圾收集器永远不会回收掉被引用的对象


### 2.软引用（Soft Reference）

描述一些还有用但并非必须的对象


对于软引用关联着的对象，在系统将要发生内存溢出异常之前，将会把这些对象列进回收范围之中进行第二次回收。如果这次回收还没有足够的内存，才会抛出内存溢出异常。在JDK1.2之后，提供了SoftReference类来实现软引用。### 3.弱引用（Weak Reference）

弱引用也是用来描述非必需对象的，但是它的强度比软引用更弱一些，被弱引用关联的对象只能生存到下一次垃圾收集发生之前。当垃圾收集器工作时，无论当前内存是否足够，都会回收掉只被弱引用关联的对象。在JDK1.2之后，提供了WeakReference类来实现弱引用。

### 4.虚引用（Phantom Reference）

虚引用也称为幽灵引用或者幻影引用，它是最弱的一种引用关系。一个对象是否有虚引用的存在，完全不会对其生存时间构成影响，也无法通过虚引用来取得一个对象实例。为一个对象设置虚引用关联的唯一目的就是能在这个对象被收集器回收时收到一个系统通知。



关于虚引用得到回收的通知：参考

- [Java幽灵引用的作用](http://www.importnew.com/20992.html)


```
package static_;
 
import java.lang.ref.PhantomReference;
import java.lang.ref.Reference;
import java.lang.ref.ReferenceQueue;
import java.lang.reflect.Field;
 
public class Test {
    public static boolean isRun = true;
 
    @SuppressWarnings("static-access")
    public static void main(String[] args) throws Exception {
        String abc = new String("abc");
        System.out.println(abc.getClass() + "@" + abc.hashCode());
        final ReferenceQueue<String> referenceQueue = new ReferenceQueue<String>();
        new Thread() {
            public void run() {
                while (isRun) {
                    Object obj = referenceQueue.poll();
                    if (obj != null) {
                        try {
                            Field rereferent = Reference.class
                                    .getDeclaredField("referent");
                            rereferent.setAccessible(true);
                            Object result = rereferent.get(obj);
                            System.out.println("gc will collect："
                                    + result.getClass() + "@"
                                    + result.hashCode() + "\t"
                                    + (String) result);
                        } catch (Exception e) {
                            e.printStackTrace();
                        }
                    }
                }
            }
        }.start();
        PhantomReference<String> abcWeakRef = new PhantomReference<String>(abc,
                referenceQueue);
        abc = null;
        Thread.currentThread().sleep(3000);
        System.gc();
        Thread.currentThread().sleep(3000);
        isRun = false;
    }
}
```

>我们用一个线程检测referenceQueue里面是不是有内容，如果有内容，打印出来queue里面的内容。

>从这个例子中，我们可以看出来，虚引用的作用是，我们可以声明虚引用来引用我们感兴趣的对象，在gc要回收的时候，gc收集器会把这个对象添加到referenceQueue，这样我们如果检测到referenceQueue中有我们感兴趣的对象的时候，说明gc将要回收这个对象了。此时我们可以在gc回收之前做一些其他事情，比如记录些日志什么的。