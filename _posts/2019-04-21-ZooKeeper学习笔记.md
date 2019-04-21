---
layout:     post
title:      ZooKeeper学习笔记
subtitle:   ZooKeeper
date:       2019-04-21
author:     Jay
header-img: img/post-bg-hacker.jpg
catalog: true
tags:
    - Zookeeper
    - middleware
---

# ZooKeeper学习笔记

### 第一章 分布式系统概念与ZooKeeper简介

##### 1.Zookeeper简介

中间件，提供协调服务。

作用于分布式系统，发挥其优势，可以为大数据服务。

支持Java，提供Java和C语言的客户端API。

##### 2.Zookeeper特性

- 一致性：数据一致性(顺序一致性)，数据按照顺序分批入库；<http://www.10tiao.com/html/616/201605/2652227239/1.html>
- 原子性：事务要么成功，要么失败，不会局部化；
- 单一视图：客户端连接zk集群中的任意一个节点，数据都是一致的；
- 可靠性：每次对zk的操作状态都会保存在服务端；
- 实时性：客户端可以读取到zk服务端的最新数据。

### 第二章 Zookeeper配置文件zoo.cfg

tickTime: 用于计算的时间单元，比如Session超时：N*tickTime

initLimit: 用于集群，允许从节点连接并同步到master节点的初始化连接时间，以tickTime的倍数来表示

syncLimit: 用于集群，master主节点与从节点之间发送消息，请求与应答时间长度。(心跳机制)

dataDir: 必须配置，内存数据的快照

dataLogDir: 事务日志目录，如果不配置会和dataDir公用

clientPort: 连接服务器的端口，默认2181

### 第三章 Zookeeper基本数据模型

##### 1.特点

- 是一个树形结构，可以理解为linux/unix的文件目录：/usr/local/...

- 每一个节点称为znode，他可以有子节点，也可以有数据

- 每个节点分为临时节点和永久节点，临时节点在客户端断开后消失

- 每个zk节点都有各自的版本号，可以通过命令行显示节点信息

- 每当节点数据发生变化，那么该节点的版本号会累加(乐观锁CAS)

- 删除/修改过时节点，版本号不匹配则会报错

- 每个zk节点存储的数据不宜过大，几K即可

- 节点可以设置权限acl，可以通过权限来限制用户的访问

##### 2.基本操作

- 客户端连接 

./zkServer.sh start

./zkCli.sh

- 查看znode结构

get /test或 stat /test

```java
[zk: localhost:2181(CONNECTED) 10] get /test
jay-xu 节点数据
cZxid = 0x2c9e 节点创建时的事务id
ctime = Thu Apr 11 10:03:57 CST 2019 节点创建时间
mZxid = 0x2c9f 节点修改时的事务id
mtime = Thu Apr 11 10:04:14 CST 2019 节点数据修改时间
pZxid = 0x2c9e 与该节点的子节点（或该节点）的最近一次 创建 / 删除 的时间戳对应。只与 本节点 / 该节点的子节点 有关；与孙子节点无关。
cversion = 0 子节点变更的版本
dataVersion = 1 数据版本
aclVersion = 0 ACL版本
ephemeralOwner = 0x0  值为0，表示该节点是持久节点；不为0时，表明这个节点是临时节点，值为会话id(sessionid)
dataLength = 6 数据长度
numChildren = 0 子节点个数
```

参考：<https://blog.csdn.net/lihao21/article/details/51810395>

- 关闭客户端连接

close

##### 3.ZK的作用

- master节点选举，主节点挂了以后，从节点就会接手工作，并且保证这个节点是唯一的，这样是所谓的首脑模式，从而保证我们集群的高可用。

- 统一配置文件管理，即只需要部署一台服务器，则可以把相同的配置文件同步更新到其他所有服务器，此操作在云计算中用的特别多。

- 发布与订阅，类似消息队列MQ，Dubbo。发布者把数据存在znode上，订阅者会读取这个数据。

- 提供分布式锁，分布式环境下不同进程之间争夺资源，类似于多线程中的锁。

- 集群管理，集群中保证数据的强一致性。

### 第四章 ZK基本特性与ZK客户端命令行学习

##### 1.ZK常用命令行操作

