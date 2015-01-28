---
layout: post
title: SNAPSHOT or RELEASE
description: 依赖版本的选择
---

目前在JAVA的世界中，maven已经成为事实上的构建标准，很多开源库的管理构建也是基于maven的，maven本身的学习曲线比较陡峭，遵循“约定优于配置”的理念，maven存在很多约定。本次我先描述下，关于版本的定义的选择，SNAPSHOT or RELEASE？

## 版本之争 ##
在maven的约定中，依赖的版本分为两类——SNAPSHOT和RELEASE。SNAPSHOT依赖泛指以-SNAPSHOT为结尾的版本号，例如1.0.1-SNAPSHOT。除此之外，所有非-SNAPSHOT结尾的版本号则都被认定为RELEASE版本，即正式版，虽然会有beta、rc之类说法，但是这些只是软件工程角度的测试版，对于maven而言，这些都是RELEASE版本。既然Maven提供了这两类版本号，那么他们之前的优劣势是什么？分别在什么场景下使用？

## 解读SNAPSHOT ##
同一个SNAPSHOT版本的依赖可以多次发布（deploy）到仓库中，也就是说同一个SNAPSHOT版本的依赖可以在仓库中存在多份，每一份都是代码在某一个特定时间的快照，这也是SNAPSHOT的含义。

很好地表达了SNAPSHOT的细节，也阐述了一个SNAPSHOT很重要观点——SNAPSHOT不是一个特定的版本，而是一系列的版本的集合，其中HEAD总是指向最新的快照，对外界可见的一般也是最新版，这种给人的假象是新的覆盖了老的，从而使得使用SNAPSHOT依赖的客户端总是通过重新构建（有时候需要-U强制更新）就可以拿到最新的代码。例如：A-->B-1.3.8-SNAPSHOT（理解为A依赖了B的1.3.8-SNAPSHOT版本），那么B-1.3.8-SNAPSHOT更新之后重新deploy到仓库之后，A只需要重新构建就可以拿到最新的代码，并不需要改变依赖B的版本。由此可见，这样达到了变更传达的透明性，这对于开发过程中的团队协作的帮助不言而喻。

## SNAPSHOT之殇 ##

SNAPSHOT版本的依赖因为存在变更传达的透明性的优势而被赏识，甚至被“溺爱”，有很多团队索性直接使用SNAPSHOT到生产环境中，这样对于变更直接生效，很方便。但是作为技术人员的我们其实应该很严谨地看待变更传达的透明性，变更就意味着风险，透明性更是把风险彻底隐藏了起来，生产环境中存在这样的现象更是心惊胆战。例如：A-->B.1.0.3-SNAPSHOT，B对一个A使用的功能实现进行了调整，直接发布到仓库，A重新构建或许就会失败，更糟糕的是构建成功，运行时异常。这个时候A甚至完全没有代码变更就突然失败了，会带来更多的困惑。

这也是maven经常遭人诟病的一个因素，对于同一份代码，构建结果却不具备确定性，让很多人沮丧。当然这个不完全是因为依赖的问题，也有maven插件的问题，maven之前的版本寻找插件策略的方式也存在不确定性，maven在版本2的时候，会去寻找最新的插件版本（如果没配置的话）来执行构建，经常会找到SNAPSHOT版本的插件，所以依赖了一个不稳定的插件来执行构建，不确定性就大大增加。不过maven在3版本就改变了这个策略，会寻找最新稳定版的插件来执行构建，使得构建具备了确定性，稳定性也好多了。说明maven本身也在SNAPSHOT的问题了摔了一跤。

归根到底，这些问题的根源就是SNAPSHOT是变化的，是不稳定的，而应用（软件）依赖于变化并且不稳定的SNAPSHOT的依赖会导致自身也在变化和不稳定中，这是稳定性的一个大忌，依赖不稳定的服务或者依赖，上述的maven2的问题就是一个典型反例。

## RELEASE简介 ##

RELEASE版本和SNAPSHOT是相对的，非SANPSHOT版本即RELEASE版本，RELEASE版本是一个稳定的版本号，看清楚咯，是一个，不是一系列，可以认为RELEASE版本是不可变化的，一旦发布，即永远不会变化。

虽然RELEASE版本是稳定不变的，但是仓库还是有策略让这个原则变得可配置，有的仓库会配置成redeploy覆盖，这样RELEASE版本就变成SNAPSHOT了，伪装成RELEASE的SNAPSHOT，会让问题更费解和棘手，我一般称这类人为“挖坑专家”。

记住，RELEASE一旦发布，就不可改变。

## 如何选择 ##

那么什么时候使用SNAPSHOT？什么时候使用RELEASE?这个可以从他们各自的特性上来看，SNAPSHOT版本的库是一直在变化的，或者说随时都会变化的，这样虽然可以获取到最新的特性，但是也存在不稳定因素，依赖一个不稳定的模块或者库会让模块自身也变得不稳定，尤其是自身对被依赖模块的变化超出掌控的情况。即使可以掌控被依赖模块的变化，也会带来不稳定的因素，因为每次变更都有引入bug的可能性。如果这么说，那么我们是不是要摒弃SANPSHOT了呢？答案肯定是否定的。

想象下，什么情况下，模块会一直变化或者变化比较剧烈？开发新特性的时候，所以对于团队之间协同开发的时候，模块之间出现依赖，变化会非常剧烈，如模块A依赖模块B，模块A必然需要最方便地获取模块B的特性，在开发期间，方便性比稳定性更重要。可以反证下，假设模块B使用RELEASE版本1.0.0，模块A依赖1.0.0，现在模块A出现了bug，需要修复下，那么A就要提供一个版本号1.0.1，这样所有依赖A模块都需要更新版本号，因为开发期间这种事情是如此多，所以会带来巨变。反观SNAPSHOT方案，如果模块B的版本是1.0.0-SNAPSHOT，模块A完全不需要修改版本号即可获取模块B的新特性。当开发进入预发布阶段，为了生产环境的稳定性，依赖应该是RELEASE版本，因为此时SNAPSHOT版本的模块自动获取新特性的特点恰恰会造成生产环境的不稳定性，生产环境上，稳定性重于一切。

## 魔幻之手 ##
现在已经很明确了，在开发期间，活跃模块的版本号使用SNAPSHOT，在生产期间，依赖RELEASE版本模块。貌似，我们找到了银弹，不过这个只是理想状态，即所有的模块的版本都在自己的掌控或者间接掌控下，只有这样你才能影响对应模块的版本号。往往是理想很丰满，现实却很骨感，如果你依赖的一个模块只有SNAPSHOT版本，并且该模块也很活跃，最无助的是模块的维护人不理会你的请求，那么是否就没辙了，只能把应用构建在不稳定模块上呢？介绍一款maven插件——[versions]，这是一个非常强大的版本管理插件，其中有个对依赖版本加锁的特性——lock-snapshots，并且提供了参数可以控制锁定的依赖，就可以实现对特定的SNAPSHOT模块锁定版本，执行的命令如下：mvn versions:lock-snapshots -DincludesList="groupId:artifactId:type:classifier:version"，执行这个命令之后，对应的版本号会变化，比如1.0.0-SNAPSHOT会变成1.0.0.20090327.172306-4，即完成了锁定，有锁定就有解锁操作，详细请看对应文档。

[versions]:   http://http//mojo.codehaus.org/versions-maven-plugin/