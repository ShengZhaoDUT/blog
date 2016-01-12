---
title: Paxos 算法
date: 2016-01-12 15:44:47
categories: [Distributed_System]
tags: [Paxos, Distributed System]
---

## 引言
有人说分布式系统里面就是可用性，可扩展性，可靠性，一致性，性能等各种特性之间的trade off，没有一个功能是能将分布式所有的问题都解决好，只是根据自己现有的应用场景，对特定的几点进行优化。但是Paxos堪称是完美的算法，基本上没有特殊的限制条件，能够很方便的应用并且已经得到Google，Micosoft和开源社区的证明。至于Paxos的段子就不多说了，很多学习Paxos的人必然会看到各种Paxos的传奇来历。

#### Paxos要解决的问题
Paxos的一个典型应用场景是在RSM（replicated state machine）中，保证每个副本都执行相同顺序的操作，使所有的副本保持一致。归根结题，Paxos保证在一次Paxos实例中，确定唯一的一个值，而且这个值能够被持久化应用与其他进程。

#### 基本要求
分布式系统能够正常运行的基本要求通常是Safety & Liveness

- Only a value that has been proposed may be chosen
- Only a single value is chosen
- A process never learns that a value has been chosen unless it actually has been.
- Ensure that some proposed value is eventually chosen and, if a value has been chosen, then a process can eventually learn the value
- Ensure the progress of the system, avoid deadlock

#### 前提假设（要求）
Paxos建立在一个异步网络，没有Byzantine问题的模型中。
- 处理器：处理器之间不同步，可能会遇到错误，但是可以提供持久化的存储来记录状态。
- 网络：各节点之间可以相互通信，但是可能遇到丢包，乱序，重发等问题。

## 算法

#### 角色：
- Client：向分布式系统发起请求并等待回复，系统内部的实现对于clien是一个黑盒，请求具体发给谁，消息流是什么样的，client并不清楚。
- Acceptor：用于chosen value时投票决议。
- Proposer：负责将client的请求封装交给系统决议。
- Learner：learner代表了用户的复制参数，一旦proposer的提案被采纳，则learner就可以学习相应的结果。
为了能够保证整个Paxos协议的progress（不形成死锁且效率提高），他是一个特殊的proposer，client都把request发给这个proposer，当系统中存在一个proposer，没有冲突效果自然好，但是因为网络分区会存在脑裂现象，这个时候使用Paxos协议依然能够保证最终决策出一个值。


#### 算法推导过程：

1. 如果我们的系统里面只有一个acceptor，那acceptor就选择它接收到的第一个proposal作为chosen value就可以了，但是这样显然存在单点故障问题。

2. 这个时候就要考虑发给多机。如果是发给全部的节点，这样会退化成两阶段提交，可用性下降。这个时候就选择用majority的节点来完成，因为任何两个majority必然有一个相同的acceptor，由于每个acceptor只接收一个值，因此只有一个proposal能够得到majority，保证了系统的Liveness。
因此我们先得到一个约束：
P1. An acceptor must accept the first proposal that it receives.

