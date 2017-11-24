# ZooKeeper入门笔记

标签（空格分隔）： ZooKeeper

*@nukil V1.0*

[TOC]

---
## ZooKeeper概述
> ZooKeeper is a centralized service for maintaining configuration information, naming, providing distributed synchronization, and providing group services.
> ZooKeeper是用于维护配置信息，命名，提供分布式同步和组的集中式服务。

### 特点
- 开源
- 高性能
- 实时性
- 单一视图：就和`HDFS`一样，屏蔽了背后的若干集群信息。无论客户端连接到哪台机器，展现给所有客户端的视图都是一致的。
- 可靠性
- 有许多好用的第三方客户端工具,比如ZkClient，curator等。
- 有着广泛的应用(`Hadoop`，`Solr`，`HBase`，`Storm`等)  

### 几个角色
- Leader
  是 `ZooKeeper` 集群中的核心，一个集群中的老大，参与选举并且选举投票超过半数的节点成为`Leader`。
- Follower
  `Follower`是`ZooKeeper`集群中Leader的下属节点，`Leader` 带领的一帮小弟，参与选举但没有成为`Leader`的节点。
- Observer
  `Observer` 是观察者，未参与选举的节点。

### Session
`ZooKeeper`中的`session`一般指的是客户端和`ZooKeeper` 集群之间的一个TCP连接。一般在客户端启动的时候就会和`ZooKeeper`集群之间建立一个长连接。

### 数据节点
整个 `ZooKeeper` 的数据模型就是一棵树。和Linux文件系统类似。有一个根节点，其下有若干子节点。这些节点可以保存数据。
另外，`ZooKeeper`集群中的一台机器，也可以称之为一个节点。上面所说的节点是指在同一台机器上而言的，请注意区分。

### Watcher

`Watcher`指的是在某些`ZooKeeper`节点上注册的**监听器**，用以监听数据的变化。

### ACL
此处的 `ACL(Access Control List)` 和 `Linux` 文件系统的中的那个访问控制列表有点类似。传统的文件系统中，`ACL`分为两个维度，一个是**属组**，一个是**权限**，子目录/文件默认继承父目录的 `ACL`。而在`ZooKeeper` 中，node的 `ACL` 是没有继承关系的，是独立控制的。`ZooKeeper`的`ACL`，可以从三个维度来理解：一是`scheme`，二是`user`，三是`permission`，通常表示为`scheme:id:permissions`。 

- scheme
  scheme对应于采用哪种方案来进行权限管理，`ZooKeeper`实现了一个pluggable的ACL方案，可以通过扩展scheme，来扩展ACL的机制。`ZooKeeper`缺省支持下面几种scheme：
1. **world**：它下面只有一个id，叫anyone，`world:anyone`代表任何人，`ZooKeeper`中对所有人有权限的结点就是属于world：anyone的
2. **auth**：它不需要id，只要是通过authentication的user都有权限（`ZooKeeper` 支持通过`kerberos`来进行authencation， 也支持`username/password`形式的authentication)
3. **digest**： 它对应的id为 `username:BASE64(SHA1(password))`，它需要先通过`username:password`形式的authentication
4. **ip**: 它对应的id为客户机的IP地址，设置的时候可以设置一个IP段，比如`ip:192.168.1.0/16`，表示匹配前16个bit的IP段
5. **super**: 在这种scheme情况下，对应的id拥有超级权限，可以做任何事情(cdrwa)

- id
  id与scheme是紧密相关的，每种scheme对应几种id均已介绍。

- permission
  `ZooKeeper`目前支持下面一些权限：
1. **CREATE(c)**：创建权限，可以在在当前node下创建child node
2. **DELETE(d)**：删除权限，可以删除当前的node
3. **READ(r)**：读权限，可以获取当前node的数据，可以list当前node所有的child nodes
4. **WRITE(w)**：写权限，可以向当前node写数据
5. **ADMIN(a)**：管理权限，可以设置当前node的permission

## ZooKeeper的用途
- 配置管理

