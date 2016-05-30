---
layout: post_layout
title: 关于View的滑动和弹性滑动的总结和扩展
time: 2015年12月1日
location: 北京
pulished: true
excerpt_separator: "1.通过view"
---
## 关于View滑动
View滑动，在Android 上几乎是应用的标配，不管是下拉刷新，还是SlidingMenu，它们的基础都是滑动。从另外一方面来说，Android手机由于屏幕较小，为了给用户呈现更多的内容，就需要使用滑动来隐藏和显示一些内容。基于上述两点，可以知道，滑动在Android开发中具有重要作用，不管一些滑动效果多么绚丽，归根到底，它们都是由不同的滑动外加一些特写组成的，因此掌握滑动原理是实现绚丽的自定义控件的基础。通过三种方式可以是实现滑动：

1.通过view本身自带的scrollTo/scrollBy 方法来实现滑动；

2.通过动画来给view添加施加平移效果来实现滑动；

3.通过改变View的layoutParams 是的重新布局从而实现滑动； 

## 关于弹性滑动
弹性滑动的优点就是在view的滑动时显得不生硬，给用户良好的体验。实现弹性滑动的原理大体上可以概括为：将一次大的滑动分成若干次小的滑动并在一个时间段内完成，弹性滑动的具体实现方式有很多种，如 scroller、Handler#postDelayed及Thread#sleep。

1.使用scroller；

2.使用动画；

3.使用延时策略；

4.使用Rect/RectF，并通过通过时间来逐渐缩小它的长/宽（扩展）；

5.使用overscroll，原理几乎同scroll，但是有很多限制（扩展）；


待续...

参考自任玉刚《Android 开发艺术探索》


