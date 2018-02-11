[TOC]


###1.精彩文章

#### 1.1原理分析

- [Android ListView工作原理完全解析，带你从源码的角度彻底理解](http://blog.csdn.net/guolin_blog/article/details/44996879)
- [Android 自己动手写ListView学习其原理 1 显示第一屏Item](http://blog.csdn.net/androiddevelop/article/details/8734255)
- [ Android ListView异步加载图片乱序问题，原因分析及解决方案](http://blog.csdn.net/guolin_blog/article/details/45586553)
- [Android ListView功能扩展，实现高性能的瀑布流布局](http://blog.csdn.net/guolin_blog/article/details/46361889)
- [Android ListView与RecyclerView对比浅析--缓存机制](http://dev.qq.com/topic/5811d3e3ab10c62013697408)
- [RecyclerView 和 ListView 使用对比分析](https://github.com/D-clock/AndroidSystemUiTraining/blob/master/note/03_AndroidSystemUI%EF%BC%9ARecyclerView%E5%92%8CListView%E4%BD%BF%E7%94%A8%E5%AF%B9%E6%AF%94%E5%88%86%E6%9E%90.md)

#### 1.2 如何使用

- [RecyclerView初探](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2015/0719/3201.html)
- [RecyclerView使用介绍](http://jcodecraeer.com/a/anzhuokaifa/androidkaifa/2014/1118/2004.html)
- [RecyclerView 和 ListView 使用对比分析](https://github.com/D-clock/AndroidSystemUiTraining/blob/master/note/03_AndroidSystemUI%EF%BC%9ARecyclerView%E5%92%8CListView%E4%BD%BF%E7%94%A8%E5%AF%B9%E6%AF%94%E5%88%86%E6%9E%90.md)
- [RecyclerView使用完全指南，是时候体验新控件了（二）](https://www.jianshu.com/p/7c3c549a0ec4)

  (非常详细的文章，总结了很多在使用RecylerView时遇到的问题和解决方法)

- [Android 优雅的为RecyclerView添加HeaderView和FooterView](http://blog.csdn.net/lmj623565791/article/details/51854533)

###2. 为RecylerView设置ItemAnimator

####2.1 开源库 

- [wasabeef/recyclerview-animators](https://github.com/wasabeef/recyclerview-animators)

- [gabrielemariotti/RecyclerViewItemAnimators](https://github.com/gabrielemariotti/RecyclerViewItemAnimators)


####2.2 ItemTouchHelper
```
	//ItemTouchHelper 用于实现 RecyclerView Item 拖曳效果的类
        ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new ItemTouchHelper.Callback() {

            @Override
            public int getMovementFlags(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder) {
                //actionState : action状态类型，有三类 ACTION_STATE_DRAG （拖曳），ACTION_STATE_SWIPE（滑动），ACTION_STATE_IDLE（静止）
                int dragFlags = makeFlag(ItemTouchHelper.ACTION_STATE_DRAG, ItemTouchHelper.UP | ItemTouchHelper.DOWN
                        | ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT);//支持上下左右的拖曳
                int swipeFlags = makeMovementFlags(ItemTouchHelper.ACTION_STATE_SWIPE, ItemTouchHelper.LEFT | ItemTouchHelper.RIGHT);//表示支持左右的滑动
                return makeMovementFlags(dragFlags, swipeFlags);//直接返回0表示不支持拖曳和滑动
            }

            /**
             * @param recyclerView attach的RecyclerView
             * @param viewHolder 拖动的Item
             * @param target 放置Item的目标位置
             * @return
             */
            @Override
            public boolean onMove(RecyclerView recyclerView, RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
                int fromPosition = viewHolder.getAdapterPosition();//要拖曳的位置
                int toPosition = target.getAdapterPosition();//要放置的目标位置
                Collections.swap(mData, fromPosition, toPosition);//做数据的交换
        		notifyItemMoved(fromPosition, toPosition);
                return true;
            }

            /**
             * @param viewHolder 滑动移除的Item
             * @param direction
             */
            @Override
            public void onSwiped(RecyclerView.ViewHolder viewHolder, int direction) {
                int position = viewHolder.getAdapterPosition();//获取要滑动删除的Item位置
                mData.remove(position);//删除数据
        		notifyItemRemoved(position);
            }

        });
        itemTouchHelper.attachToRecyclerView(mRecyclerView);

```

###3. 为RecylerView设置ItemDecoration

- [gabrielemariotti/RecyclerViewItemAnimators](https://github.com/gabrielemariotti/RecyclerViewItemAnimators)

```
<style name="AppTheme" parent="Theme.AppCompat.Light.DarkActionBar">
    <item name="colorPrimary">@color/colorPrimary</item>
    <item name="colorPrimaryDark">@color/colorPrimaryDark</item>
    <item name="colorAccent">@color/colorAccent</item>
    <item name="android:listDivider">@drawable/md_divider</item>
</style>


<?xml version="1.0" encoding="utf-8"?>
<shape xmlns:android="http://schemas.android.com/apk/res/android"
       android:shape="rectangle" >
    <solid android:color="@android:color/darker_gray"/>
    <size android:height="4dp" android:width="4dp"/>
</shape>

public class MDGridRvDividerDecoration extends RecyclerView.ItemDecoration {

    private static final int[] ATTRS = new int[]{
            android.R.attr.listDivider
    };

    /**
     * 用于绘制间隔样式
     */
    private Drawable mDivider;

    public MDGridRvDividerDecoration(Context context) {
        // 获取默认主题的属性
        final TypedArray a = context.obtainStyledAttributes(ATTRS);
        mDivider = a.getDrawable(0);
        a.recycle();
    }


    @Override
    public void onDraw(Canvas c, RecyclerView parent, RecyclerView.State state) {
        // 绘制间隔，每一个item，绘制右边和下方间隔样式
        int childCount = parent.getChildCount();
        int spanCount = ((GridLayoutManager)parent.getLayoutManager()).getSpanCount();
        int orientation = ((GridLayoutManager)parent.getLayoutManager()).getOrientation();
        boolean isDrawHorizontalDivider = true;
        boolean isDrawVerticalDivider = true;
        int extra = childCount % spanCount;
        extra = extra == 0 ? spanCount : extra;
        for(int i = 0; i < childCount; i++) {
            isDrawVerticalDivider = true;
            isDrawHorizontalDivider = true;
            // 如果是竖直方向，最右边一列不绘制竖直方向的间隔
            if(orientation == OrientationHelper.VERTICAL && (i + 1) % spanCount == 0) {
                isDrawVerticalDivider = false;
            }

            // 如果是竖直方向，最后一行不绘制水平方向间隔
            if(orientation == OrientationHelper.VERTICAL && i >= childCount - extra) {
                isDrawHorizontalDivider = false;
            }

            // 如果是水平方向，最下面一行不绘制水平方向的间隔
            if(orientation == OrientationHelper.HORIZONTAL && (i + 1) % spanCount == 0) {
                isDrawHorizontalDivider = false;
            }

            // 如果是水平方向，最后一列不绘制竖直方向间隔
            if(orientation == OrientationHelper.HORIZONTAL && i >= childCount - extra) {
                isDrawVerticalDivider = false;
            }

            if(isDrawHorizontalDivider) {
                drawHorizontalDivider(c, parent, i);
            }

            if(isDrawVerticalDivider) {
                drawVerticalDivider(c, parent, i);
            }
        }
    }

    @Override
    public void getItemOffsets(Rect outRect, View view, RecyclerView parent, RecyclerView.State state) {
        int spanCount = ((GridLayoutManager) parent.getLayoutManager()).getSpanCount();
        int orientation = ((GridLayoutManager)parent.getLayoutManager()).getOrientation();
        int position = parent.getChildLayoutPosition(view);
        if(orientation == OrientationHelper.VERTICAL && (position + 1) % spanCount == 0) {
            outRect.set(0, 0, 0, mDivider.getIntrinsicHeight());
            return;
        }

        if(orientation == OrientationHelper.HORIZONTAL && (position + 1) % spanCount == 0) {
            outRect.set(0, 0, mDivider.getIntrinsicWidth(), 0);
            return;
        }

        outRect.set(0, 0, mDivider.getIntrinsicWidth(), mDivider.getIntrinsicHeight());
    }

    /**
     * 绘制竖直间隔线
     * 
     * @param canvas
     * @param parent
     *              父布局，RecyclerView
     * @param position
     *              irem在父布局中所在的位置
     */
    private void drawVerticalDivider(Canvas canvas, RecyclerView parent, int position) {
        final View child = parent.getChildAt(position);
        final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child
                .getLayoutParams();
        final int top = child.getTop() - params.topMargin;
        final int bottom = child.getBottom() + params.bottomMargin + mDivider.getIntrinsicHeight();
        final int left = child.getRight() + params.rightMargin;
        final int right = left + mDivider.getIntrinsicWidth();
        mDivider.setBounds(left, top, right, bottom);
        mDivider.draw(canvas);
    }

    /**
     * 绘制水平间隔线
     * 
     * @param canvas
     * @param parent
     *              父布局，RecyclerView
     * @param position
     *              item在父布局中所在的位置
     */
    private void drawHorizontalDivider(Canvas canvas, RecyclerView parent, int position) {
        final View child = parent.getChildAt(position);
        final RecyclerView.LayoutParams params = (RecyclerView.LayoutParams) child
                .getLayoutParams();
        final int top = child.getBottom() + params.bottomMargin;
        final int bottom = top + mDivider.getIntrinsicHeight();
        final int left = child.getLeft() - params.leftMargin;
        final int right = child.getRight() + params.rightMargin + mDivider.getIntrinsicWidth();
        mDivider.setBounds(left, top, right, bottom);
        mDivider.draw(canvas);
    }
}

作者：王三的猫阿德
链接：https://www.jianshu.com/p/7c3c549a0ec4
來源：简书
著作权归作者所有。商业转载请联系作者获得授权，非商业转载请注明出处。


```

###4. 为RecylerView设置LayoutManager

- [devunwired/recyclerview-playground](https://github.com/devunwired/recyclerview-playground)

###5. 为RecylerView添加Header和Footer

[Android 优雅的为RecyclerView添加HeaderView和FooterView](http://blog.csdn.net/lmj623565791/article/details/51854533)


###6. 为RecylerView设置onItemClickListener

[android开发游记：RecyclerView无法添加onItemClickListener最佳的高效解决方案](http://blog.csdn.net/liaoinstan/article/details/51200600)

###7. 为RecylerView设置下拉刷新和滑动到底部加载更多

- [Android LRecyclerView实现下拉刷新，滑动到底部自动加载更多](http://jcodecraeer.com/plus/view.php?aid=4533)
- [自定义下拉刷新上拉加载控件（SwipeRefreshLayout + recyclerView）](http://jcodecraeer.com/a/anzhuokaifa/2016/1107/6753.html)

###8.ListView 与 RecyclerView差别

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

- [Android ListView与RecyclerView对比浅析--缓存机制](http://dev.qq.com/topic/5811d3e3ab10c62013697408)

- 

2.1 RecyclerView比ListView多两级缓存，支持多个离ItemView缓存，支持开发者自定义缓存处理逻辑，支持所有RecyclerView共用同一个RecyclerViewPool(缓存池)。

![](http://oa5504rxk.bkt.clouddn.com/week18_listview/2.png)

![](http://oa5504rxk.bkt.clouddn.com/week18_listview/3.jpg)

>ListView和RecyclerView缓存机制基本一致：
>
- 1). mActiveViews和mAttachedScrap功能相似，意义在于快速重用屏幕上可见的列表项ItemView，而不需要重新createView和bindView；
- 2). mScrapView和mCachedViews + mReyclerViewPool功能相似，意义在于缓存离开屏幕的ItemView，目的是让即将进入屏幕的ItemView重用.
- 3). RecyclerView的优势在于a.mCacheViews的使用，可以做到屏幕外的列表项ItemView进入屏幕内时也无须bindView快速重用；b.mRecyclerPool可以供多个RecyclerView共同使用，在特定场景下，如viewpaper+多个列表页下有优势.客观来说，RecyclerView在特定场景下对ListView的缓存机制做了补强和完善。