> 配置的管理在分布式应用环境中很常见，例如同一个应用系统需要多台PC Server 运行，但是它们运行的应用系统的某些配置项是相同的，如果要修改这些相同的配置项，那么就必须同时修改每台运行这个应用系统的PC Server，这样非常麻烦而且容易出错。 

> 像这样的配置信息完全可以交给 `Zookeeper` 来管理，将配置信息保存在 `Zookeeper` 的某个目录节点中，然后将所有需要修改的应用机器监控配置信息的状态，一旦配置信息发生变化，每台应用机器就会收到 `Zookeeper` 的通知，然后从 `Zookeeper` 获取新的配置信息应用到系统中。

- 名字服务

> 分布式应用中，通常需要有一套完整的命名规则，既能够产生唯一的名称又便于人识别和记住，通常情况下用树形的名称结构是一个理想的选择，树形的名称结构是一个有层次的目录结构，既对人友好又不会重复。说到这里你可能想到了JNDI，没错 `Zookeeper` 的 Name  Service 与 JNDI 能够完成的功能是差不多的，它们都是将有层次的目录结构关联到一定资源上，但是 `Zookeeper` 的 Name Service 更加是广泛意义上的关联，也许你并不需要将名称关联到特定资源上，你可能只需要一个不会重复名称，就像数据库中产生一个唯一的数字主键一样。

> Name Service 已经是 `Zookeeper` 内置的功能，你只要调用 `Zookeeper` 的 API 就能实现。如调用 create 接口就可以很容易创建一个目录节点。

- 集群管理

> `Zookeeper` 能够很容易的实现集群管理的功能，如有多台 `Server` 组成一个服务集群，那么必须要一个"总管"知道当前集群中每台机器的服务状态，一旦有机器不能提供服务，集群中其它集群必须知道，从而做出调整重新分配服务策略。同样当增加集群的服务能力时，就会增加一台或多台 `Server`，同样也必须让"总管"知道。

> `Zookeeper` 不仅能够帮你维护当前的集群中机器的服务状态，而且能够帮你选出一个 "总管"，让这个"总管"来管理集群，这就是 `Zookeeper` 的另一个功能 `Leader Election`。

> 它们的实现方式都是在 `Zookeeper` 上创建一个 `EPHEMERAL` 类型的目录节点，然后每个 Server 在它们创建目录节点的父目录节点上调用 `getChildren(String path, boolean watch)` 方法并设置 watch 为 `true`，由于是 `EPHEMERAL` 目录节点，当创建它的 Server 死去，这个目录节点也随之被删除，所以 Children 将会变化，这时 `getChildren` 上的 `Watcher` 将会被调用，所以其它 Server 就知道已经有某台 Server 死去了。新增 Server 也是同样的原理。

`Zookeeper` 在实现这些服务时，首先基于它设计的一种新的数据结构 —— `Znode`，然后在该数据结构的基础上定义了一些原语，也就是一些关于该数据结构的一些操作。有了这些数据结构和原语还不够，因为 `Zookeeper` 是工作在一个分布式的环境下，我们的服务是通过消息以网络的形式发送给我们的分布式应用程序，所以还需要一个通知机制 —— `Watcher`机制。

`Zookeeper` 所提供的服务主要是通过：**数据结构** + **原语** + **Watcher** 机制，三个部分来实现的。


## ZooKeeper实现机制
### 数据结构

- Znode结构

