rto Recovery Time Objective  30
RPO Recovery point objective 复原点目标 RPO=0 
传统集中数据库 劣势 ：
capex 固定资产的折旧，无形资产、递延资产摊销。
CAPEX=战略性投资+滚动型投资
opex指 企业的管理支出，运营成本。管理支出、办公室支出、员工工资支出和广告支出。
opex=维护费用+营销费用+人工成本（+折旧）。

应用场景有限，高端硬件的性能限制。
Google Spanner 数据库 OceanBase数据库
跨库分布式事务  MyCAT 
数据库引擎没有分布式并行计算能力，不能跨多台机器协同执行，需要中间件来完成
数据一致性：中间件无法100%保证多台机器的数据一致性。ACID
负载均衡
跨库复杂sql

share-Everything | Paxos
RAC,pureScale     | MPP架构
TPC-C 7.07亿 tpmc
峰值6100万次/秒
单表 最大3200亿行
准内存处理 LSM Tree  无锁结构  消除磁盘的随机写
支持 oracle mysql 生态

二、ob通过paxos协议 高可用，HTAP可以含OLTP OLAP的功能
OCP 给运维 ODC 面向开发 OMS 异构迁移
OB集群、zone observer 资源池 租户 分区
集群-zone(一批机器tag)奇数 -observer-资源池-租户-数据库-表-分区-副本
unit=2一个zone 内找2台OB Server并分配资源  每个zone 只存储一份且只有一份副本

RootService 总控服务
系统元数据管理、资源分配及调度（分区副本管理、动态负载均衡、扩缩容、全局DDL、集群数据合并）一个主、其余备
一个应用占用一个租户 系统的ID 1000以内 
一个租户在通一个server上最多有一个unit

root@sys 默认断口2881

租户创建：1create resource unit 
 2 create resource pool 
3 create tenant
创一个unit（1）  会在zonelist 分别找一台ob 吧资源要求的资源分配下去
创一个unit（2） 每个zone 找2台ob 分配资源

select * from_all_resource_poll 

系统日志：1ob erver日志   2  日志文件个数 3 日志级别

5.1 Paxos协议与负载均衡
数据分区：一个大表，拆分若干个分区，hash分区，list分区 range分区
二级分区，分区是ocean base基本单元
分区副本：分区在物理存储多份，每一份叫做分区副本
根据负载和策略，自动调度在多个server。
一个hash分区--4个range分区-每个分区生成多个副本-分区的不同副本存储在不同的zone
------
副本构成（全能型）：记录事务的日志，存储在内存的增量数据MemTable 磁盘上静态数据 SSTable
日志型：log：有  无 无
只读型：log：有  无 无

Paxos 以分区为单位，
任何一个follower完成redo-log落盘并返回给leader后，就认为redo-log完成强同步，无需等待其他folllower的反馈。

OB proxy 智能路由服务 不参与数据库引擎的计算任务，不参与事务处理，多个OB Proxy之间无联系，
与OB server共用一台服务器，也可以部署在应用服务器中。
OP proxy 根据用户的租户名和数据库名，表明一级分区ID等信息查询路由表，获取分区的主/从副本所在OB server的IP 地址信息。proxy请求路由到主副本所在机器，同时将副本的位置信息更新到location cache
OB Proxy 轻量的sql解析数据库名和标明，更新到location cache
OB Proxy 和数据库中间件不同，他是一个无状态的服务进程，不做持久化，不参与数据库引擎的计算任务。session

Primary Zone 将业务汇聚到指定Zone
zone1=zone2> zone3 适合2地三中心，两城市举例较近，分布式事务跨zone执行，zone3没有
主副本，


如果租户的unit_num=1 且primary_zone只有一个zone，不需要tablegroup、
关联的表会绑定tablegroup

如果一个ob server上的主副本挂掉了 另外两个选举一个新主副本，并将IP地址更细到路由表中，
redo-log日志可以强同步，确保了RPO=0，如果网络故障无法联系到从副本，就会些人主副本，不再承接业务，从副本就在选出一个主副本，避免脑裂。

每个城市部署一个OceanBase集群一个位主 一个位备集群，每个集群都有单独的paxos group
可达到国际灾难恢复能力6级
66页？


分布式事务、mvcc 事务隔离级别
MVCC 多用户并发访问数据库时，开启事务，互不干扰，采用MVCC技术解决读写互斥的问题。
paxos多副本之间同步redo-log redo-log落盘
两阶段提交，保障原子性，obporxy 不参与两阶段提交
1、一阶段投标阶段，协调者询问是否可以发送prepare请求与事务内容，
询问是否可以准备事务，并等待参与者的响应。向协调者返回事务操作的执行结果。
2、第二阶段是执行阶段，如果所有参与者都返回ok，协调者发送提交，并完成事务返回给应用。
如果有任何一个参与者返回no或者超时回滚。
其实是observer


多版本并发控制MVCC 解决读写互斥问题
1、采用锁机制控制读写冲突，加锁后其他事务无法进行读
导致读写竞争，影响读的并发度.MVCC可以有效解决该问题
全局统一的数据版本号管理，取自全局唯一的时间戳服务GTS
读写操作都要从GTS获取版本号，同意租户内只有一个GTS服务，可以保持全局一致性；
修改数据：事务未提交前，数据的新旧版本共存，但有用不同的版本号；
读取数据：先获取版本号，再去查找小鱼等于当前版本号的已提交的数据
写操作获取行锁，读操作不需要锁，有效避免读写锁竞争，提高读写并发读。
read-committed serializable

wal wite ahead logging (wal)日志先行
oracle：redo log 块变化 
记录逻辑操作：binlog


OB sql引擎和存储过程
双模 mysql——oracle
数据库事务，指作为单个逻辑工作单元执行的一些列操作，维护完整性
begin transaction  begin begin work 
commit  rollback
随机写、写放大的问题（就是只修改1个页内的几个字节，就要更新到硬盘表空间中，造成放大）
buffer pool 读内存没有就从硬盘中提取
修改将数据写到buffer pool 再刷新到磁盘
checkpoint 随机写和写放大
磁盘化表空间、再划分不同的extend 在划分若干页 block

OB  内存数据库+LSMTree存储，避免随机写（合并和在写）
内存分为两块：一块是MemTable（insert update）脏数据，用于写，一块是热点缓存，用于读。
把内存的一大批脏数据写到硬盘，这个过程叫转储

LSMTree 架构 数据在磁盘上默认按主键有序排列，内存数据和磁盘基线数据合并后，会重新排序。
然后批量写的形式顺序到硬盘上，可以把原有的碎片去掉。提升压缩能力。
LSMTree 压缩第一次字典和rle算法瘦身 压缩第二次 lz4压缩算法

管理范围 参数生效
参数级别 管理权限
alter  system 指定zone  仅能指定zone
zone_merge_timeout 3h    单个zone合并的超时时间
freeze_trigger_percentage  70 出发合并时，memstore使用的百分比
enable_manual_merge  是否开启手动合并 
major_freeze_duty_time 02:00 每日定时合并任务的启动时间
sql_audit)memory_limit 3G 开启sql审计功能状态，sql审计内部表最大可用内存
variables 与业务租户相关

global 级修改数据库实例共享全局变量
global variables

OCP企业级数据库管理平台
高效的ob集群管理，支持高可用性，最多可以同时管理500台主机，相应速度可达秒级








