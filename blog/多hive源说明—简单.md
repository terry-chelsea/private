#<center> Apache Kylin一套环境支持多个hive源 <center>


##为什么产生多个Hive源
 
目前Kylin只能使用一个hive作为输入源，一个hive说白了就是一套metastore，但是在有些场景下，当产品较多、表较大、分区过多的时候metastore会成为hive的瓶颈，而metastore本身也没有提供隔离的机制，因此多产品之间可能使用相互独立的metastore，这就导致当kylin作为一个基础服务提供给不同产品服务时需要对于每一套metastore部署一个集群，进而必须部署多个kylin metastore，多个hbase输出等等，这会带来相当大的管理和运维代价，所以一套kylin环境支持多个hive源的需求对于环境比较复杂的用户（例如我们）是强烈需要的。

随着公司产品的规模越来越大，最初部署的hadoop集群可能需要成倍的扩容，但是由于机房规模和最初的机架部署问题导致在需要在新的机房重新部署一套hadoop集群，而不是直接在原机房直接扩容，这导致每一个大规模的扩容就产生一个hadoop集群，将原集群的数据全部导入到新集群的代价比较大，所以维持多个hadoop集群的状态，而每一个hive只能依赖于一个hadoop集群（通过HADOOP_HOME配置指定hadoop集群的客户端），这也催生了多个hive的产生。

## Kylin遇到的问题

我们都知道，kylin依赖于shell中的hive命令找到hive的配置（hive-site.xml）和hive依赖的lib等，那么在支持多个hive源的时候如何找到其它hive源的配置文件呢？除此之外，每一个hive需要配置HADOOP_HOME指定它所依赖的hadoop客户端，而kylin只能使用一个hadoop集群客户端（通过hadoop命令找到相应的配置文件），因此可能存在跨级群提交的问题，例如kylin依赖的hadoop客户端为A集群，而需要作为输入源的hive分别依赖于A、B、C三个hadoop集群，所以需要支持向A集群提交的任务输入源在A、B、C集群中。

## 如果使用

使用多hive源的时候需要首先配置kylin.external.hive.root.directory配置项，它指定了本机一个目录，该目录下每一个目录都是一个hive客户端，每一个目录名视为该hive源的名字。例如：

	nrpt@olap-kylin-job:~/hive-root$ ls 
	binjiang_1   xiaoshan_2   liantong_77

每一个目录下需要有conf目录和bin目录，前者下面必须存在hive-site.xml文件作为改hive源的配置文件，bin目录下必须存在hive命令行作为该hive源的启动命令，当然官方hive的目录结构就是这样，只要不是刻意的修改就不会有问题。

然后在kylin中每一个project只能关联一个hive源作为输入，在创建project的时候指定hive源的名字，这样这个project下load table只能从这个hive源选择，然后完成接下来的kylin创建model和cube的流程。

考虑到不同的hive源，第一步生成hive中间表是使用源hive的命令行执行生成的，然后通过distcp命令拷贝到kylin依赖的hadoop集群，接着在默认的（命令行上hive命令）hive中创建一个一样的external表，接下来所有的任务都不再依赖源hive，和kylin之前的流程一样了。

* Create Intermediate Flat Hive Table in Local Hive：该任务使用源hive命令（绝对路径）创建临时表，并在它依赖的hadoop集群生成临时数据。
* Redistribute Flat Hive Table：使用源hive命令（绝对路径）重新生成临时数据，count计数文件存储在本地目录。
* Create Intermediate Flat Hive Table in Default Hive：使用hive命令在默认的hive下生成临时外部表。
* Copy Intermediate Table To Local Hadoop：拷贝源hive生成的数据到kylin依赖的hadoop集群相同的目录下（如果需要）

最后在Garbage Collection的时候还需要额外的垃圾清理流程，例如删除源hive的临时文件和中间表等。

除了hive客户端的部署在本地，如果需要支持多个hadoop集群，还需要修改hadoop命令的配置文件hdfs-site.xml，是它支持多个hadoop name service，这样可以使得在kylin中可以访问多个hadoop集群的数据。

## 限制

只能使用CLI的方式执行hive任务，而不能使用beeline。kylin.hive.client只能设置为cli（默认）。
hive命令的执行只能在本机执行，kylin.job.run.as.remote.cmd只能设置为false（默认）。
