# 项目背景

```
项目名称：      WGO 运输管理系统（TMS / TMS 3.0，Wilmar Global Operations）

项目背景：      自研集团型货主型 TMS，基于 Spring Cloud 微服务架构，
              能力中台+业务中台(分为不同的业务、干线、城配、船运、集装箱运输、外贸运输)、
              前端有PC、H5、小程序等

主要用户：

- 物流运营（计划员/调度员） 
- 财务（费用审核/对账） 
- 客户（货主/收货方工厂） 
- 司机
- 承运商（外部承运方）     
- 集团管理员（系统/权限/AI配置）

核心价值：
WGO（TMS）的核心价值是"以中台化架构解决集团型货主的多式联运协同难题"。把分散在各业务线的数据、单据、流程、决策统一起来，升级为"数字化全链路运输管理"。
```

 100~150 字的项目介绍。

```
丰行（TMS）是一套面向集团型货主（益海嘉里粮油）深度定制的多式联运协同平台，基于 Spring Cloud 微服务架构，实现全链路运输管理。系统通过 能力中台+业务中台，集成 SAP、JT808 车联网、高德地图、G7 等外部系统，覆盖干线、城配、船运、集装箱运输、外贸运输等全流程，服务物流运营、司机、承运商、财务、客户等多角色协同。
```



# 业务流程集装箱内贸

```text
客户下单SAP
    ↓
集运计划单据、信息预录入
    ↓
集运工作单据-初始化状态
    ↓
根据线路选择不同报价的线路进行运输线路确认
    ↓
运输线基于线路参数等进行拆分多段运输、例如 铁路、海运、干线运输单据等
    ↓
分配各个分段线路司机车辆信息
    ↓
首段司机全部装货、工作单装货完成
    ↓
首段司机全部卸货、工作单到港口重箱集港
    ↓
定时任务扫描第三方海运航程接口根据航程信息处理一程船上传、一程船船到、最终船到等事件
    ↓
末端司机在到达港进行装货到达最终目的地触发卸货回调的时候整个工作进入送货完成状态
    ↓
费用计算
    ↓
客户对账
    ↓
承运商对账
    ↓
结算
```



# 数据库高级设计

重点确认有没有：

## 唯一索引

用于：

```text
防止重复订单
防止重复费用
防止重复结算
防止重复导入
```

---

## 乐观锁

例如：

```text
lockVersion
```

搞清楚用于什么业务。

# 十一、Redis

不要只记“项目用了 Redis”。

需要搞清楚 Redis 用在哪。

例如：

```text
Token

用户信息

菜单权限

基础数据缓存

字典缓存

车辆状态

分布式锁

接口幂等

验证码

限流
```

每个地方回答：

```text
为什么使用 Redis？

Key 怎么设计？

过期时间？

怎么更新？

怎么删除？

缓存和数据库不一致怎么办？
```

重点研究：

```text
缓存穿透
缓存击穿
缓存雪崩
```





## 基础数据缓存

```
字典采用hash结构
固定key是redisDataDict
feied是字典的type
value是字典type对应值json数组内容
TTL永不过期（Hash 整体不设 expire）
```



## 数据库redis同步机制

这是这个项目最精巧的部分，没有任何 MQ/Canal 监听 binlog，完全靠"应用层主动维护 + 定时兜底"。

触发点 1：应用启动时全量加载

```
@Component
public class ApplicationStartedEventListener implements ApplicationListener<ApplicationStartedEvent> {
    @Override
    public void onApplicationEvent(ApplicationStartedEvent event) {
       DictDataUtil.startSchedule();  // 启动字典 JVM 内存定时器
        SecurityKeyDataUtils.reloadPasswordKey();
        securityKeyDataService.reloadPasswordKey();
        dictDataCacheService.cacheDictParamAll();                      // 字典 → Redis
        controlParamCacheService.cacheControlParamAll();               // 参数 → Redis
        btnPermissionsSessionCacheService.cacheAllBtnPermissions();// 全局按钮权限 → Redis
        interfaceParameterCacheService.cacheInterfaceParameterAll();// 接口参数 → Redis
    }
}
```

触发点 2：写后主动刷新（最关键）

```
public DictData save(DictDataItem dictDataItem) {
    ... saveOrUpdate(DictData.class, dictDataItem);   // 写 DB
    dictDataCacheService.cacheDictDataByKey(dictDataItem.getType());  // 写后只刷该 type
    return dictData;
}

public boolean deleteById(Object[] ids) {
    dao.delete(DictData.class, ids);
    dictDataCacheService.cacheDictDataAll();          // 删除则全量刷新
    return true;
}
```





# 十二、分布式锁

TMS 非常可能出现。

例如：

```text
同一车辆不能同时派给两个运输任务
```

场景：

```text
调度员A → 派车

调度员B → 同时派同一辆车
```

确认项目怎么解决：

```text
Redis SETNX

Redisson

数据库锁

唯一约束
```

如果使用 Redisson：

搞清楚：

```text
锁Key

锁粒度

锁超时时间

watchdog

释放锁

tryLock
```

---

# 十三、MQ

确认：

```text
Kafka
RabbitMQ
RocketMQ
其他
```

重点列出消息使用场景。

例如：

```text
订单创建
↓
异步创建运输任务

签收完成
↓
生成费用

运输状态变化
↓
通知客户

轨迹数据
↓
异步存储
```

