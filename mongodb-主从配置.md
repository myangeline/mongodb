# mongodb的主从配置

## 1. 准备工作
需要两台机器，并且装好mongodb，mongodb的安装很简单，直接下载解压即可，
创建一个data、log目录，在添加一些配置文件和配置信息

## 2. 启动主节点
windows下的启动：

    > mongod --dbpath F:\database\mongodb\data --master
    
linux下的启动
    
    > ./mongod --dbpath /mongodb/data --master

## 3. 启动从节点

    > ./mongod -dbpath /srv/mongodb/data --slave --source 192.168.0.1:27017
关键配置：指定主节点ip地址和端口 –source 192.168.0.1:27017 和 标示从节点 –source 参数。

这样就很简单的配置好了主从关系，这样的关系很不好用，除了会同步数据外，不能自动切换，需要人工的参与
所以，官网不建议这种配置
    