# Zookeeper配置

## 配置zoo.cfg

这是zookeeper的主要配置文件．因为Zookeeper是一个集群服务，集群的每个节点都需要这个配置文件。为了避免出差错，zoo.cfg这个配置文件里没有跟特定节点相关的配置，所以每个节点上的这个zoo.cfg都是一模一样的配置。这样就非常便于管理了，比如我们可以把这个文件提交到版本控制里管理起来。其实这给我们设计集群系统的时候也是个提示：**集群系统一般有很多配置，应该尽量将通用的配置和特定每个服务的配置(比如服务标识)分离，这样通用的配置在不同服务之间copy就ok了**。ok，下面来介绍一些配置点：

```
clientPort=2181   //client port，顾名思义，就是客户端连接zookeeper服务的端口。这是一个TCP port。
//zookeeper的持久化都存储在这两个目录里。dataLogDir里是放到的顺序日志(WAL)。而dataDir里放的是内存数据结构的snapshot，便于快速恢复。
//为了达到性能最大化，一般建议把dataDir和dataLogDir分到不同的磁盘上，这样就可以充分利用磁盘顺序写的特性。
dataDir=/data

dataLogDir=/datalog     //dataLogDir如果没提供的话使用的则是dataDir。
```
下面是集群中服务的列表
```
server.1=127.0.0.1:20881:30881
server.2=127.0.0.1:20882:30882
server.3=127.0.0.1:20883:30883
```
在上面的例子中，我把三个zookeeper服务放到同一台机器上。上面的配置中有两个TCP port。后面一个是用于Zookeeper选举用的，而前一个是Leader和Follower或Observer交换数据使用的。我们还注意到server.后面的数字。这个就是myid

- **initLimit**：这个配置项是用来配置 Zookeeper 接受客户端（这里所说的客户端不是用户连接 Zookeeper 服务器的客户端，而是 Zookeeper 服务器集群中连接到 Leader 的 Follower 服务器）初始化连接时最长能忍受多少个心跳时间间隔数。当已经超过 10 个心跳的时间（也就是 tickTime）长度后 Zookeeper 服务器还没有收到客户端的返回信息，那么表明这个客户端连接失败。总的时间长度就是 5*2000=10 秒

- **syncLimit**：这个配置项标识 Leader 与 Follower 之间发送消息，请求和应答时间长度，最长不能超过多少个 tickTime 的时间长度，总的时间长度就是 2*2000=4 秒

- **server.A=B：C：D**：其中 A 是一个数字，表示这个是第几号服务器；B 是这个服务器的 ip 地址；C 表示的是这个服务器与集群中的 Leader 服务器交换信息的端口；D 表示的是万一集群中的 Leader 服务器挂了，需要一个端口来重新进行选举，选出一个新的 Leader，而这个端口就是用来执行选举时服务器相互通信的端口。如果是伪集群的配置方式，由于 B 都是一样，所以不同的 Zookeeper 实例通信端口号不能一样，所以要给它们分配不同的端口号。

- **tickTime**，这是个时间单位定量。比如tickTime=1000，这就表示在zookeeper里1 tick表示1000 ms，所有其他用到时间的地方都会用多少tick来表示。
比如 syncLimit = 2 就表示fowller与leader的心跳时间是2 tick。

- **maxClientCnxns **-- 对于一个客户端的连接数限制，默认是60，这在大部分时候是足够了。但是在我们实际使用中发现，在测试环境经常超过这个数，经过调查发现有的团队将几十个应用全部部署到一台机器上，以方便测试，于是这个数字就超过了。

- **minSessionTimeout**, maxSessionTimeout -- 一般，客户端连接zookeeper的时候，都会设置一个session timeout，如果超过这个时间client没有与zookeeper server有联系，则这个session会被设置为过期(如果这个session上有临时节点，则会被全部删除，这就是实现集群感知的基础)。但是这个时间不是客户端可以无限制设置的，服务器可以设置这两个参数来限制客户端设置的范围。

- **autopurge.snapRetainCount**，autopurge.purgeInterval -- 客户端在与zookeeper交互过程中会产生非常多的日志，而且zookeeper也会将内存中的数据作为snapshot保存下来，这些数据是不会被自动删除的，这样磁盘中这样的数据就会越来越多。不过可以通过这两个参数来设置，让zookeeper自动删除数据。autopurge.purgeInterval就是设置多少小时清理一次。而autopurge.snapRetainCount是设置保留多少个snapshot，之前的则删除。

不过如果你的集群是一个非常繁忙的集群，然后又碰上这个删除操作，可能会影响zookeeper集群的性能，所以一般会让这个过程在访问低谷的时候进行，但是遗憾的是zookeeper并没有设置在哪个时间点运行的设置，所以有的时候我们会禁用这个自动删除的功能，而在服务器上配置一个cron，然后在凌晨来干这件事。

## 配置-myid

在dataDir里会放置一个myid文件，里面就一个数字，用来唯一标识这个服务。这个id是很重要的，一定要保证整个集群中唯一。zookeeper会根据这个id来取出server.x上的配置。比如当前id为1，则对应着zoo.cfg里的server.1的配置。

而且在leader选举里，这个id的大小也是有意义的。
