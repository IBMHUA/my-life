> 这篇文章是在一个微信群里和人聊天，然后整理出来的文字。当时Hulu推出了基于Yarn的Docker调度引擎。我正好那段时间也实现了一个类似的，经过交流，发现最后的实现基本是一致的。然而业界用的较多的是Mesos,这篇文章就是为了解释为什么选择用Yarn而不是Mesos来做。

## 前言

Mesos 其实我不是非常熟悉，所以有些内容可能会有失偏颇，带有个人喜好。大家也还是需要有自己的鉴别能力

Mesos和Yarn 都非常棒，都是可编程的框架。一个硬件，不能编程，就是死的，一旦可以编程就活了，就可以各种折腾，有各种奇思妙想可以实现，同样的，一个软件，只要是可编程的，基本也就活了，容易形成生态。

## Yarn VS Mesos

我先说说在做容器调度引擎的时候，为什么选择Yarn而不是Mesos.

*** 可部署性 ***

先说明下，这里探讨的是Yarn或者Mesos集群的部署,不涉其上的应用。Yarn除了依赖JDK,对操作系统没有任何依赖，基本上放上去就能跑。Mesos因为是C/C++开发的，安装部署可能会有库依赖。 这点我不知道大家是否看的重，反正我是看的相当重的。软件就应该是下下来就可以Run。所以12年的时候我就自己开发了一套Java服务框架，开发完之后运行个main方法就行。让应用包含容器，而不是要把应用丢到tomcat这些容器，太复杂，不符合直觉。

***  二次开发 *** 

Yarn 对Java/Scala工程师而言，只是个Jar包，类似索引开发包Lucene，你可以把它引入项目，做任何你想要的包装。 这是其一。

其二，Yarn提供了非常多的扩展接口，很多实现都是可插拔
可替换的，在xml配置下，可以很方便的用你的实现替换掉原来的实现，没有太大的侵入性，所以就算是未来Yarn升级，也不会有太大问题。

相比较而言，Mesos更像是一个已经做好的产品，部署了可以直接用，但是对二次开发并不友好。

*** 生态优势 *** 

Yarn 诞生于Hadoop这个大数据的“始作俑者”项目，所以在大数据领域具有先天优势

1. 底层天然就是分布式存储系统HDFS，稳定高效。
2. 其上支撑了Spark,MR等大数据领域的扛顶之座，久经考验。
3. 社区强大，最近发布版本也明显加快，对于长任务的支持也越来越优秀。


谈及长任务(long running services)的支持，其实大可不必担心，譬如现在基于其上的Spark Streaming 就是7x24小时运行的。一般而言，要支持长任务，需要考虑如下几个点：

1. Fault tolerance. 主要是AM的容错
2. Yarn Security. 如果开启了安全机制，令牌等的失效时间也是需要注意的
3. 日志收集到集群
4. 还有就是资源隔离和优先级

大家感兴趣可以先参考Jira https://issues.apache.org/jira/browse/YARN-896.
我看这个Jira 13年就开始了。说明这事很早就被重视起来了。

*** Fault tolerance *** 

* Yarn 自身高可用。目前Yarn的Master已经实现了HA.
*  AM容错，我看从2.4版本(看的源码，也可能更早的版本就已经支持)就已经支持 keep containers across attempt 的选项了。什么意思呢？就是如果AM挂掉了，在Yarn重新启动AM的过程中，所有由AM管理的容器都会被保持而不会被杀掉。除非Yarn多次尝试都没办法把AM再启动起来(默认两次)。 这说明从底层调度上来看，已经做的很好了。

*** 日志收集到集群 *** 

日志收集在2.6版本已经是边运行边收集了。

*** 资源隔离 *** 

资源隔离的话，Yarn做的不好，目前有效的是内存，对其他方面一直想做支持，但一直有限。这估计也是很多人最后选择Mesos的缘由。但是现在这点优势Mesos其实已经荡然无存，因为Docker容器在资源隔离上已经做的足够好。Yarn和Docker一整合，就互补了。

## 总结

Mesos 和 Yarn 都是非常优秀的调度框架，各有其优缺点，弹性调度，统一的资源管理是未来平台的一个趋势，类似的这种资源管理调度框架必定会大行其道。

## 一些常见的误解 

脱胎于Hadoop，继承了他的光环和生态，然而这也会给其带来一定的困惑，首先就是光环一直被Hadoop给盖住了，而且由于固有的惯性，大家会理所当然的认为Yarn只是Hadoop里的一个组件，有人会想过把Yarn拿出来单独用么？

然而，就像我在之前的一篇课程里，反复强调，Hadoop是一个软件集合，包含分布式存储，资源管理调度，计算框架三个部分。他们之间没有必然的关系，是可以独立开来的。而Yarn 就是一个资源管理调度引擎，其一开始的设计目标就是为了通用，不仅仅是跑MR。现在基于Yarn之上的服务已经非常多，典型的比如Spark。

这里还有另外一个误区，MR目前基本算是离线批量的代名词，这回让人误以为Yarn也只是适合批量离线任务的调度。其实不然，我在上面已经给出了分析，Yarn 是完全可以保证长任务的稳定可靠的运行的。
