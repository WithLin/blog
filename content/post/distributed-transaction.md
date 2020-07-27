---
title: "事务解决方案以及 .Net Core 下的实现（上）"
date:  2019-03-30T23:18:36+08:00
lastmod: 2019-03-31T20:05:51+08:00
draft: false
author: "WithLin"
tags: ["分布式事务", "saga", "中间件"]
categories: ["分布式事务", "saga"]
toc: true
comment: true
autoCollapseToc: false
---

>数据一致性是构建业务系统需要考虑的重要问题 ， 以往我们是依靠数据库来保证数据的一致性。但是在微服务架构以及分布式环境下实现数据一致性是一个很有挑战的的问题。最近在研究分布式事物，分布式的解决方案有很多解决方案，也让我在研究的同时也引发了很多思考。今天我想讲的是分布式事物解决方案是和saga有关。


#### 原文地址:[微服务场景下的数据一致性解决方案](https://opentalk.upyun.com/310.html)

#### PPT地址:[Saga分布式事务解决方案与实践](http://servicecomb.incubator.apache.org/cn/docs/distributed-transactions-saga-implementation/)
#### incubator-servicecomb-saga地址：[incubator-servicecomb-saga](https://github.com/apache/incubator-servicecomb-saga)
#### servicecomb-saga-csharp（servicecomb-saga  netcore sdk）地址：[servicecomb-saga-csharp](https://github.com/OpenSagas-csharp/servicecomb-saga-csharp)

> 根据原文做一些解释性的地方 方便更加理解
## 单体应用的数据一致性

我就给大家讲一个国外经常用到的例子吧，就是假如有一家大型的企业，下属有航空公司、租车公司、和连锁酒店。这个大公司为客户提供一站式的旅游行程规划服务，这样客户只需要提供出行目的地， 这个大公司能帮助客户预订机票、租车、以及预订酒店。从业务的角度，我们必须保证上述三个服务的预订都完成才能满足一个成功的旅游行程，否则不能成行。

我们的单体应用要满足这个需求非常简单，只需将这个三个服务请求放到同一个数据库事务中，数据库会帮我们保证全部成功或者全部回滚。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-a17d114cf8d3a0f1.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)
>这三个服务上线公司满意，客户也很满意

## 微服务场景下的数据一致性
随之时间的推移，这个大企业的行程规划服务非常成功，用户量剧增上百倍。企业的下属航空公司、租车公司、和连锁酒店也相继推出了更多服务以满足客户需求， 我们的应用和开发团队也因此日渐庞大。如今我们的单体应用已变得如此复杂，以至于没人了解整个应用是怎么运作的。更糟的是新功能的上线现在需要所有研发团队合作， 日夜奋战数周才能完成。看着市场占有率每况愈下，公司高层对研发部门越来越不满意。

