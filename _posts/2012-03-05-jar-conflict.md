---
layout: post
title: 一次完整的jar冲突解决过程
description: 记录一次完整解决jar冲突的曲折过程
---

作为Java程序员，会经常碰到jar包冲突，特别是涉及到的外部依赖越多，冲突概率就会越大。由于我们的应用代码都是使用maven来管理的，所以依赖的解决相对比较容易。不过最近碰到的一个问题，确实经历了好多步才最终定位。

## 现象 ##

应用启动过程中，*spring*容器启动失败，错误日志很明确，找不到*CollectionUtils.isEmpty()*方法，jar冲突的典型症状之一。

## Step one ##

首先，确认了这个类是apache的commons-collections中的工具类（在eclipse中ctrl+shift+T或者command+shitf+T寻找CollectionUtils是来自哪个jar中，同时也可以看看是不是含有这个方法），那么执行maven命令分析依赖树mvn dependency:tree -e -Dincludes=\*:commons-collections\*，看看应用中如何依赖commons-collections，依赖路径以及依赖版本。结果稍微出乎意料，有两个commons-collections依赖，只是groupId不同（这个时候maven是无能为力的，maven会认为这两个依赖是不同的，不会认为是一样的，因为maven对依赖的标识就是坐标，与jar中的内容无关），一个是apache-collections，另外一个是commons-collections，查看第二个中的CollectionUtils是有isEmpty方法的，第一个是没有的，而且第一个jar和第二个jar的包结构是完全相同的，apache-collections是外部系统的类库引入的，感觉好奇葩，于是果断排除之。重新查看依赖树，OK，打包重启，依然相同的错误。

## Step two ##

接下来，打开war包，看下WEB-INF/lib下的commons-collections确实是3.2.1，没有问题。说明war包没有问题，最起码war内不会冲突，war包没问题。那就是jboss有问题，由于之前碰到过类似的问题，直接去${jboss_home}/server/default/lib下查看，升级其中的commons-collections到3.2.1即可（应用的jar和容器中的冲突的话，以容器为主，因为容器被优先加载），无需打包，重启jboss。自信满满地以为问题解决了，结果错误还是没变。

## Step three ##

继续，查看jboss其他的目录，看不到commons-collections的影子。貌似到此，无路可走了。所以想想，服务器上肯定还有commons-collections-3.1存在，或许是在隐藏目录中（为毛当时没想起来用find名称搜索commons-collections）。没办法，只能通过打日志来看看CollectionUtils是从哪儿加载的了，使用了几行代码：

    URL url = getClass().getClassLoader().getResource("org/apache/commons/collections/CollectionUtils.class");
    LOG.error("CollectionUtils is loading from {}", url.getPath());
重新打包部署之后，查看日志信息，显示CollectionUtils是用WEB-INF/lib/commons-collections-3.2.1.jar中加载的，真的好意外，一直都在说，jboss中的jar优先加载啊，然后才会加载war的？那为神马会显示是用war中加载的呢？而且war中的是jar包版本是OK的，至此，基本无路可走了。除非？除非那个获取加载路径的代码是有问题的，因为和我的认知确实是冲突的。

于是就getResource()方法去问了web容器方面的“神”，给出的解释是这个方法返回的路径不一定是真实的加载路径，和具体的容器实现有关。

## Step four ##

最后，貌似打开了思路，但是什么办法可以知道类的真实记载路径呢？jvm启动参数跟踪，-XX:+TraceClassLoading加在jboss其中参数中，重启jboss，查看应用启动之后的jboss日志。好吧，日志文件很大，搜索下CollectionsUtils，确实是从一个隐藏目录加载的commons-collections，问题定位到了，解决办法不用说了。

## 写在最后 ##

如果前边真的用find（sudo find / -name "common-collections" -print）命令搜索解决了问题，那么也不会有后边对认知的颠覆，那样还会一直认为那个api是救命稻草呢，不知道要持续到什么时候呢。另外，用find命令也只能去试探性地搜索commons-collections这个文件，但是不能排除有的类库会把commons-collections的代码内嵌在自己的jar中，这种事儿见到不少。所以走了最长的路，收获一定是最多的:)。

补充一下：团队之前采取的方案一直是不采用这个方法来判断，其实也就是为了避开这个坑，这种做法多少有点不太负责任，毕竟依赖在classpath上，IDE会直接提示补充完善代码，只有踩了坑的人才知道不能用，而且还不知道根源，这样下去，我只能呵呵了。