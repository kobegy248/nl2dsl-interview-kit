# Java 后端与微服务专项（对标 JD 第 6–16 条）

> **证据边界**：本文分两栏——「你真做过的」可以直说，「你没深做的」只讲原理与选型判断，不说成项目经验。
> 真实经历依据：简历中的澎思人脸布控系统（Dubbo / Zookeeper / RabbitMQ / Redis / MongoDB / Docker / 50+ 台机器）与数据资产平台（Java + Spark SQL）。
>
> **怎么读**：先看第 1 节的现状对照表，再按缺口挑章节读。每章按「原理 → 高频问法 → 你该怎么答」组织。

---

## 1. 先对照：JD 要的 12 条，你缺哪几条

| # | JD 要求 | 你的真实情况 | 判定 |
|---|---|---|---|
| 6 | Java 基础：集合、多线程、并发、JVM 调优 | 8 年后端，简历有 Arthas / JVM 调优 / 线上排查 | 🟢 有 |
| 7 | Spring / Boot / MVC / MyBatis、IOC / AOP / 事务 | 简历有 Spring Boot / MyBatis | 🟢 有 |
| 7 | MySQL / PostgreSQL：设计、SQL 优化、索引优化 | MySQL 有；PostgreSQL 无；索引优化有实践 | 🟡 半 |
| 8 | Redis / RabbitMQ / Kafka | Redis ✅ RabbitMQ ✅；**Kafka 没写** | 🟡 半 |
| 9 | Spring Cloud / Alibaba / Nacos / OpenFeign / Gateway | **没有**（你用的是 Dubbo + Zookeeper，另一套栈） | 🔴 缺 |
| 10 | 服务发现、负载均衡、熔断限流、分布式事务、分布式锁、幂等 | 服务发现 ✅（Zookeeper）、负载均衡 ✅、其余**没系统做过** | 🟡 半 |
| 11 | Docker / K8s / 云原生 | Docker + docker-compose ✅；**K8s 无** | 🟡 半 |
| 12 | 企业级架构设计：选型、模块划分、接口设计、高可用 | NL2DSL 分层架构 + 人脸系统架构，可以讲 | 🟢 有 |
| 13 | 认证授权：RBAC / OAuth2 / JWT / Spring Security | **简历没有** | 🔴 缺 |
| 14 | RESTful API、接口集成、第三方对接 | 有（WebSocket 推送、算法 SDK 对接） | 🟢 有 |
| 15 | 技术方案评审、Code Review、单元测试、代码质量 | 有（自动化测试、测试规范） | 🟢 有 |
| 16 | 故障排查、性能分析 | **最强项**：50+ 台机器、日抓拍 1000 万级运维 | 🟢 强 |

**结论：12 条里 7 条能打，5 条半，2 条缺。缺的两条（Spring Cloud 体系、认证授权）恰好是最好补的——它们是标准模式，不需要多年经验。**

---

## 2. Java 核心：三个必考区

### 2.1 集合

| 结构 | 底层 | 关键点 |
|---|---|---|
| ArrayList | 动态数组 | 扩容 1.5 倍；随机访问 O(1)，中间插入 O(n) |
| LinkedList | 双向链表 | 插入 O(1)，查询 O(n)；**实际很少用** |
| HashMap | 数组 + 链表 + 红黑树 | 链长 > 8 且容量 ≥ 64 转红黑树；扩容 2 倍 |
| ConcurrentHashMap | 分段 + CAS + synchronized | JDK8 后锁的是桶首节点 |
| LinkedHashMap | HashMap + 双向链表 | 可实现 LRU（`accessOrder=true`） |

**高频问法**："HashMap 为什么线程不安全？""1.7 和 1.8 有什么区别？"

> 答题要点：1.7 头插法在并发扩容时会成环 → CPU 100%；1.8 改尾插 + 红黑树，但仍不安全（put 会覆盖），要并发用 ConcurrentHashMap。

### 2.2 并发

必须能讲清的四件事：

```
① 线程池七个参数 → 拒绝策略四种
② synchronized vs ReentrantLock（可中断/可超时/公平/条件变量）
③ volatile 的两个语义（可见性、禁止重排；不保证原子性）
④ ThreadLocal 内存泄漏（key 弱引用、value 强引用，用完必须 remove）
```

**线程池参数的实际用法**（结合你的抓拍场景）：

