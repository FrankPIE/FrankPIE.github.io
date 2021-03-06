---
layout: post
title: 图像处理工具ImageMagick简介
subtitle: 图像处理工具ImageMagick简介
description: > 
    关于图像处理批处理工具ImageMagick的简单介绍
tags: 
    - 图形图像
    - 效率工具
hero: /hero/ImageMagick_logo.png
overlay: red
published: true
---

## 起因

最近在做一个美术类产品项目，为了让我们产品的内容更加丰富，自然少不了到处收集一些中外名画的高清资源，于是产品同学兴冲冲的就去淘宝上跟地摊扫货一般买到了一堆百度云链接，作为一个文艺盲，对于这些中外画作简直是两眼一抹黑。这些图的艺术价值有多高我是不知道了，但是我是真的难受，真的，因为这些画实在太****大了！！！最大的一张清明上河图原件居然有2G多，合计了一下大概得有一个多T的图片资源！要不是产品同学跑得快，可能当场就被我人道毁灭了。不要说我残忍，这光resize的工作量就可以累死我们的平面，而且很低效，看着平面绝望的眼神，我决定写个工具把他拯救出来。
<!–-break-–>

## 思考

一般平面处理这种基本有几种情况，2B平面——打开PS一张一张改；普通平面——搞个PhotoShop的批处理动作，然后处理，比较快速；文艺平面——交给2B平面去做。虽然这些都是玩笑话，但是太多的美术在这种活上面浪费了太多太多的时间，可能只是修改一下图片大小，却要操作几千几万张图片，然后一天过去了，两天过去了...一个礼拜就过去了，绩效也没有了。而且对于我们今天遇到的问题，用PS都是一种非常费时的操作，打开都是上百兆甚至上G的图片，简直是内存杀手。这个时候我就在想，当PS都不适用的时候还有什么能拯救我们呢？于是我开始谷歌一下我就知道，果然一搜一水的吐槽，很多人都意识到了这个问题，也给出了很多解决方法，在搜索了一圈后发现了[ImageMagick](http://www.imagemagick.org/script/index.php)。

打开页面看见一个拿着仙女棒，穿着睡衣的变态白胡子老头的logo，作者挑选logo的审美风格奇葩的让人讨厌不起来。

简单看了看，对于各种语言的支持相当完整，又是命令行操作还跨平台，这个工具除了优雅还能说什么。

深入看了看，有一个命令行工具的文档介绍，那就万事俱备了，就差把代码写上了。

## 解决

在语言的选择上，作为一个内部效率小工具，那么python可能是最合适的了，我并没有去安装它关于python binding的库，其实如果只是想写个批处理小工具用的话，没有必要费劲巴拉的去研究它的库，这个软件的命令行工具非常完善，可以直接通过脚本外部调用系统命令行来完成相应的操作，依赖的环境条件会很少，效率也高，关键是可以不使用任何图形化界面来增加处理效率。前后代码不到三十行，简单递归遍历下根目录下的图片文件与目录，然后利用ImageMagick进行图形处理，前后写完加测试也就20来分钟。然后给平面美滋滋的让他们跑起来，效率又高又不容易出现错误。
