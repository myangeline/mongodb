# mongodb 分片集群配置

## 1. 准备
3台服务器 ip: 192.168.3.45~47
一个mongos， 一个config server， 三个shard
也可以在一台机器上测试，只要端口号不同即可，做好端口的分配

在三台机器上，端口分配：
mongos: 20000
config server: 21000
shard1: 22001
shard2: 22002
shard3: 22003


## 2. 安装mongodb
安装方式一样，下载后解压后即可
当前目录在 `/srv/mongodb/` 下
创建目录：
    
    > mkdir -p mongos/log
    > mkdir -p config/data
    > mkdir -p config/log
    > mkdir -p shard1/data
    > mkdir -p shard1/log
    > mkdir -p shard2/data
    > mkdir -p shard2/log
    > mkdir -p shard3/data
    > mkdir -p shard3/log
这些目录分别用于存储数据和日志，因为mongos不需要存储数据，所以只要一个log文件夹即可

## 3. 启动mongodb
在每一台服务器启动mongodb的配置服务器(config server)：
    
    > bin/mongod --configsvr --dbpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/config/data --port 21000 --logpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/config/log/config.log --fork
    
在每一台服务器分别启动mongodb mongos服务器：
    
    > bin/mongos --configdb 192.168.3.45:21000,192.168.3.46:21000,192.168.3.47:21000 --port 20000 --logpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/mongos/log/mongos.log --fork
    
其中 configdb 指定的是 config server的地址，一般集群中config server是有多个的，所以如果需要mongos连接都可以连接就需要在这里制定多个
 
在每台服务器上配置各个分片的副本集(shard)：
    
    > bin/mongod --shardsvr --replSet shard1 --port 22001 --dbpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard1/data --logpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard1/log/shard1.log --fork --nojournal --oplogSize 10
    > bin/mongod --shardsvr --replSet shard2 --port 22002 --dbpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard2/data --logpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard2/log/shard2.log --fork --nojournal --oplogSize 10
    > bin/mongod --shardsvr --replSet shard3 --port 22003 --dbpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard3/data --logpath /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard3/log/shard3.log --fork --nojournal --oplogSize 10
这里配置的一些参数是测试使用的，在实际中不应该这么使用

## 4. 配置副本集
登录任意一台服务器，连接副本集的mongodb，配置每个分片副本集

    > bin/mongo 127.0.0.1:22001
    > use admin
    > config = {_id:"shard1", members:[{_id:0,host:"192.168.3.45:22001"},{_id:1,host:"192.168.3.46:22001"},{_id:2,host:"192.168.3.47:22001",arbiterOnly:true}]}
    > rs.initiate(config)
    
    > bin/mongo 127.0.0.1:22002
    > use admin
    > config = {_id:"shard2", members:[{_id:0,host:"192.168.3.45:22002",arbiterOnly:true},{_id:1,host:"192.168.3.46:22002"},{_id:2,host:"192.168.3.47:22002"}]}
    > rs.initiate(config)
    
    > bin/mongo 127.0.0.1:22003
    > use admin
    > config = {_id:"shard3", members:[{_id:0,host:"192.168.3.45:22003"},{_id:1,host:"192.168.3.46:22003",arbiterOnly:true},{_id:2,host:"192.168.3.47:22003"}]}
    > rs.initiate(config)
    
## 5. 设置分片配置
连接mongos

    > mongo 127.0.0.1:20000

使用admin数据库

    > use admin

串联路由服务器与分配副本集

    > db.runCommand({addshard:"shard1/192.168.3.45:22001,192.168.3.46:22001,192.168.3.47:22001"})
    > db.runCommand({addshard:"shard2/192.168.3.45:22002,192.168.3.46:22002,192.168.3.47:22002"})
    > db.runCommand({addshard:"shard3/192.168.3.45:22003,192.168.3.46:22003,192.168.3.47:22003"})

如里shard是单台服务器，用 db.runCommand( { addshard : “[: ]” } )这样的命令加入，
如果shard是副本集，用db.runCommand( { addshard : “replicaSetName/[:port][,serverhostname2[:port],…]” });这样的格式表示 。

