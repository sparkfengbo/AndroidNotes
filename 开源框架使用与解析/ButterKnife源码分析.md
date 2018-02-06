
###1.准备知识
####1.1 Java注解
参考文章

- [Java注解（1）-基础](http://blog.csdn.net/duo2005duo/article/details/50505884)
- [Java注解（2）-运行时框架](http://blog.csdn.net/duo2005duo/article/details/50511476)
- [Java注解（3）-源码级框架](http://blog.csdn.net/duo2005duo/article/details/50541281)
- [还在用枚举？我早就抛弃了！（Android 注解详解）](http://www.jianshu.com/p/1fb27f46622c)
- [Android注解快速入门和实用解析](http://www.jianshu.com/p/9ca78aa4ab4d)
- [深入理解Java：注解（Annotation）自定义注解入门](http://www.cnblogs.com/peida/archive/2013/04/24/3036689.html)
- [Method详解](http://blog.csdn.net/zhangquanit/article/details/52927216)

####1.2 Java注解处理器

- [Java注解处理器](https://race604.com/annotation-processing/)
- [ Android 打造编译时注解解析框架 这只是一个开始](http://blog.csdn.net/lmj623565791/article/details/43452969#reply)
- [Annotation实战【自定义AbstractProcessor](http://www.cnblogs.com/avenwu/p/4173899.html)

####1.3 动手自己写一个DEMO
- [android注解入门 并来自己写一个框架](http://blog.csdn.net/qfanmingyiq/article/details/53394783) 非常入门，只用到了反射
- [ Android 打造编译时注解解析框架 这只是一个开始](http://blog.csdn.net/lmj623565791/article/details/43452969#reply)

写demo可能遇到的问题：

- 1.在AS中无法引入AbstractProcessor

  解决：新建一个module，类型是JavaModel，build.gradle中的apply参数是 java-library，此时可以引入。参考butter-knife也是一样的设置。
 
- 2.在AS中无法引入@AutoService注解
  
  解决：在build.gradle的依赖中加入
  
  ```
  dependencies {
   ...
   compileOnly 'com.google.auto.service:auto-service:1.0-rc3'
}
    ```
 
- 3.提示错误`> Annotation processors must be explicitly declared now.  The following dependencies on the compile classpath are found to contain annotation processor.  Please add them to the annotationProcessor configuration.`
  
  解决：在build.gradle的依赖中加入    
  ```
  defaultConfig {
        ...

        javaCompileOptions { annotationProcessorOptions { includeCompileClasspath = true } }
    }
  ```
 
####1.3 JavaPoet库

- [javapoet——让你从重复无聊的代码中解放出来](https://www.jianshu.com/p/95f12f72f69a)
###2.ButterKnife源码分析
在看过一些基础知识和**自己动手写过demo**之后就会对注解的使用有一个比较清晰的认识了，这个时候才可以继续往下看ButterKnife源码。
 


