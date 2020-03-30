# Metadata Analysis Service

## 背景介绍

随着HDFS集群的规模增长，用户数据量的不断增加，就需要对大量的数据进行高质量管理，从而催生了数据资产业务。用户数据量的增加同样会对集群造成更大的压力，需要更加了解数据的存储细节，从而更加有效的进行集群运维。对于HDFS集群元数据分析的需求主要包含下列几项业务：

- 数据资产管理
- 集群监控告警
- 数据存储管理

这几个业务的需求主要集中在如下几个方面：

- 查询目录及文件的基础元数据信息。
- 统计分析目录及文件的分布情况。
- 统计分析目录及文件的访问频率。
- 查询目录及文件元数据的历史变更。

HDFS本身提供了接口用于查询文件或目录的元数据信息，但是这些接口请求会占用NameNode的FSNamesystem的锁，如果大量请求调用的话，会导致NameNode锁繁忙，影响用户对于数据的访问。另外HDFS提供的元数据查询接口有时候需要查询大量的元数据信息，从而大量占用NameNode计算资源。

所以希望构建一个独立的元数据查询及分析服务，与生产环境的NameNode隔离开，满足上述业务对于元数据的查询及统计需求。

## 需求描述

### 功能性需求

接口默认响应时间为秒级。

#### 提供RestAPI接口，查询指定路径的元数据信息。

元数据信息包括：

完整路径、属主信息、数据量统计信息（元数据、存储量）、最后访问时间等。

#### 提供RestAPI接口，查询指定路径下文件大小分布。

返回结果：

- 各类型子元数据数量统计（子目录、子文件、快照等）
- 百分位数分别为10/20/30/40/50/60/70/80/90的文件大小。

#### 提供RestAPI接口，查询指定BlockID对应的文件路径。

返回结果：

- 对应的完整文件路径及相关元数据信息
- 已被删除的文件也依然能保留一段时间的块对应关系。

#### 提供RestAPI接口，查询指定路径、指定元数据类型及指定时间范围的历史数据。

查询条件：

- 指定路径
- 指定时间范围
- 指定查询数据列（元数据、大小分布）

返回结果：

- 指定查询数据列的历史数据列表。

#### 提供RestAPI接口，查询指定路径及指定时间范围的访问次数。

查询条件：

- 指定路径
- 指定时间范围

返回结果：

- 指定时间范围的历史访问次数列表

### 非功能性需求

#### 可靠性

元数据系统后续将服务于自动化存储管理系统，用户存储管理系统进行冷热数据整理，可接受的最长故障时间为1天。

元数据系统需要提供给面向用户的系统进行使用，其可用性将影响到用户满意度，需要有较高可用率，并且可接受最长故障时间为5分钟。

元数据系统需要提供给监控告警系统使用，及时性要求较高，需要有非常高的可用率，可接受的最长故障时间为5分钟。

综上，该系统是个用户及管理依赖度较高的服务，需要较高可用率，初定为99.95%，故障恢复时间为5分钟。

#### 可扩展性

目前生产环境的元数据量总共为10亿左右。

该服务需要进行数据预处理统计，所以有一定的计算量需求。

无状态服务，方便高可用及横向扩展。

该服务接受的请求主要来源为：用户请求、监控告警系统事件型请求、存储管理系统周期性请求，初步预估QPS为：1000。

#### 可维护性

主备自动切换，自动恢复，滚动升级。

无状态服务与K8S服务化，使其具备最佳可维护性。

## 架构设计

![HDFS-Metadata-architecture](Metadata.assets/HDFS-Metadata-architecture.png)

整体架构如上图所示。HDFS Metadata Service作为无状态化服务，由K8S进行管理或者裸机部署，提供相关API给上层应用。Metadata负责解析FSImage、FSEditLog及FSAuditLog数据，存储在本地RocksDB中，定期进行Checkpoint，并定期增量备份到远程HDFS中。

Metadata具体负责事项：

1. Metadata初始化时，从StandbyNamenode获取FSImage，并解析出目录树，存储到本地RocksDB中。
2. Metadata解析完FSImage之后，不断从JournalNode获取增量FSEditLog，解析并更新本地RocksDB数据。
3. Metadata根据EditLog中的具有时间戳的Op，对本地RocksDB按小时进行RocksDB Checkpoint生成。
4. Metadata从Kafka获取FSAuditLog，解析日志并更新到本地RocksDB，根据日志时间戳按小时进行RocksDB Checkpoint生成。
5. Metadata按小时周期增量备份RocksDB到HDFS中。
6. Metadata还需负责对于过期的Checkpoint进行降低精度的操作。

Metadata恢复流程：