查看分片服务器的配置

    > db.runCommand( { listshards : 1 } )
    
    {
        "shards" : [
            {
                "_id" : "shard1",
                "host" : "shard1/192.168.3.45:22001,192.168.3.46:22001"
            },
            {
                "_id" : "shard2",
                "host" : "shard2/192.168.3.46:22002,192.168.3.47:22002"
            },
            {
                "_id" : "shard3",
                "host" : "shard3/192.168.3.45:22003,192.168.3.47:22003"
            }
        ],
        "ok" : 1
    }
未列出仲裁节点


目前配置服务、路由服务、分片服务、副本集服务都已经串联起来了

## 6. 数据分片
连接在mongos上，准备让指定的数据库、指定的集合分片生效。

    > mongo 127.0.0.1:20000
    > use admin
    
指定需要分片的数据库(testdb)
    
    > db.runCommand({enablesharding: "testdb"})
    
指定需要分片的集合和片键
    
    > db.runCommand( { shardcollection : "testdb.table1",key : {id: 1} } )
我们设置testdb的 table1 表需要分片，根据 id 自动分片到 shard1 ，shard2，shard3 上面去。要这样设置是因为不是所有mongodb 的数据库和表 都需要分片
这里片键的指定基本上是根据id递增的，这样的片键可能会导致数据的不均衡分配，片键还可以指定为hash
    
    > db.runCommand({shardcollection:"test.table",key:{_id: "hashed"}})
这样的hash片键会根据 `_id` 的hash值来均匀分配到每个分片上


到此一个mongodb的分片配置基本上完成

关于客户端（程序端）连接多个mongos选择的问题，需要做好均衡负载，不然mongos的压力会很大呀...

还有关于mongodb的读写分离的问题，其实mongodb的副本集就可以做到读写分离，Primary用来写，Secondary负责读，不过mongodb的Secondary节点默认不可读，
所以需要设置，具体参考mongodb的副本集配置

## 7. 片键

### 小基数片键：
如果某个片键一共只有N个值，那最多只能有N个数据块，也最多只有个N个分片。则随着数据量的增大会出现非常大的但不可分割的chunk。
如果打算使用小基数片键的原因是需要在那个字段上进行大量的查询，请使用组合片键，并确保第二个字段有非常多的不同值。

### 递增的片键：
使用递增的分片的好处是数据的“局部性”，使得将最新产生的数据放在一起，对于大部分应用来说访问新的数据比访问老的数据更频繁，
这就使得被访问的数据尽快能的都放在内存中，提升读的性能。这类的片键比如时间戳、日期、ObjectId、自增的主键（比如从sqlserver中导入的数据）。
但是这样会导致新的文档总是被插入到“最后”一个分片(块)上去,这种片键创造了一个单一且不可分散的热点，不具有写分散性。

### 随机片键：
随机片键（比如MD5）最终会使数据块均匀分布在各个分片上，一般观点会以为这是一个很好的选择，解决了递增片键不具有写分散的问题，
但是正因为是随机性会导致每次读取都可能访问不同的块，导致不断将数据从硬盘读到内存中，磁盘IO通常会很慢。

### 组合片键：
一个理想的片键是同时拥有递增片键和随即片键的优点，这点很难做到关键是要理解自己的数据然后做出平衡。通常需要组合片键达到这种效果：

### 准升序键加搜索键 {coarselyAscending:1,search:1}

其中coarselyAscending每个值最好能对应几十到几百个数据块（比如以月为单位或天为单位），serach键应当是应用程序中通常都会依据其进行查询的字段，比如GUID。

>*注意：serach字段不能是升序字段，不然整个复合片键就下降为升序片键。这个字段应该具备非升序、分布随机基数适当的特点。

事实上，上面这种复合片键中的第一个字段保证了拥有数据局部性，第二字段则保证了查询的隔离性。同时因为第二个字段具备分布随机的特性因此又一定程度的拥有随机片键的特点。

### 哈希片键：对于哈希片键的选择官方文档中有很明确的说明：
选择哈希分片最大好处就是使得数据在各个节点分布比较均匀。2.2.5 Hased Shaeding一节对哈希片键的使用有简单的测试。

>*注意：建立哈希片键的时候不能指定唯一

>*总结：对MongoDB 单条记录的访问比较随机时，可以考虑采用哈希分片，否则范围分片可能会更好。

每次递增分片都极其不均衡，蛋疼...


