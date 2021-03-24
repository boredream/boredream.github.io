---
layout:     post
title:      "LaunchMode页面加载模式"
subtitle:   ""
date:       2014-08-25 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:
  - launchMode
---


一个应用通常(不一定)对应一个任务栈,相当于有个集合,保存了这个app里所有的页面

栈的规则是先进后出,"进"就相当于打开了一个页面,"出"就相当于返回时关闭一个页面
栈顶,则就是当前显示的页面~
所以如果有4个页面  任务栈中打开的顺序为ABCD 那一步步返回的时候就是DCBA的顺序


如果再次加载B页面 则顺序为ABCDB 虽然还是B页面 但是并非同一个对象
可以自己打印 页面对象this.toString查看信息   信息内容为: 类名@对象id
可以通过id的相等与否查看是否为同一个对象

---


在清单文件androidmanifest.xml中注册activity时,可以添加lauchmode参数,用于控制页面加载方式。
有四种方式
* standard(默认,什么都不设置时) 标准化加载方式,即上面举例子的情况

* singleTop  栈顶的页面只保留一个对象
如果栈当前是ABCD时,如果再次打开一个D页面,则标准模式时为ABCDD
当D为singleTop时则为ABCD

* singleTast 任务栈内只保留一个对象,且清空该加载类型页面顶部所有页面
比如有四个页面ABCD不停的循环跳转的话,则标准模式时就是ABCDABCDABCD....
而当B是singleTast型,则如果还是按照ABCD依次循环
则ABCDA后 再打开B时, B只会一直保持一个对象,且会清空singleTast页面栈顶上全部的页面
即ABCDA->AB

* singleInstance 新建并独享一个任务栈
多个任务栈也有个"顶/底"的概念,如果简单的理解一个应用有一个任务栈,
那当前可见的应用任务栈就是在顶部,下面例子里右边的可以理解为顶部可见栈~
还是四个页面ABCD 如果C是singleInstance, 跳转顺序还是A->B->C->D->A...
则当跳转到C时,则会新建一个任务栈,
又因为singleInstance是独享一个任务栈,所以当顺序循环跳转时为
栈1:A -> 
栈1:AB -> 
栈1:AB 栈2:C -> 
栈2:C 栈1:ABD -> 
栈2:C 栈1:ABDA
开始AB都在一个任务栈1里, 加载C时,会新建一个任务栈2,
C之后加载D时,因为C是独占一个任务栈的,所以D还是加在AB栈1里变成ABD~
注意,循环到第二次加载C时,C和之前的C为同一个对象~

当栈1:ABDA 栈2:C 后点返回时,逻辑比较特别,因为加载的顺序看上去是ABCDA...
返回时,由于栈1为顶部,所以先处理栈1, 返回时顺序为 ADBA 此时栈1清空了,
再返回就会跳转到栈2里的C了


为更好理解,有配套demo,每个页面显示页面唯一id~ singleInstance页面还会在id后添加"...任务栈id"

---

除了在清单文件中控制某个页面的加载方式外,还可以在代码中Intent跳转页面时,
通过flag控制打开页面的任务栈模式,flag可以添加多个,但是部分flag不能同时存在
使用方法:
```java
Intent intent = new Intent(this, TargetActivity.class);
intent.addFlags(Intent.FLAG_ACTIVITY_....);
startActivity(intent);
```

下面介绍一些常见的flag,一般都是Intent.FLAG...
1. FLAG_ACTIVITY_NEW_TAST
一般页面跳转时如果单加该flag,基本无用,还是默认的任务栈普通加载方式~
但是如果是在service中startActivity启动一个页面,则必须要加该flag,
因为service没有页面,是不存在于任务栈中的~所以需要加这么个flag
此外,该flag常要配合其他flag同时使用
即intent.addFlags(...NEW_TAST);
intent.addFlags(...otherFlag);

2. FLAG_ACTIVITY_CLEAR_TOP
和singleTast效果比较像,会清除目标页面顶部的全部页面
比如ABCD依次跳转,当A->B时,如果设置了这么个flag
同理,第一轮时,AB由于B在顶部,所以不会清除其他页面~
而到第二轮ABCDA时,再打开页面,则把B顶部的CDA全部清除,变成AB
看上去和singleTast一样~但是第二轮打开的B和第一轮的B不是一个对象
和NEW_TAST共用时效率更好

3. FLAG_ACTIVITY_SINGLE_TOP
和lauchmode=singletop效果一样

