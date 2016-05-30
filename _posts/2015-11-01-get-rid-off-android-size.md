---
layout: post_layout
title: 还在为Android中各种尺寸发愁吗
time: 2015年11月19日
location: 北京
pulished: true
excerpt_separator: "长乘以宽）"
---

笔者之所以决定总结这篇博文，是因为又一次晚上将将入睡之时突然惊醒，脑子中浮现出了Android，dp、sp、px等等一系列尺寸的东东(脑子潜意识回忆，怕忘记吧)，感觉很烦，（感觉有这种感受的小伙伴不止我一个），所以我决定把它记下来，供以后给自己和朋友参考。

### 首先理解一下需要理解的概念哈
> * 屏幕的大小(Screen size)： 屏幕对角线的物理尺寸（以inch为单位）
> * 屏幕密度（Screen density）：每英寸上的像素个数。通常被称作多少 dpi（dots per inch）或多少 ppi（pixels per inch）
> * 分辨率 屏幕上的物理像素个数（长乘以宽）

### 开始学习尺寸
####  dp（Density-independent Pixels）：

在不同大小、密度和分辨率的屏幕上的物理大小都近似相等的虚拟尺寸单位。
约为 1/160 英寸（为什么是约为？稍后讲解）。

#### sp（Scale-independent Pixels）

基于首选字体大小的缩放像素。与 dp 类似，但是会根据用户的首选字体大小缩放。

#### pt（Points）

1/72 英寸。

#### px（Pixels）

这就是我们说的像素。

#### mm（Millimeters）

毫米（没怎么用过）。

#### in（Inches）

>英寸，约 2.539999918 厘米。

### 开始他们之间的转换

Android 大约将屏幕密度概括为 7 种：

dpi种类 ~xxdpi ~像素密度系数

ldpi(low) ~120dpi ~0.75

mdpi(medium) ~160dpi ~1

tvdpi(tv)~ about 213dpi (手机一般不会用到，我在一些面试题中看到过) ~213/160

hdpi(high) ~240dpi ~1.5

xhdpi(extra-high) ~320dpi ~2

xxhdpi(extra-extra-high) ~480dpi ~3

xxxhdpi(extra-extra-extra-high) ~640dpi ~4

nodpi（全称是啥）所有屏幕都会适配

但是需要注意，并非表示所有Android手机只有这几个屏幕密度，比如说屏幕密度也有445dpi的手机（xxhdpi），那么他在dp 到 px 转换时，以480dpi为参照，而不是445dpi。

>像素密度（generalizedDensity） = px/dp*160
像素密度系数（scaledDensity） = 像素密度/160 =px/dp

####dp 到 px的转换代码（官方）：

```java
// The gesture threshold expressed in dp
private static final float GESTURE_THRESHOLD_DP = 16.0f;

// Get the screen's density scale
final float scale = getResources().getDisplayMetrics().density;
// Convert the dps to pixels, based on density scale
mGestureThreshold = (int) (GESTURE_THRESHOLD_DP * scale + 0.5f);

// Use mGestureThreshold as a distance in pixels...
```
####sp 转 px

1 dp = 1px = 0.00625in
dp = scaledDensity*px;
所以y = x * scaledDensity
这里 scaledDensity 获取方式为getResources().getDisplayMetrics().scaledDensity。

同时可以参考
[官方解释](http://developer.android.com/reference/android/util/DisplayMetrics.html)



    




