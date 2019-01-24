# 主要内容 RecyclerView与ListView
1. 基本用法
2. 缓存策略 [（参考腾讯博客）](https://zhuanlan.zhihu.com/p/23339185)
3. 滚动嵌套机制详解
4. 对比
    1. 布局效果
        - RecyclerView支持横向滚动（gridview，listview），支持瀑布流效果
        - RecyclerView.LayoutManager自定义可以支持更加复杂的布局（如何自定义RecyclerView.LayoutManager?）
    2. 常用功能以及api
        - ListView
            - 继承重写BaseAdapter
            - 自定义ViewHolder与convertView一起完成复用优化工作
            - 空数据以及HeaderView，footerView， setEmptyView()
            - 局部刷新notifyDataSetChanged():更新了 ListView 的数据源后需要通过 Adapter 的 notifyDataSetChanged 来通知视图更新变化，这样做比较的好处就是调用简单，坏处就是它会重绘每个 Item。
                - 手动实现真正的局部刷新
                ```
                     public void updateItemView(ListView listView, int position) {
                        //1. 换算成 Item View 在 ViewGroup 中的 index
                        int index = position - listView.getFirstVisiblePosition();
                        if (index >= 0 && index < listView.getChildCount()) {
                            //2. 更新数据
                            AuthorInfo authorInfo = mAuthorInfoList.get(position);
                            authorInfo.setNickName("Google Android");
                            authorInfo.setMotto("My name is Android .");
                            authorInfo.setPortrait(R.mipmap.ic_launcher);
                            //3. 更新单个Item
                            View itemView = listView.getChildAt(index);
                            getView(position, itemView, listView);
                        }
                    }

                ```
            - 动画效果，很简陋 [ListView动画开源库](https://github.com/nhaarman/ListViewAnimations)
            - item点击事件，方便但是容易出现问题
        - RecyclerView
            - 继承重写RecyclerView.Adapter与RecyclerView.ViewHolder，ViewHolder 的编写规范化
            - 复用 Item 的工作 Google 全帮你搞定，不再需要像 ListView 那样自己调用 setTag
            - 多出一步 LayoutManager 的设置工作
            - 空数据 相当于一个type的View，如何解决headerview,footerView呢？[参考鸿洋的优雅的为RecyclerView添加header&footer](https://blog.csdn.net/lmj623565791/article/details/51854533)
            - 局部刷新以及局部的局部刷新 notifyItemChanged(...)系列方法
            - 动画效果： RecyclerView 在做局部刷新的时候有一个渐变的动画效果。
                - 继承 RecyclerView.ItemAnimator 自定义效果；并实现相应的方法，再调用 RecyclerView 的 setItemAnimator(RecyclerView.ItemAnimator animator) 方法设置完即可实现自定义的动画效果。
                - [RecyclerView动画开源库](https://github.com/wasabeef/recyclerview-animators)
            - ItemTouchHelper是系统为我们提供的一个用于滑动和删除 RecyclerView 条目的工具类
                - 创建 ItemTouchHelper 实例，同时实现 ItemTouchHelper.SimpleCallback 中的抽象方法，用于初始化 ItemTouchHelper
                - 调用 ItemTouchHelper 的 attachToRecyclerView 方法关联上 RecyclerView 即可
                - ```
                主要重写的是 getMovementFlags 、 onMove 、 onSwiped 三个抽象方法，getMovementFlags 用于告诉系统，我们的 RecyclerView 到底是支持滑动还是拖曳。如上面的示例代码，就是表示着同时支持上下左右四个方向的拖曳和左右两个方向的滑动效果。如果时滑动，则 onSwiped 会被回调，如果是拖曳 onMove 会被回调。我们再到其中实现相应的业务操作即可。
                //ItemTouchHelper 用于实现 RecyclerView Item 拖曳效果的类
                        ItemTouchHelper itemTouchHelper = new ItemTouchHelper(new ItemTouchHelper.Callback() {
                
                            @Override
                            public int getMovementFlags(RecyclerView recyclerView,  RecyclerView.ViewHolder viewHolder) {
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
                            public boolean onMove(RecyclerView recyclerView,  RecyclerView.ViewHolder viewHolder, RecyclerView.ViewHolder target) {
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
            - item点击事件，自己实现就好，不复杂
    3. android L引入的嵌套滚动机制之后的对比
        - Android 触摸事件分发机制中，Touch 事件在进行分发的时候，由父 View 向它的子 View 传递，一旦某个子 View 开始接收进行处理，那么接下来所有事件都将由这个 View 来进行处理，它的 ViewGroup 将不会再接收到这些事件，直到下一次手指按下。而嵌套滚动机制（NestedScrolling）就是为了弥补这一机制的不足，为了让子 View 能和父 View 同时处理一个 Touch 事件。
        - 关于嵌套滚动机制（NestedScrolling），实现上相对是比较复杂的，此处就不去拓展说明，其关键在于NestedScrollingChild 和 NestedScrollingParent 两个接口，以及系统对这两个接口的实现类 NestedScrollingChildHelper 和 NestedScrollingParentHelper 
        - CollapsingToolbarLayout/AppBarLayout 结合RecyclerView，NestedScrollView可以实现；ListView，ScrollView没有嵌套滚动的实现
