# 分布式常见问题记录


<!--more-->
# 分布式幂等、分布式锁
## 概念理解：
对于相同的请求应该返回相同的结果，所以查询类接口是天然的幂等性接口。 
或者说：幂等指的是相同请求（identical request）执行一次或者多次所带来的副作 用（side-effects）是一样的。
在分布式条件下服务的设计中，如果与其他服务（或者是第三方）有复杂且密度很高的交互，且这种交互无法主动控制，那么必须使用一些策略来保证数据幂等。
## 什么常见会出现幂等？
场景1：前端调后端接口发起支付超时，然后再次发起重试。可能会导致多次支付。
场景2：页面上未做防抖进行了多次点击。 
场景3：被动订阅其他服务数据时，无法控制对方的推送，则需要在自己服务内做幂等处理。
## 总结：: 

接口的幂等性实际上就是接口可重复调用，在调用方多次调用的情况下，接口最终得到的结果是一致的 。
## 解决方案
各种各样的技术方案，总的来说都是无非就是基于语言特性（底层cpu的原子性）或者基于 CAS（Compare And Swap）的概念进行不同的处理
### 语言层面
synchronized或者java原子类 AtomicInteger， 利用CPU提供的CAS操作来保证原子性（除了AtomicInteger外，还有AtomicBoolean、AtomicLong、AtomicReference等众多原子类）
### 数据库层面
悲观锁（select …… for update）
会在数据库加上排它锁，直到事务提交或回滚时才会释放排它锁；
在此期间，如果其他线程试图更新该玩家信息或者执行select for update，会被阻塞
### 业务层面
#### 乐观锁

乐观锁即版本号机制，本身是不加锁的，存于数据库中，通过唯一标识，时间戳、唯一索引等，在操作数据时先插叙，再进行比较，从而判断数据是否应该处理

#### 分布式锁

即：SET EX|PX NX  + 校验唯一随机值
EX参数用于设置键的过期时间，单位为秒。
PX参数用于设置键的过期时间，单位为毫秒。
NX: 如果key不存在，则会将key设置为value，并返回1；如果key存在，不会有任务影响，返回0。

锁命名规范：命名空间（Namespace）：业务标识（Business Identifier）：版本号（Versioning）：节点标识（Node Identifier）：唯一标识（结合业务自定义）
eg: SJZT:USER_UPDATE:v1:NODE1（数据中台：用户更新：版本v1：节点1：自定义标识）

链接参考：
https://www.cnblogs.com/linjiqin/p/8003838.html
https://blog.csdn.net/weixin_43844718/article/details/126463959
#### 示例代码  

使用的时候很简单，spring中封装了redisson，需要注意解锁的处理

~~~
RLock   lock        = this.redissonClient.getLock(MessageTypeEnum.getRedisLockName(msgType) + dto.getId());
boolean lockSuccess = false;  
try {  
    lockSuccess = lock.tryLock(5L , TimeUnit.SECONDS);  
    if (! lockSuccess) {  
        return MessageBuilder.failed("添加用户：数据校验失败，获取锁失败！");  
    }
    // do somthing
    if (lock.isLocked() && lock.isHeldByCurrentThread()) {  
        lock.unlock();  
    }    
} catch (InterruptedException e) {  
    throw new DataException(e.getMessage());  
} finally {  
    if (lockSuccess && lock.isLocked() && lock.isHeldByCurrentThread()) {  
        lock.unlock();  
    }}
~~~

## 一些问题

### **乐观锁和悲观锁优缺点和适用场景**

乐观锁和悲观锁并没有优劣之分，它们有各自适合的场景；下面从两个方面进行说明。

功能限制 与悲观锁相比，乐观锁适用的场景受到了更多的限制，无论是CAS还是版本号机制。

例如，CAS只能保证单个变量操作的原子性，当涉及到多个变量时，CAS是无能为力的，而synchronized则可以通过对整个代码块加锁来处理。

再比如版本号机制，如果query的时候是针对表1，而update的时候是针对表2，也很难通过简单的版本号来实现乐观锁。

竞争激烈程度 如果悲观锁和乐观锁都可以使用，那么选择就要考虑竞争的激烈程度

当竞争不激烈 (出现并发冲突的概率小)时，乐观锁更有优势，因为悲观锁会锁住代码块或数据，其他线程无法同时访问，影响并发，而且加锁和释放锁都需要消耗额外的资源。