经过数轮讨论，领导最终决定将庞大的单体应用一分为四：机票预订服务、租车服务、酒店预订服务、和支付服务。服务各自使用自己的数据库，并通过HTTP协议通信。 负责各服务的团队根据市场需求按照自己的开发节奏发版上线。如今我们面临新的挑战：如何保证最初三个服务的预订都完成才能满足一个成功的旅游行程， 否则不能成行的业务规则？现在服务有各自的边界，而且数据库选型也不尽相同，通过数据库保证数据一致性的方案已不可行。
![image.png](https://upload-images.jianshu.io/upload_images/5590759-f849ffb54f2fd434.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Sagas
经过一段时间的查找，我发现了一篇论文，1987年Hector & Kenneth 发表论文 Sagas[论文地址](https://www.cs.cornell.edu/andru/cs711/2002fa/reading/sagas.pdf)

>Saga是一个长活事务(Long Live Transaction (LLT))，可被分解成可以交错运行的子事务集合。其中每个子事务都是一个保持数据库一致性的真实事务(LLT = T1 + T2 + T3 + ... + Tn)。每个本地事务Tx 有对应的补偿 Cx。

在大企业的业务场景下，一个行程规划的事务就是一个Saga，其中包含四个子事务：机票预订、租车、酒店预订、和支付。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-0213414bba5afd06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

根据上面提到的公式
>当每个saga子事务 T1, T2, …, Tn 都有对应的补偿定义 C1, C2, …, Cn-1, 那么saga系统可以保证 [1]子事务序列 T1, T2, …, Tn得以完成 (最佳情况)或者序列 
T1, T2, …, Tj, Cj, …, 
C2, C1, 0 < j < n, 
得以完成

![image.png](https://upload-images.jianshu.io/upload_images/5590759-f807ffb4405f563c.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

换句话说，通过上述定义的事务/补偿，saga保证满足以下业务规则：

所有的预订都被执行成功，如果任何一个失败，都会被取消
如果最后一步付款失败，所有预订也将被取消，这些取消就是所谓的补偿。

## Saga的恢复方式
原论文中描述了两种类型的Saga恢复方式：

>向后恢复 补偿所有已完成的事务，如果任一子事务失败。向前恢复 重试失败的事务，假设每个子事务最终都会成功

显然，向前恢复没有必要提供补偿事务，如果你的业务中，子事务（最终）总会成功，或补偿事务难以定义或不可能，向前恢复更符合你的需求。

理论上补偿事务永不失败，然而，在分布式世界中，我们来想想极端的情况，无非就是往三种可能去考虑，成功，失败，超时(有可能成功，也有可能失败)。那么服务器可能会宕机，网络可能会失败，甚至数据中心也可能会停电。在这种情况下我们能做些什么？ 最后的手段是提供回退措施，比如人工干预。

>补充说明:ACID与SAGA
- 原子性(Atomicity)：Saga只提供ACD保证,原子性（通过Saga协调器实现）
- 一致性(Consistency)：本地事务 + Saga log
- 隔离性（Isolation）：Saga不保证
- 持久性（Durability）：Saga log 提供

##### 有很多朋友会说怎么不提供隔离性啊？
>例子地址:[地址](https://microservices.io/microservices/general/2018/03/22/microxchg-sagas.html)
- 两个Saga事务同时操作一个资源会出现数据语义不一致的的情况
- 两个Saga事务同时操作一个订单 ，彼此操作会覆盖对方（更新丢失）
- 两个Saga事务同时访问扣款账号，无法看到退款 （脏读取问题）
- 在一个Saga事务内，数据被其他事务修改前后的读取值不一致（模糊读取问题）

##### 面对以上问题我们应该如何应对隔离性问题呢？
>下面给出对应的解决方案
- 隔离的本质是控制并发，防止并发事务操作相同资源而引起结果错乱
- 在应用层面加入逻辑锁的逻辑。
- 业务层面采用预先冻结资金的方式隔离此部分资金。
- 业务操作过程中通过及时读取当前状态的方式获取更新。

## 使用Saga的条件
Saga看起来很有希望满足我们的需求。所有长活事务都可以这样做吗？这里有一些限制：

Saga只允许两个层次的嵌套，顶级的Saga和简单子事务 [1]
在外层，全原子性不能得到满足。也就是说，sagas可能会看到其他sagas的部分结果 [1]
每个子事务应该是独立的原子行为 [2] 
在我们的业务场景下，航班预订、租车、酒店预订和付款是自然独立的行为，而且每个事务都可以用对应服务的数据库保证原子操作。
我们在行程规划事务层面也不需要原子性。一个用户可以预订最后一张机票，而后由于信用卡余额不足而被取消。同时另一个用户可能开始会看到已无余票， 接着由于前者预订被取消，最后一张机票被释放，而抢到最后一个座位并完成行程规划。

补偿也有需考虑的事项：

补偿事务从语义角度撤消了事务Ti的行为，但未必能将数据库返回到执行Ti时的状态。（例如，如果事务触发导弹发射， 则可能无法撤消此操作）
但这对我们的业务来说不是问题。其实难以撤消的行为也有可能被补偿。例如，发送电邮的事务可以通过发送解释问题的另一封电邮来补偿。

现在我们有了通过Saga来解决数据一致性问题的方案。它允许我们成功地执行所有事务，或在任何事务失败的情况下，补偿已成功的事务。 虽然Saga不提供ACID保证，但仍适用于许多数据最终一致性的场景。那我们如何设计一个Saga系统？




## Saga Log
Saga保证所有的子事务都得以完成或补偿，但Saga系统本身也可能会崩溃。Saga崩溃时可能处于以下几个状态：

- Saga收到事务请求，但尚未开始。因子事务对应的微服务状态未被Saga修改，我们什么也不需要做。
- 一些子事务已经完成。重启后，Saga必须接着上次完成的事务恢复。
- 子事务已开始，但尚未完成。由于远程服务可能已完成事务，也可能事务失败，甚至服务请求超时，saga只能重新发起之前未确认完成的子事务。这意味着子事务必须幂等。
- 子事务失败，其补偿事务尚未开始。Saga必须在重启后执行对应补偿事务。


补偿事务已开始但尚未完成。解决方案与上一个相同。这意味着补偿事务也必须是幂等的。
所有子事务或补偿事务均已完成，与第一种情况相同。
为了恢复到上述状态，我们必须追踪子事务及补偿事务的每一步。我们决定通过事件的方式达到以上要求，并将以下事件保存在名为saga log的持久存储中：

- Saga started event 保存整个saga请求，其中包括多个事务/补偿请求
- Transaction started event 保存对应事务请求
- Transaction ended event 保存对应事务请求及其回复
- Transaction aborted event 保存对应事务请求和失败的原因
- Transaction compensated event 保存对应补偿请求及其回复
- Saga ended event 标志着saga事务请求的结束，不需要保存任何内容

![image.png](https://upload-images.jianshu.io/upload_images/5590759-77aa010a58216b13.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

通过将这些事件持久化在saga log中，我们可以将saga恢复到上述任何状态。

由于Saga只需要做事件的持久化，而事件内容以JSON的形式存储，Saga log的实现非常灵活，数据库（SQL或NoSQL），持久消息队列，甚至普通文件可以用作事件存储， 当然有些能更快得帮saga恢复状态。

##Saga 请求的数据结构
在我们的业务场景下，航班预订、租车、和酒店预订没有依赖关系，可以并行处理，但对于我们的客户来说，只在所有预订成功后一次付费更加友好。 那么这四个服务的事务关系可以用下图表示：

![image.png](https://upload-images.jianshu.io/upload_images/5590759-0213414bba5afd06.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

将行程规划请求的数据结构实现为有向非循环图恰好合适。 图的根是saga启动任务，叶是saga结束任务。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-641064ea0e37231b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

## Parallel Saga
如上所述，航班预订，租车和酒店预订可以并行处理。但是这样做会造成另一个问题：如果航班预订失败，而租车正在处理怎么办？我们不能一直等待租车服务回应， 因为不知道需要等多久。

最好的办法是再次发送租车请求，获得回应，以便我们能够继续补偿操作。但如果租车服务永不回应，我们可能需要采取回退措施，比如手动干预。

超时的预订请求可能最后仍被租车服务收到，这时服务已经处理了相同的预订和取消请求。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-baf7c4d0f5a4a37b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

因此，服务的实现必须保证补偿请求执行以后，再次收到的对应事务请求无效。 Caitie McCaffrey在她的演讲Distributed Sagas: A Protocol for Coordinating Microservices中把这个称为可交换的补偿请求 (commutative compensating request)。


## 分布式saga架构

>分布式的Saga借鉴了zipkin的思想，Omega就是类似探针的形式，上报saga事件，然后Alpha是属于Saga的ProcessManager.也就是协调器的东西。

- alpha充当协调者的角色，主要负责对事务的事件进行持久化存储以及协调子事务的状态，使其得以最终与全局事务的状态保持一致。
- omega是微服务中内嵌的一个agent，负责对网络请求进行拦截并向alpha上报事务事件，并在异常情况下根据alpha下发的指令执行相应的补偿操作。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-eb24889a43dea369.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


接下来我们看下Omega的内部实现
>omega是微服务中内嵌的一个agent。当服务收到请求时，omega会将其拦截并从中提取请求信息中的全局事务id作为其自身的全局事务id（即Saga事件id），并提取本地事务id作为其父事务id。在预处理阶段，alpha会记录事务开始的事件；在后处理阶段，alpha会记录事务结束的事件。因此，每个成功的子事务都有一一对应的开始及结束事件。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-cc7c0e5cb6ce5b0b.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

我们再看下 他们是如何通信的
>服务间通信的流程与[Zipkin](https://github.com/openzipkin/zipkin)的类似。在服务生产方，omega会拦截请求中事务相关的id来提取事务的上下文。在服务消费方，omega会在请求中注入事务相关的id来传递事务的上下文。通过服务提供方和服务消费方的这种协作处理，子事务能连接起来形成一个完整的全局事务。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-f95ae49886de4b9a.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

> 借助zipkin的思想就可以让整一个事务组形成一个链式结构。

![image.png](https://upload-images.jianshu.io/upload_images/5590759-6489ed6ec15b803e.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


## Saga 具体处理流程
Saga处理场景是要求相关的子事务提供事务处理函数同时也提供补偿函数。Saga协调器alpha会根据事务的执行情况向omega发送相关的指令，确定是否向前重试或者向后恢复。

成功场景
成功场景下，每个事务都会有开始和有对应的结束事件。
![image.png](https://upload-images.jianshu.io/upload_images/5590759-56adac1e4320361f.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

异常场景
异常场景下，omega会向alpha上报中断事件，然后alpha会向该全局事务的其它已完成的子事务发送补偿指令，确保最终所有的子事务要么都成功，要么都回滚。
![image.png](https://upload-images.jianshu.io/upload_images/5590759-b431be4f7aaf3fc7.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)

超时场景 (需要调整）
超时场景下，已超时的事件会被alpha的定期扫描器检测出来，与此同时，该超时事务对应的全局事务也会被中断。
![image.png](https://upload-images.jianshu.io/upload_images/5590759-e94e86fcc75be11d.png?imageMogr2/auto-orient/strip%7CimageView2/2/w/1240)


以上都是介绍完[incubator-servicecomb-saga](https://github.com/apache/incubator-servicecomb-saga) 总体架构。我觉得它的idea很nice，所以我和[水哥](https://github.com/jonechenug),还有[老杜](https://github.com/djl394922860)做了一个很有趣的事情。什么事情呢？就是实现了Omega这个客户端，github地址在这里:[servicecomb-saga-csharp](https://github.com/OpenSagas-csharp/servicecomb-saga-csharp),目前实现上面的三种场景。

下篇结合实际的sample和大家讲解下netcore下的实现，这篇文章让大家整体的了解什么是saga。