4. FLAG_ACTIVITY_CLEAR_TASK
必须和NEW_TAST一起使用
会将页面所在任务栈中的全部页面清空,并新建一个任务栈将跳转至的页面放进去
如 ABCD依次跳转, 如果startActivity打开B页面时使用了此方式,即会清除其他所在栈中已存页面
并将B放在新任务栈中~此时在B页面如果按一下back键 就会发现直接退出整个任务栈了~
注:因为老的任务栈清空了,所以任务栈也相当于移除了,新建的任务栈的id还是会和之前任务栈一样
但是实际上同样的任务栈id其实不是一个任务栈了~
如果打印任务栈id观察到还是同一个id的话不要疑惑

5. FLAG_ACTIVITY_CLEAR_WHEN_TASK_RESET
要和FLAG_ACTIVITY_RESET_TASK_IF_NEEDED搭配使用
这两个是搭配使用的
当使用WHEN_TAST_RESET打开时,相当于在该页面中设置了一个标记
而下次该页面被携带NEEDED的flag启动时,则会将该页面其顶部所有的页面都关闭
清空逻辑和CLEAR_TOP一样~区别在于CLEAR_TOP每次都清顶部全部页面,
而这里只有在CLEAR_WHEN_TAST_REST设置标记位后再用NEEDED触发时才清

6. FLAG_ACTIVITY_REORDER_TO_FRONT
如果启动某页面时添加了此flag,则会将页面移至栈顶~还是同一页面对象
比如仍然是ABCDABCD..依次循环启动
如果A启动B时添加了此flag,则会在ABCDA再打开B时变成ACDAB,将B移至了栈顶

7. FLAG_ACTIVITY_NO_HISTORY
如果启动某页面时使用了该flag,则该页面不会存在于任务栈中
即当打开此页面后,一旦离开页面,它就已经关闭finish了
比如ABCD依次加载,A打开B时添加此flag,
则再当B跳转到C后,在C页面按back键就无法回到B页面了,会直接back回A
相当于B打开C时加了行finish()的代码


其他的flag,要不是系统不推荐直接使用,要不是我也看不懂具体用处- - 
可以在代码中 Intent.FLAG再按提示建,鼠标点击某条flag查看弹出的介绍框


还有,使用flag一般都是去startActivity启动一个页面,基本分两种
一种是重新打开已有页面,即每次启动的对象都是同一个,比如SINGLE_TOP
另一种则是重新创建页面,比如CLEAT_TOP
重新创建的,一般是普通的走onCreate生命周期
而打开已有的不重新创建的,则会走onNewIntent周期,即原有页面内的数据也会保存
demo中flag包下,B页面内有输入框,可以试着输入一些内容,看下次再打开时是否保存

---

最重要的部分
使用场景
可能覆盖不全,大部分为个人经验

1. singleTast
一种比如是以前最常用的tabActivity+activity结构,如果打开的页面中有返回主页功能
就可以将tabActivity的lauchMode直接设为singleTask~
一般TabActivity都是在子activity栈底部的,singleTast模式跳至TabActivity主页时,
就会将其顶部所有页面(一般都是打开的子模块页面)移除了

2. FLAG_ACTIVITY_CLEAR_TOP
和singleTast感觉场景挺像的,比如某个设置功能,是瀑布型流程,即设置部分信息后点下一步,
然后再设置其他部分点击下一步~
如果再这些步骤页面里都有一个重新设置功能,跳转到设置的第一页~
就可以使用此flag启动第一页~ 
这样就不会出现到第一页后再点返回又跳回去某步骤的奇怪逻辑了

3. FLAG_ACTIVITY_NEW_TAST ,  singleInstance
以前做过一个功能是同时使用到两个加载模式的,就放到一起介绍了,不是必须要求两者一起使用
程序锁功能,有点涉及到不同进程间交互的,即有多个任务栈存在的情况
比如程序锁功能是一个service在后台监听当前打开应用,如果当前打开应用是要求锁住的
则会打开程序锁密码输入页面~
service中打开页面时,就需要添加一个NEW_TAST的标识

而我的程序锁应用是有自己页面和任务栈的,比如设置哪些程序需要锁等页面
如果在打开其他页面前先打开了程序锁应用设置一些信息,未关闭程序锁的部分页面就直接到桌面
然后打开了某个要锁住的程序,则启动的输入密码页面是属于程序锁应用的,
所以会将其直接加载已存在的程序锁任务栈栈顶,此时程序锁的任务栈会置于顶部
输入密码正确关闭程序锁页面后,会跳转至程序锁页面中的上一个页面~而非需要打开的应用
跳转逻辑明显不对
而如果给输入密码页面设置了singleInstance,则其会创建并独享一个任务栈,不会加在程序锁已有的栈顶
这时再密码正确关闭后,就会正常显示需要打开的程序页面了
