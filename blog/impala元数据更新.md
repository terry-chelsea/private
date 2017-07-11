#<center>Impala中的invalidate metadata和refresh<center>

## 前言

Impala采用了比较奇葩的多个impalad同时提供服务的方式，并且它会由catalogd缓存全部元数据，再通过statestored完成每一次的元数据的更新到impalad节点上，Impala集群会缓存全部的元数据，这种缓存机制就导致通过其他手段更新元数据或者数据对于Impala是无感知的，例如通过hive建表，直接拷贝新的数据到HDFS上等，Impala提供了两种机制来实现元数据的更新，分别是INVALIDATE METADATA和REFRESH操作，本文将详细介绍这两个操作。

## 使用方式

INVALIDATE METADATA是用于刷新全库或者某个表的元数据，包括表的元数据和表内的文件数据，它会首先清楚表的缓存，然后从metastore中重新加载全部数据并缓存，该操作代价比较重，主要用于在hive中修改了表的元数据，需要同步到impalad，例如create table/drop table/alter table add columns等。

INVALIDATE METADATA 语法：

	INVALIDATE METADATA;                   //重新加载所有库中的所有表
	INVALIDATE METADATA [table]            //重新加载指定的某个表

REFRESH是用于刷新某个表或者某个分区的数据信息，它会重用之前的表元数据，仅仅执行文件刷新操作，它能够检测到表中分区的增加和减少，主要用于表中元数据未修改，数据的修改，例如INSERT INTO、LOAD DATA、ALTER TABLE ADD PARTITION、LLTER TABLE DROP PARTITION等，如果直接修改表的HDFS文件（增加、删除或者重命名）也需要指定REFRESH刷新数据信息。

REFRESH 语法：
 
	REFRESH [table]                             //刷新某个表
	REFRESH [table] PARTITION [partition]       //刷新某个表的某个分区

## INVALIDATE METADATA原理

对于INVALIDATE METADATA操作，由客户端将查询提交到某个impalad节点上，执行如下的操作：

* 1.获取需要执行INVALIDATE METADATA的表，如果没指定表则不设置表示全部表（不考虑这种情况）。
* 2.请求catalogd执行resetMetadata操作，并将isFresh参数设置为false。
* 3.catalogd接收到该请求之后执行invalidateTable操作，将该表的缓存清除，然后重新生成该表的缓存对象，新生成的对象只包含表名+库名的信息，为新生成的表对象生成一个新的catalog版本号（假设新的version=1），将这部分信息返回给调用方（impalad），然后异步执行元数据和数据的加载。
* 4.impalad收到catalogd的返回值，返回值是更新之后的表缓存对象+版本号，但是这是一个不完整的表元数据，impalad将这个元数据应用到本地元数据缓存。
* 5.INVALIDATE METADATA执行完成。

INVALIDATE METADATA操作带来的副作用是生成一个新的未完成的元数据对象，对于操作请求的impalad（称它为impalad-A），能够立马获取到该对象，对于其它的impalad需要通过statestored同步，因此执行完该操作，处理该操作的impalad对于该表的缓存是一个新的但是不完整的对象，其余的impalad保存的是旧的元数据。

对于后续的该表查询操作，分为如下四种情况：

* 如果catalogd已经完成该表所有元数据加载，会对该表生成一个新的版本号（假设version=2），然后更新到statestored，由statestored广播到各个impalad节点上，此时所有的查询都查询到最新的元数据和数据。
* 如果catalogd尚未完成表的元数据加载或者statestored未广播完成，并且接下来请求到impalad-A（之前执行INVALIDATE METADATA的节点），此时impalad在执行语义分析的时候能够检测到表的元数据不完整（因为当前只有表名和库名，没有任何其余的元数据），impalad会直接请求catalogd获取该表最新的元数据，如果catalogd尚未完成元数据加载，则该请求会等到直到catalogd加载完成并返回impalad最新的元数据。
* 如果catalogd尚未完成表的元数据加载或statestored未广播完成，接下来请求到了其他的impalad节点，如果接受请求的impalad尚未通过statestored同步新的不完整的表元数据（version=1），则该impalad中缓存的关于该表的元数据是执行INVALIDATE METADATA之前的，因此根据旧的元数据处理该查询（可能因为文件被删除导致错误）。
* 如果catalogd尚未完成表的元数据加载，接下来请求到了其他的impalad节点，如果接受请求的impalad已经通过statestored同步新的不完整的表元数据（version=1），那么接下来会像第二种情况一样处理。

从INVALIDATE METADATA的实现来看，该操作不仅仅会全量加载表的元数据和分区、文件元数据，还会影响后面关于该表的查询。


## REFRESH原理

对于REFRESH操作，由客户端将查询提交到某个impalad节点上，执行如下的操作：

* 1.获取需要执行REFRESH的表和分区信息。
* 2.请求catalogd执行resetMetadata操作，并将isFresh参数设置为true。
* 3.catalogd接收到该请求之后判断是否指定分区，如果指定了分区则执行reload partition操作，如果未指定则执行reload table操作，对于reloadPartition则从metastore中读取partition最新的元数据，然后刷新该partition拥有的所有文件的元数据（大小，权限，数据分布等）；对于reloadTable则从metadata中读取全部的partition信息，然后和缓存中的partition进行比对判断是否有分区需要增加和删除，对于其余的分区则执行元数据的更新。
* 4.impalad收到catalogd的返回值，返回值是更新之后该表的缓存数据，impalad会将该数据更新到自己的缓存中。因此接受请求的impalad能够将当前元数据缓存。
* 5.REFRESH执行完成。

对于后续的查询，分为如下两种情况：

* 如果查询提交到到执行REFRESH的impalad节点，那么查询能够使用最新的元数据。
* 如果查询提交到其他impalad节点，需要依赖于该表0更新后的缓存是否已经同步到impalad中，如果已经完成了同步则可以使用最新的元数据，如果未完成则使用旧的元数据。

可以看出REFRESH操作较之于INVALIDATE METADATA是轻量级的操作，如果更改只涉及到一个分区设置可以只刷新一个分区的元数据，并且它是同步的，对于之后查询的影响较小。

## 使用原则

如果在使用过程中涉及到了元数据或者数据的更新，则需要使用这两者中的一个操作完成，具体如何选择需要根据如下原则：

* invalidate metadata操作比refresh要重量级
* 如果涉及到表的schema改变，使用invalidate metadata [table]
* 如果只是涉及到表的数据改变，使用refresh [table]
* 如果只是涉及到表的某一个分区数据改变，使用refresh [table] partition [partition]
* 禁止使用invalidate metadata什么都不加，宁愿重启catalogd。

## 总结

REFRESH和INVALIDATE METADATA对于impala而言是比较重要的两个操作，分别处理数据和元数据的修改，其中REFRESH操作是同步的，INVALIDATE METADATA是异步的，本文详细介绍了两种语句的适用场景和执行原理，以及可能造成的影响，最重要的是，需要谨记这两种查询使用场景。