每个消息搞清楚：

```text
Producer是谁：

Consumer是谁：

为什么不用同步调用：

消息体是什么：

失败怎么办：

是否重试：

是否有死信队列：

如何防止重复消费：

如何保证幂等：
```

MQ 是简历重点。







# 与外部系统对接事务一致性问题设计



寻找一个典型场景。

例如：

```text
运输订单确认
    ↓
创建运输任务
    ↓
锁定车辆
    ↓
生成运单
```

如果第二步成功，第三步失败怎么办？

确认项目使用：

```text
本地事务

Seata

MQ最终一致性

补偿任务

定时任务

人工处理
```

一定要找到至少一个真实案例。









```

这个合单接口其实没有用分布式事务框架，而是靠**「多层兜底保证最终一致性」**：
1. 主流程：先在本地用 Redisson 分布式锁 + 数据库事务完成合单业务，再同步调用 OMS 接口，最后根据 OMS 返回结果更新提单的「同步状态」（成功/失败）。本地库和 OMS 之间没有原子绑定，存在「OMS调成功但本地状态没改」或反之的短暂不一致。
2. 异步重试：定时器定期扫 task_asynchronous 表里「同步失败/异常」的数据，重新发到 task-asyncStatus 队列，由 MQ 消费者再调一次合单接口；另一条定时器链路则直接同步重试不经过 MQ。
3. 状态机防护：提单同步状态有 同步中/成功/失败/异常 四态，重试时过滤掉「同步中」和「成功」的数据，防止重复调用；同时定时器会把超时卡在「同步中」的数据强制改为「异常」，确保不会被无限挂起。
4. 反向回调：OMS→SAP 链路完成后，OMS 会回调 handleOmsSyncSapResult 接口更新最终 SAP 同步状态，并用 docId 校验防止错乱。
5. 兜底入口：管理后台提供 updateSynchronizeFail

```

**核心思路**：放弃强一致性（XA/Seata），用 **Redis锁防并发 + MQ 异步解耦 + 消息任务表补偿** 三件套实现最终一致性。这是物流系统典型做法 ——业务强约束（如"同提单不被并发改"）用锁保证；跨系统数据同步（如 SAP 回写）允许短暂不一致，由补偿机制兜底。



本地日志表实现分布式事务

```
提单合单、加单、减单、回撤操作，不要求操作的时候，需要收到外部接口的信息，所以采用本地事务+分布式锁+MQ实现。

本地进行运单号生成数据保存，记录mq消息表当前操作待处理、处理中、成功、失败、本地事务结束后发MQ、

MQ内消费消息进行外部系统接口调用，调用成功更新消息表状态。 数据一致性保证，如果处理失败、会有定时任务定时触发补偿、记录重试次数、上次重试时间，失败原因等。后续定时任务扫描到了又会发起尝试，直到重试多次都无法成功后，进行人为排查
```





# 千万级数据报表导出性能与稳定性设计

```
我们有一个数据看板的报表导出功能，数据量比较大的时候会导出几十万甚至更多数据。最开始为了避免 HTTP 请求长时间阻塞，我设计了任务表加 MQ 的异步导出方案，前端提交导出任务以后，由消费者异步生成文件。

这个方案一开始效果还可以，但是随着业务数据增长，导出越来越慢，而且测试环境模拟大数据量时出现过 OOM。后来分析发现，虽然任务异步化了，但是消费者内部还是一次性查询和组装大量数据，所以异步只是解决了请求阻塞问题，并没有解决内存问题。

第二个版本我把全量查询改成了分页分批查询，每次查固定数量的数据，处理完一批再查下一批。这样内存占用稳定下来了，OOM 问题基本解决。

但是上线之后，通过慢 SQL 日志发现一个新的问题：导出到后面几页的时候 SQL 越来越慢。比如前面可能几十、100、200毫秒，越往后可能几秒、5秒、6秒甚至更高。

后来分析发现是 MySQL 的深分页问题。类似：

SELECT *
FROM xxx
WHERE ...
LIMIT 1000000, 1000;

MySQL 并不是直接跳到第 100 万条，而是需要扫描前面的数据，再把前 offset 条丢弃。offset 越大，扫描量越大。如果查询字段不能被索引覆盖，还可能带来大量回表，所以后面的分页会越来越慢。

当时也评估过延迟关联方案，比如先通过覆盖索引分页查询主键：
SELECT id
FROM xxx
WHERE ...
LIMIT 1000000, 1000;

再根据这批 id 查询完整数据。这个方案可以减少回表次数，但它并没有解决 offset 本身仍然要扫描大量索引记录的问题。

最终我们改成了游标分页。第一次确定查询范围，然后每一批记录上一批最后一条数据的主键：

SELECT *
FROM xxx
WHERE ...
  AND id > #{lastId}
ORDER BY id
LIMIT 1000;

第一批从最小 id 开始，后续直接使用上一批最后一个 id。这样 MySQL 可以利用 B+Tree 索引直接定位下一批数据，不需要扫描和丢弃前面的几十万、几百万条记录。

最终解决了三个问题：异步化解决接口阻塞，分批查询解决 OOM，游标分页解决深分页慢 SQL。上线之后后段分页耗时不再随着页码增加明显上升，也没有再出现相关异常和投诉。

性能效果： 在千万级报表导出场景下，原深分页方案后段单批查询耗时由百毫秒级逐步恶化至5s及以上；改造为游标分页后，单批查询耗时稳定在 100～300ms，后段分页性能提升约一个数量级，整体导出耗时由 20+ 分钟降低至 5～8 分钟，并消除 OOM 问题。
```