```java
// 人脸底库导入：CPU 密集 + IO 混合，队列不能无界
new ThreadPoolExecutor(
    8,                              // core：CPU 核数
    16,                             // max：IO 等待时扩容
    60L, TimeUnit.SECONDS,
    new ArrayBlockingQueue<>(2000),  // 有界队列，防止 OOM
    new ThreadFactoryBuilder().setNameFormat("import-%d").build(),
    new ThreadPoolExecutor.CallerRunsPolicy()  // 满了让调用线程自己跑，起背压作用
);
```

> 你能讲的故事：**2000 万级静态底库导入用多线程批处理**——这就是线程池 + 有界队列 + 批量提交的真实场景，比背八股强。

### 2.3 JVM

| 区域 | 内容 | 常见问题 |
|---|---|---|
| 堆 | 新生代（Eden + 2 Survivor）、老年代 | OOM、频繁 Full GC |
| 方法区/元空间 | 类元数据 | Metaspace OOM |
| 栈 | 线程私有 | StackOverflow |
| 直接内存 | NIO | 堆外 OOM |

**GC 选型**：JDK8 默认 Parallel；大堆低延迟用 G1（`-XX:+UseG1GC`）；超大堆用 ZGC。

**线上排查三板斧**（你的强项，直接讲经验）：

```bash
jps -l                          # 找进程
jstat -gcutil <pid> 1000        # 看 GC 频率和耗时
jstack <pid> > stack.log        # 线程 dump，查死锁/阻塞
jmap -histo:live <pid> | head   # 对象统计
# 线上更常用 Arthas：
thread -n 3                     # CPU 最高的 3 个线程
trace com.xxx.Service method    # 方法耗时链路
```

> 面试时讲"我用 Arthas 的 trace 定位过接口超时，发现是某个 SQL 没走索引 + 连接池打满"，比背 jstat 参数有用得多。

---

## 3. Spring 体系：四个必答点

### 3.1 IoC 与 Bean 生命周期

```
实例化 → 属性注入 → BeanNameAware/BeanFactoryAware
→ BeanPostProcessor#before → @PostConstruct
→ InitializingBean#afterPropertiesSet → BeanPostProcessor#after
→ 使用 → @PreDestroy → DisposableBean#destroy
```

### 3.2 AOP 与代理

- **JDK 动态代理**：目标类实现了接口（生成接口的代理类）
- **CGLIB**：没接口，生成子类（Spring Boot 2.x 默认）
- **经典坑**：同类内部方法调用不走代理 → `@Transactional` 失效。解法：注入自身、`AopContext.currentProxy()`、或拆类。

### 3.3 事务传播（高频）

| 传播行为 | 含义 | 场景 |
|---|---|---|
| REQUIRED（默认） | 有就加入，没有就新建 | 绝大多数 |
| REQUIRES_NEW | 总是新建，挂起外层 | 记日志不想被回滚 |
| NESTED | 嵌套，用 savepoint | 局部失败局部回滚 |
| SUPPORTS / NOT_SUPPORTED | 有就加入 / 挂起事务 | 查询优化 |

**事务失效的五种情况**（能答全就是资深）：

```
① 方法非 public
② 同类内部调用（不走代理）
③ 异常被 catch 没抛出
④ 抛的是检查异常（默认只回滚 RuntimeException）
⑤ 数据库引擎不支持事务（MyISAM）
```

### 3.4 Spring Boot 启动流程

```
SpringApplication.run()
→ 加载 ApplicationContextInitializer / Listener
→ 准备 Environment（配置文件、命令行参数）
→ 创建 ApplicationContext
→ refresh()：BeanFactory 准备 → BeanPostProcessor 注册 → 实例化单例
→ 启动内嵌容器 → 发布 ApplicationReadyEvent
```

---

## 4. 微服务：Spring Cloud 那套（你缺的，但好补）

### 4.1 组件全景

| 能力 | Netflix/早期 | Spring Cloud Alibaba（JD 要的） | 你的经历 |
|---|---|---|---|
| 注册发现 | Eureka | **Nacos** | Zookeeper ✅（原理相通） |
| 配置中心 | Config | **Nacos Config** | 无 |
| 负载均衡 | Ribbon | **LoadBalancer** | ✅ 有概念 |
| 服务调用 | Feign | **OpenFeign** | Dubbo ✅（RPC 思路一致） |
| 网关 | Zuul | **Gateway** | Nginx 反向代理 ✅ |
| 熔断限流 | Hystrix | **Sentinel** | 无 |
| 分布式事务 | — | **Seata** | 无 |

