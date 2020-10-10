---
layout: post
title: iOS底层(1)-Class & isa
date: 2020-10-09
tag: iOS
---

## 类初始化内存分配
NSObject占用内存大小为16个字节，但是NSObjct只有一个属性就是isa只占用8个字节，由于底层设计原因最少需要16个字节。  
结构体内存对齐，(x+7)& ~x 是将x按照8字节对齐。不满8就补齐8字节
malloc 中的内存分配也会有内存对齐规则，mac中是16的倍数，如果传入只需要24字节，alloc后系统会分配32个字节空间即内存对齐。   
class_getInstanceSize 返回的是对象需要的内存， NSObjcet中的isa占用8， 加上继承来的每个属性的空间。
malloc_size 返回的是实际分配的空间大小

## OC中的对象类型
分为3种
- 实例对象
    - isa
    - 成员变量值
- 类对象
    - isa
    - superclass
    - 属性、对象方法、协议、成员变量
- 元类对象
    - isa
    - superclass
    - 类方法

    
类对象是由元类对象描述的，即class->isa = metaclass, 但是二者结构一致，只是存储的数据不同。      
`object_getClass(id)` 传入是对象，则返回类对象，传入是类对象，返回元类对象
`objc_getClass(char * className)` 只会返回类对象。         

## 对象之间的关系
有3个类 NSObjcect -> Person -> Student，
`[stu studentInstanceMethod];` stu->isa = Student类寻找对象方法来执行
`[stu personInstanceMethod]; ` stu->isa = Student->superClass = Person寻找对象方法
`[stu load]; `stu->isa = Student->superClass = Person->superClass = NSObject寻找对象方法

`[Student studentClassMethod];` Student->isa = Meta_Student 寻找Student的类方法     
`[Student personClassMethod];` Student->isa = Meta_Student->superclass = Meta_Person 寻找person的类方法  
`[Student copy]; `Student->isa = Meta_Student->superclass = Meta_Person->superclass=Meta_NSObject 寻找NSObject的类方法     
一旦中途找到，则停止后续寻找，这就是为什么override会覆盖父类同名方法

isa指针地址需要& isa_mask才能得到真实的class的地址
superClass地址 直接就是真实superclass地址
**总结**
instance调用实例方法轨迹
isa找到class,方法不存在，就通过superclass找到父类(类对象)

class调用类方法轨迹
isa找meta-class，方法不存在，就通过superclass找父类(元类对象)

## Class的结构

<img src='http://image.smartjames.cn/mweb/20201009/16022266359919.png' style='zoom=50%'>

class_rw_t 中存放的是可读写的所有数据，class_ro_t保存的是只读的数据，主要是主类中生成的方法变量协议等property_list_t一维数组。class_rw_t中存放的是包含类别变量协议方法的一个大的二维数组property_array_t，存放主类和类别的小数组数据。

<img src='http://image.smartjames.cn/mweb/20201009/16022266740521.jpg' style='zoom=50%' >

## 方法缓存
class中有个cache_t cache, 用来做方法的缓存，使用哈希表实现。第一次在rw_t中的methods中遍历查找方法实现，找到就放入cache中，节省查找时间。
插入和查询时分别用SEL按位与 mask,得到哈希表的index,解决哈希表冲突的方式是开放地址法，就是往前寻找一位。

<img src='http://image.smartjames.cn/mweb/20201009/16022267154421.png' style='zoom=50%' >

在arm64之前，isa是一个普通指针，指向class和meta-class地址
arm64之后， ISA经过优化，使用union数据结构，使用位域来增加存储信息，增加内存利用率
下图中bits为真正存放数据的值，struct中只是申明bits中分别多少位是什么数据。在取值时将bits按位与得到指定位数的值是多少而不受其他位数据的影响。例如shiftcls是存放类地址的，使用了33位。

![1](http://image.smartjames.cn/mweb/20201009/16022267618486.jpg)
![2](http://image.smartjames.cn/mweb/20201009/16022267618501.jpg)
![3](http://image.smartjames.cn/mweb/20201009/16022267618514.jpg)
