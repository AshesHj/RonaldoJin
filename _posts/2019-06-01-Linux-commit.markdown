---
layout: post
title: Redis主从安装配置
date: 2019-06-01 00:00:00 +0300
description: Redis主从安装配置 # Add post description (optional)
img: # Add image post (optional)
tags: [java, Linux] # add tag
---

### 参考：
http://www.yunweipai.com/archives/20444.html
http://blog.csdn.net/woniu211111/article/details/54646755
http://redis.majunwei.com/topics/sentinel.html

### 服务器：
10.6.60.149    Redis Master / Sentinel
10.6.60.150    Redis Slave / Sentinel

### 安装：
1. Standalone
* 创建redis安装目录 
`mkdir /usr/local/redis`
* 下载Redis安装包
`wget http://download.redis.io/releases/redis-3.2.8.tar.gz`
* 安装编译安装redis所需要的软件
`yum install -y wget make gcc tcl`
编译安装redis
```
    tar xvf redis-3.2.8.tar.gz
    cd redis-3.2.8
    make PREFIX=/usr/local/redisMALLOC=libc install
```
安装完成后会在 /usr/local/redis看到一个bin目录，有以下文件

![I and My friends]({{site.baseurl}}/assets/img/redis/bin.jpg)

启动redis服务 redis-server /usr/local/src/redis-3.2.8/redis.conf出现以下证明安装成功

![I and My friends]({{site.baseurl}}/assets/img/redis/redis.jpg)

Redis配置信息在redis.conf中包含守护进程模式，端口、日志存储位置、数据存储位置等，
根据服务器规划自行设置。
2. redis主从配置
修改主配置文件以下两项：
* 绑定局域网ip
bind 127.0.0.1 10.6.60.149
* 设置密码
requirepass camelot123
* master的密码 按说主是不用配置这个的，但是因为主从可能互相切换，所以要配置
masterauth camelot123
修改从配置文件以下两项：
### 绑定局域网ip
bind 127.0.0.1 10.6.60.150
### 设置密码
requirepass camelot123
### 设置master的ip和端口
slaveof 10.6.60.149 6379
### 设置master的密码
masterauth camelot123
由于我们设置了密码，所以需要修改service文件，不然重启有问题。因为关闭需要密码：
vim /etc/init.d/redis_6379
找到这一行，添加-a "camelot123"
$CLIEXEC -a "camelot123" -p $REDISPORT shutdown
重启主从安装完成

测试：

### 在主上插入测试数据
```
    [root@bogon bin]# redis-cli -h 10.6.60.149
    10.6.60.149:6379> auth camelot123
    OK
    10.6.60.149:6379> set name "123"
    OK
```
### 在从上获取在主上插入的数据
```
    [root@bogon bin]# redis-cli -h 10.6.60.150
    10.6.60.150:6379> auth camelot123
    OK
    10.6.60.150:6379> get name  
    "123"
```

3. Sentinel（哨兵）

Sentinel的职责：

监控(Monitoring)：Sentinel 会不断地检查你的主服务器和从服务器是否运作正常。

提醒(Notification)：当被监控的某个Redis服务器出现问题时， Sentinel可以通过API向管理员或者其他应用程序发送通知。

自动故障迁移(Automatic failover)：当一个主服务器不能正常工作时， Sentinel会开始一次自动故障迁移操作，它会将失效主服务器的其中一个从服务器升级为新的主服务器，并让失效主服务器的其他从服务器改为复制新的主服务器；当客户端试图连接失效的主服务器时，集群也会向客户端返回新主服务器的地址，使得集群可以使用新主服务器代替失效服务器。

安装（两台机器上都要安装）
切换到redis的解压目录，拷贝sentinel.conf到/etc/redis

```
    cd redis-3.2.8
    cp sentinel.conf /usr/local/redis/
```

修改配置如下：

```
    vim /usr/local/redis/sentinel.conf
    
    daemonize yes
    port 26379
    bind 127.0.0.1 10.6.60.149
    sentinel monitor redis-master 10.6.60.149 6379 2
    sentinel down-after-milliseconds redis-master 10000
    sentinel parallel-syncs redis-master 2
    sentinel failover-timeout redis-master 180000
    sentinel auth-pass redis-master camelot123
    logfile /var/log/redis-sentinel.log
```

配置项说明：

`daemonize yes`

后台运行

`port 26379`

哨兵端口

`bind 127.0.0.1 10.6.60.149`

绑定本机和局域网ip

`sentinel monitor redis-master 10.6.60.149 6379 2`

redis-master：是主数据库的别名，考虑到故障恢复后主数据库的地址和端口号会发生变化，哨兵提供了命令可以通过别名获取主数据库的地址和端口号。

10.6.60.149 6379：初次配置时主数据库的地址和端口号，当主数据库发生变化时，哨兵会自动更新这个配置，不需要我们去关心。

2：该参数用来表示执行故障恢复操作前至少需要几个哨兵节点同意，一般设置为N/2+1(N为哨兵总数)。

`sentinel down-after-milliseconds redis-master 10000`

如果master在多少秒内无反应哨兵会开始进行master-slave间的切换，使用“选举”机制，默认30s

`sentinel failover-timeout redis-master 180000`

如果在多少毫秒内没有把宕掉的那台Master恢复，那Sentinel认为这是一次真正的宕机。在下一次选取时排除该宕掉的Master作为可用的节点，然后等待一定的设定值的毫秒数后再来探测该节点是否恢复，如果恢复就把它作为一台Slave加入Sentinel监测节点群，并在下一次切换时为他分配一个”选取号”。

`sentinel parallel-syncs redis-master 2`

表示一次性允许多少slave指向新的new master. 这里默认为1，如果该数值过大会导致新的master服务器IO剧增，保持默认1即可。

`sentinel auth-pass redis-master camelot123`

当Master设置了密码时，Sentinel连接Master和Slave时需要通过设置参数auth-pass配置相应密码。

`logfile /var/log/redis/redis-sentinel.log`

日志位置

启动sentinel

`redis-sentinel /usr/local/redis/sentinel.conf`

测试Failover，我们让10.6.60.149主机上的redis-master主动休眠60秒来观察failover过程：

`redis-cli -p 6379 -h 10.6.60.149 -a camelot123 DEBUG sleep 60`

查看sentinel日志

`tail -fn200 /var/log/redis-sentinel.log`