> **关键说法**：你用的是 Dubbo + Zookeeper，JD 要 Spring Cloud Alibaba。**不要硬说做过 Nacos**，讲"我用 Dubbo/ZK 做过服务注册发现和动态扩容，Nacos 的思路是一样的：服务注册 + 心跳 + 本地缓存 + 推送更新"。

### 4.2 注册发现原理（你能讲，因为做过 ZK 版）

```
服务启动 → 注册到注册中心（临时节点/心跳保活）
消费者启动 → 拉取服务列表 → 本地缓存
服务变更 → 注册中心推送 → 消费者更新缓存
服务下线 → 临时节点消失 / 心跳超时剔除
```

你的真实经验：**"基于 Zookeeper 完成算法 SDK 服务集群化，包括服务注册、负载均衡、扩容机制和更新推送"**——这就是注册发现的全部要点，只是实现是 ZK 而不是 Nacos。

### 4.3 熔断限流（缺，但要会讲）

| 概念 | 含义 | 实现 |
|---|---|---|
| **限流** | 限制 QPS，保护自己 | 令牌桶 / 漏桶 / 滑动窗口 |
| **熔断** | 下游挂了快速失败，别拖死自己 | 关闭 → 打开 → 半开 三态 |
| **降级** | 返回兜底结果 | 默认值 / 缓存 / 简化逻辑 |

```java
// Sentinel 风格
@SentinelResource(value = "query", blockHandler = "onBlock")
public Result query(String q) { ... }

public Result onBlock(String q, BlockException ex) {
    return Result.degraded("当前查询量过大，请稍后重试");
}
```

---

## 5. 分布式四件套（JD 第 10 条，必须能讲）

### 5.1 分布式锁

| 方案 | 原理 | 坑 |
|---|---|---|
| Redis（SETNX + 过期） | 抢 key | 过期时间不好定、误删别人的锁 |
| **Redisson** | 看门狗自动续期 + Lua 原子解锁 | 生产首选 |
| Zookeeper | 临时顺序节点 | 性能低于 Redis，但更可靠 |

**正确实现三要素**：

```java
// 1. 原子加锁（SET NX EX，不能分两步）
SET lock:order:123 <uuid> NX EX 30

// 2. 解锁要校验持有者（Lua 脚本，防止删别人的锁）
if redis.call("get", KEYS[1]) == ARGV[1] then
    return redis.call("del", KEYS[1])
end

// 3. 业务没完 → 看门狗续期（Redisson 自动做）
```

### 5.2 分布式事务

| 方案 | 一致性 | 性能 | 场景 |
|---|---|---|---|
| 2PC/XA | 强 | 差 | 老系统、金融 |
| **TCC** | 最终一致 | 中 | 需要精细控制 |
| **Saga** | 最终一致 | 好 | 长流程 |
| **本地消息表** | 最终一致 | 好 | **最常用**，配合 MQ |
| Seata AT | 最终一致 | 中 | 侵入小 |

**本地消息表（最值得讲）**：

```
① 业务操作 + 写消息表，同一个本地事务
② 定时任务扫消息表 → 发 MQ → 标记已发
③ 消费方幂等处理
④ 失败重试 + 告警
```

> 好处：不依赖分布式事务框架，靠"本地事务 + 重试 + 幂等"做到最终一致。

### 5.3 幂等（最常考）

| 方案 | 做法 |
|---|---|
| **唯一索引** | 最可靠，插入冲突即重复 |
| **token 机制** | 先领 token，提交时校验并删除 |
| **状态机** | 只允许 待支付→已支付，重复请求直接返回 |
| **分布式锁 + 请求 ID** | 同一请求 ID 只处理一次 |

```java
// 唯一索引方案
try {
    orderMapper.insert(order);          // order_no 唯一索引
} catch (DuplicateKeyException e) {
    return orderMapper.selectByNo(order.getNo());  // 返回已有结果
}
```

### 5.4 服务降级与故障转移

你真实做过：**算法 SDK 服务集群化 + 扩容**。可以讲成："SDK 服务多实例注册到 ZK，调用方负载均衡；某实例挂了下游自动摘除，请求转移到其他实例。"

---

## 6. 中间件：Redis / RabbitMQ / Kafka

### 6.1 Redis 三大问题（必考）

| 问题 | 现象 | 解法 |
|---|---|---|
| **缓存穿透** | 查不存在的数据，每次都打 DB | 布隆过滤器 / 空值缓存 |
| **缓存击穿** | 热点 key 过期瞬间，大量请求打 DB | 互斥锁重建 / 逻辑过期 |
| **缓存雪崩** | 大量 key 同时过期 | 过期时间加随机数 / 多级缓存 / 熔断 |

