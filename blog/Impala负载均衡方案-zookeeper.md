#<center>Impala负载均衡方案——zookeeper<center>

## 缘来

之前根据Impala官方的文档尝试使用haproxy实现impalad节点的负载均衡，但是这种方案存在一些弊端，例如haproxy本身也是单点的，虽然可以通过keeplived实现haproxy的高可用，但是这样的配置难免有点太重了，实现impala负载均衡的同时还需要多部署两个组件，增大了系统运维的复杂度。在大数据生态圈中zookeeper是一个必不可少的自身具有高可用保证的组件，本文探讨如何使用zookeeper实现impalad的负载均衡。

## HS2方案

众所周知，hiveserver2中可以集成zookeeper实现高可用，毕竟hiveserver2也可以视为无状态的服务，如果配置了zookeeper则在进程启动的时候向zk注册，可以同时注册多个hiveserver2服务，在jdbc连接的时候hive-jdbc可以识别获取服务的配置，然后再向这个服务发起连接，hiveserver2注册时会指定一个zookeeper的注册的根目录，每一个节点在注册的时候在该节点下写入一个节点，节点名为：

	serverUri=db-87.photo.163.org:21050;version=1.2.1;sequence=0000000013

serverUri由hive.server2.thrift.bind.host和hive.server2.thrift.port配置决定的，version等于当前hiveserver2的版本号，sequence保持一个全局递增的序列号。

每一个节点的内容如下：

	db-87.photo.163.org:21050
	cZxid = 0x2500018ab0
	ctime = Wed Dec 28 11:22:33 CST 2016
	mZxid = 0x2500018ab0
	mtime = Wed Dec 28 11:22:33 CST 2016
	pZxid = 0x2500018ab0
	cversion = 0
	dataVersion = 0
	aclVersion = 0
	ephemeralOwner = 0x358997ea0460d74
	dataLength = 25
	numChildren = 0

节点里面保存了注册时的一些信息，最主要的是第一行服务的IP和端口，这样在jdbc创建连接的时候客户端会随机选择一个服务节点并且用其信息替换掉jdbc url中原始的主机和端口信息，然后再尝试使用替换之后的url创建连接，如果创建失败则将其加入到error列表中，继续随机获取下一个节点，直到创建连接成功或者全部服务节点都不可用。

## ZK方案

既然hiveserver2原生的带有这种特性，而impala又可以兼容hive-jdbc客户端，那么理所当然impala也可以使用这种方案实现负载均衡，现在的问题是impalad在启动的时候并不会向zookeeper注册，虽然客户端提供了该机制，没有服务端的配合也不可能work，此时，我们的方案是通过不侵入impala代码的方式独立的启动一个注册和监控进程，它的作用是为impalad节点注册，但是由于hiveserver2向zk注册时创建的是EPHEMERAL类型的临时节点，所以当session关闭时这个节点也会自动被删除，这样客户端就不能获取到该服务信息，因此这个注册进程必须和impalad进程拥有共同的生命周期（至少不能比impalad的生命周期长）。

我们的做法是创建一个独立的进程，可以在impalad脚本启动的时候随着impalad进程一块启动，该进程启动时向配置的zookeeper注册一个节点和改服务的信息，然后周期的检查impalad是否存在（目前的方法是通过查看/proc/pid目录查看），如果impalad进程不存在则退出，zookeeper上注册的临时节点也随之被删除。

该进程直接调用hiveserver2的注册和删除方法，但是由于HiveServer2中存在的方法都是private的，因此需要使用反射进程变量的设置和函数调用。