# 高并发接口架构设计

```
tomcat连接池不够用
接口处理的太慢


```





# 多级缓存系统架构设计

```
TMS 中针对字典、系统参数、行政区域、运输地、港口、承运商等高频读取、低频更新的基础数据设计缓存机制，减少数据库重复查询，提升创建单据、基础信息查询、及各类数据看板检索数据的性能，并通过缓存失效和消息通知机制保证配置变更后的数据一致性。

基础配置与运价计算使用 Caffeine + Redis 两级缓存。应用优先访问本地缓存，未命中访问 Redis，Redis 未命中再访问 MySQL；配置发生变更后通过 MQ/Redis PubSub 通知各实例删除本地缓存，从而降低数据库和 Redis 压力，同时解决多实例本地缓存一致性问题。

缓存kv设计

字典Hash结构 key filed value
dict:transport_type
    ROAD -> 公路
    SEA  -> 海运
    
系统参数等参数类的也用hash结构

港口、承运商、运输地等基础信息用k、v结构，k存id或者编码，value存json格式

```



````
TMS 系统中的字典、系统参数、行政区域、运输地、港口、承运商、运价配置等数据具有高频读取、低频更新的特点。初期直接查询 MySQL，随着业务量增长，出现重复 SQL 多、数据库压力大、接口响应变慢的问题。

因此先引入 Redis 缓存：字典、系统参数使用 Hash，港口、承运商、运输地等对象采用 String + JSON 存储，降低数据库访问压力。

后续由于热点数据频繁访问 Redis，进一步引入 Caffeine + Redis 两级缓存：

```text
Caffeine -> Redis -> MySQL
```

优先读取 JVM 本地缓存，本地未命中再访问 Redis，从而减少 Redis 网络调用和 QPS。

多实例部署后又出现本地缓存数据不一致问题，例如配置修改后，部分实例仍读取旧数据。最终采用 Cache Aside 模式：

```text
更新 MySQL
    ↓
删除 Redis 缓存
    ↓
MQ / Redis PubSub 广播失效消息
    ↓
各实例删除 Caffeine 本地缓存
```

同时通过 TTL、消息重试和热点 Key 防击穿机制进行兜底。

最终方案既降低了 MySQL 和 Redis 的访问压力，又解决了多实例环境下缓存数据不一致的问题，提升了单据创建、运价计算、基础数据查询和数据看板等场景的性能与稳定性。

````





````
### 项目背景

TMS 系统中存在大量高频读取、低频更新的基础数据，包括字典、系统参数、行政区域、运输地、港口、承运商、运价基础配置等。

这些数据会被创建单据、运单查询、运价计算、运输路由、数据看板等多个业务模块频繁访问。随着系统业务量增长，如果每次请求都直接查询 MySQL，会产生大量重复 SQL，增加数据库连接和 IO 压力，并进一步影响核心业务接口响应时间。

因此针对该类基础数据设计缓存体系，在降低数据库压力的同时，需要重点解决缓存穿透、热点数据访问以及多实例部署下的数据一致性问题。

---

### 一、项目初期问题

系统初期主要依赖 MySQL 查询基础配置数据。

例如创建一张运输单据时，需要依次获取：

* 字典配置
* 系统基础参数
* 行政区域
* 起运地、目的地
* 港口信息
* 承运商信息
* 运价及相关配置

单次查询耗时并不明显，但在高并发场景下，大量请求会反复访问相同基础数据。

例如：

```text
创建单据
  ↓
查询行政区域
  ↓
查询运输地
  ↓
查询承运商
  ↓
查询系统参数
  ↓
查询运价配置
  ↓
MySQL
```

随着请求量增加，出现以下问题：

1. 基础数据 SQL 重复执行，数据库 QPS 持续升高。
2. 大量简单查询占用了数据库连接池资源。
3. 运价计算等复杂业务需要关联多个配置表，接口响应时间逐渐增加。
4. 数据看板和批量业务查询进一步放大数据库压力。
5. 部分热点基础数据在高峰期形成集中访问。

因此首先引入 Redis 作为分布式缓存。

---

### 二、第一阶段：Redis 单级缓存

针对不同类型的数据，根据业务查询特征设计不同 Redis 数据结构。

#### 1. 字典数据

字典天然属于一组 Key-Value 数据，因此采用 Hash：

```text
dict:transport_type

ROAD -> 公路运输
SEA  -> 海运
AIR  -> 空运
```

通过：

```text
HGET dict:transport_type ROAD
```

即可快速获取单个字典值。

---

#### 2. 系统参数

系统配置同样采用 Hash：

```text
system:config

default_currency -> CNY
auto_dispatch    -> true
track_timeout    -> 30
```

对于不同业务模块，也可进一步拆分：

```text
system:config:order
system:config:freight
system:config:transport
```

避免单个 Hash 过大。

---

#### 3. 港口、承运商、运输地等基础对象

这类数据通常通过 ID 或业务编码查询，因此采用 String + JSON：

