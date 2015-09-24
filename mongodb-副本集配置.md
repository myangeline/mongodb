# mongodb 副本集配置

## 1. 准备

首先准备好三台机器，最少需要三个副本集，因为两台的话，如果PRIMARY掉线的话，不能进行选举生成新的PRIMARY

## 2. 启动副本集
在每台机器上运行下面的启动命令:

    > ./mongod --dbpath /srv/mongodb/data --replSet replset
    
    replSet:表示副本集的名称
    启动的时候需要replSet的值是一样的才是同一个副本集

## 3. 初始化配置

随便进入一台机器进行初始化操作，完成后机器变成PRIMARY

### 连接mongodb
    
    # 使用admin数据库
    > use admin
    
    # 定义副本集配置变量，这里的 _id:”repset” 和上面命令参数“ –replSet repset” 要保持一样。
    >config = { _id:"repset", members:[
    ... {_id:0,host:"192.168.0.1:27017"},
    ... {_id:1,host:"192.168.0.2:27017"},
    ... {_id:2,host:"192.168.0.3:27017"}]
    ... }
    
    # 输出
    {
        "_id" : "repset",
        "members" : [
                {
                        "_id" : 0,
                        "host" : "192.168.0.98:27017"
                },
                {
                        "_id" : 1,
                        "host" : "192.168.0.61:27017"
                },
                {
                        "_id" : 2,
                        "host" : "192.168.1.54:27017"
                }
        ]
    }

### 初始化副本集配置
    
    >rs.initiate(config)

### 查看集群节点的状态 

    >rs.status()

### 添加副本，在登录到主节点下输入
    
    >rs.add("ip:port")

### 删除副本 
    
    >rs.remove("ip:port")
    
    
## 4. 设置SECONDARY为可读
在主节点读取查询都可以正常进行，但是副本节点默认不允许读，更不许写，所以需要设置为可读，
在每次副本节点的查询前设置如下命令即可：
    
    > rs.slaveOk()
    
在每个连接中只需要设置一次即可，且需要在查询之前进行设置
之后进行正常的查询操作即可

可以设置一个 replStart.js 文件, 里面写入 `rs.slaveOk()`，
每次连接mongodb的时候加一个参数 `--shell replStart.js`即可
    
    > mongo -h host --shell replStart.js
    

StackOverflow中描述如下：

>* You only need to set slaveok when querying from secondaries, and only once per session.
   A note about "eventual consistency": under normal circumstances, replica set secondaries 
   have all the same data as primaries within a second or less. Under very high load, 
   data that you've written to the primary may take a while to replicate to the secondaries. 
   This is known as "replica lag", and reading from a lagging secondary is known as an "eventually consistent" read, 
   because, while the newly written data will show up at some point (barring network failures, etc), 
   it may not be immediately available.

>* Create a file named replStart.js, containing one line: rs.slaveOk()
   Then include --shell replStart.js when you launch the Mongo shell. Of course, 
   if you're connecting locally to a single instance, this doesn't save any typing.




