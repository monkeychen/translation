# ZooKeeper administrator's guide中文版

> 本文档主要介绍ZooKeeper部署及日常管理

## 安装部署
本章节包含与ZooKeeper部署有关的内容，具体来说包含下面三部分内容：

* [系统软硬件需求](#1.1)
* [集群部署（安装）](#1.2)
* [单机开发环境部署（安装）](#1.3)

前两部分主要介绍如何在数据中心等生产环境上安装部署ZooKeeper，第三部分则介绍如何在非生产环境上（如为了评估、测试、开发等目的）安装部署ZooKeeper。

### 系统软硬件需求<span id="1.1"></span>
#### 支持的OS平台
ZooKeeper框架由多个组件组成，有的组件支持全部平台，而还有一些组件只支持部分平台，详细支持情况如下：

* **Client：**它是一个`Java`客户端连接库，上层应用系统通过它连接至ZooKeeper集群。
* **Server：**它是运行在ZooKeeper集群节点上的一个`Java`后台服务程序。
* **Native Client：**它是一个用`C`语言实现的客户端连接库，其与`Java`客户端库一样，上层应用（非`Java`实现）通过它连接至ZooKeeper集群。
* **Contrib：**它是指多个可选的扩展组件。

|操作系统|Client|Server|Native Client|Contrib|
|:-|:-:|:-:|:-:|:-:|
|GNU/Linux|D + P|D + P|D + P|D + P|
|Solaris|D + P|D + P|/|/|
|FreeBSD|D + P|D + P|/|/|
|Windows|D + P|D + P|/|/|
|Mac OS X|D|D|/|/|

<!--More-->

> **D：支持开发环境， P：支持生成环境， /：不支持任何环境**

上表中未显式注明支持的组件在相应平台上可能不能正常运行。虽然ZooKeeper社区会尽量修复在未支持平台上发现的BUG，但并无法保证会修复全部BUG。

#### 软件要求
ZooKeeper需要运行在JDK6或以上版本中。若ZooKeeper以集群模式部署，则推荐的节点数至少为3，同时建议部署在独立的服务器上。在Yahoo!，ZooKeeper通常部署在运行RHEL系统的服务器上（服务器配置：双核CPU、2G内存、80G容量IDE硬盘）。

### 集群部署（安装）<span id="1.2"></span>
为了保证ZooKeeper服务的可靠性，您应该以集群模式部署ZooKeeper服务。只要半数以上的集群节点在线，服务将是可用的。因为ZooKeeper需要半数以上节点同意才能选举出Leader，所以建议ZooKeeper集群的节点数为奇数个。举个例子，对于有四个节点的集群只能应付一个节点宕机的异常，如果有两个节点宕机，则剩下两个节点未达到法定的半数以上选票，ZooKeeper服务将变为不可用。而如果集群有五个节点，则集群就可以应付二个节点宕机的异常。
> **提示：**
> 正如[《ZooKeeper快速入门》][1]文档中所提到的，至少需要三个节点的ZooKeeper集群才具备容灾特性，因此我们强烈建议集群节点数为奇数。
> 通常情况下，生产环境下，集群节点数只需要三个。但如果为了在服务维护场景下也保证最大的可靠性，您也许会部署五个节点的集群。原因很简单，如果集群节点为三个，当你对其中一个节点进行维护操作，将很有可能因维护操作导致集群异常。而如果集群节点为5个，那你可以直接将维护节点下线，此时集群仍然可正常提供服务（就算四个节点中的任意一个突然宕机）。
> 您的冗余措施应该包括生产环境的各个方面。如果你部署三个节点的ZooKeeper集群，但你却将这三个节点都连接至同一个网络交换机，那么当交换机宕掉时，你的集群服务也一样是不可用的。

关于如何配置服务器使其成为集群中的一个成员节点的详细步骤如下（每个节点的操作类似）：

* 1.安装JDK，你可以使用本地包管理系统安装，也可以从JDK官网[下载][2]
* 2.设置Java堆相关参数，这对于避免内存swap来说非常重要。因为频繁的swap将严重降低ZooKeeper集群的性能。您可以通过使用压力负载测试来决定一个合适的值，确保该值刚好低于触发swap的阈值。保守的做法是：当节点拥有4G的内存，则设置-xmx=3G。
* 3.安装ZooKeeper，您可以从[官网][3]下载：http://zookeeper.apache.org/releases.html
* 4.创建一个配置文件，这个文件可以随便命名，您可以先使用如下设置参数：

```
tickTime=2000
dataDir=/var/lib/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=zoo1:2888:3888
server.2=zoo2:2888:3888
server.3=zoo3:2888:3888
```

> 您可以在[配置参数](#2.10)章节中找到上面这些参数及其他参数的解释说明。这里简要说一下：集群中的每个节点都需要知道集群中其它节点成员的连接信息。通过上述配置文件中格式为`server.id=host:port:port`的若干行配置信息您就可以实现这个目标。`host`与`port`意思很简单明了，不多作说明。而`server.${id}`代表节点ID，你需要为每个节点创建一个名为`myid`的文件，该文件存放于参数`dataDir`所指向的目录下。

* 5.`myid`文件的内容只有一行，其值为配置参数`server.${id}`中`${id}`的实际值。即服务器1对应的`myid`文件的内容为1，这个值在集群中必须保证其唯一性，同时必须处于[1, 255]之间。
* 6.如果你已经创建好配置文件（如zoo.cfg），你就可以启动ZooKeeper服务：

```
$ java -cp zookeeper.jar:lib/slf4j-api-1.6.1.jar:lib/
slf4j-log4j12-1.6.1.jar:lib/log4j-1.2.15.jar:conf \
org.apache.zookeeper.server.quorum.QuorumPeerMain zoo.cfg
```

> QuorumPeerMain启动一个ZooKeeper服务，JMX管理bean也同时被注册，通过这些JMX管理bean，你就可以在JMX管理控制台上对ZooKeeper服务进行监控管理。ZooKeeper的[JMX文档][4]有更详细的介绍信息。同时，你也可以在`$ZOOKEEPER_HOME/bin`目录下找到ZooKeeper的启动脚本`zkServer.sh`。

* 7.接下来你就可以连接至ZooKeeper节点来测试部署是否成功。
在`Java`环境下，你可以运行下面的命令连接至已启动的ZooKeeper服务节点并执行一些简单的操作

```
$ bin/zkCli.sh -server 127.0.0.1:2181
```

### 单机开发环境部署介绍<span id="1.3"></span>
如果你想部署ZooKeeper以便于开发测试，那你可以使用单机部署模式。然后安装Java或C客户端连接库，同时在配置文件中将服务器信息与开发机绑定。
具体安装部署步骤也集群部署模式类似，唯一不同的是zoo.cfg配置文件更简单一些。你可以从[《ZooKeeper快速入门》][1]文档中的相关章节获取详细的操作步骤。
关于安装客户端连接库的相关信息，你可以从[ZooKeeper Programmer's Guide][5]文件的Bindings章节中获取。

## 维护管理
这部分包含ZooKeeper运行与维护相关的信息，其包含如下几个主题：
 
* [ZooKeeper集群部署规划](#2.1)
* [Provisioning](#2.2)
* [Things to Consider: ZooKeeper Strengths and Limitations](#2.3)
* [Administering](#2.4)
* [Maintenance](#2.5)
* [Supervision](#2.6)
* [Monitoring](#2.7)
* [Logging](#2.8)
* [Troubleshooting](#2.9)
* [Configuration Parameters](#2.10)
* [ZooKeeper Commands: The Four Letter Words](#2.11)
* [Data File Management](#2.12)
* [Things to Avoid](#2.13)
* [Best Practices](#2.14)

### ZooKeeper集群部署规划<span id="2.1"></span>
ZooKeeper可靠性依赖于两个基本的假设：

* 一个集群中只会有少数服务器会出错，这里的出错是指宕机或网络异常而使得服务与集群中的多数服务器失去联系。
* 已部署的机器可以正常运行，所谓正常运行是指所有代码可以正确的执行，有适当且一致的工作时钟、存储、网络。

为了最大可能的保证上述两个前提假设能够成立，这里有几个点需要考虑。一些是关于跨服务器（节点之间）需求的，而另一些是关于集群中每个节点服务器的考虑


[1]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperStarted.html
[2]: http://java.sun.com/javase/downloads/index.jsp
[3]: http://zookeeper.apache.org/releases.html
[4]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperJMX.html
[5]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperProgrammers.html
[6]: https://zookeeper.apache.org/doc/r3.4.10/api/index.html
[7]: http://logging.apache.org/log4j/1.2/manual.html#defaultInit

