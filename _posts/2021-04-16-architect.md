---
layout:     post
title:      "MVC MVP MVVM MVI"
subtitle:   ""
date:       2021-04-16 12:00:00
author:     "boredream"
catalog: false
header-style: text
tags:
- 架构

---

Android中的架构一路进化 MVC -> MVP -> MVVM 还有最新的方向 MVI

为什么架构会不停的换呢？
有什么区别？
新的一定比旧的好吗？
如何选择？如何使用？

以下主要以Android中各架构为例  

### MVC
传统写法是在Activity中，在点击等事件后，先「处理数据」再「设置视图」
此时Activity就相当于C，数据、视图逻辑都要做，在业务复杂时就会臃肿
  
设计架构的目标是？
让代码阅读性更好，问题更少修改扩展更容易。因此追求模块解耦，保证功能单一性。
而这里C既要处理M又要处理V就不合适了，要将其分开，因此就诞生了MVP的概念。

### MVP
让Presenter去处理M数据业务逻辑
让Actvity/Fragment去具体实现V视图逻辑
  
这样数据业务类问题都去p定位，ui类问题都去v定位。还可以分别实现测试需求。

#### MVP 具体实现
MVP架构只是一种思想，可以有不同实现，谷歌官方给了最基础的实现和一些修改版
* [谷歌官方MVP Demo](https://github.com/android/architecture-samples/tree/todo-mvp)

* [谷歌官方MVP-Clean Demo](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)
基于Clean架构思想，对MVP进一步分层，新建一个Domain层在P和M之间，里面以业务逻辑为粒度分成多个UseCase。因此对于复杂的业务，可以把多个Presenter中重复的代码提取出来。

#### MVP 实践经验
首先简单的业务就直接MVC了，如拉取数据然后无加工的直接显示到页面上。需要进一步处理数据进行复杂展示时才会使用MVP。
1. 数据处理
官方demo里其实对Model部分有进一步优化，不同业务都有对应的Repository类，里面进一步对数据进行了 缓存/远程/数据 不同来源的处理。我做的项目中因为只需要从服务器拉取，因此就省略了Repository类，直接在Presenter中处理远程数据。
2. View生命周期
官方Demo里基本会在Activity上加一层Fragment作为View的实现，这步实际开发中也省略了，除非Fragment有复用的情况。又因远程拉取数据有延迟，过程中如果页面销毁则再更新会有问题，Demo中是利用 fragment.isAdded 判断，我在实际开发使用了Retrofit+RxJava+RxLifeCircle 实现的
3. BaseView封装
对于显示/隐藏加载框，显示toast等通用视图操作，可以抽取到BaseView中。且在BaseActivity中具体实现，这样其它Activity在继承各自View时，无需重复处理这些通用视图逻辑。
4. 测试
UI自动化测试适合稳定模块的回归测试。新开发的复杂模块可以针对Presenter进行单元测试，直接mock view只测试数据部分，可以模拟远程数据测试本地逻辑也可以真实调用接口做调试工作。
5. 类文件过多
每个模块的MVP实现都要写很多类文件，使用AndroidStudio的File and Code Templates自定义新建几个Contract Presenter等的模板类，之后直接用模板创建接口。也可以网上找下AS相关插件，或自行编写生成模板代码脚本。

#### MVP 问题
1. 类文件多 + View生命周期。这个是已经有方案简化的，但还是需要我们自己处理。
2. P和V之间是通过协议类里的接口交互的，接口方法的颗粒度不好控制。且互相持有接口引用还是有一定耦合性。


### MVVM
看上去和MVP没区别，都解决了M和V的隔离，VM类似于P用于处理业务逻辑。最大的不同在于VM和V是双向绑定的。

#### MVVM 具体实现
* [谷歌官方MVVM-DataBinding Demo](https://github.com/android/architecture-samples/tree/todo-mvvm-databinding)
* [谷歌官方MVVM-Live Demo](https://github.com/android/architecture-samples/tree/todo-mvvm-databinding)

不同在于数据绑定的具体实现，是DataBinding，还是LiveData等。

那么问题来了，为什么使用MVVM？ 
MVC到MVP是因为要分离V和M，优化代码且可以单元测试，为什么一定要绑定V和M呢？
#### MVVM 优势
1. 无需通过接口交互，不用创建那么多文件
2. V与VM单向依赖，不像MVP里V和P互相持有接口引用，进一步解耦
3. 节省代码量，开发效率更高

### 总结
回头开头的问题
#### 为什么架构会不停的换呢？
因为随着业务的逐步复杂，旧的架构耦合性强，代码越来越臃肿。为了提升代码阅读性，迭代开发更容易，因此追求模块更高的解耦程度。

#### 有什么区别？
前面已经挨个介绍过了。MVC MVP MVVM 其实本质上都差不多，将视图、逻辑、数据进行拆分解耦。只是MVP MVVM相对于MVC在视图和逻辑数据层间解耦的更彻底。

#### 新的一定比旧的好吗？
不一定，越来越细化的分层也会带来新的问题，如MVP中文件过多，MVVM的调试更麻烦以及性能也有损耗等。

#### 如何选择？
为什么架构要进化，是因为业务上太复杂已有的架构无法满足。所以当你的业务比较简单时，MVC即可。或者项目里有的模块复杂可以MVVM，有的模块简单就直接MVC。此外从实际开发角度，还要考虑社区完善度，如MVVM有个优势是，谷歌官方Jetpack库里包含的各种具体方案。

#### 如何使用？
首先自然是根据实际情况选不同架构，然后可以基于官方的架构Demo去修改，不用完整copy，可按需组装。如不需要数据库/缓存部分就不用写Repository，如果有业务逻辑要复用可以参考Clean架构的UseCase设计，也可以用Repository实现复用。如果不喜欢Databinding LiveData的方式，也可以自己RxJava等方式实现等等。主要是掌握思想，不要太限制于具体实现方式，抓住核心即可 解耦、单一、复用。

### 发展
21年初官方推出了Jetpack Compose的Beta版，是一种新的声明式UI方案。看了谷歌compose开发团队的采访，提到了新的MVI的架构设计方向，期待+观望~