当竞争激烈(出现并发冲突的概率大)时，悲观锁更有优势，因为乐观锁在执行更新时频繁失败，需要不断重试，浪费CPU资源。

### CAS中的ABA问题

1.ABA问题 假设有两个线程——线程1和线程2，两个线程按照顺序进行以下操作：

(1)线程1读取内存中数据为A；
(2)线程2将该数据修改为B；
(3)线程2将该数据修改为A；
(4)线程1对数据进行CAS操作
在第(4)步中，由于内存中数据仍然为A，因此CAS操作成功，但实际上该数据已经被线程2修改过了。这就是ABA问题。

在AtomicInteger的例子中，ABA似乎没有什么危害。

但是在某些场景下，ABA却会带来隐患，例如栈顶问题：一个栈的栈顶经过两次(或多次)变化又恢复了原值，但是栈可能已发生了变化。

对于ABA问题，比较有效的方案是引入版本号，内存中的值每发生一次变化，版本号都+1；

在进行CAS操作时，不仅比较内存中的值，也会比较版本号，只有当二者都没有变化时，CAS才能执行成功。

Java中的AtomicStampedReference类便是使用版本号来解决ABA问题的。

2.高竞争下的开销问题 在并发冲突概率大的高竞争环境下，如果CAS一直失败，会一直重试，CPU开销较大。

针对这个问题的一个思路是引入退出机制，如重试次数超过一定阈值后失败退出。

当然，更重要的是避免在高竞争环境下使用乐观锁。

3.功能限制 CAS的功能是比较受限的，例如CAS只能保证单个变量（或者说单个内存值）操作的原子性，这意味着：

(1)原子性不一定能保证线程安全，例如在Java中需要与volatile配合来保证线程安全；

(2)当涉及到多个变量(内存值)时，CAS也无能为力。

除此之外，CAS的实现需要硬件层面处理器的支持，在Java中普通用户无法直接使用，只能借助atomic包下的原子类使用，灵活性受到限制。



# 分布式事务
这个讲的很清楚
https://www.zhihu.com/question/64921387/answer/225784480

【以下为复制】
关于分布式事务，工程领域主要讨论的是强一致性和最终一致性的解决方案。典型方案包括：

