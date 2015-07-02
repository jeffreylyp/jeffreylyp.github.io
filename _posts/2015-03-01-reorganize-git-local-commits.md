---
layout: post
title: 重新组织git本地提交
description: 描述怎么整理git的本地提交
---

## 前言 ##

以git为首的SVCS现在已经很流行了，目前越来的企业和开源组织都在使用或者迁移到git上，github让git更加出色和流行。每次提到git，都会和SVN进行对比，当然这其中可以对比的因素。我这次主要就可以git可以本地提交的特性讲述如何在push之前对本地的commit进行重组。

## 本地commit重组的必要性 ##

在开发过程中，有很多优秀的实践指导着我们，其中有一条是“每次提交一个功能点”，意思是说每次提交应该是一个独立的单元，还有另外一条优秀实践是“频繁提交，小提交”，意思是说要频繁提交代码，并且最好是小的提交。其实这两条都没错，但是总是感觉有点冲突和矛盾。如果想完美做到第一条，那么写代码之前要好好规划，每次提交要修改哪些文件，然后才能一次提交一个完整的单元，但是这么做又会产生大commit，和第二条有冲突。考虑到很难在开发之前，设计那么清楚，就像设计大功能一样，可以边做边重构，甚至是做完重构。

所以针对git本地的commit也是一样，可以先比较频繁地执行小提交，这样每个小提交可以达到一个提交一个功能了。这样本地会产生很多commit，这些commit比较散乱，逻辑关联性也不强，甚至有些commit是本地测试用的，有的commit需要拆成两个，有个多个commit可以合并，commit之间顺序需要调换等等，经过这些过程，本地commit可以调整得很有序，而且组织性特别好。

## 案例 ##

需要开发三个功能function-1、function-2以及function-3，由于自己没有想清楚，开发有点散乱，提交之后的git log如下：

    8e68ded function-3-all-big
    aad65ef function-2[2/2]
    721d5e9 function-1[2/2]
    8ac73c9 function-2[1/2]
    f1927eb modified properties to local test
    b29c96f function-1[1/2]
从log可以看到，function-1被分在了两个commit提交，而且中间夹杂着一个为了测试修改配置的commit（这个commit不应该push到代码库），仔细去看，function-1和function-2穿插着提交了。而function-3是一个比较大的提交，提交之后我觉得可以function-3完全可以拆成两个子功能分开提交。

## 如何实现 ##

第一步：先把function-1和function-2的穿插问题解决掉，就是调换function-2[1/2]和function-1[2/2]顺序，因为是两个功能，所以代码不会很大的冲突。

执行git rebase -i进入交互模式，自动打开vim，内容如下：

      pick b29c96f function-1[1/2]
      pick f1927eb modified properties to local test
      pick 8ac73c9 function-2[1/2]
      pick 721d5e9 function-1[2/2]
      pick aad65ef function-2[2/2]
      pick ac10187 function-3-all-big
从这个看上去和git log的输出很相似，只是顺序恰好是倒置的，最先提交的commit在最上边。现在调换function-2[1/2]和function-1[2/2]顺序。

      pick b29c96f function-1[1/2]
      pick f1927eb modified properties to local test
      pick 721d5e9 function-1[2/2]
      pick 8ac73c9 function-2[1/2]
      pick aad65ef function-2[2/2]
      pick ac10187 function-3-all-big
保存，退出编辑器，git会自动执行rebase操作，之后执行git log观察下输出：
      
      b761ac1 function-3-all-big
      4152266 function-2[2/2]
      9c09ceb function-2[1/2]
      c9fed87 function-1[2/2]
      f1927eb modified properties to local test
      b29c96f function-1[1/2]
发现function-2[1/2]和function-1[2/2]顺序已经调换了。

第二步：删除modified properties to local test这个commit，使得本地测试代码不会push到远程代码库。这个比较容易，执行git rebase -i，在编辑器中直接删除pick f1927eb modified properties to local test这一行，保存并退出编辑器，执行rebase操作。此后可以执行git log看下输出内容：

      998a582 function-3-all-big
      b67b728 function-2[2/2]
      ff9f827 function-2[1/2]
      1a7ebaa function-1[2/2]
      b29c96f function-1[1/2]
不需要push的那个提交已经被踢出了。

第三步：把需要合并的提交合并掉使其变成一个更内聚的提交。首先合并function-1[1/2]和function-1[2/2]，执行git rebase -i，输出如下：

      pick b29c96f function-1[1/2]
      pick 1a7ebaa function-1[2/2]
      pick ff9f827 function-2[1/2]
      pick b67b728 function-2[2/2]
      pick 998a582 function-3-all-big
这个只需要把第二行的pick改成s，保存退出编辑器，这个时候会在编辑器中重新编辑前面两个commit的comment，于是修改正function-1即可。看下git log输出：

      ac884c8 function-3-all-big
      b992119 function-2[2/2]
      9549400 function-2[1/2]
      150ff1a function-1
这样看来function-1的两部分被合并了，变成一个单一的commit了。同样的方式来处理function2的两个commit。最后的git log输出：

      236128f function-3-all-big
      1188917 function-2
      150ff1a function-1
function-1和function-2都合并了。

第四步：拆分function-3为function-3-module-1和function-3-module-2两个独立commit。先执行git reset --soft HEAD^，这样先回退一个commit，变成function-3提交前的状态，这样可以commit的内容还在，只是出于未提交状态。这样就可以自由选择把未提交的内容分成几次提交了，我们做成两个commit。现在git log看下结果：

      3a27021 function-3-module-2
      7829bd8 function-3-module-1
      1188917 function-2
      150ff1a function-1

第五步：结果已经完美了，执行git push，当然先执行git pull --rebase可能是个更好的习惯。

## 总结 ##

因为git有local commit，既可以“随意”提交小提交，又可以在push之前修整成很漂亮的功能单元，一个commit一个单元，从而让我们有“后悔药”，也是为了做出更好的代码质量。不过一定要注意，rebase应该只操作还未push到远程仓库的commit，一旦push到了远程仓库，那么不允许再修改commit，不然会给其他开发带来很多麻烦。
