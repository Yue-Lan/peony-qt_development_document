# 构建与测试

首先给出peony-qt目前所在的仓库地址——[https://github.com/explorer-cs/peony-qt](https://github.com/explorer-cs/peony-qt "Peony-Qt")

对于刚刚接触这个项目的开发和测试人员而言，这一章的内容可能比较重要。目前的peony-qt主要有3个分支，它们分别是libfm-qt-based、master和file-operation-test，我对这3个分支的内容做一下简单的阐述。

## libfm-qt-based

这是一个基于libfm-qt提供的api快速构建的demo分支，libfm-qt提供了许多可以进行快速开发的工具类，比如侧边栏、地址栏、文件视图、文件操作前后端、搜索框架等等。这个分支的主要工作就是对这些控件进行重新布局，增加新的交互逻辑，定义新的信号连接方式，形成一个接近UKUI3.0文件管理器设计稿的demo。

可能描述起来过于抽象，我们结合图片来说：

![](/assets/libfm-qt-based.png)

整个demo尽量复用了libfm-qt的api，比如说中间的侧边栏，地址按钮，完全没有进行二次封装，对于工具栏，则使用qt的QToolBar进行封装，并结合了libfm-qt以及其之外的一些接口实现。文件视图主要也是复用了libfm-qt的model和view，稍微对一些无法直接复用的地方做了适配。将这些控件基于现有的信号重新连接起来，执行对应的逻辑，就形成了一个新的文件管理器。

libfm-qt-based的peony-qt可以算是一个问题多多但是勉强能够称之为文件管理器的应用，更多的功能可以在终端中使用帮助选项查看。

![](/assets/libfm-qt-based2.png)

## master

master分支是一个长期分支，目前整个peony-qt处于重构和新开发状态，可能会有许多不同侧重方向的分支。我会在这些分支成熟之后将其代码合并至master，目前master分支实现了大部分文件管理器所需要的gobject类型的封装，并且在在此之上实现了基于gvfs的model。

![](/assets/master.png)

一些信息的获取还有些许小问题，但是基本上已经趋于稳定了。

## file-operation-test

这个分支是我目前正在开发的分支，它包含了master分支，并且增加了file operation相关的api，当这个分支稳定之后，我会将它与master进行合并。

目前这个分支提供了文件操作以及异常处理的接口，并且提供了一个测试用例。

![](/assets/file-operation-test.png)

## 选择一个分支构建

### 编译依赖

不同的分支的编译依赖可能会不同，比如libfm-qt-based，就需要libfm-qt的编译依赖，由于master和file-operation-test的依赖暂时一致，并且和libfm-qt的编译依赖属于包含关系，为了简单，在Debian系的发行版上，我建议使用

* sudo apt build-dep libfm-qt-dev

拉取所有分支都能够编译通过的编译环境必要包。

### 拉取源码仓库

* git clone [https://github.com/explorer-cs/peony-qt.git](https://github.com/explorer-cs/peony-qt.git)

### 切换一个分支并构建

我们使用分支管理，由于默认clone下来的分支处于master，我们需要切换到想要的分支再进行构建。比如我们想使用libfm-qt-based分支进行构建，首先执行

* git checkout libfm-qt-based

然后，为了保持项目的整洁，还是创建一个build目录再进行构建

* mkdir build

* cd build

* qmake ..

* make

在确保编译依赖的情况下可以完成build，如果想要切换分支，请注意解决git的冲突问题，如果觉得麻烦也可以多clone几个，每一个对应切换一个分支。

### 测试

对于每个分支，测试的用例可能会不一样，针对的方向也不同。由于所有的分支都使用qmake，每一个测试用例都有对应的pro文件，为了简单我建议在切换分之后使用qtcreator进行构建和测试（终端交互除外）。

* libfm-qt-based 测试用例主要是在构建目录（build）的src/peony-qt

* master的测试用例目前主要是model的测试，在构建目录的libpeony-qt/model/model-test/model-test，另外还有一个插件测试。

* file-operation-test的测试用例在libpeony-qt/file-operation/file-operation-test/file-operation-test

## 测试用例

在peony-qt中项目的测试用例是不稳定的，我专门创建了一个项目用于测试peony-qt的api，这个项目也可能对你入门和了解peony-qt有不小的帮助，详见：

[https://github.com/Yue-Lan/libpeony-qt-development-examples](https://github.com/Yue-Lan/libpeony-qt-development-examples)