```text
port:CNSHA
    -> {港口JSON}

carrier:10001
    -> {承运商JSON}

transport_place:20001
    -> {运输地JSON}
```

这样可以实现：

* 单对象独立缓存
* 独立 TTL
* 单对象独立更新
* 避免巨大 Hash Key
* Java 对象直接 JSON 序列化 / 反序列化

---

### 三、Redis 单级缓存遇到的新问题

引入 Redis 后，MySQL 查询压力明显下降，但随着系统访问量进一步提升，又出现新的性能瓶颈。

应用每次获取基础配置都需要经过网络访问 Redis：

```text
业务请求
   ↓
Redis
   ↓
返回配置
```

即使 Redis 本身响应较快，也仍然存在：

* 网络 RTT
* Redis 连接池开销
* JSON 序列化 / 反序列化成本
* Redis 集群网络开销
* 大量热点 Key 集中访问 Redis

例如字典、港口、行政区域、系统配置等数据，在单个应用实例内部可能一分钟被访问数千甚至数万次。

这些数据大多数几分钟甚至数小时都不会发生变化，但是每次仍然访问 Redis，造成大量无意义的网络调用。

因此将缓存进一步升级为：

```text
Caffeine + Redis + MySQL
```

三级数据访问体系。

---

### 四、最终方案：Caffeine + Redis 两级缓存

缓存访问流程：

```text
业务请求
   ↓
Caffeine 本地缓存
   ↓ miss
Redis 分布式缓存
   ↓ miss
MySQL
   ↓
写 Redis
   ↓
写 Caffeine
   ↓
返回结果
```

优先访问应用 JVM 内部的 Caffeine。

由于 Caffeine 属于进程内缓存：

```text
应用
 ↓
内存
```

无需网络请求，访问速度远高于 Redis。

对于字典、系统参数、港口、承运商、运输地、行政区域等高频数据，大部分请求可以直接在本地完成。

例如：

```text
原方案：

10000 次基础数据查询
≈ 10000 次 Redis 请求


多级缓存方案：

10000 次查询

假设 Caffeine 命中率 95%

Redis 请求：
10000 × 5%
≈ 500 次
```

从而进一步降低 Redis QPS 和网络开销。

---

### 五、项目核心难点：多实例缓存数据不一致

引入 Caffeine 后，出现了一个新的核心问题：

**本地缓存无法天然保证多实例之间的数据一致性。**

例如生产环境部署：

```text
          Nginx

     /      |      \

  App-1   App-2   App-3

Caffeine Caffeine Caffeine

       \    |    /
          Redis
            |
          MySQL
```

假设承运商信息：

```text
carrier:10001
```

已经分别存在于：

```text
App-1 Caffeine
App-2 Caffeine
App-3 Caffeine
Redis
```

此时后台管理系统修改了承运商信息。

数据库已经更新：

```text
MySQL：新数据
```

Redis 也已经删除或更新：

```text
Redis：新数据
```

但是 App-2 和 App-3 本地 Caffeine 中可能仍然保存旧数据。

这样就会出现：

```text
请求落到 App-1 -> 新数据

请求落到 App-2 -> 旧数据

请求落到 App-3 -> 旧数据
```

最终表现为线上非常典型的问题：

> 同一个接口，多次请求偶尔返回新数据、偶尔返回旧数据。

由于请求经过负载均衡分发到不同实例，该问题具有较强的随机性，定位也比较困难。

---

### 六、最终一致性方案：Cache Aside + 消息通知

最终采用：

```text
MySQL
+
Redis
+
Caffeine
+
MQ / Redis PubSub
```

结合 Cache Aside Pattern 实现缓存一致性。

更新流程：

```text
修改基础数据
      ↓
更新 MySQL
      ↓
删除 Redis Cache
      ↓
发送缓存失效消息
      ↓
App-1 删除 Caffeine
App-2 删除 Caffeine
App-3 删除 Caffeine
```

例如修改：

```text
carrier:10001
```

处理流程：

```text
UPDATE carrier
       ↓
DELETE Redis carrier:10001
       ↓
发送消息
CacheEvictEvent
{
    type: "carrier",
    key: "10001"
}
       ↓
各应用实例消费消息
       ↓
删除本机 Caffeine
carrier:10001
```

下一次访问：

```text
Caffeine miss
      ↓
Redis miss
      ↓
MySQL
      ↓
加载最新数据
      ↓
回填 Redis
      ↓
回填 Caffeine
```

这样避免主动维护数据库、Redis、Caffeine 三份数据，而是统一通过“失效 + 懒加载”的方式获取最新数据。

---

### 七、为什么采用删除缓存，而不是直接更新缓存

在数据变更时，没有直接同时更新：

```text
MySQL
Redis
Caffeine
```

而采用：

```text
更新 DB
+
删除缓存
```

主要原因是缓存属于数据库数据的副本。

如果同时维护多个缓存副本：

```text
DB
Redis
App-1
App-2
App-3
```

一次业务修改就需要保证多个数据源全部成功更新。

一旦其中一个节点更新失败，就容易产生脏数据。

因此采用 Cache Aside：

```text
写操作：

DB Update
     ↓
Cache Delete
```

读操作：

```text
Cache
 ↓ miss
DB
 ↓
Cache Set
```

整体逻辑更加简单，也更容易保证最终一致性。

---

### 八、异常场景处理

