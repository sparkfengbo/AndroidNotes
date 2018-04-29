- **参考：**[Android Studio多Module使用aar编译报错的解决方案](http://leehong2005.com/2016/08/28/android-studio-use-aar-issues/)




-------


### 1.关于 AAR

>在 Android Studio 之前，如果用引用第三方的库，一般使用 jar 包，它只包含了class，没有包含对应的资源、so库等，所以引用起来就不方便，特别是一些 UI 库，第三方在使用的时候，还需要自己单独导入对应的资源（字符串、图片等）。
>现在 Android 中引入了 aar 这种包结构，它其实也是一个zip包，它里面包含了很多资源，具体的结构如下所示：
>

```
The file extension is .aar, and the maven artifact type should be aar as well, but the file itself a simple zip file with the following entries:

	/AndroidManifest.xml (mandatory)
	/classes.jar (mandatory)
	/res/ (mandatory)
	/R.txt (mandatory)
	/assets/ (optional)
	/libs/*.jar (optional)
	/jni//*.so (optional)
	/proguard.txt (optional)
	/lint.jar (optional)
These entries are directly at the root of the zip file.

The R.txt file is the output of aapt with –output-text-symbols.
```

### 2.在Gradle中如何使用aar

>Android Studio (AS) 中使用aar，通常是把aar文件放到 libs 目录下面，需要在 MODULE 目录下面 build.gralde 中添加如下配置：

```
repositories {
    flatDir {
        dirs 'libs'
    }
}
```

>另外，还需要在 dependencies block中添加如下代码：


```
dependencies {
	compile(name: 'aar-file-name', ext: 'aar')  
	// 其中aar-file-name不用后缀名
}
```

### 3.多Module中使用aar编译报错

```
allprojects {
    repositories {
        jcenter()

        flatDir {
            // 由于Library module中引用了 gif 库的 aar，在多 module 的情况下，
            // 其他的module编译会报错，所以需要在所有工程的repositories
            // 下把Library module中的libs目录添加到依赖关系中
            dirs project(':AppLibrary').file('libs')  
        }
    }
}
```