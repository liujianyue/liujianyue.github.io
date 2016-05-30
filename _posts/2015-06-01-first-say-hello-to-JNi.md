---
layout: post_layout
title: JNI 初学使用一般步骤
time: 2015年6月01日
location: 北京
pulished: true
excerpt_separator: ""
---

> * 新建java文件，并在根目录执行javah -classpath .\bin\classses -d jni 包名.类名 生成 .h 文件；
> * c 与c++有些不同，首先在项目中properties/c/c++build中add NDK/ 目录，（env 在c++ 中不使用指针方式，c 不支持类，使用结构体）；
> * 配置Android.mk 文件（可以从之前的项目中copy）；
> * 若报错，清除obj文件夹；
> * 需要静态加载类库 static {system.loadLiberary（"...."）};
(可能需要新建新的builder)

参考如下文章，感谢作者：
### [Android开发之JNI(一)--HelloWorld及遇到的错误解析](http://blog.csdn.net/yangdev/article/details/41313533)

### [Android中JNI创建实例](http://blog.csdn.net/wikiday/article/details/42403953)

### [史上最易懂的Android jni开发资料--NDK环境搭建](http://www.cnblogs.com/yejiurui/p/3476565.html)