实际生产环境不能完全依赖消息通知，因此增加多层容错机制。

#### 1. Caffeine TTL

本地缓存设置较短 TTL，例如：

```text
5min
10min
30min
```

即使 MQ 通知失败，旧数据也不会永久存在。

---

#### 2. Redis TTL

Redis 同样设置合理 TTL：

```text
30min
1h
2h
```

避免异常情况下脏数据长期存在。

---

#### 3. 消息重试

MQ 消息消费失败时执行重试：

```text
第一次消费失败
    ↓
Retry
    ↓
仍失败
    ↓
进入死信队列 / 告警
```

降低单次消息异常导致缓存长期不一致的概率。

---

#### 4. 防止缓存击穿

缓存失效后，大量并发请求可能同时访问数据库。

例如：

```text
carrier:10001 失效

1000 个并发请求同时进入

Caffeine miss
Redis miss
      ↓
1000 个请求查询 MySQL
```

对于热点 Key，通过 Caffeine LoadingCache、Redis 分布式锁或 SingleFlight 思路控制缓存重建。

实现效果：

```text
1000 个请求

第一个请求
   ↓
查询数据库并重建缓存

其他请求
   ↓
等待 / 复用结果
```

避免数据库瞬时被大量请求击穿。

---

### 九、最终缓存架构

最终系统形成：

```text
                  用户请求
                     |
                     v
              Spring Boot
                     |
                     v
             Caffeine L1
              /        \
           hit         miss
            |            |
          返回          Redis L2
                        /     \
                      hit     miss
                       |        |
                     返回     MySQL
                                |
                                v
                           回填 Redis
                                |
                                v
                          回填 Caffeine
```

缓存更新链路：

```text
管理后台修改配置
        |
        v
      MySQL
        |
        v
   删除 Redis
        |
        v
   MQ / PubSub
        |
   +----+----+
   |    |    |
 App1 App2 App3
   |    |    |
 删除本地 Caffeine
```

形成：

> **Caffeine 解决进程内高频访问性能问题，Redis 解决多实例共享缓存问题，MySQL 作为最终数据源，MQ/PubSub 负责多节点缓存失效通知。**

---

### 十、项目技术亮点总结

针对 TMS 系统字典、系统参数、行政区域、运输地、港口、承运商及运价配置等高频读取、低频更新的基础数据，设计并逐步演进缓存体系。

初期通过 Redis 缓存减少 MySQL 重复查询；随着访问量增长，发现大量热点配置访问 Redis 仍产生较高网络和 Redis QPS，因此进一步引入 Caffeine 构建本地 + 分布式两级缓存。

针对多实例部署下 Caffeine 数据无法同步的问题，采用 Cache Aside Pattern，通过“更新数据库 + 删除 Redis + MQ/PubSub 广播缓存失效消息”的方式通知各应用实例清理本地缓存，并结合 TTL、消息重试以及热点 Key 防击穿机制保证异常场景下的最终一致性和系统稳定性。

最终形成：

```text
Caffeine
   ↓
Redis
   ↓
MySQL

          +
          
MQ/PubSub 缓存失效广播
```

有效降低数据库 QPS、Redis 网络访问和热点数据查询开销，同时解决多实例本地缓存产生的线上新旧数据随机读取问题，提高了创建单据、运价计算、基础数据查询及数据看板等核心业务场景的整体性能和稳定性。

````





# 并发编排与异步化改造

```
串行匹配响应慢 → 并行查询 → 解决子线程鉴权 → MQ 异步化与状态管理 → 控制重复操作。
其中，多线程解决任务内部执行效率，MQ 解决 HTTP 等待和业务操作体验，状态管理让异步任务可查询、可人工重新触发。

报价匹配功能。
工作单需要根据运输条款、起讫地址、货主、船公司等条件，匹配应收和应付报价，再把报价转换成实际费用。
这个功能的业务复杂度比较高。一条运输路线可能涉及始驳、海运和末端配送，报价既可能按整条线路维护，也可能需要分段组合。
拼箱和地址映射还会产生多个候选地址组合，每个组合都需要远程调用报价服务。

最初我们采用串行处理，一个地址组合查完再查下一个。随着业务场景增加，远程调用次数不断累积，接口经常需要 30 秒，甚至超时。

串行执行时，总耗时会叠加每次远程查询的等待时间，所以第一轮改造是在地址组合这一层引入并发。
我们把各个地址组合拆成独立任务，通过线程池并行执行，用 ConcurrentHashMap 保存各组合的查询结果，再通过 CountDownLatch 等待子任务完成，最后统一做结果汇总和优先级处理。

这里有一个业务上的要求：结果不能按哪个任务先完成就选哪个。原始地址匹配到的报价应该优先使用，只有原始地址没有结果，才采用映射地址报价。因此并发只改变查询的执行方式，最终仍然按统一的业务规则选择结果。

引入多线程后，又遇到了子线程远程调用鉴权失败的问题。排查发现，主线程里的 SessionContext 能拿到用户 Token，但子线程拿不到，因为鉴权上下文是绑定在当前线程上的。
当时匹配任务使用的是每次新建、执行后关闭的线程池，所以我们在提交任务前，从 SessionContext 提取 Token，写入 InheritableThreadLocal，让新创建的工作线程继承。远程调用层在拿不到 SessionContext 时，从这里读取 Token，放入请求头。主线程在 finally 中清理上下文并关闭线程池。

这个方案依赖工作线程是在设置 Token 之后创建的。InheritableThreadLocal 不是每次提交任务都会重新传递上下文，所以如果后续改成共享线程池，就需要改成任务提交时捕获、执行时设置、执行结束后清理或恢复的方式。后续再也没有出现过用户投诉等，这次改造是页面中必须要看到结果的，所以是必须同步响应结果到页面上的。


后续业务方又需要加功能，必须修改了信息之后支持重新匹配报价。我采用状态机+MQ方案，在前端提交以后，后端按工作单发送匹配消息，并记录处理中状态；消费者执行匹配，完成后更新成功或失败状态。业务人员可以查看处理结果，在失败或调整业务条件后手动重新触发匹配，不再完全依赖研发排查和处理。

异步化以后，还需要考虑用户重复点击、重新匹配以及人工选择报价之间的并发冲突，因为这些操作会修改同一工作单的报价和费用。我们对相关入口使用工作单维度的 Redisson 分布式锁，让同一工作单的相关修改互斥，不同工作单仍然可以并行。

对于需要跟随数据库事务触发的消息和报价引用回调，我们采用事务提交后发送，避免事务还没提交、消费者就开始处理，或者事务回滚后消息已经发出去。

最终这个模块从一个长时间阻塞的同步接口，迭代成了内部并行查询、外部异步触发、处理状态可查询、业务人员可以手动重新匹配的流程。
```





