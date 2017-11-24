# Kafka副本同步机制

标签（空格分隔）： Kafka

*@nukil V1.0*

[TOC]

## 前言

`Kafka`分区下有可能有很多个副本`replica`用于实现冗余，从而进一步实现高可用。副本根据角色的不同可分为3类：

- leader副本：响应clients端读写请求的副本
- follower副本：被动地备份leader副本中的数据，不能响应clients端读写请求
- Isr副本：包含了leader副本和所有与leader副本保持同步的follower副本

**每个**`Kafka`副本对象都有两个重要的属性：`LEO(Log End Offset)`和`HW(High Watermark)`。是**所有的**副本，而不只是leader副本。

- LEO：即日志末端位移`Log End Offset`，记录了该副本底层日志log中下一条消息的位移值。是下一条消息，也就是说，如果LEO=10，那么表示该副本保存了10条消息，位移值范围是[0, 9]。
- HW：即水位值`WaterMark`。对于同一个副本对象而言，其HW值不会大于LEO值。小于等于HW值的所有消息都被认为是"已备份"的。

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171123162435.png)

上图中，HW值是4，表示位移是0~7的所有消息都已经处于"已备份状态"（committed），而LEO值是9，那么5~8的消息就是尚未完全备份（fully replicated）——为什么没有9？因为LEO指向的是下一条消息到来时的位移，故上图使用深色表示。我们知道`consumer`无法消费未提交消息。这句话如果用以上名词来解读的话，应该表述为：`consumer`无法消费分区下leader副本中位移值大于分区HW值的任何消息。

## HW与LEO何时更新

### follower副本何时更新LEO

上面说过，follower副本只是被动地向leader副本请求数据，具体表现为follower副本不停地向leader副本所在的`broker`发送FETCH请求，一旦获取消息后写入自己的日志中进行备份。

首先需要知道的是，`Kafka`有 **两套** follower副本LEO（敲黑板，划重点）：

- 一套LEO保存在follower副本所在`broker`的副本管理机中；
- 另一套LEO保存在leader副本所在`broker`的副本管理机中，也就是说，leader副本机器上保存了所有的follower副本的LEO。

为什么要保存两套？这是因为`Kafka`使用前者帮助follower副本更新其HW值；而利用后者帮助leader副本更新其HW使用。

- follower副本端的follower副本LEO何时更新？

  follower副本端的LEO值就是其底层日志的LEO值，也就是说每当新写入一条消息，其LEO值就会被更新（类似于LEO += 1）。当follower发送FETCH请求后，leader将数据返回给follower，此时follower开始向底层log写数据，从而自动地更新LEO值

- leader副本端的follower副本LEO何时更新？

  leader副本端的follower副本LEO的更新发生在leader在处理follower FETCH请求时。一旦leader接收到follower发送的FETCH请求，它首先会从自己的log中读取相应的数据，但是在给follower返回数据之前它先去更新follower的LEO（即上面所说的第二套LEO）

### follower副本何时更新HW

follower更新HW发生在其更新LEO之后，一旦follower向log写完数据，它会尝试更新它自己的HW值。具体算法就是比较当前LEO值与响应中leader的HW值，取两者的小者作为新的HW值。这告诉我们一个事实：如果follower的LEO值超过了leader的HW值，那么follower HW值是不会越过leader HW值的。

### leader副本何时更新LEO

和follower更新LEO道理相同，leader写log时就会自动地更新它自己的LEO值。

### leader副本何时更新HW值

leader的HW值就是分区HW值，因此何时更新这个值是最重要的，因为它直接影响了分区数据对于`consumer`的可见性 。以下4种情况下leader会 **尝试** 去更新分区HW，是尝试，因为有可能因为不满足条件而不做任何更新：

- 副本成为leader副本时：当某个副本成为了分区的leader副本，Kafka会尝试去更新分区HW。这是显而易见的道理，毕竟分区leader发生了变更，这个副本的状态是一定要检查的
- `broker`出现崩溃导致副本被踢出ISR时：若有`broker`崩溃则必须查看下是否会波及此分区，因此检查下分区HW值是否需要更新是有必要的
- `producer`向leader副本写入消息时：因为写入消息会更新leader的LEO，故有必要再查看下HW值是否也需要修改（敲黑板）
- leader处理follower FETCH请求时：当leader处理follower的FETCH请求时首先会从底层的log读取数据，之后会尝试更新分区HW值（敲黑板）

特别注意上面4个条件中的最后两个。他们说明当`Kafka broker`都正常工作时，分区HW值的更新时机有两个：    