1. 两阶段提交（2PC, Two-phase Commit）方案
2. eBay [事件队列方案](https://www.zhihu.com/search?q=%E4%BA%8B%E4%BB%B6%E9%98%9F%E5%88%97%E6%96%B9%E6%A1%88&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)
3. TCC 补偿模式
4. 缓存数据最终一致性

## **一、一致性理论**

分布式事务的目的是保障分库数据一致性，而跨库事务会遇到各种不可控制的问题，如个别节点永久性宕机，像单机事务一样的ACID是无法奢望的。另外，业界著名的[CAP理论](https://www.zhihu.com/search?q=CAP%E7%90%86%E8%AE%BA&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)也告诉我们，对分布式系统，需要将数据一致性和系统可用性、分区容忍性放在天平上一起考虑。

两阶段提交协议（简称2PC）是实现分布式事务较为经典的方案，但2PC 的可扩展性很差，在分布式架构下应用代价较大，eBay [架构师](https://www.zhihu.com/search?q=%E6%9E%B6%E6%9E%84%E5%B8%88&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)Dan Pritchett 提出了BASE 理论，用于解决大规模分布式系统下的数据一致性问题。BASE 理论告诉我们：可以通过放弃系统在每个时刻的强一致性来换取系统的可扩展性。

### **1、CAP理论**

在分布式系统中，一致性（Consistency）、可用性（Availability）和[分区容忍性](https://www.zhihu.com/search?q=%E5%88%86%E5%8C%BA%E5%AE%B9%E5%BF%8D%E6%80%A7&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)（Partition Tolerance）3 个要素最多只能同时满足两个，不可兼得。其中，分区容忍性又是不可或缺的。

![](https://pic1.zhimg.com/80/v2-8bb62ba9e7ce9f35199da032e17f9bb7_720w.webp?source=2c26e567)

- 一致性：分布式环境下多个节点的数据是否强一致。
- 可用性：分布式服务能一直保证可用状态。当用户发出一个请求后，服务能在有限时间内返回结果。
- 分区容忍性：特指对[网络分区](https://www.zhihu.com/search?q=%E7%BD%91%E7%BB%9C%E5%88%86%E5%8C%BA&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)的容忍性。

举例：Cassandra、Dynamo 等，默认优先选择AP，弱化C；HBase、MongoDB 等，默认优先选择CP，弱化A。

### **2、BASE 理论**

核心思想：

- **基本可用（Basically**  
    **Available）**：指分布式系统在出现故障时，允许损失部分的可用性来保证核心可用。
- **软状态（Soft**  
    **State）**：指允许分布式系统存在中间状态，该中间状态不会影响到系统的整体可用性。
- **最终一致性（Eventual**  
    **Consistency）**：指分布式系统中的所有副本数据经过一定时间后，最终能够达到一致的状态。

## **二、一致性模型**

数据的一致性模型可以分成以下 3 类：

1. **强一致性**：数据更新成功后，任意时刻所有副本中的数据都是一致的，一般采用同步的方式实现。
2. **弱一致性**：数据更新成功后，系统不承诺立即可以读到最新写入的值，也不承诺具体多久之后可以读到。
3. **最终一致性**：弱一致性的一种形式，数据更新成功后，系统不承诺立即可以返回最新写入的值，但是保证最终会返回上一次更新操作的值。

分布式系统数据一致性模型可以通过Quorum NRW算法分析。

不少朋友向我们反馈处理企业数据一致性时成本过高、事务执行状态不透明，如果您也遇到同样的情况，欢迎了解网易数帆【分布式事务GTXS】，限时领取定制化企业方案，帮您解决数据一致性处理难题：

[](https://xg.zhihu.com/plugin/db84483f1f7f2966b6bc4b5893db3d67?BIZ=ECOMMERCE)

## **三、分布式事务解决方案**

### **1、2PC方案——强一致性**

2PC的核心原理是通过提交分阶段和记日志的方式，记录下事务提交所处的阶段状态，在组件宕机重启后，可通过日志恢复事务提交的阶段状态，并在这个状态节点重试，如Coordinator重启后，通过日志可以确定提交处于Prepare还是PrepareAll状态，若是前者，说明有节点可能没有Prepare成功，或所有节点Prepare成功但还没有下发Commit，状态恢复后给所有节点下发RollBack；若是PrepareAll状态，需要给所有节点下发Commit，数据库节点需要保证[Commit幂](https://www.zhihu.com/search?q=Commit%E5%B9%82&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)等。

![](https://picx.zhimg.com/80/v2-21a220437e3d499aeada2d20efc7e083_720w.webp?source=2c26e567)

2PC方案的问题：

1. 同步阻塞。
2. 数据不一致。
3. 单点问题。

升级的3PC方案旨在解决这些问题，主要有两个改进：

1. 增加超时机制。
2. 两阶段之间插入准备阶段。

但[三阶段提交](https://www.zhihu.com/search?q=%E4%B8%89%E9%98%B6%E6%AE%B5%E6%8F%90%E4%BA%A4&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)也存在一些缺陷，要彻底从协议层面避免数据不一致，可以采用[Paxos](https://link.zhihu.com/?target=https%3A//en.wikipedia.org/wiki/Paxos_%28computer_science%29)或者[Raft 算法](https://link.zhihu.com/?target=https%3A//raft.github.io/)。

### **2、eBay 事件队列方案——最终一致性**

eBay 的架构师Dan Pritchett，曾在一篇解释BASE 原理的论文《[Base：An Acid Alternative](https://link.zhihu.com/?target=http%3A//queue.acm.org/detail.cfm%3Fid%3D1394128)》中提到一个eBay [分布式](https://www.zhihu.com/search?q=%E5%88%86%E5%B8%83%E5%BC%8F&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)系统一致性问题的解决方案。它的核心思想是将需要分布式处理的任务通过消息或者日志的方式来异步执行，消息或日志可以存到本地文件、数据库或[消息队列](https://www.zhihu.com/search?q=%E6%B6%88%E6%81%AF%E9%98%9F%E5%88%97&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)，再通过业务规则进行失败重试，它要求各服务的接口是[幂等](https://www.zhihu.com/search?q=%E5%B9%82%E7%AD%89&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)的。

描述的场景为，有用户表user 和交易表transaction，用户表存储用户信息、总销售额和总购买额，交易表存储每一笔交易的流水号、买家信息、卖家信息和[交易金额](https://www.zhihu.com/search?q=%E4%BA%A4%E6%98%93%E9%87%91%E9%A2%9D&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)。如果产生了一笔交易，需要在交易表增加记录，同时还要修改用户表的金额。

![](https://picx.zhimg.com/80/v2-5532dabf9dad7650e892065022de30f8_720w.webp?source=2c26e567)

论文中提出的解决方法是将更新交易表记录和用户表更新消息放在一个本地事务来完成，为了避免重复消费用户表更新消息带来的问题，增加一个操作记录表updates_applied来记录已经完成的交易相关的信息。

![](https://pica.zhimg.com/80/v2-65ac9055dd96aee7be2984f6093c0933_720w.webp?source=2c26e567)

这个方案的核心在于第二阶段的重试和幂等执行。失败后重试，这是一种[补偿机制](https://www.zhihu.com/search?q=%E8%A1%A5%E5%81%BF%E6%9C%BA%E5%88%B6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)，它是能保证系统最终一致的关键流程。

### **3、TCC （Try-Confirm-Cancel）补偿模式——最终一致性**

某业务模型如图，由服务 A、服务B、服务C、服务D 共同组成的一个[微服务架构](https://www.zhihu.com/search?q=%E5%BE%AE%E6%9C%8D%E5%8A%A1%E6%9E%B6%E6%9E%84&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)系统。服务A 需要依次调用服务B、服务C 和服务D 共同完成一个操作。当服务A 调用服务D 失败时，若要保证整个系统数据的一致性，就要对服务B 和服务C 的invoke 操作进行回滚，执行反向的revert 操作。回滚成功后，整个微服务系统是数据一致的。

![](https://picx.zhimg.com/80/v2-48e3f206b2b0a8ce1bae05409651ccd8_720w.webp?source=2c26e567)

实现关键要素：

1. 服务调用链必须被记录下来。
2. 每个服务提供者都需要提供一组业务逻辑相反的操作，互为补偿，同时回滚操作要保证幂等。
3. 必须按失败原因执行不同的回滚策略。

### **4、缓存数据最终一致性**

在我们的业务系统中，缓存（Redis 或者Memcached）通常被用在数据库前面，作为数据读取的缓冲，使得I/O 操作不至于直接落在数据库上。以[商品详情页](https://www.zhihu.com/search?q=%E5%95%86%E5%93%81%E8%AF%A6%E6%83%85%E9%A1%B5&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)为例，假如卖家修改了[商品信息](https://www.zhihu.com/search?q=%E5%95%86%E5%93%81%E4%BF%A1%E6%81%AF&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)，并写回到数据库，但是这时候用户从商品详情页看到的信息还是从缓存中拿到的过时数据，这就出现了缓存系统和[数据库系统](https://www.zhihu.com/search?q=%E6%95%B0%E6%8D%AE%E5%BA%93%E7%B3%BB%E7%BB%9F&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)中的数据不一致的现象。

要解决该场景下缓存和数据库数据不一致的问题我们有以下两种解决方案：

1. 为缓存数据设置过期时间。当缓存中数据过期后，业务系统会从数据库中获取数据，并将新值放入缓存。这个过期时间就是系统可以达到最终一致的容忍时间。
2. 更新数据库数据后同时清除缓存数据。数据库数据更新后，同步删除缓存中数据，使得下次对商品详情的获取直接从数据库中获取，并同步到缓存。

> 网易数帆分布式事务，高性能、高可靠、低成本，保障数据一致性，目前已成熟运用于金融、能源等行业。欢迎体验，一起保卫企业数据安全~

[](https://xg.zhihu.com/plugin/db84483f1f7f2966b6bc4b5893db3d67?BIZ=ECOMMERCE)

## **四、选择建议**

最后，面临数据一致性问题的时候，主要从业务需求的角度出发，确定我们对于3 种一致性模型的接受程度，再通过具体场景来决定解决方案。

从应用角度看，分布式事务的现实场景常常无法规避，在有能力给出其他解决方案前，2PC也是一个不错的选择。对购物转账等电商和金融业务，[中间件](https://www.zhihu.com/search?q=%E4%B8%AD%E9%97%B4%E4%BB%B6&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)层的2PC最大问题在于业务不可见，一旦出现不可抗力或意想不到的一致性破坏，如数据节点永久性宕机，业务难以根据2PC的日志进行补偿。金融场景下，数据一致性是命根，业务需要对数据有百分之百的掌控力，建议使用TCC这类[分布式事务模型](https://www.zhihu.com/search?q=%E5%88%86%E5%B8%83%E5%BC%8F%E4%BA%8B%E5%8A%A1%E6%A8%A1%E5%9E%8B&search_source=Entity&hybrid_search_source=Entity&hybrid_search_extra=%7B%22sourceType%22%3A%22answer%22%2C%22sourceId%22%3A225784480%7D)，或基于消息队列的柔性事务框架，这两种方案都在业务层实现，业务开发者具有足够掌控力，可以结合SOA框架来架构，包括Dubbo、Spring Cloud等


# 分布式ID


唯一性：确保生成的ID是全网唯一的。 
有序递增性：确保生成的ID是对于某个用户或者业务是按一定的数字有序递增的。
高可用性：确保任何时候都能正确的生成ID。 
带时间：ID里面包含时间，一眼扫过去就知道哪天的交易。

## 1. UUID 
算法的核心思想是结合机器的网卡、当地时间、一个随记数来生成UUID。 
优点：本地生成，生成简单，性能好，没有高可用风险 
缺点：长度过长，存储冗余，且无序不可读，查询效率低 
## 2. 数据库自增ID 
使用数据库的id自增策略，如 MySQL 的 auto_increment。并且可以使用两台数据库分别设置不同 步长，生成不重复ID的策略来实现高可用。 
优点：数据库生成的ID绝对有序，高可用实现方式简单 
缺点：需要独立部署数据库实例，成本高，有性能瓶颈
## 批量生成ID 
一次按需批量生成多个ID，每次生成都需要访问数据库，将数据库修改为最大的ID值，并在内存中 记录当前值及最大值。 
优点：避免了每次生成ID都要访问数据库并带来压力，提高性能 
缺点：属于本地生成策略，存在单点故障，服务重启造成ID不连续

## Redis生成ID
 Redis的所有命令操作都是单线程的，本身提供像 incr 和 increby 这样的自增原子命令，所以能保 证生成的 ID 肯定是唯一有序的。 
 优点：不依赖于数据库，灵活方便，且性能优于数据库；数字ID天然排序，对分页或者需要排序的结果很有帮助。 
 缺点：如果系统中没有Redis，还需要引入新的组件，增加系统复杂度；需要编码和配置的工作 量比较大。 
 考虑到单节点的性能瓶颈，可以使用 Redis 集群来获取更高的吞吐量。假如一个集群中有5台 Redis。可以初始化每台 Redis 的值分别是1, 2, 3, 4, 5，然后步长都是 5

Redis 的自增功能，通过 INCR 命令来实现。你可以为每个需要生成唯一 ID 的实体创建一个自增的计数器。
例如，对于用户，你可以创建一个键为 "user:id" 的计数器，每次需要生成新用户的 ID 时，可以使用 INCR 命令获取下一个唯一 ID。
~~~
127.0.0.1:6379> SET user:id 1000   # 设置初始 ID
OK
127.0.0.1:6379> INCR user:id       # 生成下一个唯一 ID
(integer) 1001
127.0.0.1:6379> INCR user:id       # 再次生成下一个唯一 ID
(integer) 1002
~~~

## 雪花算法 Snowflake 
Snowflake 算法由下面几部分组成：
1位符号位： 由于 long 类型在 java 中带符号的，最高位为符号位，正数为 0，负数为 1，且实际系统中所使用 的ID一般都是正数，所以最高位为 0。
41位时间戳（毫秒级）： 需要注意的是此处的 41 位时间戳并非存储当前时间的时间戳，而是存储时间戳的差值（当前时间 戳 - 起始时间戳），这里的起始时间戳一般是ID生成器开始使用的时间戳，由程序来指定，所以41 位毫秒时间戳最多可以使用 (1 << 41) / (1000x60x60x24x365) = 69年 。
10位数据机器位： 包括5位数据标识位和5位机器标识位，这10位决定了分布式系统中最多可以部署 1 << 10 = 1024 个节点。超过这个数量，生成的ID就有可能会冲突。
12位毫秒内的序列： 这 12 位计数支持每个节点每毫秒（同一台机器，同一时刻）最多生成 1 << 12 = 4096个ID 加起来刚好64位，为一个Long型。 
优点：高性能，低延迟，按时间有序，一般不会造成ID碰撞 
缺点：需要独立的开发和部署，依赖于机器的时钟

# 限流和熔断

参考博客
https://zhuanlan.zhihu.com/p/391779142
这个示例很清楚
https://blog.csdn.net/qq_38322527/article/details/106253143
官方文档
https://github.com/alibaba/Sentinel/wiki/%E4%BB%8B%E7%BB%8D


【以下为复制】
当系统的处理能力不能应对外部请求的突增流量时，为了不让系统崩溃，必须采取限流的措施。


在分布式系统中，如果某个服务节点发生故障或者网络发生异常，都有可能导致调用方被阻塞等待，如果超时时间设置很长，调用方资源很可能被耗尽。这又导致了调用方的上游系统发生资源耗尽的情况，最终导致系统雪崩。

如下图：

  

![](https://pic1.zhimg.com/80/v2-3b78261dea18954c52eb0a7b70fe3b64_720w.webp)

  

如果D服务发生了故障不能响应，B服务调用D时只能阻塞等待。假如B服务调用D服务设置超时时间是10秒，请求速率是每秒100个，那10秒内就会有1000个请求线程被阻塞等待，如果B的线程池大小设置1000，那B系统因为线程资源耗尽已经不能对外提供服务了。而这又影响了入口系统A的服务，最终导致系统全面崩溃。

提高系统的整体容错能力是防止系统雪崩的有效手段。

在Martin Fowler和James Lewis的文章 《Microservices: a definition of this new architectural term》[1]中，提出了微服务的9个特征，其中一个是容错设计。

要防止系统发生雪崩，就必须要有容错设计。如果遇到突增流量，一般的做法是对非核心业务功能采用熔断和服务降级的措施来保护核心业务功能正常服务，而对于核心功能服务，则需要采用限流的措施。

今天我们来聊一聊系统容错中的限流、熔断和服务降级。

## 1 限流

当系统的处理能力不能应对外部请求的突增流量时，为了不让系统崩溃，必须采取限流的措施。

## 1.1 限流指标

## 1.1.1 TPS

系统吞吐量是衡量系统性能的关键指标，按照事务的完成数量来限流是最合理的。

但是对实操性来说，按照事务来限流并不现实。在分布式系统中完成一笔事务需要多个系统的配合。比如我们在电商系统购物，需要订单、库存、账户、支付等多个服务配合完成，有的服务需要异步返回，这样完成一笔事务花费的时间可能会很长。如果按照TPS来进行限流，时间粒度可能会很大大，很难准确评估系统的响应性能。

## 1.1.2 HPS

每秒请求数，指每秒钟服务端收到客户端的请求数量。

> 如果一个请求完成一笔事务，那TPS和HPS是等同的。但在分布式场景下，完成一笔事务可能需要多次请求，所以TPS和HPS指标不能等同看待。

## 1.1.3 QPS

服务端每秒能够响应的客户端查询请求数量。

> 如果后台只有一台服务器，那HPS和QPS是等同的。但是在分布式场景下，每个请求需要多个服务器配合完成响应。  
> 目前主流的限流方法多采用HPS作为限流指标。

## 1.2 限流方法

## 1.2.1 流量计数器

这是最简单直接的方法，比如限制每秒请求数量100，超过100的请求就拒绝掉。

但是这个方法存在2个明显的问题：

- 单位时间(比如1s)很难把控，如下图：这张图上，从下面时间看，HPS没有超过100，但是从上面看HPS超过100了。
- 有一段时间流量超了，也不一定真的需要限流，如下图，系统HPS限制50，虽然前3s流量超了，但是如果都超时时间设置为5s，并不需要限流。

## 1.2.2 滑动时间窗口

滑动时间窗口算法是目前比较流行的限流算法，主要思想是把时间看做是一个向前滚动的窗口，如下图：

  

![](https://pic2.zhimg.com/80/v2-51a1f8b92f3db584c38d60bf9648b601_720w.webp)

  

开始的时候，我们把t1~t5看做一个时间窗口，每个窗口1s，如果我们定的限流目标是每秒50个请求，那t1~t5这个窗口的请求总和不能超过250个。

这个窗口是滑动的，下一秒的窗口成了t2~t6，这时把t1时间片的统计抛弃，加入t6时间片进行统计。这段时间内的请求数量也不能超过250个。

滑动时间窗口的优点是解决了流量计数器算法的缺陷，但是也有2个问题：

- 流量超过就必须抛弃或者走降级逻辑
- 对流量控制不够精细，不能限制集中在短时间内的流量，也不能削峰填谷

## 1.2.3 漏桶算法

漏桶算法的思想如下图：

  

![](https://pic3.zhimg.com/80/v2-b4247d0be521a4b10e5c2b19868d017a_720w.webp)

  

在客户端的请求发送到服务器之前，先用漏桶缓存起来，这个漏桶可以是一个长度固定的队列，这个队列中的请求均匀地发送到服务端。

如果客户端的请求速率太快，漏桶的队列满了，就会被拒绝掉，或者走降级处理逻辑。这样服务端就不会受到突发流量的冲击。

漏桶算法的优点是实现简单，可以使用消息队列来削峰填谷。

但是也有3个问题需要考虑:

- 漏桶的大小，如果太大，可能给服务端带来较大处理压力，太小可能会有大量请求被丢弃。
- 漏桶给服务端的请求发送速率。
- 使用缓存请求的方式，会使请求响应时间变长。

> **❝**  
> 漏桶大小和发送速率这2个值在项目上线初期都会根据测试结果选择一个值，但是随着架构的改进和集群的伸缩，这2个值也会随之发生改变。  
> ❞

## 1.2.4 令牌桶算法

令牌桶算法就跟病人去医院看病一样，找医生之前需要先挂号，而医院每天放的号是有限的。当天的号用完了，第二天又会放一批号。

算法的基本思想就是周期性地执行下面的流程：

  

![](https://pic1.zhimg.com/80/v2-5f6c104e85dd9dbca800822b44f75c2c_720w.webp)

  

客户端在发送请求时，都需要先从令牌桶中获取令牌，如果取到了，就可以把请求发送给服务端，取不到令牌，就只能被拒绝或者走服务降级的逻辑。如下图：

  

![](https://pic2.zhimg.com/80/v2-1428778f3a233aded15e37b59d7e2a41_720w.webp)

  

> **❝**  
> 令牌桶算法解决了漏桶算法的问题，而且实现并不复杂，使用信号量就可以实现。在实际限流场景中使用最多，比如google的guava中就实现了令牌桶算法限流，感兴趣可以研究一下。  
> ❞

## 1.2.5 分布式限流

如果在分布式系统场景下，上面介绍的4种限流算法是否还适用呢？

以令牌桶算法为例，假如在电商系统中客户下了一笔订单，如下图：

  

![](https://pic2.zhimg.com/80/v2-ac290d69df157f083da16493e27a00e1_720w.webp)

  

如果我们把令牌桶单独保存在一个地方(比如redis中)供整个分布式系统用，那客户端在调用组合服务，组合服务调用订单、库存和账户服务都需要跟令牌桶交互，交互次数明显增加了很多。

有一种改进就是客户端调用组合服务之前首先获取四个令牌，调用组合服务时减去一个令牌并且传递给组合服务三个令牌，组合服务调用下面三个服务时依次消耗一个令牌。

## 1.2.6 hystrix限流

hystrix可以使用信号量和线程池来进行限流。

## 1.2.6.1 信号量限流

hystrix可以使用信号量进行限流，比如在提供服务的方法上加下面的注解。这样只能有20个并发线程来访问这个方法，超过的就被转到了errMethod这个降级方法。

```text
@HystrixCommand(
 commandProperties= {
   @HystrixProperty(name="execution.isolation.strategy", value="SEMAPHORE"),
   @HystrixProperty(name="execution.isolation.semaphore.maxConcurrentRequests", value="20")
 },
 fallbackMethod = "errMethod"
)
```

## 1.2.6.2 线程池限流

hystrix也可以使用线程池进行限流，在提供服务的方法上加下面的注解，当线程数量

```text
@HystrixCommand(
    commandProperties = {
            @HystrixProperty(name = "execution.isolation.strategy", value = "THREAD")
    },
    threadPoolKey = "createOrderThreadPool",
    threadPoolProperties = {
            @HystrixProperty(name = "coreSize", value = "20"),
   @HystrixProperty(name = "maxQueueSize", value = "100"),
            @HystrixProperty(name = "maximumSize", value = "30"),
            @HystrixProperty(name = "queueSizeRejectionThreshold", value = "120")
    },
    fallbackMethod = "errMethod"
)
```

> **❝**  
> 这里要注意：在java的线程池中，如果线程数量超过coreSize，创建线程请求会优先进入队列，如果队列满了，就会继续创建线程直到线程数量达到maximumSize，之后走拒绝策略。但在hystrix配置的线程池中多了一个参数queueSizeRejectionThreshold，如果queueSizeRejectionThreshold < maxQueueSize,队列数量达到queueSizeRejectionThreshold就会走拒绝策略了，因此maximumSize失效了。如果queueSizeRejectionThreshold > maxQueueSize,队列数量达到maxQueueSize时，maximumSize是有效的，系统会继续创建线程直到数量达到maximumSize。Hytrix线程池设置坑[2]  
> ❞

# 熔断

相信大家对断路器并不陌生，它就相当于一个开关，打开后可以阻止流量通过。比如保险丝，当电流过大时，就会熔断，从而避免元器件损坏。

服务熔断是指调用方访问服务时通过断路器做代理进行访问，断路器会持续观察服务返回的成功、失败的状态，当失败超过设置的阈值时断路器打开，请求就不能真正地访问到服务了。

为了更好地理解，我画了下面的时序图：

  

![](https://pic1.zhimg.com/80/v2-5d8ce80db03f9995c5ef4a28266caed8_720w.webp)

  

## 2.1 断路器的状态

断路器有3种状态：

- CLOSED：默认状态。断路器观察到请求失败比例没有达到阈值，断路器认为被代理服务状态良好。
- OPEN：断路器观察到请求失败比例已经达到阈值，断路器认为被代理服务故障，打开开关，请求不再到达被代理的服务，而是快速失败。
- HALF OPEN：断路器打开后，为了能自动恢复对被代理服务的访问，会切换到半开放状态，去尝试请求被代理服务以查看服务是否已经故障恢复。如果成功，会转成CLOSED状态，否则转到OPEN状态。

断路器的状态切换图如下：

  

![](https://pic4.zhimg.com/80/v2-1442e6c8cde54f73119c4667f5ea4607_720w.webp)

  

## 2.2 需要考虑的问题

使用断路器需要考虑一些问题：

- 针对不同的异常，定义不同的熔断后处理逻辑。
- 设置熔断的时长，超过这个时长后切换到HALF OPEN进行重试。
- 记录请求失败日志，供监控使用。
- 主动重试，比如对于connection timeout造成的熔断，可以用异步线程进行网络检测，比如telenet，检测到网络畅通时切换到HALF OPEN进行重试。
- 补偿接口，断路器可以提供补偿接口让运维人员手工关闭。
- 重试时，可以使用之前失败的请求进行重试，但一定要注意业务上是否允许这样做。

## 2.3 使用场景

- 服务故障或者升级时，让客户端快速失败
- 失败处理逻辑容易定义
- 响应耗时较长，客户端设置的read timeout会比较长，防止客户端大量重试请求导致的连接、线程资源不能释放

## 3 服务降级

前面讲了限流和熔断，相比来说，服务降级是站在系统全局的视角来考虑的。

在服务发生熔断后，一般会让请求走事先配置的处理方法，这个处理方法就是一个降级逻辑。

服务降级是对非核心、非关键的服务进行降级。

## 3.1 使用场景

- 服务处理异常，把异常信息直接反馈给客户端，不再走其他逻辑
- 服务处理异常，把请求缓存下来，给客户端返回一个中间态，事后再重试缓存的请求
- 监控系统检测到突增流量，为了避免非核心业务功能耗费系统资源，关闭这些非核心功能
- 数据库请求压力大，可以考虑返回缓存中的数据
- 对于耗时的写操作，可以改为异步写
- 暂时关闭跑批任务，以节省系统资源

## 3.2 使用hystrix降级

## 3.2.1 异常降级

hystrix降级时可以忽略某个异常，在方法上加上@HystrixCommand注解：

下面的代码定义降级方法是errMethod，对ParamErrorException和BusinessTypeException这两个异常不做降级处理。

```text
@HystrixCommand(
 fallbackMethod = "errMethod",
 ignoreExceptions = {ParamErrorException.class, BusinessTypeException.class}
)
```

## 3.2.2 调用超时降级

专门针对调用第三方接口超时降级。

下面的方法是调用第三方接口3秒未收到响应就降级到errMethod方法。

```text
@HystrixCommand(
    commandProperties = {
            @HystrixProperty(name="execution.timeout.enabled", value="true"),
            @HystrixProperty(name="execution.isolation.thread.timeoutInMilliseconds", value="3000"),
    },
    fallbackMethod = "errMethod"
)
```

## 总结

限流、熔断和服务降级是系统容错的重要设计模式，从一定意义上讲限流和熔断也是一种服务降级的手段。

熔断和服务降级主要是针对非核心业务功能，而核心业务如果流程超过预估的峰值，就需要进行限流。

对于限流，选择合理的限流算法很重要，令牌桶算法优势很明显，也是使用最多的限流算法。

在系统设计的时候，这些模式需要配合业务量的预估、性能测试的数据进行相应阈值的配置，而这些阈值最好保存在配置中心，方便实时修改。

# 