# MQ前置后置处理架构开发









# 项目JVM配置

```
G1垃圾回收器 4核16G

```









# 报价匹配业务

## 一、功能亮点介绍

```
项目:WGO 集运报价匹配系统  技术栈:Java 8 / Spring Cloud / MySQL / RabbitMQ / Redisson

1. 设计 InheritableThreadLocal 上下文透传机制,解决线程池子线程 RPC 调用 Session Token 丢失问题
   - 背景:报价匹配涉及拼箱场景 N1×N2 地址组合并行,子线程需远程调用报价服务,普通 ThreadLocal 无法跨线程透传
   - 方案:主线程从 SessionContext 提取 Token → 写入 InheritableThreadLocal → 子线程继承 → finally 强制 remove 防内存泄漏
   - 收益:替代显式参数透传,业务代码零侵入,避免线程复用场景下的 Token 串号风险

2. 设计拼箱场景地址组合爆炸的并行匹配框架
   - 背景:门到门运输场景下,起始/目的地址各 N 个备选,笛卡尔积 N² 种组合,串行 RPC 耗时 10s+
   - 方案:ThreadPoolExecutor + CountDownLatch + ConcurrentHashMap + CopyOnWriteArrayList 组合
   - 收益:匹配耗时降低 60%+,StopWatch 性能埋点便于线上追踪

3. 设计 L3~L8 多级漏斗降级匹配算法
   - 背景:实际业务中地址精度不足,单一 SQL 无法命中报价
   - 方案:基于区域/箱型/货主/船公司/港口/详细地址 6 级粒度漏斗,每级独立查询逐级放宽
   - 收益:报价命中率显著提升,业务方 0 投诉
   
4. 设计 Redisson 注解驱动的分布式锁防重机制
- 方案:@RedissonLock AOP 注解 + leaseTime=120s/waitTime=25s,锁粒度到工作单级别
- 收益:跨工作单并行不互斥,避免重复匹配导致的脏数据

5. 设计 sendAfterTx 事务后置 MQ 投递
   - 方案:RabbitMQ + 状态机三态(PROCESSING/SUCCESS/FAILURE)驱动前端可视化
   - 收益:解决事务回滚与消息发送的数据一致性问题
```

## 二、面试话术版(可展开讲)

📌 Q1:为什么要用 InheritableThreadLocal 而不是 ThreadLocal?

```
回答模板:
物流报价匹配里有一个典型场景:拼箱 + 门到门 业务,工作单的起始地址有 1~N 个备选、目的地址也有 1~N 个备选,笛卡尔积后会产生 N² 种地址组合,每种组合都要远程调用报价服务查询。
一开始我用 ThreadLocal<String> TOKEN 来传递用户 Token,但发现子线程调 RPC 时一直鉴权失败 —— 因为普通 ThreadLocal 在子线程里 get 不到父线程的值。
改用 InheritableThreadLocal 后,子线程在创建时(从线程池借出时)会自动继承父线程的 Token,问题就解决了。
但这里有个陷阱:线程池里的线程是复用的,如果不在 finally 里 remove(),下一个任务拿到这个线程时,Token 可能是上一个用户的,就会 Token 串号,造成越权访问。
所以最终落地是 三段式:
// 1. 主线程 set
QuotationRequestTokenThreadLocal.setUserToken(SessionContext.getSessionUserInfo().getToken());
try {
    // 2. 提交 N 个子任务,子线程自动继承
    threadPool.submit(() -> doMatch(...));
    latch.await();
} finally {
    // 3. 强制 remove,防止线程复用导致泄漏
    QuotationRequestTokenThreadLocal.removeUserToken();
    threadPool.shutdown();
}

替代方案对比:也可以用 TransmittableThreadLocal(阿里开源) + TTL Agent 解决线程池复用场景的传递问题,但这个项目用的是 JDK 自带的 InheritableThreadLocal,够用了。
```

