
这篇简单介绍下怎么将一个现有的`Flutter`项目转成`Flutter Web`项目。

开始之前先浇一盆冷水，我们理想中的一套代码、多端运行的愿望是要破灭了，至少目前版本的`Flutter Web SDK`是没法做到的。不过没关系，谷歌爸爸已经在[官网](https://github.com/flutter/flutter_web)中降低了我们的预期：
1. 项目目前只是处于`technical preview`状态
2. 项目是从`Flutter`主项目`fork`出来的，目前还没准备好合入主项目
3. 不是所有的`Flutter API`都有对应的`Flutter Web`实现
4. 目前`Flutter Web`的代码跑起来很卡，性能优化工作才刚刚开始

因此`Flutter Web`目前只是个半成品，踩到坑是必然的，但不妨碍我们试着玩一玩。正好手里有一个之前开发的`Flutter`项目，看看转成`Flutter Web`要做哪些事。

首先假定读者之前已经搭好了`Flutter`开发环境，如果没有的话可以先看看谷歌的文档。再此基础上我们先搭个`Flutter Web`的环境。

`Flutter Web`要求`Flutter SDK`的版本至少要`1.5.4`，先跑下`flutter upgrade`升级下`Flutter`的版本。

然后安装`Flutter Web`的编译工具`webdev`：

```
flutter pub global activate webdev
```

具体步骤和问题可以参考[官网](https://github.com/flutter/flutter_web)，这里就不多说了。

然后我们开始迁移项目。

由于目前`Flutter Web`和`Flutter`是两个不同的`SDK`，两者在项目中只能二选一，所以要支持`Flutter Web`就不能用`Flutter SDK`。因此对于`Flutter Web`项目，目前只能新开一个项目，把原有项目中大部分搬过去。`VS Code`也提供了搭建新项目的脚手架：

![vs_flutter.png](https://upload-images.jianshu.io/upload_images/138119-9b005d114616c3ac.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

先用这个方法创建一个新项目。可以看到`pubspec.yaml`中已经不再依赖`flutter`了，改为依赖`flutter_web`:

```dart
name: hello_world_web
description: An app built using Flutter for web

environment:
  # You must be using Flutter >=1.5.0 or Dart >=2.3.0
  sdk: '>=2.3.0 <3.0.0'

dependencies:
  flutter_web: any
  flutter_web_ui: any

dev_dependencies:
  build_runner: ^1.5.0
  build_web_compilers: ^2.1.0
  pedantic: ^1.7.0

dependency_overrides:
  flutter_web:
    git:
      url: https://github.com/flutter/flutter_web
      path: packages/flutter_web
  flutter_web_ui:
    git:
      url: https://github.com/flutter/flutter_web
      path: packages/flutter_web_ui

```

然后把老项目中`lib`文件夹下的所有代码搬过去。这样以后肯定会有一堆编译问题，我们一个个来处理。

首先我们要把`import`方式全部改掉，比如以前的`flutter`要换成`flutter_web`：

``` dart
import 'package:flutter/material.dart'; /// 不再适用
import 'package:flutter_web/material.dart'; /// 现在用这种

import 'dart:ui'; /// 不再适用
import 'package:flutter_web_ui/ui.dart'; /// 现在用这种
```


另外由于项目名也变了，比如之前是`hello_world`，现在是`hello_world_web`，所有相关的`import`也要改下。


这么做以后，如果你的项目没有任何第三方依赖的话，编译问题基本算是解决了。如果有第三方依赖，嘿嘿，这就比较麻烦了。我们的项目就用到了如下依赖：

``` dart
  percent_indicator: ^2.1.1
  pull_to_refresh: ^1.5.1
  fluttertoast: ^3.1.0
  flutter_spinkit: "^3.1.0"
  modal_progress_hud: ^0.1.3
  sticky_headers: "^0.1.8"
```

直接把这些依赖添加到`Flutter Web`项目中会报错：

```
Resolving dependencies...
Because percent_indicator >=1.0.6 depends on flutter any from sdk which is forbidden, percent_indicator >=1.0.6 is forbidden.
So, because iwords_web depends on percent_indicator ^2.1.1, version solving failed. 
```

也就是说，这些依赖库都是基于`Flutter`开发的，在`Flutter Web`项目中不能用。。。 
于是去[pub.dev](https://pub.dev/)上找有没有对应的`Web`版本依赖，结果发现一个都没有。。。

为了能最终看到跑起来的效果，再恶心的事情也得干。遂决定把所有的依赖库全部源码引入工程中，然后把依赖库逐个改成支持`Flutter Web`的版本，这是个纯体力活，此处省略1000字。。。

这一步做完，编译问题应该没有了。接下来要处理资源问题了。

以前是通过在`pubspec.yaml`中指定`assets`路径的方式，现在同样的方法试了试不起作用。然后官网查了一圈也没说资源怎么指定，最后在`Flutter Web`官方的`Sample`[代码](https://github.com/flutter/samples/tree/master/web)中找到了方案：

![web_assets.png](https://upload-images.jianshu.io/upload_images/138119-92689c6798e5416d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

于是把原项目中的`assets`文件夹搬到新项目的`web`文件夹下，然后引用资源的路径也调整了下：

``` dart
/// 老路径
FadeInImage.assetNetwork(
placeholder:'assets/images/book_placeholder.png',
image: item.book.coverImageUrl,
fit: BoxFit.cover)

/// 新路径
FadeInImage.assetNetwork(
placeholder:'images/book_placeholder.png',
image: item.book.coverImageUrl,
fit: BoxFit.cover)
```

这样图片资源就能显示出来了。

然后我们跑一把项目：

```
flutter pub get # 获取第三方依赖
webdev serve  # 把编译好的js部署到web server
```
如果`webdev serve`这一步出错的话，可以重启下机器试试。

然后默认是通过`localhost:8080`打开页面。如果需要指定不同端口号，可以用下面命令：

```
webdev serve web:3002
```

打开页面后，发现所有的网络请求都失败，查了下是跨域问题，接口都是跨域访问的。最终确认是服务端需要做如下调整：

1. 接口需要支持`OPTIONS mode`
2. `OPTIONS mode`的接口`reponse header`需要支持跨域

在服务端改接口之前，可以用`Charles`的`rewrite`功能来`mock`下，让所有接口都支持跨域：

![charles_rewrite.png](https://upload-images.jianshu.io/upload_images/138119-1e21b9237f81db9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

好了，到这里，迁移过程基本完成。

还留了一些坑，等`Flutter Web`稳定了以后再看看：

1. 有些文字的布局出现了问题
2. 有些图片显示有问题
3. 比较卡顿，特别是窗口缩放的时候，感觉是页面渲染卡主线程了，导致浏览器拖放卡顿

最后简单梳理下迁移需要做的事情：

1. 用新的脚手架创建一个`Flutter Web`项目，把代码搬过去
2. `import`方式调整
3. 第三方依赖处理，本文简单粗暴的选择了源码引入方式，读者可以选择更优雅的方式
4. 搬资源，并调整资源路径
5. 接口跨域处理

完。