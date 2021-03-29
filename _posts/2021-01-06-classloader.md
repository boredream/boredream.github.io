---
layout:     post
title:      "ClassLoader 类加载器"
subtitle:   ""
date:       2021-01-06 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:


---


### 是什么？
虚拟机设计团队把类加载阶段中的“通过一个类的全限定名来获取描述此类的二进制字节流”这个动作放到Java虚拟机外部去实现，以便让应用程序自己决定如何去获取所需要的类。实现这个动作的代码模块称为“类加载器”。

### 为什么虚拟机不自己加载类？
更灵活，可以热部署、代码加密等。  

注：  
不同类加载器加载的类，及时是相同类名的，也不会Class.equals()相等。


### 双亲委派模型
![classloader1](https://github.com/boredream/boredream.github.io/blob/master/img/in-post/classloader1.png?raw=true)
* 启动类：
C++语言实现（其它都是Java实现），虚拟机的一部分，加载 \lib 下的类库。
* 扩展类：
加载 \lib\ext 目录下类库。
* 应用程序：
加载用户类路径上所有指定类库，ClassLoader.getSystemClassLoader() 返回的就是它。

#### 工作过程是：  
如果一个类加载器收到了类加载的请求，它首先不会自己去尝试加载这个类，而是把这个请求委派给父类加载器去完成，每一个层次的类加载器都是如此，因此所有的加载请求最终都应该传送到顶层的启动类加载器中，只有当父加载器反馈自己无法完成这个加载请求（它的搜索范围中没有找到所需的类）时，子加载器才会尝试自己去加载。

#### 为什么要这样的结构？
Java类随着类加载器也具备了优先级层级关系。所以无论哪个类加载器加载如Object，则都是委托给顶端的加载，保证相等。


#### OSGi 面向java的动态模型系统
在双亲委派模型上进行了修改
1. 将以java.*开头的类委派给父类加载器加载。
2. 否则，将委派列表名单内的类委派给父类加载器加载。
3. 否则，将Import列表中的类委派给Export这个类的Bundle的类加载器加载。
4. 否则，查找当前Bundle的ClassPath，使用自己的类加载器加载。
5. 否则，查找类是否在自己的Fragment Bundle中，如果在，则委派给Fragment Bundle的类加载器加载。
6. 否则，查找Dynamic Import列表的Bundle，委派给对应Bundle的类加载器加载。
7. 否则，类查找失败。

1、2符合双亲委派，其它都是平级查找。