Q5:整套并发设计里你怎么保证正确性?
```
三道防线:
外层 Redisson 分布式锁:@RedissonLock(lockKey = 工作单ID) 保证同一个工作单不会被两个线程同时匹配;
内层 CountDownLatch:保证主线程等所有子任务完成后再汇总结果;
状态机隔离:PROCESSING / SUCCESS / FAILURE 三态落库,即使匹配过程异常也能从 DB 查到状态;
MQ 异步化:HTTP 接口立即返回 "匹配中",实际匹配在 MQ 消费者里跑,前端轮询状态,这是 削峰 + 解耦 的双重设计。
```

在 WGO 报价匹配模块中,设计 InheritableThreadLocal + 线程池 + CountDownLatch + MQ 事务后置 + Redisson 分布式锁 的五合一并发编排框架,支撑拼箱场景 N² 地址组合的并行匹配,匹配耗时降低 60%+,且具备完善的异常兜底与状态可视化能力。

## 三、阿里风格介绍这个模块

```
项目:WGO 智能物流中台 - 运价引擎  角色:核心开发  技术栈:Java 8 / Spring Cloud / MySQL / RabbitMQ / Redisson

1. 沉淀分布式上下文透传组件,赋能高并发场景链路鉴权
   • 背景:运价匹配引擎在拼箱 N² 地址组合并行场景下,子线程 RPC 调用丢失 Session 鉴权信息,频繁触发网关拦截
   • 方案:封装基于 InheritableThreadLocal 的 UserContext 组件,统一 set-use-remove 三段式生命周期,搭配 try-finally 兜底杜绝线程复用导致的越权风险
   • 成果:作为平台基础能力被 5+ 业务线复用,业务代码零侵入,链路鉴权成功率 100%

2. 设计多粒度漏斗式运价召回算法,显著提升地址精度缺失场景下的命中率
   • 背景:物流地址录入精度参差不齐,单一匹配策略难以兼顾准确率与召回率
   • 方案:基于"区域/箱型/货主/船公司/港口/详细地址"6 级粒度,设计 L3~L8 漏斗召回,逐级放宽约束,搭配段间报价拼接策略(1/2/3 段组合)
   • 成果:运价召回率提升至行业领先水平,业务方客诉率显著下降

3. 构建运价组合空间爆炸的并行化求解框架
   • 背景:门到门运输条款下,起始/目的地址各 N 个备选,笛卡尔积 N² 种组合串行执行,平均耗时 10s+
   • 方案:ThreadPoolExecutor + CountDownLatch + ConcurrentHashMap 三件套,搭配 StopWatch 性能埋点与异常隔离,实现地址组合级别的并发求解
   • 成果:匹配耗时降低 65%,系统吞吐能力提升 3 倍,成为高优 SLA 业务的标配能力

4. 沉淀声明式分布式锁组件,提供细粒度并发控制能力
   • 方案:基于 Redisson 封装 @RedissonLock AOP 注解,锁粒度下沉至工作单维度,leaseTime=120s/waitTime=25s 双重防雪崩
   • 成果:杜绝并发匹配脏数据,跨工作单并行不互斥,组件已成为平台并发控制标准

5. 设计最终一致性事务消息方案,实现运价匹配状态可观测可干预
   • 方案:基于 RabbitMQ + sendAfterTx 事务后置投递,搭配 PROCESSING/SUCCESS/FAILURE 三态状态机驱动前端可视化
   • 成果:彻底解决事务回滚与消息发送的数据一致性问题,匹配异常可被前端实时感知并人工介入
```









# 自定义注解审计模块

```
1. 自研「字段级变更审计日志」框架（AOP + 注解 + 反射）
位置：FieldLogUtil.java / WorkTaskLogAspect.java
做了什么：
自定义 @FieldLog 字段注解 + @WorkTaskLog 方法注解
通过 Spring SpEL 表达式 动态提取方法参数（如 #workTaskId、#tasks）
利用 Java反射 读取实体字段的 @FieldLog 注解，按字段类型（String/Date/BigDecimal）差异化处理
字典字段自动调用 DictDataService.getDictValueByCode 转换为可读值
BigDecimal 使用 compareTo 而非 equals（避免 scale 差异导致误报）
记录"修改X从A改为B"格式的变更说明，存入 WorkTaskLog 表
价值：业务人员无需手写变更日志代码，零侵入式实现全字段审计追踪。


2. 「策略 + 工厂」模式实现可扩展日志体系
位置：WorkTaskLogStrategyFactory.java / log/ 目录下10+ 策略类

做了什么：
定义 WorkTaskLogStrategy 接口，10 个具体策略类：PrimaryWorkTaskLogStrategy、ContainerWorkTaskLogStrategy、BookingTaskWorkTaskLogStrategy、DeclarationWorkTaskLogStrategy、ExpenseWorkTaskLogStrategy 等
WorkTaskLogStrategyFactory 实现 InitializingBean，启动时自动收集所有 WorkTaskLogStrategy 
实现类按 WorkTaskLogType 枚举（PRIMARY/CONTAINER/BOOKING_TASK/DECLARATION/QUOTATION/EXPENSE/INSURANCE/ATTACHMENT 等）路由到对应策略

策略间可相互注入组合（如 PrimaryWorkTaskLogStrategy 内部调用 ContainerWorkTaskLogStrategy）
价值：新增日志类型只需新增一个策略类即可，符合 OCP（开闭原则），是教科书级的策略 + 工厂应用。
```



