# mongodb遇到的一些问题

## 1. ERROR: child process failed, exited with error number 100

这个问题查看对应的日志文件，可以发现下面的内容

    2015-10-21T09:39:04.588+0800 W -        [initandlisten] Detected unclean shutdown - /srv/mongodb/mongodb-linux-x86_64-3.0.6/shard1/data/mongod.lock is not empty.
    2015-10-21T09:39:04.599+0800 I STORAGE  [initandlisten] **************
    Unclean shutdown detected.
    Please visit http://dochub.mongodb.org/core/repair for recovery instructions.
    *************
    2015-10-21T09:39:04.599+0800 I STORAGE  [initandlisten] exception in initAndListen: 12596 old lock file, terminating
    2015-10-21T09:39:04.599+0800 I CONTROL  [initandlisten] dbexit:  rc: 100
    
如果是这样的话，那是因为mongod.lock文件没有删除的原因，直接去删掉即可 `rm mongod.lock`
出现这个问题不止这一个原因，可能还有journal文件太大存储空间不足导致的，可以选择关闭也可以解决问题，设置 `nojournal = true`即可