**缓存一致性**（高频）：**先更新数据库，再删缓存**（Cache Aside）。为什么不先删缓存？因为并发下会出现"删了缓存 → 另一个线程读到旧值写回缓存 → 脏数据常驻"。

### 6.2 RabbitMQ 可靠性三件事

```
① 生产者可靠：publisher confirm + 失败重投
② 消息不丢：交换机/队列持久化 + 消息持久化
③ 消费可靠：手动 ACK + 幂等消费 + 死信队列
```

你的真实经验：**"通过 RabbitMQ 接收相机抓拍数据 → 算法解析 → 入库 → 布控比对 → 告警；告警再经 MQ 发到主工程，WebSocket 按权限推送"**——这就是完整的消息驱动架构，讲的时候把"消息积压怎么处理"带上（增加消费者、优化消费逻辑、临时扩容）。

### 6.3 Kafka（你没写，但要能讲）

| 对比 | RabbitMQ | Kafka |
|---|---|---|
| 定位 | 业务消息、可靠投递 | 高吞吐日志流 |
| 吞吐 | 万级 | 十万~百万级 |
| 顺序 | 队列内有序 | **分区内有序** |
| 消费 | 推/拉 | 拉（消费者组） |

**高频问法**："Kafka 怎么保证顺序？"→ 同一 key 发到同一分区，分区内有序。**"怎么处理积压？"** → 加分区 + 加消费者（不超过分区数）+ 提高单批拉取量。

---

## 7. 数据库：索引与优化

### 7.1 索引原理

- **B+ 树**：非叶子只存索引，叶子存数据 + 双向链表 → 范围查询快、树矮（3~4 层能存千万级）。
- **聚簇索引**（InnoDB 主键）：叶子存整行；**二级索引**叶子存主键 → 查非索引列要**回表**。
- **覆盖索引**：查询列都在索引里，不用回表。

### 7.2 索引失效的六种情况（必考）

```
① 对索引列做函数/运算：where date(create_time) = '2024-01-01'
② 隐式类型转换：where phone = 13800138000（phone 是 varchar）
③ 前导模糊：like '%abc'
④ 违反最左前缀：联合索引 (a,b,c)，查 b
⑤ OR 连接非索引列
⑥ 优化器判断全表更快（数据量小 / 区分度低）
```

### 7.3 慢查询排查

```sql
EXPLAIN SELECT ...;
-- 重点看：type（ref/range 好，ALL 差）、key（用了哪个索引）、
--         rows（扫描行数）、Extra（Using index 覆盖 / Using filesort 需优化）
```

> 结合你的真实经验讲："数据资产平台的大表 Join 慢，通过**分区裁剪 + 谓词下推 + 中间结果复用**把耗时降下来"——这是 Spark SQL 语境，但优化思路（减少扫描量、减少 shuffle）和 MySQL 是相通的。

---

## 8. Docker 与 K8s

### 8.1 Docker 三件套

```
镜像（image）：只读模板，分层存储
容器（container）：镜像的运行实例，可写层
仓库（registry）：镜像分发
```

你真实做过：**docker-compose 编排多个业务模块**——能讲清 `services` / `volumes` / `depends_on` / `networks` 就够。

### 8.2 K8s 核心概念（缺，但要会讲）

| 概念 | 作用 |
|---|---|
| **Pod** | 最小调度单位，一个或多个容器 |
| **Deployment** | 管理 Pod 副本、滚动更新、回滚 |
| **Service** | 稳定访问入口 + 负载均衡 |
| **Ingress** | 七层路由（域名/路径 → Service） |
| **ConfigMap / Secret** | 配置与密钥 |
| **HPA** | 按 CPU/QPS 自动扩缩容 |

**滚动更新**：`maxSurge=25%`（先起新的）、`maxUnavailable=25%`（旧的不全停）→ 服务不中断。

> 说法："我们用的是 docker-compose 单机编排；K8s 的思路我理解——Deployment 管副本和滚动更新，Service 做服务发现和负载均衡，和我在 ZK 上做的注册发现是同一层问题。"

---

## 9. 认证授权（JD 第 13 条，你完全缺）

### 9.1 Spring Security + JWT 最小落地

```
① 登录接口：校验用户名密码 → 签发 JWT
② 过滤器：每个请求解析 Authorization: Bearer xxx → 验签 → 塞进 SecurityContext
③ 授权：@PreAuthorize("hasRole('ADMIN')")
```

