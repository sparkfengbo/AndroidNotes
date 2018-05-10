#### 参考文章

- [Android App 优化之 ANR 详解](https://juejin.im/post/582582df0ce4630058bbbad2)
- [浅谈ANR及log分析ANR](http://blog.csdn.net/itachi85/article/details/6918761)

#### ANR类型

- 1：KeyDispatchTimeout(5 seconds) --主要类型

按键或触摸事件在特定时间内无响应

- 2：BroadcastTimeout(10 seconds)

BroadcastReceiver在特定时间内无法处理完成

- 3：ServiceTimeout(20 seconds) --小概率类型

Service在特定的时间内无法处理完成

#### ANR产生原因
超时时间的计数一般是从按键分发给app开始。超时的原因一般有两种：

- (1)当前的事件没有机会得到处理（即UI线程正在处理前一个事件，没有及时的完成或者looper被某种原因阻塞住了）

- (2)当前的事件正在处理，但没有及时完成


三种ANR发生时都会在log中输出错误信息，你会发现各个应用进程和系统进程的函数堆栈信息都输出到了一个/data/anr/traces.txt的文件中

- 从LOG可以看出ANR的类型，CPU的使用情况，如果CPU使用量接近100%，说明当前设备很忙，有可能是CPU饥饿导致了ANR.如果CPU使用量很少，说明主线程被BLOCK了.如果IOwait很高，说明ANR有可能是主线程在进行I/O操作造成的

```
// libcore/libart/src/main/java/java/lang/Thread.java
    /**
     * A representation of a thread's state. A given thread may only be in one
     * state at a time.
     */
    public enum State {
        /**
         * The thread has been created, but has never been started.
         */
        NEW,
        /**
         * The thread may be run.
         */
        RUNNABLE,
        /**
         * The thread is blocked and waiting for a lock.
         */
        BLOCKED,
        /**
         * The thread is waiting.
         */
        WAITING,
        /**
         * The thread is waiting for a specified amount of time.
         */
        TIMED_WAITING,
        /**
         * The thread has been terminated.
         */
        TERMINATED
    }
```

```
Thread状态说明： 
ThreadState (defined at “dalvik/vm/thread.h “) 
THREAD_UNDEFINED = -1, /* makes enum compatible with int32_t */ 
THREAD_ZOMBIE = 0, /* TERMINATED */ 
THREAD_RUNNING = 1, /* RUNNABLE or running now */ 
THREAD_TIMED_WAIT = 2, /* TIMED_WAITING in Object.wait() */ 
THREAD_MONITOR = 3, /* BLOCKED on a monitor */ 
THREAD_WAIT = 4, /* WAITING in Object.wait() */ 
THREAD_INITIALIZING= 5, /* allocated, not yet running */ 
THREAD_STARTING = 6, /* started, not yet on thread list */ 
THREAD_NATIVE = 7, /* off in a JNI native method */ 
THREAD_VMWAIT = 8, /* waiting on a VM resource */ 
THREAD_SUSPENDED = 9, /* suspended, usually by GC or debugger */
```