最后，hiveserver2依赖的配置信息：

	需要注册到zookeeper的根节点
	HIVE_SERVER2_ZOOKEEPER_NAMESPACE("hive.server2.zookeeper.namespace", "hiveserver2",
	        "The parent node in ZooKeeper used by HiveServer2 when supporting dynamic service discovery."),
	
	注册的端口号
	HIVE_SERVER2_THRIFT_PORT("hive.server2.thrift.port", 10000,
	        "Port number of HiveServer2 Thrift interface when hive.server2.transport.mode is 'binary'."),
	
	注册的机器IP，如果不指定默认是本机的hostname
	HIVE_SERVER2_THRIFT_BIND_HOST("hive.server2.thrift.bind.host", "",
	        "Bind host on which to run the HiveServer2 Thrift service."),          
	
	注册的zookeeper地址，可以使多个地址通过‘,’分割
	HIVE_ZOOKEEPER_QUORUM("hive.zookeeper.quorum", "",
	        "List of ZooKeeper servers to talk to. This is needed for: \n" +
	        "1. Read/write locks - when hive.lock.manager is set to \n" +
	        "org.apache.hadoop.hive.ql.lockmgr.zookeeper.ZooKeeperHiveLockManager, \n" +
	        "2. When HiveServer2 supports service discovery via Zookeeper.\n" +
	        "3. For delegation token storage if zookeeper store is used, if\n" +
	        "hive.cluster.delegation.token.store.zookeeper.connectString is not set"),
	
	注册的zookeeper的端口号
	HIVE_ZOOKEEPER_CLIENT_PORT("hive.zookeeper.client.port", "2181",
	        "The port of ZooKeeper servers to talk to.\n" +
	        "If the list of Zookeeper servers specified in hive.zookeeper.quorum\n" +
	        "does not contain port numbers, this value is used.")
	
可以通过将这些配置信息存储在一个hive-site.xml文件下，并且在进程启动时指定该文件在进程的classpath下，或者在启动参数中指定--hiveconf设置这些参数，这样可以脱离配置文件进行启动。

代码地址：[点这里](https://github.com/terry-chelsea/bigdata/tree/master/impala)
使用实例：

	java -cp .:target/impala-tools-0.0.1.jar:./tools/lib/* com.netease.impala.service.RegisterImpalaTools -pid 12345 --hiveconf hive.server2.zookeeper.namespace=impala-dev --hiveconf hive.server2.thrift.port=16124 --hiveconf hive.server2.thrift.bind.host=db-87.photo.163.org --hiveconf hive.zookeeper.quorum=classb-bigdata4.server.163.org --hiveconf hive.zookeeper.client.port=2181

注册之后可以直接在beeline上创建jdbc连接，使用实例：

	!connect jdbc:hive2://classb-bigdata4.server.163.org:2181/default;serviceDiscoveryMode=zooKeeper;zooKeeperNamespace=impala-dev;principal=impala/db-87.photo.163.org@HADOOP.HZ.NETEASE.COM;

最后需要注意的是不太的impalad对外部客户端提供的principal可能不相同，例如每一个机器使用的principal是impala/_HOST@realm，这样的话无论对于那种方式的代理都是不可以的，因此需要修改服务端的--principal配置，一般使用上文介绍的合并keytab的方式，将两个keytab合并成一个，使得同一个keytab可以使用不同的principal认证，对外提供相同的一个principal，impalad内部通信则使用impala/_HOST@realm的principal。

## 总结

本文介绍并且实现了另外一种基于zookeeper的方式实现impalad节点高可用的方案，该方案不需要依赖繁重的haproxy+keeplived繁重的组建以来，并且利用原生jdbc的特性实现impalad的负载均衡和高可用，但是这种方案也存在一定的缺陷，例如独立的监控进程谁来保证，如果该进程down了那么它所监控的impalad也不能对外提供服务了（因为zookeeper上该服务对应的节点会被删除），如何保证该进程的生命周期比它所监控的impalad进程更久是该方案面临的最大问题。

不过这个监控进程毕竟是非常轻量级的，相对于impalad它down的几率还是比较小的，即使它挂掉了而impalad仍然存在，也只是意味着在这个监控进程重启之前该impalad不能对外提供服务，如果在多个impalad能够同时提供服务的集群中，这种影响是可以接受的。

最后，使用该工具只能讲hs2的接口注册到zk，并且通过beeline或者hive-jdbc访问，但是对于直接使用thrift接口还是比较麻烦的，接下来还需要寻找如何将hs2接口转成thrift接口的方式。
