#### 关于 ViewPager 的使用就不多说了，相信大家都会玩.主要目的在于总结一下 PageTransform 该怎么玩. implements ViewPager.PageTransformer

#### 常见问题
1. FragmentPagerAdapter 和 FragmentStatePagerAdapter 的区别
    - FragmentPagerAdapter 对于不再需要的 fragment，选择调用 onDetach() 方法，仅销毁视图，不会销毁 fragment 实例。
    - FragmentStatePagerAdapter 会销毁不再需要的 fragment，当当前事务提交以后，会彻底的将 fragment 从当前 FragmentManager 中移除，state 表明，销毁时，会将其 onSaveInstanceState(Bundle outState) 中的 bundle 信息保存下来，当用户切换回来，可以通过该 bundle 恢复生成新的 fragment，也就是说，你可以在 onSaveInstanceState(Bundle outState) 方法中保存一些数据，在 onCreate 中进行恢复创建。
    - 由此可见，使用 FragmentStatePagerAdapter 更省内存，但是销毁后新建也是需要时间的。一般情况下，如果是结合 TabLayout 展示四五个 Tab 的话，可以直接使用 FragmentPagerAdapter，如果 ViewPager 展示特别多的条目时，建议使用 FragmentStatePagerAdapter。