```
工作单操作日志系统。业务背景是：一票外贸工作单会牵涉订舱、报关、保险、货柜、费用等 10 来个子业务，业务侧强烈需要追溯"谁在什么时候把什么改成了什么"，而且记录要像人话一样能直接读懂，比如'修改目的港从上海改为宁波'。
我把它做成了一个声明式的审计日志框架：业务方法上打一个注解，AOP 切面在方法执行前后自动查库对比差异，按策略分发到不同业务域，最后生成自然语言日志。业务代码零侵入，只记真实变更，全程不影响主流程。"
```

```
"比如业务员把一票货的目的港从上海改成宁波、同时把柜子封号改了——系统会落一条：'修改目的港从上海改为宁波;修改封号从 A 改为 B;'，操作人、时间都带上。要是这批货后面出了问题，业务可以精确回放每一步操作。"
```

```
"我讲讲工作单日志这个模块，它是我觉得设计上比较完整的一块。
业务上：外贸工作单是核心单据，牵涉订舱、报关、保险、货柜、费用等十几个子域。业务要求对任何修改都能追溯——谁改的、什么时候、把什么从什么改成了什么，而且要能直接看懂。
设计上：我用注解 + AOP + 策略三件事解决。注解声明式埋点，SpEL 从方法参数里取工作单 ID，业务代码零侵入；AOP 做环绕，执行前查库快照、执行后再查库对比——以数据库为基准而不是以入参为基准，因为业务方法可能级联改多张表，只有重查才能拿到完整的新状态。快照必须浅拷贝，否则对象引用被业务改动污染，旧值就不准了。10 个业务域差异大，用策略模式按枚举分发，每个策略自己编排该域的对比，工厂靠 Spring 构造注入自动注册，新增类型不动老代码。
细节上：字段级 diff 引擎只处理打注解的字段，BigDecimal 用 compareTo 防精度误报、空值归一成'空'、字典 code 批量翻译成中文、只记有变化的字段；集合用主键归并算法区分新增/删除/修改，还能带上柜号这种业务标识；批量标记场景先脏检查再记日志，关联数据全部批量查询防 N+1。
健壮性上：日志写失败只记 error 绝不拖垮主业务；系统自动操作操作人自动记 system；一次改货柜的操作，会通过策略组合把工作单主体、货柜、货柜商品三层变更一次留痕。
效果是 24 个业务方法一行注解接入，全程无侵入，日志话术业务人员直接可读。这个模块也让我把 AOP、注解、策略、反射这些平时'会背不会用'的东西真正落地了一遍，踩了不少坑，比如 BigDecimal 的 equals 陷阱、对象快照被引用污染，都是实际生产里才遇得到的。"
```





# 自定义注解数据同步框架

```
3. 分布式锁 + 数据同步自研框架
位置：WorkTaskSyncLockAspect.java / WorkTaskSyncStrategy.java

做了什么：
自研 @WorkTaskSyncLock 注解，支持 SpEL 提取工作单 ID，支持配置 waitTime/leaseTime/syncDataTypes
切面通过 Redisson RLock.tryLock(...) 加锁
精妙设计：selectRelatedWorkTask(workTaskId) 查询拼箱/拼票关联工作单后，取最小 ID 作为锁 key，避免多个关联工作单互相等待导致死锁

锁释放后，根据 syncDataTypes 枚举路由到对应的 WorkTaskSyncStrategy 实现类执行同步
Spring 自动注入所有 WorkTaskSyncStrategy 实现（Map<String, WorkTaskSyncStrategy>），通过枚举名匹配- 同步失败隔离不影响主流程（try-catch 包住每个同步服务）

价值：将「加锁 + 数据同步」从业务代码中彻底解耦，复杂的并发控制一个注解搞定。
```



# 项目简历

```
项目描述

集运 TMS 系统（内贸 + 外贸一体化），涵盖工作单、订舱、报关、保险、费用、委托函等全链路物流业务。基于 SpringCloud + Spring Boot + MyBatis-Plus + Redisson + Apollo + XXL-JOB + RabbitMQ 技术栈，对接 SAP、OMS、中远 COSCO、安通、Dify AI 等多个外部系统。

核心职责

设计并实现集运线路报价匹配全链路方案：基于线程池、并发包、MQ等技术方案，针对物流报价匹配中地址纯在的多种笛卡尔积组合以及多种报价组合方案进行编排并优化处理性能于响应体验

设计并实现系统字段级审计日志：基于 Spring AOP + 自定义注解 + 工厂模式 + 策略模式，实现零侵入式变更追踪，覆盖工作单/货柜/订舱/报关等 10+ 业务实体

设计并实现物流拼箱、拼票场景数据同步：基于自定义注解，通过 Redisson 加锁 + 策略模式分发同步任务，解决拼箱、拼票场景下的数据同步并发与数据一致性

设计并实现物流配送核心业务对象工作单：进行模块化服务拆分、状态集中控制管理、拼箱拼票单动作级联传播、高危业务操作三段式校验等

设计并实现项目中分布式锁：基于redission框架+自定义注解+Spring AOP

设计并实现项目多级缓存架构：使用本地缓存+redis缓存提升系统吞吐量
```







