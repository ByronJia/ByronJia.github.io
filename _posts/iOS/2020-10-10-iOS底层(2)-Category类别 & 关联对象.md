---
layout: post
title: iOS底层(2)-Category类别 & 关联对象
date: 2020-10-10
tag: iOS
---
## Category
分类在编译的时候变成`category_t` 结构体，其中包含所有方法，协议，属性等。
运行的时候把Category数组合并到Class的大数组中，后面参与编译的在数组的前面，也解释了为什么类别能够覆盖主类的方法，因为在方法查找时先找到类别的方法而已。  
- `+laod()`是在app启动时加载类、分类的时候调用。
调用原理与普通的category方法不同，是直接按照主类，类别的方法调用专门存取load方法的指针来调用load方法，即调用顺序一定是主类，分类1，分类2。。。而且如果父类还未load，先调用父类load。
- `+initialize()`在类第一次接受消息时调用。且方法是通过`objc_msgSend()`方法调用，则会从大数组中寻找方法，找到第一个肯定是类别的方法，则调用完毕结束。若父类还未初始化，则先调用父类initialize

## 关联对象
可以使用runtime的关联对象给categoty添加成员变量，`objc_setAssociatedObject`, `objc_getAssociatedObject`,其中key的参数很重要，必须是不变的一个地址，有多种实现方式，可以用静态变量例如@“key”, 因为存在常量区，所以地址不变。可以用`static void * KEYNAEM = &KEYNAME`， 取出定义的全局变量的指针用作key
 
原理: 全局保存一个`AssociationsManager`，和其中的`hashMap`,保存所有的对象和对象需要监听的属性。

<img src="http://image.smartjames.cn/mweb/20201010/16022999875266.png" style="zoom=50%">