- ls 列出子节点
- ls2 列出子节点和当前节点信息
- stat 获取当前节点信息
- get 获取当前节点信息和数据
- create -e -s 创建节点， -e 临时节点 -s顺序节点
- set path data [version] ，version可以实现乐观锁(带version更新，version No is not valid : /imooc)
- delete path [version]，version可以实现乐观锁(带版本删除)

##### 2.Session的基本原理

- 客户端与服务端之间的连接存在会话
- 每个会话都可以设置一个超时时间
- 服务端超过一定的时间没有收到来自客户端的心跳，session过期
- Session过期，则临时节点znode会被删除
- 心跳机制，客户端向服务端发送ping包请求

##### 3.watcher机制

- 针对每个节点的操作，都会有一个有监督者watcher
- 当监控的某个对象znode发送变化，则触发watcher事件
- zk中的watcher是一次性的，触发后立即销毁
- 父节点、子节点增删改都能触发其watcher
- 针对不同类型的操作，触发的watcher事件是不同的：
  - (子)节点创建事件
  - (子)节点删除事件
  - (子)节点数据变化事件



###### (1)watcher命令行学习

- 通过get path [watch] / stat path [watch]设置watcher
- 父节点事件
  - 创建父节点触发：NodeCreated
  - 修改父节点数据触发：NodeDataChanged
  - 删除父节点触发：NodeDeleted
- 子节点事件
  - ls为父节点设置watcher，创建子节点触发：NodeChildrenChanged
  - ls为父节点设置watcher，删除子节点触发：NodeChildrenChanged
  - ls为父节点(/imooc)设置watcher，修改子节点(/imooc/xyz)不触发事件，需要为子节点(/imooc/xyz)设置watcher(get /imooc/xyz watch)，才能触发事件

###### (2)watcher使用场景

统一资源配置

##### 4.ACL(access control lists)权限控制

- 针对节点可以设置相关读写等权限crwda，目的是为了保障数据安全性
- 权限permissions可以指定不同的权限范围和角色

###### (1)ACL命令行

- getAcl：获取某个节点的acl权限信息
- setAcl：设置某个节点的acl权限信息
- addAuth：输入认证授权信息，注册时输入明文密码(登录)，但是在zk系统里，密码是以加密的形式存在的

###### (2)ACL的构成

- zk的acl通过`[scheme:id:perimssions]`来构成权限列表

  - scheme 代表采用的某种权限机制
  - id 代表允许访问的用户
  - permissions 权限组合字符串

- scheme

  - world: world下只有一个id，即只有一个用户，也就是anyone，那么组合的写法就是：world:anyone:[permissions]
  - auth:  auth不使用任何 id ，代表任何已确认用户。setAcl之前，需要先addauth登录。形式为auth:user:pwd:[permissions]
  - digest: setAcl时需要对密码加密，组合形式为digest:username:BASE64(SHA1(password)):[permissions]
  - 简而言之，auth与digest的区别是前者明文，后者密文，setAcl /path auth:lee:lee:cdwra 与setAcl /path digest:lee:BASE64(SHA1(password)):cdwra是等价的，在通过addauth digest lee:lee后都能拥有操作指定节点的权限
  - ip: 当设置ip为指定的ip时，此时限制ip进行访问，比如ip:192.168.0.1:[permissions]
  - super: 代表超级管理员，拥有所有的权限

- permissions

  权限字符串缩写：crdwa

  - CREATE: 创建子节点
  - READ: 获取节点/子节点
  - WRITE: 设置节点数据
  - DELETE: 删除子节点
  - ADMIN: 设置节点权限

- world:

  world:anyone:crdwa

- auth

  auth:user:pwd:crdwa。auth不使用任何 id ，代表任何已确认用户。setAcl之前，需要先addauth

  digest:user:BASE64(SHA1(pwd)):cdrwa

  addauth digest user:pwd

- ip

  ip:192.168.1.1:cdrwa

- super

  1.修改zkServer.sh，增加super管理员

  2.重启zkServer.sh

###### (3)ACL的使用场景

- 开发和测试环境隔离，开发者无法操作测试库的节点，只能看
- 生产环境上控制指定ip的服务可以访问相关节点，防止混乱

###### (4)参考资料

