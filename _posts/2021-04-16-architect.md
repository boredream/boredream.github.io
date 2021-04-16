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
 
有什么区别？  
为什么会进化？ 
如何选择使用？  

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
看上去和MVP没区别，都解决了M和V的隔离，VM类似于P用于处理业务逻辑。最大的不同在于VM和V是双向绑定的，那只多了个绑定有啥意义？  
MVP最本质的问题其实是在于：处理业务逻辑的P和处理视图的V相互引用，不符合单向依赖的原则。  
而MVVM解决了这个问题，尤其从下面的MVVM-RxJava的代码就可以感受到，V持有VM的引用 而 VM对V是无直接感知的，不像MVP里V和P是互相持有。  
![MVVM架构图](https://developer.android.com/topic/libraries/architecture/images/final-architecture.png)

#### MVVM 优势
1. 不用像MVP那样创建很多文件
2. V与VM单向依赖，符合单向依赖原则
3. 数据绑定部分可以节省代码量，提升开发效率
4. 官方支持，教程、组件、Demo [Android官方应用架构指南](https://developer.android.com/jetpack/guide)

#### MVVM 具体实现
* [谷歌官方MVVM-DataBinding Demo](https://github.com/android/architecture-samples/tree/todo-mvvm-databinding)  
* [谷歌官方MVVM-Live Demo](https://github.com/android/architecture-samples/tree/todo-mvvm-databinding)
* [谷歌官方MVVM-RxJava Demo](https://github.com/android/architecture-samples/tree/dev-todo-mvvm-rxjava)


### 总结
回头开头的问题  

#### 有什么区别？
前面已经挨个介绍过了。MVC MVP MVVM 其实本质上都差不多，将视图、逻辑、数据进行拆分解耦。MVP在MVC的基础上对数据和视图分离的更清楚。而MVVM则又在MVP基础上，更清晰的实现了单向依赖，还做了其它优化。

#### 为什么会进化？  
因为随着业务的逐步复杂，旧的架构耦合性强，代码越来越臃肿。为了提升代码阅读性，迭代开发更容易，所以要不停进化架构。

#### 如何选择？  
1. 看实际项目需求，不一定用最新的，也不一定只用一种。当你的业务比较简单时，MVC即可，如果有的模块复杂可以MVVM。  
2. 不能一味追求新的架构，新架构也会有新的问题，如MVP中文件过多，MVVM的调试更麻烦以及性能也有损耗等。  
3. 从实际开发角度，还要考虑社区完善度，如MVVM有个优势是，谷歌官方的支持。

#### 如何使用？
建议MVC+MVVM混合，具体到模块据实际情况选择。  
MVVM部分基于官方的架构，不用完整copy，可按需组装。如不需要数据库/缓存部分就不用写Repository，如果有业务逻辑要复用可以参考Clean架构的UseCase设计，也可以直接用Repository实现。如果不喜欢Databinding LiveData的方式，也可以自己RxJava等方式实现等等。  
主要是掌握思想，不要太限制于具体实现方式，比如MVC + UseCase这样可以行。抓住核心即可 模块解耦、功能单一、方便复用。

### 发展
21年初官方推出了Jetpack Compose的Beta版，是一种新的声明式UI方案。看了谷歌compose开发团队的采访，提到了新的MVI的架构设计方向，期待+观望~