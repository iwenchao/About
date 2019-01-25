### 为RecyclerView添加分割线

1. 那么如何创建分割线呢，创建一个类并继承RecyclerView.ItemDecoration，重写以下两个方法：
    - onDraw()或者onDrawOver: 绘制分割线样式。
    - getItemOffsets(Rect outRect, View view, RecyclerView parent,
        RecyclerView.State state): 设置分割线的宽、高。
        - outRect是当前item四周的间距，类似margin属性,确定分割线的大小的(这个大小指的是高度，宽度)
    - 然后使用RecyclerView通过addItemDecoration()方法添加item之间的分割线
2. 那么getItemOffsets()是怎么被调用的呢？
    1. RecyclerView继承了ViewGroup，并重写了measureChild()，该方法在onMeasure()中被调用，用来计算每个child的大小，计算每个child大小的时候就需要加上getItemOffsets()设置的外间距：
    ```
        public void measureChild(View child, int widthUsed, int heightUsed) {
            final LayoutParams lp = (LayoutParams) child.getLayoutParams();

            final Rect insets = mRecyclerView.getItemDecorInsetsForChild(child);
            widthUsed += insets.left + insets.right;
            heightUsed += insets.top + insets.bottom;
            ......
        }

    ```
3. 那么接着onDraw()以及onDrawOver()，两者的作用是什么呢以及两者之间有什么关系呢？
    ```
    public class RecyclerView extends ViewGroup {
        @Override
        public void draw(Canvas c) {
            super.draw(c);

            final int count = mItemDecorations.size();
            
            for (int i = 0; i < count; i++) {
                mItemDecorations.get(i).onDrawOver(c, this, mState);
            }
            ......
        }

        @Override
        public void onDraw(Canvas c) {
            super.onDraw(c);

            final int count = mItemDecorations.size();
            for (int i = 0; i < count; i++) {
                mItemDecorations.get(i).onDraw(c, this, mState);
            }
        }
    }

    ```
    根据view的绘制流程，首先调用RecyclerView重写的draw()方法，然后super.draw()即调用父类View的draw()，该方法会调用onDraw()。注意这里RecyclerView重写了onDraw()，然后再调用dispatchDraw()绘制children。因此，ItemDecoration的onDraw()在绘制item之前调用，ItemDecoration的onDrawOver()在绘制Item之后调用。