- leader处理PRODUCE请求时和leader处理FETCH请求时。另外，leader是如何更新它的HW值的呢？前面说过了，leader broker上保存了一套follower副本的LEO以及它自己的LEO。当尝试确定分区HW时，它会选出所有满足条件的副本，比较它们的LEO（当然也包括leader自己的LEO），并选择最小的LEO值作为HW值。这里的满足条件主要是指副本要满足以下两个条件之一：
  1. 前期副本同步正常，处于ISR中
  2. 副本LEO落后于leader LEO的时长不大于`replica.lag.time.max.ms`参数值（默认是10s）

乍看上去好像这两个条件说得是一回事，毕竟ISR的定义就是第二个条件描述的那样。但某些情况下`Kafka`的确可能出现副本已经"追上"了leader的进度，但却不在ISR中——比如某个从failure中恢复的副本。如果`Kafka`只判断第一个条件的话，确定分区HW值时就不会考虑这些未在ISR中的副本，但这些副本已经具备了"立刻进入ISR"的资格，因此就可能出现分区HW值越过ISR中副本LEO的情况——这肯定是不允许的，因为分区HW实际上就是ISR中所有副本LEO的最小值。

## 示例

假设有1个`topic`，有一个`partition`，2个`replicas` ，

下图是初始状态：初始时leader和follower的HW和LEO都是0（严格来说源代码会初始化LEO为-1）。leader中的Follower LEO指的就是leader端保存的follower LEO，也被初始化成0。此时，`producer`没有发送任何消息给leader，而follower已经开始不断地给leader发送FETCH请求了，但因为没有数据因此什么都不会发生。特别的，follower发送过来的FETCH请求因为无数据而暂时会被寄存到leader端的`purgatory`中，待replica.fetch.wait.max.ms（默认500ms）超时后会强制完成。倘若在寄存期间`producer`端发送过来数据，那么会`Kafka`会自动唤醒该FETCH请求，让leader继续处理。

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171123174827.png)

follower发送FETCH请求在leader处理完PRODUCE请求之后，`producer`给该`topic`分区发送了一条消息。此时的状态如下图所示：

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171123174859.png)

如图所示，leader接收到PRODUCE请求主要做两件事情：

- 把消息写入写底层log（同时也就自动地更新了leader的LEO）

- 尝试更新leader HW值。假设此时follower尚未发送FETCH请求，那么leader端保存的remote LEO依然是0

  因此leader会比较它自己的LEO值和Follower LEO值，发现最小值是0，与当前HW值相同，故不会更新分区HW值

所以，PRODUCE请求处理完成后leader端的HW值依然是0，而LEO是1，Follower LEO是0。

假设此时follower发送了FETCH请求(或者说follower早已发送了FETCH请求，只不过在`broker`的请求队列中排队)，那么状态变更如下图所示：

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171123174917.png)

此时leader端的处理依次是：

- 读取底层log数据
- 更新Follower LEO = 0（为什么是0？ 因为此时follower还没有写入这条消息。leader如何确认follower还未写入呢？这是通过follower发来的FETCH请求中的fetch offset来确定的）
- 尝试更新分区HW——此时leader LEO = 1，Follower LEO = 0，故分区HW值= min(leader LEO, follower remote LEO) = 0
- 把数据和当前分区HW值（依然是0）发送给follower副本

而follower副本接收到Response后依次执行下列操作：

- 写入本地log（同时更新follower LEO）
- 更新follower HW——比较本地LEO和当前leader LEO取小者，故follower HW = 0

此时，第一轮FETCH RPC结束，我们会发现虽然leader和follower都已经在log中保存了这条消息，但分区HW值尚未被更新。实际上，它是在第二轮FETCH RPC中被更新的，如下图所示：

![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/TIM%E6%88%AA%E5%9B%BE20171123174930.png)

上图中，follower发来了第二轮FETCH请求，leader端接收到后仍然会依次执行下列操作：

- 读取底层log数据
- 更新Follower LEO = 1（这次为什么是1了？ 因为这轮FETCH RPC携带的fetch offset是1，那么为什么这轮携带的就是1了呢，因为上一轮结束后follower LEO被更新为1了）
- 尝试更新分区HW——此时leader LEO = 1，remote LEO = 1，故分区HW值= min(leader LEO, follower remote LEO) = 1。
- 把数据（实际上没有数据）和当前分区HW值（已更新为1）发送给follower副本

同样地，follower副本接收到FETCH response后依次执行下列操作：

- 写入本地log，当然没东西可写，故follower LEO也不会变化，依然是1
- 更新follower HW——比较本地LEO和当前leader LEO取小者。由于此时两者都是1，故更新follower HW = 1 