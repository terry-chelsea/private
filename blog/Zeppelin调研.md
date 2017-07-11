#<center>Zeppelin调研</center>

## Zeppelin是什么

Apache Zeppelin是一个开源的基于web的notebook的实现，通过它可以在web上与spark/hive/impala等实现交互式的分析查询，并且可以支持图表展现，结果下载/分享等。Notebook的交互方式更类似于用户通过spark/hive客户端查询数据的方式，用户可以随时获取之前的查询和结果，而在web上集成多个服务，也大大降低了用户的使用代价。

目前Zeppelin支持的引擎类型还是比较丰富的，基本上涵盖了日常使用到的大数据工具，如下图：

<center>![这里写图片描述](http://img.blog.csdn.net/20161109202003905)</center>

## Zeppelin运行原理

和其他类似的系统（HUE）一样，zeppelin对于每一类支持的引擎都可以创建多个配置，每一个插件的配置称为一个Interpreter，相同类型的Interpreter称为一个Interpreter group，一个group内的所有Interpreter可以共享一个JVM进程，如下图：

<center>![这里写图片描述](http://img.blog.csdn.net/20161109110358933)</center>

在zeppelin启动时启动ZeppelinServer进程，当用户选择使用某一个interpreter的时候会通过lazy的方式启动一个进程负责该interperter的请求，ZeppelinServer和interpreter进程之间的通信通过thrift完成的。整体来看zeppelinServer类似于猛犸的web server，每一个interpreter进程类似于猛犸的执行服务器。

## Zeppelin使用方式

如果对于一个普通用户部署和配置一套自己的zeppelin而言，通常需要配置自己的interpreter，下面以配置一个hive interpreter的方式展示zeppelin的使用方式：

1、创建一个hive interpreter

<center>![这里写图片描述](http://img.blog.csdn.net/20161109110419871)</center>

hive属于jdbc interpreter group，然后配置一些该interpreter的配置项，这里需要配置url和driver，由于使用kerberos认证则不需要user和password

2、创建notebook

<center>![这里写图片描述](http://img.blog.csdn.net/20161109110644539)</center>

3、设置notebook binding的interpreter

<center>![这里写图片描述](http://img.blog.csdn.net/20161109155632992)</center>

这里取消了默认的jdbc interpreter，而选择使用自己创建的hive interpreter，因此对于一个notebook，对于一种类型的服务只能指定一个interpreter，否则可能出现找不到正确的配置的问题。

4、在notebook中执行操作或者查询

<center>![这里写图片描述](http://img.blog.csdn.net/20161109172915814)</center>

</center>![这里写图片描述](http://img.blog.csdn.net/20161109173123455)</center>

由于把hive这个interpreter移到了第一位，所以如果不加任何的%xxx则默认使用它，也可以使用%jdbc访问，访问spark需要使用%spark来指定。

## 猛犸使用Zeppelin

基于zeppelin的特性，可以将猛犸数仓的查询，甚至spark任务的执行放在zeppelin上，前者包括hive(on spark or on MR),impala,kylin,对于zeppelin而言，这些都是可以通过jdbc的方式访问。zeppelin对于spark分析（代码）和scala的执行是它较为显著的特点，所以在zeppelin上也可以支持spark的分析。

## 猛犸使用可能遇到的问题

通过对Zeppelin的了解和使用，结合猛犸的现状，总结出如果猛犸使用Zeppelin需要解决的问题。

* 用户接入

zeppelin有自己的一套用户认证机制，基于apache shiro框架，由于猛犸的认证是基于openID的，所以用户认证方面还需要重新定义自己的Realm，用户的角色可以通过猛犸中获取，这部分还需要详细了解猛犸的用户认证体系。

* 多个spark环境

zeppelin显著的特点就是支持spark分析，但是由于我们环境中现存多个yarn集群，而每一个zeppelin环境只能配置一个SPARK_HOME来引用spark的配置文件，因此如果支持全部集群的spark分析则需要每一个集群部署一个zeppelin环境。

* 多个hive/impala/kylin

zeppelin访问这些服务都是通过jdbc的方式完成的，impala是完全兼容hive的，所以在猛犸上可以将impala（独立的）看做是一个所有用户都能够看到的hive server，这样每一个用户至少存在两个server（一个hive server，一个impala），可能有的用户存在多个hive server的情况，对于jdbc连接而言，如果不希望每次查询都携带数据库则需要url中指定database，这样每一个数据库就对应着一套jdbc的配置（url不同），如何让用户透明的使用自己的hive server是一个不得不考虑的问题。除此之外，由于目前ranger的权限依赖于hive server的url加入了corp邮箱，这样就导致每一个用户需要一个url（如果需要细粒度的访问权限），这更进一步增加了url的数量。

* notebook存储与隔离

目前notebook是存储在本地的，并且可以支持本地git，S3和Azure上，如果存储在其他引擎上则需要进行扩展，notebook之间的隔离目前是notebook的创建者自己进行授权的（分为读/写权限）。

* interpreter之间的隔离

目前来看，接入猛犸不同用户之间的数仓肯定需要在zeppelin上创建多个interpreter，而在zeppelin中，interpreter是全局可见的，也没有类似管理员授权某些用户可以看到使用某些interpreter配置的操作，如果在接入猛犸时需要考虑如何将不同的interpreter和用户关联起来，不同的用户只能访问自己的interpreter。

* kerberos访问

截止到0.6.2版本，zeppelin还是不能很好地支持kerberos，在未发布的0.7.0版本上支持了（ZEPPELIN-1146）kerberos访问，但是对于猛犸的使用远远不止这样，猛犸需要使用超级用户代理的方式访问hive server和impala。

* 性能问题

zeppelin执行语句的时候是通过zeppelin server提交到后端的interpreter process执行的，对于猛犸而言需要jdbc和spark两种interpreter，spark使用zeppelin的时候只能使用yarn-client的方式提交，需要消耗一定的本地资源，而jdbc的interpreter process如果被所有的notebook共享，是否会存在性能问题还有待验证。

* 元数据展示

由于zeppelin的notebook并不是简单的支持jdbc的查询，所以并没有做database/table/column的展示，需要用户建立好interpreter之后手动的show xxx以获取相应的信息。