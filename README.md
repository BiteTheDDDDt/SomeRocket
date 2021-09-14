### TODOLIST
* DiskStorage实现类kafka的page分割
* PMEM cache方案设计

### 参考资料
* llpl库资料 - https://github.com/pmem/llpl
* JDK核心JAVA源码解析（5） - JAVA File MMAP原理解析 - https://zhuanlan.zhihu.com/p/258934554
* Linux中的Page Cache [一] - https://zhuanlan.zhihu.com/p/68071761
* Kafka存储模型 - https://blog.csdn.net/FlyingAngelet/article/details/84761466

### 资料总结

* pmem一次读写块大小256byte时读写性能较好。
* pmem写入线程数为8时可以达到最大性能。

### 当前存储结构

##### 内存(8G)
* TopicQueueIdMap/StorageEngine 等内存对象
* 系统级的cache
##### PMEM(126G)

##### SSD(400G)
* 冷数据
  - binary data
  - binary offset

### 提交方式
```
git remote add tianchi git@code.aliyun.com:952130278/mq-sample.git
git fetch tianchi
git checkout tianchi/master
git rebase origin/main
git push -f
```


>写在前面: 
> 1.在开始coding前请仔细阅读以下内容

## 1. 赛题描述
Apache RocketMQ作为的一款分布式的消息中间件，历年双十一承载了万亿级的消息流转，为业务方提供高性能低延迟的稳定可靠的消息服务。其中，实时读取写入数据和读取历史数据都是业务常见的存储访问场景，而且会在同一时刻同时出现，因此针对这个混合读写场景进行优化，可以极大的提升存储系统的稳定性。同时英特尔® 傲腾™ 持久内存作为一款与众不同的独立存储设备，可以缩小传统内存与存储之间的差距，有望给RocketMQ的性能再次飞跃提供一个支点。
## 2 题目内容
  实现一个单机存储引擎，提供以下接口来模拟消息的存储场景：
  
  写接口

    - long append(String topic, int queueId, ByteBuffer data)
    - 写不定长的消息至某个 topic 下的某个队列，要求返回的offset必须有序

    例子：按如下顺序写消息

      {"topic":a,"queueId":1001, "data":2021}
      {"topic":b,"queueId":1001, "data":2021}
      {"topic":a,"queueId":1000, "data":2021}
      {"topic":b,"queueId":1001, "data":2021}

    返回结果如下：

      0
      0
      0
      1

  读接口

    -Map<Integer, ByteBuffer> getRange(String topic, int queueId, long offset, int fetchNum)
    -返回的 Map<offset, bytebuffer> 中 offset 为消息在 Map 中的顺序偏移，从0开始
      
    例子：
      
      getRange(a, 1000, 1, 2)
      Map{}

      getRange(b, 1001, 0, 2)
      Map{"0": "2021", "1": "2021"}

      getRange(b, 1001, 1, 2)
      Map{"0": "2021"}

  评测环境中提供128G的傲腾持久内存，鼓励选手用其提高性能。

## 3 语言限定
JAVA


## 4.  程序目标

仔细阅读demo项目中的MessageStore，DefaultMessageStoreImpl两个类。

你的coding目标是实现DefaultMessageStoreImpl

注：
日志请直接打印在控制台标准输出。评测程序会把控制台标准输出的内容搜集出来，供用户排错，但是请不要密集打印日志，单次评测，最多不能超过100M，超过会截断

## 5. 测试环境描述

1、    4核8G规格ECS，配置400G的ESSD PL1云盘（吞吐可达到350MiB/s），配置126G傲腾™持久内存。

2、    限制Java语言级别到Java 8。

3、    允许使用 llpl 库，预置到环境中，选手以provided方式引入。

4、    允许使用 log4j2 日志库，预置到环境中，选手以provided方式引入。

5、    不允许使用JNI和其它三方库。

## 6. 性能评测
评测程序会创建20~40个线程，每个线程随机若干个topic（topic总数<=100），每个topic有N个queueId（1 <= N <= 10,000），持续调用append接口进行写入；评测保证线程之间数据量大小相近（topic之间不保证），每个data的大小为100B-17KiB区间随机（伪随机数程度的随机），数据几乎不可压缩，需要写入总共150GiB的数据量。

保持刚才的写入压力，随机挑选50%的队列从当前最大点位开始读取，剩下的队列均从最小点位开始读取（即从头开始），再写入总共100GiB后停止全部写入，读取持续到没有数据，然后停止。

评测考察维度包括整个过程的耗时。超时为1800秒，超过则强制退出。

最终排行开始时会更换评测数据。

## 7. 正确性评测
写入若干条数据。

**重启ESC，并清空傲腾盘上的数据**。

再读出来，必须严格等于之前写入的数据。

## 8. 排名和评测规则
通过正确性评测的前提下，按照性能评测耗时，由耗时少到耗时多来排名。

我们会对排名靠前的代码进行review，如果发现大量拷贝别人的代码，将酌情扣减名次。

所有消息都应该进行按实际发送的信息进行存储，不可以压缩，不能伪造。

程序不能针对数据规律进行针对性优化, 所有优化必须符合随机数据的通用性。

如果发现有作弊行为，比如hack评测程序，绕过了必须的评测逻辑，则程序无效，且取消参赛资格。

## 9. 参赛方法
在阿里天池找到"云原生编程挑战赛"，并报名参加。

在code.aliyun.com注册一个账号，新建一个仓库，将大赛官方账号cloudnative-contest 添加为项目成员，权限为Reporter 。

fork或者拷贝mqrace2021仓库的代码到自己的仓库，并实现自己的逻辑；注意实现的代码必须放在指定package下，否则不会被包括在评测中。

在天池提交成绩的入口，提交自己的仓库git地址，等待评测结果。

及时保存日志文件，日志文件只保存1天。
