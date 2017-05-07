# ZooKeeper Administrator's Guide中文版

> 本文档主要介绍ZooKeeper部署及日常管理

## 1. 安装部署
本章节包含与ZooKeeper部署有关的内容，具体来说包含下面三部分内容：
* [系统软硬件需求](#1.1)
* [集群部署（安装）](#1.2)
* [单机开发环境部署（安装）](#1.3)

前两部分主要介绍如何在数据中心等生产环境上安装部署ZooKeeper，第三部分则介绍如何在非生产环境上（如为了评估、测试、开发等目的）安装部署ZooKeeper。

<h3 id="1.1">1.1. 系统软硬件需求</h3>
#### 1.1.1. 支持的OS平台
ZooKeeper框架由多个组件组成，有的组件支持全部平台，而还有一些组件只支持部分平台，详细支持情况如下：
- **Client：**它是一个`Java`客户端连接库，上层应用系统通过它连接至ZooKeeper集群。
- **Server：**它是运行在ZooKeeper集群节点上的一个`Java`后台服务程序。
- **Native Client：**它是一个用`C`语言实现的客户端连接库，其与`Java`客户端库一样，上层应用（非`Java`实现）通过它连接至ZooKeeper集群。
- **Contrib：**它是指多个可选的扩展组件。

|操作系统|Client|Server|Native Client|Contrib|
|:-|:-:|:-:|:-:|:-:|
|GNU/Linux|D + P|D + P|D + P|D + P|
|Solaris|D + P|D + P|/|/|
|FreeBSD|D + P|D + P|/|/|
|Windows|D + P|D + P|/|/|
|Mac OS X|D|D|/|/|

> **D：支持开发环境， P：支持生成环境， /：不支持任何环境**

上表中未显式注明支持的组件在相应平台上可能不能正常运行。虽然ZooKeeper社区会尽量修复在未支持平台上发现的BUG，但并无法保证会修复全部BUG。

### 1.1.2. 软件要求
ZooKeeper需要运行在JDK6或以上版本中。若ZooKeeper以集群模式部署，则推荐的节点数至少为3，同时建议部署在独立的服务器上。在Yahoo!，ZooKeeper通常部署在运行RHEL系统的服务器上（服务器配置：双核CPU、2G内存、80G容量IDE硬盘）。

<h3 id="1.2">1.2. 集群部署（安装）</h3>
为了保证ZooKeeper服务的可靠性，您应该以集群模式部署ZooKeeper服务。只要半数以上的集群节点在线，服务将是可用的。因为ZooKeeper需要半数以上节点同意才能选举出Leader，所以建议ZooKeeper集群的节点数为奇数个。举个例子，对于有四个节点的集群只能应付一个节点宕机的异常，如果有两个节点宕机，则剩下两个节点未达到法定的半数以上选票，ZooKeeper服务将变为不可用。而如果集群有五个节点，则集群就可以应付二个节点宕机的异常。
> **提示：**
> 正如[《ZooKeeper快速入门》][1]文档中所提到的，至少需要三个节点的ZooKeeper集群才具备容灾特性，因此我们强烈建议集群节点数为奇数。
> 通常情况下，生产环境下，集群节点数只需要三个。但如果为了在服务维护场景下也保证最大的可靠性，您也许会部署五个节点的集群。原因很简单，如果集群节点为三个，当你对其中一个节点进行维护操作，将很有可能因维护操作导致集群异常。而如果集群节点为5个，那你可以直接将维护节点下线，此时集群仍然可正常提供服务（就算四个节点中的任意一个突然宕机）。
> 您的冗余措施应该包括生产环境的各个方面。如果你部署三个节点的ZooKeeper集群，但你却将这三个节点都连接至同一个网络交换机，那么当交换机宕掉时，你的集群服务也一样是不可用的。

关于如何配置服务器使其成为集群中的一个成员节点的详细步骤如下（每个节点的操作类似）：
1. 安装JDK，你可以使用本地包管理系统安装，也可以从JDK官网[下载][2]
2. 设置Java堆相关参数，这对于避免内存swap来说非常重要。因为频繁的swap将严重降低ZooKeeper集群的性能。您可以通过使用压力负载测试来决定一个合适的值，确保该值刚好低于触发swap的阈值。保守的做法是：当节点拥有4G的内存，则设置-xmx=3G。
3. 安装ZooKeeper，您可以从[官网][3]下载：http://zookeeper.apache.org/releases.html
4. 创建一个配置文件，这个文件可以随便命名，您可以先使用如下设置参数：

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

您可以在[配置参数](#2.10)章节中找到上面这些参数及其他参数的解释说明。这里简要说一下：集群中的每个节点都需要知道集群中其它节点成员的连接信息。通过上述配置文件中格式为`server.id=host:port:port`的若干行配置信息您就可以实现这个目标。`host`与`port`意思很简单明了，不多作说明。而`server.${id}`代表节点ID，你需要为每个节点创建一个名为`myid`的文件，该文件存放于参数`dataDir`所指向的目录下。
5. `myid`文件的内容只有一行，其值为配置参数`server.${id}`中`${id}`的实际值。即服务器1对应的`myid`文件的内容为1，这个值在集群中必须保证其唯一性，同时必须处于[1, 255]之间。
6. 如果你已经创建好配置文件（如zoo.cfg），你就可以启动ZooKeeper服务：

```
$ java -cp zookeeper.jar:lib/slf4j-api-1.6.1.jar:lib/
slf4j-log4j12-1.6.1.jar:lib/log4j-1.2.15.jar:conf \
org.apache.zookeeper.server.quorum.QuorumPeerMain zoo.cfg
```

QuorumPeerMain启动一个ZooKeeper服务，JMX管理bean也同时被注册，通过这些JMX管理bean，你就可以在JMX管理控制台上对ZooKeeper服务进行监控管理。ZooKeeper的[JMX文档][4]有更详细的介绍信息。同时，你也可以在`$ZOOKEEPER_HOME/bin`目录下找到ZooKeeper的启动脚本`zkServer.sh`。
7. 接下来你就可以连接至ZooKeeper节点来测试部署是否成功。
在`Java`环境下，你可以运行下面的命令连接至已启动的ZooKeeper服务节点并执行一些简单的操作

```
$ bin/zkCli.sh -server 127.0.0.1:2181
```

<h3 id="1.3">1.3. 单机开发环境部署介绍</h3>
如果你想部署ZooKeeper以便于开发测试，那你可以使用单机部署模式。然后安装Java或C客户端连接库，同时在配置文件中将服务器信息与开发机绑定。
具体安装部署步骤也集群部署模式类似，唯一不同的是zoo.cfg配置文件更简单一些。你可以从[《ZooKeeper快速入门》][1]文档中的相关章节获取详细的操作步骤。
关于安装客户端连接库的相关信息，你可以从[ZooKeeper Programmer's Guide][5]文件的Bindings章节中获取。

## 2. 维护管理
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

<h3 id="2.1">2.1. ZooKeeper集群部署规划</h3>
ZooKeeper可靠性依赖于两个基本的假设：
1. 一个集群中只会有少数服务器会出错，这里的出错是指宕机或网络异常而使得服务与集群中的多数服务器失去联系。
2. 已部署的机器可以正常运行，所谓正常运行是指所有代码可以正确的执行，有适当且一致的工作时钟、存储、网络。

为了最大可能的保证上述两个前提假设能够成立，这里有几个点需要考虑。一些是关于跨服务器（节点之间）需求的，而另一些是关于集群中每个节点服务器的

#### 2.1.1 Cross Machine Requirements
For the ZooKeeper service to be active, there must be a majority of non-failing machines that
can communicate with each other. To create a deployment that can tolerate the failure of F
machines, you should count on deploying 2xF+1 machines. Thus, a deployment that consists
of three machines can handle one failure, and a deployment of five machines can handle two
failures. Note that a deployment of six machines can only handle two failures since three
machines is not a majority. For this reason, ZooKeeper deployments are usually made up of
an odd number of machines.

To achieve the highest probability of tolerating a failure you should try to make machine
failures independent. For example, if most of the machines share the same switch, failure of
that switch could cause a correlated failure and bring down the service. The same holds true
of shared power circuits, cooling systems, etc.

#### 2.1.2 Single Machine Requirements
If ZooKeeper has to contend with other applications for access to resourses like storage
media, CPU, network, or memory, its performance will suffer markedly. ZooKeeper has
strong durability guarantees, which means it uses storage media to log changes before the
operation responsible for the change is allowed to complete. You should be aware of this
dependency then, and take great care if you want to ensure that ZooKeeper operations aren’t
held up by your media. Here are some things you can do to minimize that sort of degradation:

* ZooKeeper's transaction log must be on a dedicated device. (A dedicated partition is not
enough.) ZooKeeper writes the log sequentially, without seeking Sharing your log device
with other processes can cause seeks and contention, which in turn can cause multisecond
delays.
* Do not put ZooKeeper in a situation that can cause a swap. In order for ZooKeeper to
function with any sort of timeliness, it simply cannot be allowed to swap. Therefore,
make certain that the maximum heap size given to ZooKeeper is not bigger than the
amount of real memory available to ZooKeeper. For more on this, see Things to Avoid
below.

<h3 id="2.2">2.2. Provisioning</h3>

<h3 id="2.3">2.3. Things to Consider: ZooKeeper Strengths and Limitations</h3>

<h3 id="2.4">2.4. Administering</h3>

<h3 id="2.5">2.5. Maintenance</h3>
Little long term maintenance is required for a ZooKeeper cluster however you must be aware
of the following:

#### 2.5.1 Ongoing Data Directory Cleanup
The ZooKeeper Data Directory contains files which are a persistent copy of the znodes
stored by a particular serving ensemble. These are the snapshot and transactional log
files. As changes are made to the znodes these changes are appended to a transaction log,
occasionally, when a log grows large, a snapshot of the current state of all znodes will be
written to the filesystem. This snapshot supercedes all previous logs.

A ZooKeeper server will not remove old snapshots and log files when using the default
configuration (see autopurge below), this is the responsibility of the operator. Every serving
environment is different and therefore the requirements of managing these files may differ
from install to install (backup for example).

The PurgeTxnLog utility implements a simple retention policy that administrators can use.
The [API docs][6] contains details on calling conventions (arguments, etc...).

In the following example the last count snapshots and their corresponding logs are retained
and the others are deleted. The value of` <count>` should typically be greater than 3 (although
not required, this provides 3 backups in the unlikely event a recent log has become
corrupted). This can be run as a cron job on the ZooKeeper server machines to clean up the
logs daily.

```
$ java -cp zookeeper.jar:lib/slf4j-api-1.6.1.jar:lib/slf4j-log4j12-1.6.1.jar:lib/
log4j-1.2.15.jar:conf org.apache.zookeeper.server.PurgeTxnLog <dataDir> <snapDir> -n <count>
```

Automatic purging of the snapshots and corresponding transaction logs was introduced
in version 3.4.0 and can be enabled via the following configuration parameters
`autopurge.snapRetainCount` and `autopurge.purgeInterval`. For more on this, see
Advanced Configuration below.

#### 2.5.2 Debug Log Cleanup (log4j)
See the section on logging in this document. It is expected that you will setup a rolling file
appender using the in-built log4j feature. The sample configuration file in the release tar's
conf/log4j.properties provides an example of this.

<h3 id="2.6">2.6. Supervision</h3>
You will want to have a supervisory process that manages each of your ZooKeeper server
processes (JVM). The ZK server is designed to be "fail fast" meaning that it will shutdown
(process exit) if an error occurs that it cannot recover from. As a ZooKeeper serving cluster
is highly reliable, this means that while the server may go down the cluster as a whole is still
active and serving requests. Additionally, as the cluster is "self healing" the failed server
once restarted will automatically rejoin the ensemble w/o any manual interaction.

Having a supervisory process such as daemontools or SMF (other options for supervisory
process are also available, it's up to you which one you would like to use, these are just two
examples) managing your ZooKeeper server ensures that if the process does exit abnormally
it will automatically be restarted and will quickly rejoin the cluster.

<h3 id="2.7">2.7. Monitoring</h3>
The ZooKeeper service can be monitored in one of two primary ways; 1) the command
port through the use of 4 letter words and 2) JMX. See the appropriate section for your
environment/requirements.

<h3 id="2.8">2.8. Logging</h3>
ZooKeeper uses log4j version 1.2 as its logging infrastructure. The ZooKeeper
default log4j.properties file resides in the conf directory. Log4j requires that
log4j.properties either be in the working directory (the directory from which
ZooKeeper is run) or be accessible from the classpath.

For more information, see [Log4j Default Initialization Procedure][7] of the log4j manual.
<h3 id="2.9">2.9. Troubleshooting</h3>
**Server not coming up because of file corruption**
A server might not be able to read its database and fail to come up because of some
file corruption in the transaction logs of the ZooKeeper server. You will see some
IOException on loading ZooKeeper database. In such a case, make sure all the other
servers in your ensemble are up and working. Use "stat" command on the command port
to see if they are in good health. After you have verified that all the other servers of the
ensemble are up, you can go ahead and clean the database of the corrupt server. Delete all
the files in datadir/version-2 and datalogdir/version-2/. Restart the server.

<h3 id="2.10">2.10. 配置参数</h3>
ZooKeeper's behavior is governed by the ZooKeeper configuration file. This file is designed
so that the exact same file can be used by all the servers that make up a ZooKeeper server assuming the disk layouts are the same. If servers use different configuration files, care must
be taken to ensure that the list of servers in all of the different configuration files match.

#### 2.10.1 Minimum Configuration
Here are the minimum configuration keywords that must be defined in the configuration file:
**clientPort**
the port to listen for client connections; that is, the port that clients attempt to connect to.
**dataDir**
the location where ZooKeeper will store the in-memory database snapshots and, unless
specified otherwise, the transaction log of updates to the database.
> Be careful where you put the transaction log. A dedicated transaction log device is key
to consistent good performance. Putting the log on a busy device will adversely effect
performance.

**tickTime**
the length of a single tick, which is the basic time unit used by ZooKeeper, as measured
in milliseconds. It is used to regulate heartbeats, and timeouts. For example, the
minimum session timeout will be two ticks.

#### 2.10.2 Advanced Configuration
The configuration settings in the section are optional. You can use them to further fine tune
the behaviour of your ZooKeeper servers. Some can also be set using Java system properties,
generally of the form zookeeper.keyword. The exact system property, when available, is
noted below.
**dataLogDir**
(No Java system property)
This option will direct the machine to write the transaction log to the dataLogDir
rather than the dataDir. This allows a dedicated log device to be used, and helps avoid
competition between logging and snaphots.
> Having a dedicated log device has a large impact on throughput and stable latencies. It is highly
recommened to dedicate a log device and set dataLogDir to point to a directory on that device,
and then make sure to point dataDir to a directory not residing on that device.

**globalOutstandingLimit**
(Java system property: zookeeper.globalOutstandingLimit.)
Clients can submit requests faster than ZooKeeper can process them, especially if
there are a lot of clients. To prevent ZooKeeper from running out of memory due
to queued requests, ZooKeeper will throttle clients so that there is no more than
globalOutstandingLimit outstanding requests in the system. The default limit is 1,000.


<h3 id="2.11">2.11.  ZooKeeper Commands: The Four Letter Words</h3>
ZooKeeper responds to a small set of commands. Each command is composed of four letters.
You issue the commands to ZooKeeper via telnet or nc, at the client port.
Three of the more interesting commands: "stat" gives some general information about the
server and connected clients, while "srvr" and "cons" give extended details on server and
connections respectively.

> **conf**
New in 3.3.0: Print details about serving configuration.
**cons**
New in 3.3.0: List full connection/session details for all clients connected to this server.
Includes information on numbers of packets received/sent, session id, operation latencies,
last operation performed, etc...
**crst**
New in 3.3.0: Reset connection/session statistics for all connections.
**dump**
Lists the outstanding sessions and ephemeral nodes. This only works on the leader.
**envi**
Print details about serving environment
**ruok**
Tests if server is running in a non-error state. The server will respond with imok if it is
running. Otherwise it will not respond at all.
A response of "imok" does not necessarily indicate that the server has joined the quorum,
just that the server process is active and bound to the specified client port. Use "stat" for
details on state wrt quorum and client connection information.
**srst**
Reset server statistics.
**srvr**
New in 3.3.0: Lists full details for the server.
**stat**
Lists brief details for the server and connected clients.
**wchs**
New in 3.3.0: Lists brief information on watches for the server.
**wchc**
New in 3.3.0: Lists detailed information on watches for the server, by session.
This outputs a list of sessions(connections) with associated watches (paths). Note,
depending on the number of watches this operation may be expensive (ie impact server
performance), use it carefully.
**wchp**
New in 3.3.0: Lists detailed information on watches for the server, by path. This outputs
a list of paths (znodes) with associated sessions. Note, depending on the number of
watches this operation may be expensive (ie impact server performance), use it carefully.
**mntr**
New in 3.4.0: Outputs a list of variables that could be used for monitoring the health of
the cluster.

```
$ echo mntr | nc localhost 2185
zk_version 3.4.0
zk_avg_latency 0
zk_max_latency 0
zk_min_latency 0
zk_packets_received 70
zk_packets_sent 69
zk_outstanding_requests 0
zk_server_state leader
zk_znode_count 4
zk_watch_count 0
zk_ephemerals_count 0
zk_approximate_data_size 27
zk_followers 4 - only exposed by the Leader
zk_synced_followers 4 - only exposed by the Leader
zk_pending_syncs 0 - only exposed by the Leader
zk_open_file_descriptor_count 23 - only available on Unix platforms
zk_max_file_descriptor_count 1024 - only available on Unix platforms
```

The output is compatible with java properties format and the content may change over
time (new keys added). Your scripts should expect changes.
ATTENTION: Some of the keys are platform specific and some of the keys are only
exported by the Leader.
The output contains multiple lines with the following format:
`key \t value`

Here's an example of the ruok command:

```
$ echo ruok | nc 127.0.0.1 5111
imok
```


<h3 id="2.12">2.12. Data File Management</h3>
ZooKeeper stores its data in a data directory and its transaction log in a transaction log
directory. By default these two directories are the same. The server can (and should) be
configured to store the transaction log files in a separate directory than the data files.
Throughput increases and latency decreases when transaction logs reside on a dedicated log
devices.

#### 2.12.1 The Data Directory
This directory has two files in it:
* myid - contains a single integer in human readable ASCII text that represents the server
id.
* snapshot.<zxid> - holds the fuzzy snapshot of a data tree.

Each ZooKeeper server has a unique id. This id is used in two places: the myid file and
the configuration file. The myid file identifies the server that corresponds to the given data
directory. The configuration file lists the contact information for each server identified by
its server id. When a ZooKeeper server instance starts, it reads its id from the myid file and
then, using that id, reads from the configuration file, looking up the port on which it should
listen.

The snapshot files stored in the data directory are fuzzy snapshots in the sense that during
the time the ZooKeeper server is taking the snapshot, updates are occurring to the data tree.
The suffix of the snapshot file names is the zxid, the ZooKeeper transaction id, of the last
committed transaction at the start of the snapshot. Thus, the snapshot includes a subset of
the updates to the data tree that occurred while the snapshot was in process. The snapshot,
then, may not correspond to any data tree that actually existed, and for this reason we refer
to it as a fuzzy snapshot. Still, ZooKeeper can recover using this snapshot because it takes
advantage of the idempotent nature of its updates. By replaying the transaction log against
fuzzy snapshots ZooKeeper gets the state of the system at the end of the log.

#### 2.12.2 The Log Directory
The Log Directory contains the ZooKeeper transaction logs. Before any update takes place,
ZooKeeper ensures that the transaction that represents the update is written to non-volatile
storage. A new log file is started each time a snapshot is begun. The log file's suffix is the
first zxid written to that log.

#### 2.12.3 File Management
The format of snapshot and log files does not change between standalone ZooKeeper servers
and different configurations of replicated ZooKeeper servers. Therefore, you can pull these
files from a running replicated ZooKeeper server to a development machine with a standalone
ZooKeeper server for trouble shooting.

Using older log and snapshot files, you can look at the previous state of ZooKeeper servers
and even restore that state. The LogFormatter class allows an administrator to look at the
transactions in a log.

The ZooKeeper server creates snapshot and log files, but never deletes them. The retention
policy of the data and log files is implemented outside of the ZooKeeper server. The serveritself only needs the latest complete fuzzy snapshot and the log files from the start of that
snapshot. See the maintenance section in this document for more details on setting a retention
policy and maintenance of ZooKeeper storage.

> The data stored in these files is not encrypted. In the case of storing sensitive data in ZooKeeper,necessary measures need to be taken to prevent unauthorized access. Such measures are external to ZooKeeper (e.g., control access to the files) and depend on the individual settings in which it is being deployed.

<h3 id="2.13">2.13. Things to Avoid</h3>
Here are some common problems you can avoid by configuring ZooKeeper correctly:

**inconsistent lists of servers**
The list of ZooKeeper servers used by the clients must match the list of ZooKeeper servers that each ZooKeeper server has. Things work okay if the client list is a subset of the real list, but things will really act strange if clients have a list of ZooKeeper servers that are in different ZooKeeper clusters. Also, the server lists in each Zookeeper server
configuration file should be consistent with one another.

**incorrect placement of transasction log**
The most performance critical part of ZooKeeper is the transaction log. ZooKeeper syncs
transactions to media before it returns a response. A dedicated transaction log device
is key to consistent good performance. Putting the log on a busy device will adversely
effect performance. If you only have one storage device, put trace files on NFS and
increase the snapshotCount; it doesn't eliminate the problem, but it should mitigate it.

**incorrect Java heap size**
You should take special care to set your Java max heap size correctly. In particular, you
should not create a situation in which ZooKeeper swaps to disk. The disk is death to
ZooKeeper. Everything is ordered, so if processing one request swaps the disk, all other
queued requests will probably do the same. the disk. DON'T SWAP.

Be conservative in your estimates: if you have 4G of RAM, do not set the Java max heap
size to 6G or even 4G. For example, it is more likely you would use a 3G heap for a 4G
machine, as the operating system and the cache also need memory. The best and only
recommend practice for estimating the heap size your system needs is to run load tests,
and then make sure you are well below the usage limit that would cause the system to
swap.

**Publicly accessible deployment**
A ZooKeeper ensemble is expected to operate in a trusted computing environment. It is
thus recommended to deploy ZooKeeper behind a firewall.

<h3 id="2.14">2.14. Best Practices</h3>
For best results, take note of the following list of good Zookeeper practices:
For multi-tennant installations see the section detailing ZooKeeper "chroot" support, this can
be very useful when deploying many applications/services interfacing to a single ZooKeeper
cluster.






[1]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperStarted.html
[2]: http://java.sun.com/javase/downloads/index.jsp
[3]: http://zookeeper.apache.org/releases.html
[4]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperJMX.html
[5]: https://zookeeper.apache.org/doc/r3.4.10/zookeeperProgrammers.html
[6]: https://zookeeper.apache.org/doc/r3.4.10/api/index.html
[7]: http://logging.apache.org/log4j/1.2/manual.html#defaultInit
