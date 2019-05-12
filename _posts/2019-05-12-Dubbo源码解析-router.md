---
layout:     post
title:      Dubbo源码分析-router
subtitle:   
date:       2019-05-12
author:     Jay
header-img: img/home-bg-art.jpg
catalog: true
tags:
    - Dubbo
    - middleware
---

# Dubbo源码分析-router

<font color=red>**转自[肥朝](https://www.jianshu.com/p/278e782eef85)**，个人觉得Dubbo router过程分析得很清楚。</font>

### 前言

估算了一下,`dubbo`里面涉及的东西还是比较多的.比如谈到框架的时候,设计模式都是一个老生常谈的话题,再比如我们开发中我们不常用的一些概念,`spi`、`javassist`,以及和zookeeper相关的一些知识,比如`ZKClient`的使用,这些和`dubbo`关系很密切,但是这些假如我不做一些前戏铺垫就直接把源码贴出来,那真的没啥意义.因为看源码还是要有一些基础,所以我的目标是,即使看不懂源码,但是看我的分析思路和穿插的一些面试题,都能有一些收获.

从标题就知道,这次我讲的是集群容错中的第二个关键词`Router`,中文意思就是`路由`,这个`路由`是个很有意思的词汇.因为前端的`路由`和后端的`路由`他们是不同的,但是思想是基本一致的.鉴于很多技术文章都有一个诟病,就是只讲`概念`,却不讲`应用场景`,其实`Router`在`应用隔离`,`读写分离`,`灰度发布`中都有它的影子.因此本篇用`灰度发布`的例子来做前期的铺垫。

### 灰度发布

先看看百度百科的概念

> 灰度发布是指在黑与白之间，能够平滑过渡的一种发布方式。AB test就是一种灰度发布方式，让一部分用户继续用A，一部分用户开始用B，如果用户对B没有什么反对意见，那么逐步扩大范围，把所有用户都迁移到B上面来。灰度发布可以保证整体系统的稳定，在初始灰度的时候就可以发现、调整问题，以保证其影响度。

说人话就是,你发布应用的时候,不停止对外的服务,也就是让用户感觉不到你在发布.那么下面演示一下灰度发布

1.首先在`192.168.56.2`和`192.168.56.3`两台机器上启动`Provider`,然后启动`Consumer`,如下图

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router1.png)

2.假设我们要升级`192.168.56.2`服务器上的服务,接着我们去dubbo的控制台配置路由,切断`192.168.56.2`的流量,配置完成并且启动之后,就看到此时只调用`192.168.56.3`的服务

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router2.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router3.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router4.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router5.png)

3.假设此时你在`192.168.56.2`服务器升级服务,升级完成后再次将启动服务.

4.由于服务已经升级完成,那么我们此时我们要把刚才的禁用路由取消点,于是点了`禁用`,但是此时dubbo的这个管理平台就出现了`bug`,如下图所示

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router6.png)

惊奇的发现点了`禁用`,数据就变两条了,继续点`禁用`,还是两条,而且删除还删除不了,这样就很蛋疼了...但是一直删不了也不是办法,解决办法也是有的,那就是去zookeeper上删除节点

5.由于我之前是从事iOS开发的,所以一直用的是Mac电脑,Mac上好像没有特别好用的zookeeper可视化客户端工具,于是我就用了这个`idea`的zookeeper插件,只要将这个zookeeper节点删除

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router7.png)

然后刷新控制台的界面,如下图那么就只剩下一条了

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router8.png)

6.那么此时我们再看控制台的输出,已经恢复正常,整个灰度发布流程结束

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router9.png)

### 直入主题

我们先来看看`Router`的继承体系图

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router10.png)

从图中可以看出,他有三个实现类,分别是`ConditionRouter`,`MockInvokersSelector`,`ScriptRouter`

`MockInvokersSelector`在[dubbo源码解析-集群容错架构设计](https://www.jianshu.com/p/8e007012367e)中提到这里就暂时不多做叙述。**这个类主要用于服务降级时选择`mock  invokers`，见[Dubbo 服务降级](https://xuanjian1992.top/2019/05/07/Dubbo-%E6%9C%8D%E5%8A%A1%E9%99%8D%E7%BA%A7%E5%88%86%E6%9E%90/)。**

`ScriptRouter`在`dubbo`的测试用例中就有用到,这个类的源码不多,也就124行.引用官网的描述

> 脚本路由规则 支持 JDK 脚本引擎的所有脚本，比如：javascript, jruby, groovy 等，通过 type=javascript 参数设置脚本类型，缺省为 javascript。

当然看到这里可能你可能还是没有感觉出这个类有什么不可替代的作用,你注意一下这个类中有个`ScriptEngine`的属性,那么我可以举一个应用场景给你

假如有这么个表达式如下:

```java
double d = (1+1-(2-4)*2)/24;//没有问题 

"(1+1-(2-4)*2)/24"//但是假如这个表达式是这样的字符串格式,或者更复杂的运算,那么你就不好处理了,然后这个ScriptEngine类的eval方法就能很好处理这类字符串表达式的问题
```

本篇主要讲讲`ConditionRouter(条件路由)`,条件路由主要就是根据dubbo管理控制台配置的路由规则来过滤相关的`invoker`,当我们对路由规则点击`启用`的时候,就会触发`RegistryDirectory`类的`notify`方法

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router11.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router12.png)

其实我觉得看技术类文章更重要的是看分析的思路,看的是思考过程,比如为什么这个`notify`方法传入的是`List<URL>`呢?如果看过我前两篇dubbo源码解析[dubbo源码解析-集群容错架构设计](https://www.jianshu.com/p/8e007012367e)和[dubbo源码解析-directory](https://www.jianshu.com/p/3d47873f8ad3)就明白,我的分析过程都是以`官方文档`为依据,所以这个问题的答案自然也在官方文档.下面引用一段官网文档的描述

> 所有配置最终都将转换为 URL 表示，并由服务提供方生成，经注册中心传递给消费方，各属性对应 URL 的参数，参见配置项一览表中的 "对应URL参数" 列

其实对于`Router`来说,我们最关心的就是他是怎么过滤的.所以下面这些流程代码我们先走一遍

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router13.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router14.png)

这个条件路由有一个特点,就是他的`getUrl`是有值的,同时这里分享一个`IDEA`中`debug`查看表达式内容的技巧,比如`router.getUrl()`表达式的值,如下图所示

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router15.png)

从这里我们看到,此时实现类是`ConditionRouter`,由于接下来的逻辑如果直接让大家看源码图可能不够清晰,所以我又把这个核心的筛选过程用了一个高清无码图,并且用序号标注

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router16.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router17.png)

最后的筛选结果如下,因为我们在管理后台配置了禁用`192.168.56.2`,所以最后添加进`invokers`的就只有`192.168.56.3`

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router18.png)

![](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/dubbo/router/router19.png)

### 写在末尾

这篇文章准备了好久,因为每天加班回到家都是11点多,但是我认为越是时间紧张,才能越看出一个人做事的决心,无论加班多忙,或者过年放假,每周一篇的承诺始终不变,鉴于本人才疏学浅,不对的地方还望斧正.

### 参考文章

- [路由规则(官方文档介绍)](https://dubbo.gitbooks.io/dubbo-user-book/demos/routing-rule.html)
- [dubbo源码解析-集群容错架构设计](https://www.jianshu.com/p/8e007012367e)
- [dubbo源码解析-directory](https://www.jianshu.com/p/3d47873f8ad3)