1. Metadata从HDFS下载RocksDB备份数据。
2. 从备份数据的EditLog transaction ID开始从JournalNode增量拉取EditLog，更新到本地RocksDB。
3. 从备份数据记录的AuditLog最后一条记录开始，从Kafka增量拉取AuditLog，更新到本地RocksDB。

上层应用通过K8S服务发现获取Metadata的主服务地址，调用API获取详细数据。如果是裸机部署，则使用Nginx提供API服务。

## 详细设计

### 数据存储结构

#### 目录树元数据表

Key为：目录树路径

value为：元数据对象

**元数据对象包括：**

type：该路径类型，包括目录、文件、符号链接三种

username/groupname：属主信息

permission：权限信息

nsquota/dsquota：配额信息，路径为目录时有效

replication/preferredblocksize：文件实际副本数及预期副本数

modificationtime/accesstime：最后变更时间戳

filesize/blockscount：文件大小及副本数；如果为目录则为当前目录下的所有文件大小及数据块总数（不递归）。

fileinode/directoryinode/symlinkinode：目录下文件数量及目录数量（不递归）。

totalfilesize/totalblockscount/totalfileinode/totaldirectoryinode/totalsymlinkinode：递归子目录的总文件大小，总块数，总文件数量，总目录数量。

filesizepctl10/filesizepctl20/filesizepctl30/filesizepctl40/filesizepctl50/filesizepctl60/filesizepctl70/filesizepctl80/filesizepctl90：目录下文件大小的百分位数

#### 目录树热度表

Key为：目录树路径

value为：访问热度对象

**访问热度对象包括**：

lastreadtime/lastwritetime：最后一次读写时间戳。

readcount/writecount：读写累积次数；如果为目录，则为目录下所有文件累积。

#### 数据块映射表

Key为：数据块ID

Value为：对应的文件路径

该表数据在文件删除时不进行删除，对已删除数据块数据保存一定的时间之后再清理。

#### Checkpoint表

Key为：表与Checkpoint组合串

Value为：源数据偏移量

如果为元数据表，则偏移量为EditLog的transition ID；如果为热度表，则为Kafka上AuditLog的消费偏移量。

#### TODO：删除及历史数据问题

有如下场景需要解决：

- 在Checkpoint之间，新建文件并且很快就删除了，元数据信息将无法保留在表中，查询不到其元数据信息。需要提供延迟删除的能力，保障短暂存在的文件也能查找到元数据。
- 延迟删除带来的问题是以路径为Key的表数据冲突现象。同个文件删除再创建，该如何表示需要思考。

### 主要流程描述

#### FSImage解析流程

只有在初始化元数据服务的时候才需要进行FSImage解析，服务已经存在的只需要获取RocksDB备份，再通过EditLog进行增量同步数据即可恢复服务。

1. Metadata通过HDFS接口从StandbyNameNode同步FSImage到本地，再通过FSImage相关接口进行解析。
2. FSImage包含的数据只是较为原始的数据，并没有包括元数据统计相关的信息，需要经过多次统计，可以参考HugeImageTextWriter类的解析方式。

FSImage的解析需要消耗1~2个小时时间，时间消耗很长，所以后续的数据变更及恢复不再依赖于FSImage重新解析。

从StandbyNameNode获取FSImage需要hdfs管理员用户，如果达不到运维安全需求，可以通过从HDFS下载FSImage的方式来替代。

#### FSEditLog解析流程

1. FSImage是基于某一个EditLog的Transaction ID进行合并的，当FSImage解析完成后即可通过该ID从JournalNode获取之后的增量EditLog。
2. Metadata解析EditLog，解析出每个Operation，更新本地的RocksDB元数据。
3. Mkdir/Rename/Delete/Concat/Truncate这些Operation里面带有时间戳，当解析到这几个Op的时候，查看时间戳，检查是否需要创建Checkpoint。
4. 如果需要创建Checkpoint，则创建RockDB Checkpoint并保存到以时间命名的分区目录中，更新Checkpoint表，写入当前解析到的Transaction ID。

**TODO：**

Checkpoint的时间来源于特定Op的时间戳，如果某段时间一直没有这类型Op，Checkpoint将不会创建。需要深入查看EditLog，是否有其它机制来得到每个Op的时间戳，使Checkpoint更加准确。

#### FSAuditLog解析流程

目前AuditLog已经由Smilodon Beat采集到Kafka，所以可以从Kafka获取AuditLog，并进行解析。

1. Metadata从Checkpoint表中获取热度表当前Kafka消费的偏移量，使用该偏移量从Kafka消费AuditLog。如果没有记录偏移量，则从头开始消费。
2. 解析AuditLog，获取时间戳，检查是否创建热度表的Checkpoint；如果需要创建Checkpoint，则创建RockDB Checkpoint并保存到以时间命名的分区目录中，更新Checkpoint表，写入当前AuditLog的记录偏移量。
3. 解析AuditLog，过滤无关操作，只保留读写相关的操作，递归增加路径节点的热度表数据，相关路径访问次数累加。