3. 只有P1约束可以chosen value吗？显然不行，下面看这样一个例子：假设有两个proposal同时发起请求。
if 所有的请求能够定序，那我们自然能够选择确定的一个（比如第一个）达成一致。
    else， 乱序到达，我们需要majority来接受这个proposal。此时如果有A1,A2,A3,A4,A5，5个acceptor，其中P1被A1，A2首先accept，P2被A3，A4accept，不幸的是A5挂掉了，此时全局没有majority，算法失败（何以谈safety and liveness？）。

    4. 因此，根据3中描述的问题，能够得到结论，acceptor必须允许接受多个proposal。因此我们先对这些proposal标号（这个标号表明在全局的全序关系）。

    5. 当一个acceptpr能够接受多个proposal时，为了能够支持“基本要求”中的第二条，则必须保证所有被chosen的proposal要有相同的value。
    P2. If a proposal with value v is chosen, then every higher-numbered proposal that is chosen has value v.

    6. 批准一个value意味着多个acceptor接受了该value。因此，可以对P2进行加强。
    P2a. If a proposal with value v is chosen, then every higher-numbered proposal accepted by any acceptor has value v.(P2a：一旦一个具有value v的提案被批准（chosen），那么之后任何acceptor再次接受（accept）的提案必须具有value v。)

    7. 为了维护P1，由于通信时异步的，我们可以考虑有如下场景，一个acceptor c从来没有参与过投票，而其他acceptors已经chosen一个proposal P，此时如果又有一个proposer发出一个Proposal Q，正好发给c，根据P1，c必须接受Q，但是P和Q有不同的value值，违反了P2a，因此我们需要继续对P2a进行约束。
    P2b. If a proposal with value v is chosen, then every higher-numbered proposal issued by any proposal proposal has value v.

    8. 由于P2b不好实现，因此提出了一个更强的约束。
    我们要保证的是如果v被chosen，以后任何proposal只能是v。这就要求但凡有一个（n，v）能提出来，则要不就是之前从来没有value被chosen，那现在chosen什么样的值都不违背P2b，要不就是之前已经chosen，但是是标号小于n的最大proposal number里面包含v。
    P2c. For any v and n, if a proposal with value v ad number n is issued, the there is a set S consisting of a majority of acceptors such that either (a) no acceptors in S has accepted any proposal numbered less than n, or (b) v is the value of the highest-numbered proposal among all proposals numbered less than n accepted by the acceptors in S.
    假设具有value v的提案m获得批准，当n=m+1时，采用反证法，假如提案n不具有value v，而是具有value w，根据P2c，则存在一个多数派S1，要么他们中没有人接受过编号小于n的任何提案，要么他们已经接受的所有编号小于n的提案中编号最大的那个提案是value w。由于S1和通过提案m时的多数派C之间至少有一个公共的acceptor，所以以上两个条件都不成立，导出矛盾从而推翻假设，证明了提案n必须具有value v；
    再做一个略形式化的证明，采用反证法
    若（m+1）..（N-1）所有提案都具有value v，采用反证法，假如新提案N不具有value v，而是具有value w',根据P2c，则存在一个多数派S2，要么他们没有接受过m..（N-1）中的任何提案，要么他们已经接受的所有编号小于N的提案中编号最大的那个提案是value w'。由于S2和通过m的多数派C之间至少有一个公共的acceptor，所以至少有一个acceptor曾经接受了m，从而也可以推出S2中已接受的所有编号小于n的提案中编号最大的那个提案的编号范围在m..（N-1）之间，而根据初始假设，m..（N-1）之间的所有提案都具有value v，所以S2中已接受的所有编号小于n的提案中编号最大的那个提案肯定具有value v，导出矛盾从而推翻新提案n不具有value v的假设。根据数学归纳法，我们证明了若满足P2c，则P2b一定满足。

    9. 为了维护P2c，一个标号为n的proposal必须学习小于n的已经被chosen的最大的proposal-numbered对应的v，而这个值被chosen可能是已经发生了，也可能是将要发生（因为通信异步，还有一些acceptor没有给出答案），如果学习已经确定的这很简单（majority中肯定有），但是如果学习将来要chosen的比较难，因此这个地方又做了一个限制。
    The proposer requests that the acceptors not accept any more proposals numbered less than n.
    P1a. An acceptor can accept a proposal numbered n iff it has not responded to a prepare request having a number greater than n.

    整理一下上述证明过程，能够得到的结果如下：

#### 算法描述

prepare阶段：
- proposer选择一个提案编号n并将prepare请求发送给acceptors中的一个多数派
- acceptor收到prepare消息后，如果当前的编号大于它已经回复的所有prepare消息，则acceptor将自己上次接受的提案回复给proposer，并承诺不再回复小于n的提案

批准阶段：
- 当一个proposer收到了多数acceptors对prepare恢复后，就进入批准阶段。他要向acceptors发送accept请求，包括编号n和根据P2c决定的value，如果根据P2c没有已经接受的value，那么它可以自由决定value
- 在不违背自己向其他proposer的承诺的前提下，acceptor收到acceptor请求后即回复这个请求。
（阶段1和阶段2所请求的acceptors可以不同）