<center>![](https://raw.githubusercontent.com/Nukil/Bimg/master/images/netposa/kafka/zookeeper.png)</center>

`Znode`在结构上和标准文件系统的非常相似，都是采用这种树形层次结构，`ZooKeeper`树中的每个节点被称为——`Znode`。和文件系统的目录树一样，`Znode` 树中的每个节点可以拥有子节点。但也有不同之处：

1. 引用方式
  `Znode` 通过路径引用，如同`Unix`中的文件路径。路径必须是绝对的，因此必须由斜杠字符来开头。除此以外，他们必须是唯一的，也就是说每一个路径只有一个表示，因此这些路径不能改变。在 `Znode` 中，路径由`Unicode`字符串组成，并且有一些限制。字符串"/zookeeper"用以保存管理信息，比如关键配额信息。
2. `Znode`结构
  `Znode` 兼具文件和目录两种特点。既像文件一样维护着数据、元信息、`ACL`、时间戳等数据结构，又像目录一样可以作为路径标识的一部分。上图中的每个节点称为一个 `Znode` 节点。 每个 `Znode` 节点由3部分组成：
    ① stat：状态信息，描述该 `Znode` 节点的版本，权限等信息
    ② data：与该 `Znode` 节点关联的数据
    ③ children：该 `Znode` 节点下的子节点
  `Znode` 虽然可以关联一些数据，但并没有被设计为常规的数据库或者大数据存储，相反的是，它用来管理调度数据，比如分布式应用中的配置文件信息、状态信息、汇集位置等等。这些数据的共同特性就是它们都是很小的数据，通常以KB为大小单位。`ZooKeeper` 的服务器和客户端都被设计为严格检查并限制每个 `Znode` 的数据大小至多**1M**，但常规使用中应该远小于此值。
3. 数据访问
  `Znode` 中的每个节点存储的数据要被原子性的操作。也就是说读操作将获取与节点相关的所有数据，写操作也将替换掉节点的所有数据。另外，每一个节点都拥有自己的 `ACL` (访问控制列表)，这个列表规定了用户的权限，即限定了特定用户对目标节点可以执行的操作。
4. 节点类型
  `Znode` 中的节点有两种，分别为**临时节点**和**永久节点**。节点的类型在创建时即被确定，并且不能改变。
  ① 临时节点：该节点的生命周期依赖于创建它们的会话。一旦会话 `Session` 结束，临时节点将被自动删除，当然可以也可以手动删除。虽然每个临时的 `Znode` 都会绑定到一个客户端会话，但他们对所有的客户端还是可见的。另外，`Znode` 临时节点不允许拥有子节点。
  ② 永久节点：该节点的生命周期不依赖于会话，并且只有在客户端显示执行删除操作的时候，它们才能被删除。
5. watch
  客户端可以在节点上设置 `Watcher`，称之为**监视器**。当节点状态发生改变时( `Znode` 的增、删、改)将会触发 `Watcher` 所对应的操作。当 `Watcher` 被触发时，`ZooKeeper` 将会向客户端发送且仅发送一条通知，因为 `ZooKeeper` 只能被触发一次，这样可以减少网络流量。

&nbsp;

- Znode状态信息

一个节点自身拥有表示其状态的许多重要属性，如下表：
|       属性       |                  描述                  |
| :------------: | :----------------------------------: |
|     cZxid      |             该节点被创建时的Zxid             |
|     mZxid      |           该节点最近一次被修改时的Zxid           |
|     ctime      |              该节点被创建时的时间              |
|     mtime      |              该节点被修改时的时间              |
|     pZxid      |           子节点最近一次修改时的Zxid            |
|    cversion    |   节点拥有的子节点被修改的版本号，如果没有子节点则等于cZxid    |
|   aclversion   |              节点的ACL版本号               |
| ephemeralOwner | 如果该节点为临时节点，那么它的值为该节点拥有者的会话ID，否则它的值为0 |
|   dataLength   |                节点数据长度                |
|  numChildren   |               节点的子节点数目               |


1. Zxid
   <p style="text-align:justify">使`Znode`节点状态改变的每一个操作都将使节点接收到一个Zxid格式的事物ID，并且该ID全局有序。也就是说，每个对节点的改变都将产生一个唯一的Zxid。如果Zxid1的值小于Zxid2的值，那么Zxid1所对应的事件发生在Zxid2所对应的事件之前。实际上，`ZooKeeper`的每个节点维护者三个Zxid值，为别为：cZxid、mZxid、pZxid。实现中Zxid是一个64位的数字，它高32位是epoch用来标识leader关系是否改变，每次一个leader被选出来，它都会有一个新的epoch。低32位是个递增计数器。</p>

2. 版本号
  对节点的每一个操作都将致使这个节点的版本号增加。每个节点维护着三个版本号，他们分别为：
  ① dataVersion：节点数据版本号
  ② cversion：子节点版本号
  ③ aclVersion：ACL版本号

```shell
#在zk bin目录下
./zkCli.sh

[zk: localhost:2181(CONNECTED) 8] get /brokers/ids/1
{"jmx_port":-1,"timestamp":"1509358386183","endpoints":["PLAINTEXT://kafka1:9092"],"host":"kafka1","version":3,"port":9092}
cZxid = 0x200000046
ctime = Mon Oct 30 03:13:06 PDT 2017
mZxid = 0x200000046
mtime = Mon Oct 30 03:13:06 PDT 2017
pZxid = 0x200000046
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x35f6b3770030006
dataLength = 123
numChildren = 0
```

### 原语
我们知道，每个`Znode`由`stat`，`data`和`children`组成，我们可以通过`ZooKeeper`提供的一些基础命令操作`Znode`，达到获取/修改`Znode`信息的目的。常用的命令如下：
|               命令               |                    描述                    |
| :----------------------------: | :--------------------------------------: |
|       stat path [watch]        |          列出指定节点的状态信息，或者说是元数据信息           |
|    set path data [version]     | 修改操作。 path：节点路径。data：新数据。version：可选，版本号，要么不写，要么和上一次查询出的版本号一致。该操作会影响节点的mZxid、dataVersion和mtime属性 |
|        ls path [watch]         |         列出指定节点的子节点，类似于linux的ls命令         |
|     delquota [-n\|-b] path     |                 删除quota                  |
|        ls2 path [watch]        |        是ls的升级版，列出子节点的同时列出节点的状态信息         |
|        setAcl path acl         |               更改指定节点的ACL信息               |
|    setquota -n\|-b val path    | 对节点增加限制(配额)，n：表示子节点的最大个数。b：表示数据值的最大长度。val：子节点最大个数或数据值的最大长度。path：节点路径 |
|            history             |                  列出命令历史                  |
|           redo cmdno           |  该命令可以重新执行指定命令编号的历史命令，命令编号可以通过history查看  |
|        printwatches on         |                   off                    |
|     delete path [version]      |            删除操作，只能删除不含子节点的节点             |
|           sync path            | 把所有在sync之前的更新操作都进行同步，达到每个请求都在半数以上的ZooKeeper Server上生效 |
|         listquota path         |               列出指定节点的quota               |
|            rmr path            |          删除操作，可以递归删除节点，从子节点开始删除          |
|        get path [watch]        |               可以列出指定节点的数据                |
| create [-s] [-e] path data acl | 创建节点。s：可选，表示该节点为顺序节点。e：可选，表示该节点为临时节点。path：节点路径data：节点数据。acl：访问控制列表。 |
|      addauth scheme auth       |                 注册会话授权信息                 |
|              quit              |               退出zk client                |
|          getAcl path           |               获取指定节点的ACL信息               |
|             close              |                 关闭与zk的连接                 |
|       connect host:port        |                 连接到指定的zk                 |

操作示例：
```shell
# 在zk bin目录下
./zkCli.sh

# 列出所有可以使用的命令
[zk: localhost:2181(CONNECTED) 0] help
ZooKeeper -server host:port cmd args
	stat path [watch]
	set path data [version]
	ls path [watch]
	delquota [-n|-b] path
	ls2 path [watch]
	setAcl path acl
	setquota -n|-b val path
	history 
	redo cmdno
	printwatches on|off
	delete path [version]
	sync path
	listquota path
	rmr path
	get path [watch]
	create [-s] [-e] path data acl
	addauth scheme auth
	quit 
	getAcl path
	close 
	connect host:port
```

- 查询操作
```shell
# ls path [watch]
[zk: localhost:2181(CONNECTED) 1] ls /
[controller_epoch, controller, brokers, zookeeper, admin, isr_change_notification, consumers, config]

# stat path [watch]
[zk: localhost:2181(CONNECTED) 2] stat /
cZxid = 0x0
ctime = Wed Dec 31 16:00:00 PST 1969
mZxid = 0x0
mtime = Wed Dec 31 16:00:00 PST 1969
pZxid = 0x200000081
cversion = 40
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 8

# get path [watch]
[zk: localhost:2181(CONNECTED) 6] get /brokers/ids/1
{"jmx_port":-1,"timestamp":"1509358386183","endpoints":["PLAINTEXT://kafka1:9092"],"host":"kafka1","version":3,"port":9092}
cZxid = 0x200000046
ctime = Mon Oct 30 03:13:06 PDT 2017
mZxid = 0x200000046
mtime = Mon Oct 30 03:13:06 PDT 2017
pZxid = 0x200000046
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x35f6b3770030006
dataLength = 123
numChildren = 0

# ls2 path [watch]
[zk: localhost:2181(CONNECTED) 7] ls2 /
[controller_epoch, controller, brokers, zookeeper, admin, isr_change_notification, consumers, config]
cZxid = 0x0
ctime = Wed Dec 31 16:00:00 PST 1969
mZxid = 0x0
mtime = Wed Dec 31 16:00:00 PST 1969
pZxid = 0x200000081
cversion = 40
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 0
numChildren = 8
```

- 创建操作
```shell
# create [-s] [-e] path data acl
[zk: localhost:2181(CONNECTED) 10] create /likun hello,world world:anyone:crw
Created /likun
[zk: localhost:2181(CONNECTED) 11] get /likun
hello,world
cZxid = 0x20000008e
ctime = Wed Nov 01 01:09:28 PDT 2017
mZxid = 0x20000008e
mtime = Wed Nov 01 01:09:28 PDT 2017
pZxid = 0x20000008e
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 11
numChildren = 0
[zk: localhost:2181(CONNECTED) 13] getAcl /likun
'world,'anyone
: crw

[zk: localhost:2181(CONNECTED) 16] create -e /likun hello,world world:anyone:crw
Created /likun
[zk: localhost:2181(CONNECTED) 17] get /likun
hello,world
cZxid = 0x200000090
ctime = Wed Nov 01 01:12:15 PDT 2017
mZxid = 0x200000090
mtime = Wed Nov 01 01:12:15 PDT 2017
pZxid = 0x200000090
cversion = 0
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x35f6b3770030010
dataLength = 11
numChildren = 0

```

- 修改操作
```shell
# set path data [version]
[zk: localhost:2181(CONNECTED) 18] set /likun hello,likun
cZxid = 0x200000090
ctime = Wed Nov 01 01:12:15 PDT 2017
mZxid = 0x200000091
mtime = Wed Nov 01 01:18:22 PDT 2017
pZxid = 0x200000090
cversion = 0
dataVersion = 1
aclVersion = 0
ephemeralOwner = 0x35f6b3770030010
dataLength = 11
numChildren = 0
```

- 删除操作
```shell
# delete path [version]
[zk: localhost:2181(CONNECTED) 21] delete /likun                                   
[zk: localhost:2181(CONNECTED) 22] ls /
[controller_epoch, controller, brokers, zookeeper, admin, isr_change_notification, consumers, config]
```
- quota
```shell
# 创建节点
[zk: localhost:2181(CONNECTED) 23] create /likun hello,likun world:anyone:crwa
Created /likun

# 指定其子节点最大数为2
[zk: localhost:2181(CONNECTED) 26] setquota -n 2 /likun
Comment: the parts are option -n val 2 path /likun

# 创建第2个子节点
[zk: localhost:2181(CONNECTED) 31create /likun/likun1 1
Created /likun/likun1
[zk: localhost:2181(CONNECTED) 32create /likun/likun2 2
Created /likun/likun2
[zk: localhost:2181(CONNECTED) 0] ls2 /likun
[likun2, likun1]
cZxid = 0x200000094
ctime = Wed Nov 01 01:25:00 PDT 2017
mZxid = 0x200000094
mtime = Wed Nov 01 01:25:00 PDT 2017
pZxid = 0x20000009a
cversion = 2
dataVersion = 0
aclVersion = 0
ephemeralOwner = 0x0
dataLength = 11
numChildren = 2

# 创建第3个子节点
[zk: localhost:2181(CONNECTED) 1] create /likun/likun3 3
Created /likun/likun3

2017-11-01 01:29:23,372 [myid:3] - WARN  [CommitProcessor:3:DataTree@301] - Quota exceeded: /likun count=3 limit=2

# listquota 列出指定节点的quota
[zk: localhost:2181(CONNECTED) 8] listquota /likun
absolute path is /zookeeper/quota/likun/zookeeper_limits
# 子节点个数为2,数据长度-1表示没限制
Output quota for /likun count=2,bytes=-1
# 当前信息，节点数为4(超额了),数据长度为18(包含子节点的数据长度)
Output stat for /likun count=4,bytes=18

#delquota 删除quota
[zk: localhost:2181(CONNECTED) 10] delquota -n /likun   
[zk: localhost:2181(CONNECTED) 11] listquota /likun
absolute path is /zookeeper/quota/likun/zookeeper_limits
# count=-1表示不做限制
Output quota for /likun count=-1,bytes=-1
Output stat for /likun count=4,bytes=18
```
- ACL
```shell
# getAcl 获取指定节点的ACL信息
[zk: localhost:2181(CONNECTED) 12] getAcl /likun
'world,'anyone
: crwa
# setAcl 更改ACL 用户名likun，密码123
[zk: localhost:2181(CONNECTED) 14] setAcl /likun digest:likun:ZHnF3uAot4y8j/cnKagCvRp/u2Y=:crw
cZxid = 0x200000094
ctime = Wed Nov 01 01:25:00 PDT 2017
mZxid = 0x200000094
mtime = Wed Nov 01 01:25:00 PDT 2017
pZxid = 0x20000009d
cversion = 3
dataVersion = 0
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 11
numChildren = 3
[zk: localhost:2181(CONNECTED) 15] getAcl /likun
'digest,'likun:ZHnF3uAot4y8j/cnKagCvRp/u2Y=
: crw
[zk: localhost:2181(CONNECTED) 16] set /likun hello
Authentication is not valid : /likun

# addauth 注册会话授权信息
[zk: localhost:2181(CONNECTED) 23] addauth digest likun:123    
[zk: localhost:2181(CONNECTED) 24] set /likun hello        
cZxid = 0x200000094
ctime = Wed Nov 01 01:25:00 PDT 2017
mZxid = 0x2000000aa
mtime = Wed Nov 01 02:15:57 PDT 2017
pZxid = 0x20000009d
cversion = 3
dataVersion = 1
aclVersion = 1
ephemeralOwner = 0x0
dataLength = 5
numChildren = 3
```

zk提供了`org.apache.zookeeper.server.auth.DigestAuthenticationProvider`的 `java API`用于快速生成密文，其实现如下：
```java
DigestAuthenticationProvider.generateDigest("likun:123");
public static String generateDigest(String idPassword) throws NoSuchAlgorithmException {
    String[] parts = idPassword.split(":", 2);
    byte[] digest = MessageDigest.getInstance("SHA1").digest(idPassword.getBytes());
    return parts[0] + ":" + base64Encode(digest);
}
```

### Watcher机制
`ZooKeeper`可以为所有的读操作设置`Watcher`，这些读操作包括：`exists()`、`getChildren()`及`getData()`。`Watcher`事件是一次性的触发器，当Watch的对象状态发生改变时，将会触发此对象上Watch所对应的事件，`Watcher`事件将被异步地发送给客户端。

当客户端向`ZooKeeper`服务器注册`Watcher`的同时，会将`Watcher`对象存储在客户端的`WatchManager`中。当`ZooKeeper`服务器端触发`Watcher`事件后，会向客户端发送通知，客户端线程从`WatchManager`中取出对应的`Watcher`对象来执行回调逻辑。

- Watcher的特点：

1. Watches通知是一次性的，必须重复注册。
2. 发生`CONNECTIONLOSS`之后，只要在`session_timeout`之内再次连接上（即不发生`SESSIONEXPIRED`），那么这个连接注册的`Watches`依然在。
3. 节点数据的**版本变化**会触发`NodeDataChanged`。只要成功执行了`setData()`方法，无论内容是否和之前一致，都会触发`NodeDataChanged`。
4. 对某个节点注册了`Watcher`，但是节点被删除了，那么注册在这个节点上的`Watches`都会被移除。
5. 同一个zk客户端对某一个节点注册相同的`Watcher`，只会收到一次通知。

- 状态码
  服务端触发`Watcher`并通知客户端时，会同时将`Watcher`的状态和发生的事件类型告诉客户端。

<center><label>watchEvent的state类型</label></center>
| state |    描述     |
| :---: | :-------: |
|  -1   |   未知状态    |
| -112  |  会话超时状态   |
|   0   |   失去连接    |
|   1   | 未同步建立连接状态 |
|   3   |  同步已建立状态  |
|   4   |   认证失败    |
|   5   |   只读状态    |
|   6   | Sasl认证状态  |
<center><label>watchEvent的type类型</label></center>
| type |              描述               |
| :--: | :---------------------------: |
|  1   |      创建节点事件 NodeCreated       |
|  2   |      删除节点事件 NodeDeleted       |
|  3   |    更改节点事件 NodeDataChanged     |
|  4   | 子节点列表变化事件 NodeChildrenChanged |
|  -1  |          会话session事件          |
|  -2  |            监控被移除事件            |


- Watcher触发规则

父节点的变更以及孙节点的变更都不会触发`Watcher`，而对`Watcher`本身节点以及子节点的变更会触发`Watcher`。
```java
package zookeeper;
import org.apache.zookeeper.*;
import org.apache.zookeeper.data.Stat;

class MyWatcher implements Watcher {
    @Override
    public void process(WatchedEvent watchedEvent) {
        System.out.println(watchedEvent.getPath() + " " + watchedEvent.getType());
    }
}

public class ZKClient {
    public static void main(String[] args) {
        try {
            String  address = "kafka1:2181,kafka2:2181,kafka3:2181";
            int timeout = 3000000;
            Stat st = new Stat();
            ZooKeeper zkClient = new ZooKeeper(address, timeout, new Watcher() {
                @Override
                public void process(WatchedEvent watchedEvent) {
                    // nothing
                }
            });

            zkClient.create("/father", "hello world!".getBytes(), ZooDefs.Ids.OPEN_ACL_UNSAFE, CreateMode.PERSISTENT);
            zkClient.getData("/father", new MyWatcher(), st);
            zkClient.delete( "/father", -1);
            zkClient.close();
        } catch (Exception e) {
            System.out.println(e.getMessage() + e);
        }
    }
}

# 输出
/father NodeDeleted
[2017-11-01 21:10:23,301] INFO Session: 0x35f6b3770030022 closed (org.apache.zookeeper.ZooKeeper)
[2017-11-01 21:10:23,302] INFO EventThread shut down (org.apache.zookeeper.ClientCnxn)

```



## ZooKeeper四字命令
`ZooKeeper`提供了一些简单的四字命令用于获取`ZooKeeper`的状态或者配置信息，使用前请安装`telnet`或者`nc`，如下：
```shell
[root@kafka3 ~]# echo conf | nc 192.168.62.152 2181
clientPort=2181
dataDir=/home/data/version-2
dataLogDir=/home/data/version-2
tickTime=2000
maxClientCnxns=60
minSessionTimeout=4000
maxSessionTimeout=40000
serverId=2
initLimit=10
syncLimit=5
electionAlg=3
electionPort=13888
quorumPort=12888
peerType=0
```
|                   功能描述                   |  命令  |
| :--------------------------------------: | :--: |
|              输出相关服务配置的详细信息               | conf |
| 列出所有连接到服务器的客户端的连接/会话的详细信息，包括"接受/发送"的包数量，会话 id ，操作延迟，最后的操作执行等等信息 | cons |
|              列出未经处理的会话和临时节点              | dump |
|         输出关于服务环境的详细信息(区别于conf命令)         | envi |
|                列出未经处理的请求                 | reqs |
|  测试服务是否处于正常状态，如果是，那么服务返回"imok"，否则不做任何相应  | ruok |
|             输出关于性能和连接的客户端的列表             | stat |
|             列出服务器 watch的详细信息             | wchs |
| 通过session列出服务器watch的详细信息，它的输出是一个与watch相关的会话的列表 | wchc |
| 通过路径列出服务器watch的详细信息，它输出一个与session相关的路径。  | wchp |
