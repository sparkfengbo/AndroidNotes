- [Android ListView工作原理完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/44996879)
- [Android 自己动手写ListView学习其原理 1 显示第一屏Item](http://blog.csdn.net/androiddevelop/article/details/8734255)
- [ Android ListView异步加载图片乱序问题，原因分析及解决方案](http://blog.csdn.net/guolin_blog/article/details/45586553)
- [Android ListView功能扩展，实现高性能的瀑布流布局](http://blog.csdn.net/guolin_blog/article/details/46361889)

- [Android ListView与RecyclerView对比浅析--缓存机制](http://dev.qq.com/topic/5811d3e3ab10c62013697408)
- [RecyclerView 和 ListView 使用对比分析](https://github.com/D-clock/AndroidSystemUiTraining/blob/master/note/03_AndroidSystemUI%EF%BC%9ARecyclerView%E5%92%8CListView%E4%BD%BF%E7%94%A8%E5%AF%B9%E6%AF%94%E5%88%86%E6%9E%90.md)



###1.ListView

1.RecycleBin（Adapter中的子类）

1.1两级缓存列表

```
private ArrayList<View>[] mScrapViews; 
 
private ArrayList<View> mCurrentScrap;  

```

- RecycleBin当中使用mScrapViews和mCurrentScrap这两个List来存储废弃View。
- Adapter的setViewTypeCount方法会根据view的类型数量创建mScrapViews。
- mCurrentScrap指向mScrapViews中的某一个元素

###2.RecyclerView与ListView的差别


2.1 RecyclerView比ListView多两级缓存，支持多个离ItemView缓存，支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool(缓存池)。

![](http://oa5504rxk.bkt.clouddn.com/week18_listview/2.png)

![](http://oa5504rxk.bkt.clouddn.com/week18_listview/3.jpg)

>ListView和RecyclerView缓存机制基本一致：
>
- 1). mActiveViews和mAttachedScrap功能相似，意义在于快速重用屏幕上可见的列表项ItemView，而不需要重新createView和bindView；
- 2). mScrapView和mCachedViews + mReyclerViewPool功能相似，意义在于缓存离开屏幕的ItemView，目的是让即将进入屏幕的ItemView重用.
- 3). RecyclerView的优势在于a.mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；b.mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpaper+多个列表页下有优势.客观来说，RecyclerView在特定场景下对ListView的缓存机制做了补强和完善。

