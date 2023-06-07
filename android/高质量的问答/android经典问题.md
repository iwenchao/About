### 如何设计一个模仿微信右滑删除的功能？

1. 是否熟练掌握手势与动画的运用？
2. 是否了解窗口绘制的内部原理？
3. 是否对activity的窗口有深入认识？

- 如何实现？
    - 没有明说UI的类型，activity或者fragment？
    - fragment实现简单，重点说activity
        - 对于fragment的控制相对简单。不涉及window的控制，只是对view级别的操作；实现view跟随手势一起移动的效果；实现手势结束后判断取消或者返回执行归位的动画。
        - 对于activity的实现：设置activity的window的style（背景透明true），
    - 考虑如何设计这样一个组件
    - 考虑如何降低接入成本