- [Zookeeper权限管理之坑](<https://www.jianshu.com/p/147ca2533aff>)
- [ZooKeeper ACL权限控制](http://www.yoonper.com/post.php?id=47)

##### 5.四字命令

zk可以通过自身提供的简写命令与服务器进行交互。

echo [command] | nc ip port

- echo stat | nc localhost 2181 查看zk的状态信息，以及mode(单例或集群)
- echo ruok  | nc localhost 2181 查看当前zk Server是否启动，返回imok
- echo dump  | nc localhost 2181 列出未经处理的会话和临时节点
- echo conf  | nc localhost 2181 查看服务器配置
- echo cons  | nc localhost 2181 展示连接到服务器的客户端信息
- echo envi | nc localhost 2181 查看环境变量
- echo mntr  | nc localhost 2181  监控zk健康信息
- echo wchs  | nc localhost 2181 展示watch的信息
- echo wchc  | nc localhost 2181 展示session与watch信息
- echo wchp  | nc localhost 2181 展示path与watch信息

参考：

- <http://zookeeper.apache.org/doc/r3.4.10/zookeeperAdmin.html#sc_zkCommands>

- <https://blog.csdn.net/u013673976/article/details/47279707>

### 第五章 选举模式与Zookeeper集群的安装

##### 1.zookeeper集群搭建

zk集群，主从节点，心跳机制(选举模式)。

```
// zoo.cfg文件
server.1=127.0.0.1:2222:2223  // 2222表示数据同步端口，2223表示选举端口，下同。
server.2=127.0.0.1:3333:3334
server.3=127.0.0.1:4444:4445
```

##### 2.Zookeeper集群节点数量为什么要是奇数个？

- 防止由脑裂造成的集群不可用
- 在容错能力相同的情况下，奇数台更节省资源

参考：<https://blog.csdn.net/u010476994/article/details/79806041>

### 第六章 使用ZooKeeper原生Java API进行客户端开发

- 会话连接与恢复
- 节点的增删改查
- watch与acl相关的操作

### 第七章 Apache Curator客户端的使用

- 会话连接与关闭
- 节点的增删改查
- watch与acl相关的操作

### 第八章 分布式锁

- 死锁：一个服务占有数据的锁，其他服务不能进行任何操作，包括读取数据
- 活锁：一个服务占有数据的锁并且读取数据，其他服务可以进行读取操作，类似读写锁



##### 分布式锁开发

分布式锁获取流程图：

![image-20190421150406222](https://alvin-jay.oss-cn-hangzhou.aliyuncs.com/middleware/ZK/%E5%88%86%E5%B8%83%E5%BC%8F%E6%89%80%E8%8E%B7%E5%8F%96%E6%B5%81%E7%A8%8B%E5%9B%BE.png)

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.recipes.cache.PathChildrenCache;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheEvent;
import org.apache.curator.framework.recipes.cache.PathChildrenCacheListener;
import org.apache.zookeeper.CreateMode;
import org.apache.zookeeper.ZooDefs;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

import java.util.concurrent.CountDownLatch;

// 分布式锁，创建节点成功，表示获得锁
public class DistributedLock {

    private static final Logger LOGGER = LoggerFactory.getLogger(DistributedLock.class);

    private CuratorFramework client;

    /** 未获取分布式锁的线程等待 */
    private static CountDownLatch zkLockLatch = new CountDownLatch(1);

    private static final String IMOOC_LOCK_PROJECT = "imooc-locks";

    private static final String DISTRIBUTED_LOCK = "distributed_lock";

    // init
    public DistributedLock(CuratorFramework client) {
        this.client = client;
        // 使用命名空间
        this.client.usingNamespace("ZKlokcs-Namespace");
        try {
            if (this.client.checkExists().forPath("/" + IMOOC_LOCK_PROJECT) == null) {
                client.create()
                        .creatingParentsIfNeeded()
                        .withMode(CreateMode.PERSISTENT)
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                        .forPath("/" + IMOOC_LOCK_PROJECT);
            }
            // 监听分布式锁节点的删除事件
            addWatcherToLock("/" + IMOOC_LOCK_PROJECT);
        } catch (Exception e) {
            LOGGER.warn("客户端连接ZK服务失败，请重连...");
        }
    }

    // 增加分布式锁节点监听，用于唤醒等待线程
    private void addWatcherToLock(String path) throws Exception {
        final PathChildrenCache pathChildrenCache = new PathChildrenCache(client, path, true);
        pathChildrenCache.start(PathChildrenCache.StartMode.POST_INITIALIZED_EVENT);

        pathChildrenCache.getListenable().addListener(new PathChildrenCacheListener() {
            @Override
            public void childEvent(CuratorFramework client, PathChildrenCacheEvent event) throws Exception {
                // 节点删除，即释放分布式锁
                if (event.getType() == PathChildrenCacheEvent.Type.CHILD_REMOVED) {
                    String path = event.getData().getPath();
                    LOGGER.info("上一个会话已经释放分布式锁或者会话已断开，节点路径为" + path);
                    if (path.contains(DISTRIBUTED_LOCK)) {
                        LOGGER.info("释放计数器，让其他等待线程来获取分布式锁...");
                        zkLockLatch.countDown();
                    }
                }
            }
        });
    }

    // 获取分布式锁
    public void getLock() {
        // 循环尝试
        while (true) {
            try {
                client.create()
                        .creatingParentsIfNeeded()
                        // 临时节点，防止客户端会话断开未释放分布式锁的情况
                        .withMode(CreateMode.EPHEMERAL)
                        .withACL(ZooDefs.Ids.OPEN_ACL_UNSAFE)
                        .forPath("/" + IMOOC_LOCK_PROJECT + "/" + DISTRIBUTED_LOCK);
                LOGGER.info(Thread.currentThread().getName() + " 获取分布式锁成功");
                // 获取锁成功，就退出循环
                return;
            } catch (Exception e) {
                // 节点已存在，表示已经有线程获取锁
                LOGGER.info(Thread.currentThread().getName() + " 获取分布式锁失败");
                try {
                    if (zkLockLatch.getCount() <= 0) {
                        zkLockLatch = new CountDownLatch(1);
                    }
                    // 阻塞等待获取锁
                    zkLockLatch.await();
                } catch (InterruptedException ex) {
                    ex.printStackTrace();
                }
            }
        }
    }

    // 释放分布式锁
    public boolean releaseLock() {
        try {
            if (client.checkExists().forPath("/" + IMOOC_LOCK_PROJECT + "/" + DISTRIBUTED_LOCK) != null) {
                client.delete().forPath("/" + IMOOC_LOCK_PROJECT + "/" + DISTRIBUTED_LOCK);
            }
        } catch (Exception e) {
            e.printStackTrace();
            return false;
        }
        LOGGER.info(Thread.currentThread().getName() + " 分布式锁释放完毕");
        return true;
    }

}
```

分布式锁工具测试：

```java
import org.apache.curator.framework.CuratorFramework;
import org.apache.curator.framework.CuratorFrameworkFactory;
import org.apache.curator.retry.ExponentialBackoffRetry;

import java.util.Random;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;
import java.util.concurrent.ThreadFactory;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicLong;

/**
 * 分布式锁测试
 *
 * @author xuanjian
 */
public class DistributedLockTest {

    private static final ExecutorService EXECUTOR_SERVICE = Executors.newFixedThreadPool(5, new ThreadFactory() {
        private AtomicLong counter = new AtomicLong(0);

        @Override
        public Thread newThread(Runnable r) {
            Thread t = new Thread(r);
            t.setName("dslock-test-" + counter.incrementAndGet());
            return t;
        }
    });
    private static Random random = new Random();
    private static DistributedLock distributedLock;

    static {
        CuratorFramework client = CuratorFrameworkFactory.builder()
                .connectString("localhost:2181")
                .sessionTimeoutMs(50000)
                .connectionTimeoutMs(50000)
                .retryPolicy(new ExponentialBackoffRetry(1000, 5))
                .build();
        client.start();

        distributedLock = new DistributedLock(client);
    }

    public static void main(String[] args) {
        // 模拟分布式场景，多个进程争用分布式锁
        int i = 0;
        while (i < 5) {
            EXECUTOR_SERVICE.submit(() -> {
                distributedLock.getLock();
                try {
                    // 操作
                    TimeUnit.SECONDS.sleep(random.nextInt(5) + 1);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                } finally {
                    distributedLock.releaseLock();
                }
            });
            i++;
        }

        EXECUTOR_SERVICE.shutdown();
        System.out.println("all done!!!");
    }

}
```

### 参考

- [ZooKeeper分布式专题与Dubbo微服务入门](https://coding.imooc.com/class/chapter/201.html#Anchor)