```java
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    @Bean
    SecurityFilterChain filterChain(HttpSecurity http) throws Exception {
        http.csrf().disable()
            .authorizeHttpRequests(auth -> auth
                .requestMatchers("/api/v1/query/**").hasAnyRole("ANALYST", "ADMIN")
                .requestMatchers("/api/v1/admin/**").hasRole("ADMIN")
                .anyRequest().authenticated())
            .addFilterBefore(jwtFilter, UsernamePasswordAuthenticationFilter.class);
        return http.build();
    }
}
```

### 9.2 RBAC 四张表

```
user / role / permission
user_role / role_permission
```

> 你项目里的 `permissions.yaml`（用户→行级过滤 + 允许维度）其实就是一个轻量 ABAC，可以拿来类比："我做的是数据级的 ABAC，应用级 RBAC 是同一套思路，只是判定对象从'行/列'变成'接口'。"

---

## 10. 把你真实的经历"挂"到这些知识点上

| 面试问 | 你讲什么（真实） |
|---|---|
| 分布式/服务治理 | **Zookeeper 做算法 SDK 集群化**：注册、负载均衡、扩容、更新推送 |
| 消息驱动 | **RabbitMQ 接收抓拍 → 解析 → 入库 → 告警 → WebSocket 推送**；处理过消息积压 |
| 并发/性能 | **2000 万级底库导入**用多线程 + 批处理；EasyExcel 多线程导出 |
| 线上可靠性 | **50+ 台机器、日抓拍 1000 万级**集群运维，排查超时/积压/延迟 |
| 架构设计 | **NL2DSL 分层架构**（Agent 层 + Graph 层）+ 人脸系统多模块 |
| 数据库优化 | 大表 Join 优化、分区、谓词下推（Spark 语境，思路通用） |
| 容器化 | **docker-compose** 编排多模块，提升现场部署效率 |

**缺的直接说缺**：Spring Cloud Alibaba、K8s、Seata 分布式事务、OAuth2 —— "这几块我没在生产用过，但我理解它们的模型和取舍"。**这句话比硬编一个好**。

---

## 11. 面试高频 15 题速答

| 题 | 一句话答 |
|---|---|
| HashMap 为什么 1.8 加红黑树 | 链长 8 时查询退化到 O(n)，树化后 O(log n) |
| 线程池怎么设参数 | CPU 密集 N+1；IO 密集 2N；队列必须有界 |
| ThreadLocal 为什么会泄漏 | key 弱引用被回收，value 强引用还在，必须 remove |
| 事务失效的五种情况 | 非 public / 内部调用 / 吞异常 / 检查异常 / 引擎不支持 |
| 怎么保证消息不丢 | 生产 confirm + 持久化 + 消费手动 ACK + 死信 |
| 怎么处理消息积压 | 加消费者（不超分区/队列数）+ 优化消费逻辑 + 临时扩容 |
| 缓存一致性怎么做 | 先更新库再删缓存；强一致上分布式锁或订阅 binlog |
| 缓存穿透/击穿/雪崩 | 布隆或空值 / 互斥重建 / 过期加随机 |
| 分布式锁怎么实现 | SET NX EX + Lua 解锁 + 看门狗续期 |
| 分布式事务怎么选 | 强一致 2PC；最终一致用本地消息表 / Saga / TCC |
| 幂等怎么做 | 唯一索引 / token / 状态机 / 请求 ID + 锁 |
| 索引失效的常见原因 | 函数、隐式转换、前导模糊、违反最左前缀 |
| 怎么排查线上 CPU 高 | jstack / Arthas thread -n 3 找热点线程，看栈 |
| 怎么排查接口慢 | trace 链路 → SQL 慢查 → 连接池 → 下游超时 |
| 滚动更新怎么不中断 | maxSurge + maxUnavailable + 健康检查 + 优雅停机 |

---

## 12. 速查

| 项 | 要点 |
|---|---|
| 你的强项 | 故障排查、消息驱动、服务注册（ZK）、并发批处理、架构分层 |
| 你的缺口 | Spring Cloud 体系、K8s、分布式事务框架、OAuth2/JWT |
| 缺口怎么讲 | "没在生产用过，但理解模型和取舍"+ 讲同类经验 |
| 最容易被问穿 | 硬说自己做过 Nacos/Seata |
| 补的顺序 | JWT/RBAC（1 周）→ Spring Cloud 概念（1 周）→ K8s 概念（3 天） |
| 一句话 | **7 条能打、2 条缺，缺的是标准模式不是经验——补概念 + 诚实交底就够。** |