#### Checkpoint备份流程

Metadata周期性（一小时）把RocksDB新生成的Checkpoint增量备份到HDFS中。RocksDB本身具备对Checkpoint进行全量或者增量备份的能力。

#### Checkpoint降精度

Metadata每个小时创建一个RocksDB的Checkpoint，时间久了以后会很多，所以需要定期进行Checkpoint的降精度。小时精度保留一个月，对于一个月前的Checkpoint，由小时精度降为天精度。

1. Metadata服务线程每天检查Checkpoint列表，对于超过一个月的Checkpoint，删除多余Checkpoint，每天只保留一份。
2. 检查Checkpoint表，清除多余的Checkpoint记录数据。

#### 服务恢复流程

当需要对服务进行升级、扩容、迁移等操作时，需要涉及到恢复流程。服务恢复不需要重新解析整个FSImage，只需要从备份数据进行恢复即可。

1. Metadata服务启动，对比本地数据目录与HDFS备份目录数据，从HDFS备份目录同步增量数据到本地。
2. 从Checkpoint表去除当前Checkpoint对应的数据偏移量，启动EditLog和AuditLog的解析线程，开始解析增量数据到本地。

#### 用户API实现

上层应用的点查请求相对比较简单：

1. 解析请求路径，确定相应集群。
2. 打开对应的RocksDB数据，获取结果。

需要查询历史数据的请求：

1. 需要解析请求路径，确定对应集群。
2. 解析时间范围，确定需要访问的所有Checkpoint数据。
3. 查询每个Checkpoint数据，并返回。

### 非功能性需求实现

#### 高可用

Metadata服务解析EditLog和AuditLog生成数据，元数据来源一致，分析方式也是一致的，形成的本地数据最终将一致，所以可以认为Metadata服务具有幂等性。具有幂等性的服务可以比较简单的进行多实例部署。

Metadata多实例也存在冲突点：

- Metadata需要备份数据到HDFS中，从而提供其它实例进行快速恢复；备份操作在多实例间会冲突。

- 多实例之间的数据只具备最终一致性，如果使用负载均衡模式，不同实例获取的数据可能存在不一致。

备份冲突的解决方式是：

1. Metadata备份线程先检查即将备份的Checkpoint点是否已经备份到HDFS中。
2. 如果未备份，则检查是否有该Checkpoint的lock文件，表明已有Metadata服务正在准备进行备份。
3. 如果存在lock文件，则检查lock文件是否超时，如果已经超时则删除lock文件及未完成的Checkpoint数据；如果未超时，则跳过该Checkpoint。
4. 如果未有实例正在备份，则创建lock文件并写入当前时间戳，开始进行备份。

多实例可能读取数据不一致的解决方式是：上层应用请求的转发使用主备方式，单点读。可以通过Nginx的简单主从热备实现，或者K8S的主备切换实现。

#### 多集群

Metadata服务不需要绑定到具体某一个HDFS集群，可以为所有的HDFS集群服务，只需要把数据保存在不用DB目录即可。处理用户请求的时候，可以从请求路径解析对应的集群并读取相应DB的数据。

#### 性能

RocksDB点查性能在毫秒级别，所以接口的响应速度不会出现性能瓶颈。

性能需要关注的地方在于：

- EditLog及AuditLog的数据量，Metadata的处理速度。
- 生成的RocksDB数据量，Checkpoint的性能（本地磁盘支持硬链接，速度快；云存储并不支持硬链接，使用拷贝数据方式，速度慢）。
- 增量Checkpoint的数据量及备份速度。
- EditLog从JournalNode获取，需要关注JournalNode的负载，避免影响线上业务。

EditLog事实上也可以考虑使用Beat进行采集，再进行分析，规避JournalNode的安全问题及性能问题。

#### 可维护性

应用K8S体系，将获得便捷的可维护性，包括：服务发现、主备切换、扩容、迁移等。但是也存在一个问题，该服务极度依赖于RocksDB，如果容器使用云存储，在进行Checkpoint的时候将会受到性能影响，需要进行测试验证。

该服务应用于Mammut和Smilodon，需要考虑商业化问题，商业化目前采用物理机或者云主机部署，所以该服务也需要能支持裸机部署，通过Nginx提供请求转发。

## 后续扩展

第一期Metadata的服务主要满足上层应用对于指定路径元数据的点查，后续还需要去满足元数据的统计分析能力。Metadata把数据存储在RocksDB，并备份到HDFS，RocksDB提供SST Dump工具，可以把RocksDB导出。导出后的数据同样可以使用大数据数仓体系进行有效的分析。