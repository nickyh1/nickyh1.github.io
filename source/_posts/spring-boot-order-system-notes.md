---
title: Spring Boot 订单系统开发笔记：从排错到原理
date: 2026-05-28
tags:
  - Spring Boot
  - RabbitMQ
  - MyBatis-Plus
  - Redis
  - Java
categories:
  - 知识笔记
---

## RabbitMQ 消费失败：Order not found 排查

在开发订单超时取消功能时，`OrderTimeoutConsumer` 消费延迟队列消息后一直报 `BusinessException: Order not found`。SQL 日志显示 `SELECT ... WHERE order_no = ? → Total: 0`，说明数据库里确实没有这条订单记录。

根因有两种可能。第一种是测试类跑完后做了数据清理或事务回滚，订单数据没了，但延迟消息已经发到 RabbitMQ 了，15 分钟后消息到期、消费者查库自然找不到。第二种是订单创建事务未提交就发了 MQ 消息，事务最终回滚但消息已在队列中。

修复方式是在消费者里加防御性判断，查不到订单时直接 ack 消息、记 warn 日志，不抛异常、不重试。因为订单不存在这种情况，重试一万次也不会成功。

```java
OrderInfo order = orderService.findByOrderNo(orderNo);
if (order == null) {
    log.warn("Order not found, skip timeout: {}", orderNo);
    return;
}
```

项目里已经用了 `TransactionSynchronization.afterCommit()` 来确保事务提交后才发消息，这个做法是正确的。但如果应用在 afterCommit 执行时崩了，消息还是会丢，所以配合本地消息表（`order_message_log`）做兜底。

## RabbitMQ 清空队列的操作

如果需要临时止血，可以清空队列里的积压消息：

```bash
docker exec <容器名> rabbitmqctl purge_queue order.timeout.queue
```

但代价是队列里可能有合法的超时取消消息也会被清掉。更稳妥的方式是先修复代码再让积压消息自然消费完。

## 压测变慢的原因：数据库行锁串行化

500 个线程同时下单、抢同一个商品的库存，但实际执行变成了一条一条慢慢跑。原因是 `createOrder` 整个方法在 `@Transactional` 内，而 `deductStock` 会对库存行加锁。锁从拿到到事务提交才释放，期间还要做插入订单、保存消息日志等操作，锁持有时间很长，后面的线程全在排队等锁。

优化方向是缩短锁持有时间——把库存扣减用乐观锁做成独立的短事务，和订单创建分开，让锁持有时间从"整个 createOrder"缩短到"一条 UPDATE"。

另外，如果 IDE 里主应用和测试类同时运行，两者连同一个数据库和 RabbitMQ，主应用的消费者也在抢资源，也会拖慢压测速度。

## MyBatis-Plus 的关联机制

`productMapper.selectPage(page, wrapper)` 涉及三个组件的协作：

- **BaseMapper 的泛型 `<Product>`** 告诉框架操作哪个实体，框架通过反射读取 `Product` 类上的 `@TableName("product")` 得到表名，从字段映射得到列名。
- **Entity 的字段** 决定 SELECT 哪些列、Java 字段到数据库列的映射关系（如 `createTime` → `create_time`），以及查询结果的承载。
- **Wrapper** 只是条件容器，存储 WHERE 条件、排序、分组等片段，本身不知道表名。

最终拼 SQL 时，BaseMapper 提供 FROM 和 SELECT，Wrapper 提供 WHERE 和 ORDER BY。

关于泛型 `T`：在定义 `BaseMapper<T>` 时 T 是占位符，使用时必须填具体类型如 `BaseMapper<Product>`，否则框架无法确定查哪张表。泛型可以层层传递，但最终一定要有一个地方替换成具体实体类。

## ObjectMapper 的作用

`ObjectMapper` 是 Jackson 库提供的 JSON 序列化/反序列化工具。

- `objectMapper.writeValueAsString(product)` 把 Java 对象转成 JSON 字符串（序列化），用于存入 Redis。
- `objectMapper.readValue(cached, Product.class)` 把 JSON 字符串转回 Java 对象（反序列化），用于从 Redis 读取。

## 延迟双删不会阻塞请求

```java
public void evictCacheWithDelay(Long productId) {
    evictCache(productId);
    new Thread(() -> {
        Thread.sleep(500);
        evictCache(productId);
    }).start();
}
```

`Thread.sleep(500)` 在新线程里执行，不阻塞当前请求线程。`updateProduct` 调完就立刻返回了。

但 `new Thread()` 每次创建销毁线程有开销，更好的做法是用 `ScheduledExecutorService`：

```java
private final ScheduledExecutorService scheduler =
    Executors.newSingleThreadScheduledExecutor();

public void evictCacheWithDelay(Long productId) {
    evictCache(productId);
    scheduler.schedule(() -> evictCache(productId), 500, TimeUnit.MILLISECONDS);
}
```

线程池作为 Service 的成员变量，随实例初始化一次，之后所有延迟任务复用同一个线程，没有反复创建销毁的开销。`ScheduledExecutorService` 是支持定时/延迟任务的线程池，比普通 `ExecutorService` 多了延迟执行和周期执行的能力。

## 幂等控制的时机

用户进入确认订单页时，前端请求一个一次性 token 并暂存。用户点"提交订单"时，token 随请求发给后端。`createOrder` 第一步就校验 token：去 Redis 检查是否存在，存在就删除并放行，不存在就拒绝。

这样用户连点两次下单按钮，第一次消费了 token，第二次直接被拦截。返回再进入确认页会重新获取新 token。

## 本地消息表的设计目的

`order_message_log` 表同时存 `ORDER_CREATED` 和 `ORDER_TIMEOUT` 两条记录，因为它们对应两条不同的 MQ 消息。`ORDER_CREATED` 通知下游系统订单已创建，`ORDER_TIMEOUT` 发到延迟队列用于 15 分钟后自动取消未支付订单。

这张表的核心作用是保证消息一定能发出去。消息在 `afterCommit` 里发送，如果那一刻应用崩了或 MQ 不可用，消息就丢了。有了消息表，`MessageRetryTask` 每 30 秒扫描 PENDING 状态的记录重新发送，最多重试 3 次，用数据库的可靠性兜底 MQ 的不可靠性。

## Redis + DB 库存扣减的一致性

当前策略是先扣 Redis 再扣 DB，如果 DB 扣减失败则回滚 Redis。但存在一个缺口：DB 扣减成功后，`createOrder` 后续步骤失败导致事务回滚，DB 库存被还原了，但 Redis 已经扣了没人补回来。

可以注册事务回滚钩子来兜底：

```java
TransactionSynchronizationManager.registerSynchronization(
    new TransactionSynchronization() {
        @Override
        public void afterCompletion(int status) {
            if (status == STATUS_ROLLED_BACK) {
                stockCacheService.rollback(productId, quantity);
            }
        }
    }
);
```

实际电商场景中，Redis 库存短暂少扣（少卖几件）比超卖更可接受。少卖意味着有货卖不出去，等对账修正就好；超卖意味着收了钱发不出货，需要退款赔偿，代价大得多。

## 消费失败的兜底机制

当前项目有两层兜底：RabbitMQ 自动重试（消费者抛异常后消息 requeue）和 `MessageRetryTask` 定时扫描重发。但前者没有限制重试次数会导致无限刷屏，后者只保证消息发出去、不保证消费成功。

完整的方案是配置 RabbitMQ 重试策略加死信队列：

```yaml
spring:
  rabbitmq:
    listener:
      simple:
        retry:
          enabled: true
          max-attempts: 3
          initial-interval: 5000
        default-requeue-rejected: false
```

消费失败最多重试 3 次，仍然失败就进死信队列，由人工排查或死信消费者告警。

## Spring AMQP 的定位

Spring AMQP 是 Spring 对 AMQP 协议的封装，主要用来对接 RabbitMQ，屏蔽连接管理、通道创建、消息序列化、ACK 确认等底层细节。`RabbitTemplate`（发消息）、`@RabbitListener`（收消息）都是它提供的。Spring AMQP 之于 RabbitMQ，就像 MyBatis-Plus 之于 MySQL。

## Lombok 常用注解

- **`@Data`**：自动生成 getter、setter、toString、equals、hashCode。DTO 和 Entity 都用，因为前者需要 getter/setter 做 JSON 序列化，后者需要做字段映射。
- **`@NoArgsConstructor`**：生成无参构造器。Jackson 反序列化时需要先创建空对象再通过 setter 填值。
- **`@AllArgsConstructor`**：生成全参构造器。允许 `new CacheDeleteMessage(permKey, 0)` 一行创建并赋值。

三个经常一起用是因为 `@AllArgsConstructor` 加上后 Java 不再提供默认无参构造器，必须补 `@NoArgsConstructor`，否则 Jackson 反序列化会报错。三个注解各自独立，不存在依赖关系。

## 前端渲染与数据加载

现代前端框架先渲染页面骨架（skeleton），然后组件挂载后异步请求数据，数据回来后自动填充到对应位置。商品详情页一般并行发两个请求：一个拿商品信息（走缓存，快），一个拿库存（实时查询）。分开是因为商品信息变化少适合重度缓存，库存变化频繁需要更接近实时。

## MQ 消息 DTO 的设计

RabbitMQ 的消息体通常用专门的 DTO 类承载，不同用途的消息各自写一个。`CacheDeleteMessage` 只需要 `cacheKey` 和 `retryCount`，`TicketNotifyMessage` 需要 `ticketId`、`oldStatus`、`newStatus`，数据结构完全不同。核心原则是按消息的数据结构来分，不是机械地按消息类型一对一，如果两种消息结构恰好一样，共用一个也可以。

## Mockito 动态 Agent 警告

JDK 21 开始限制动态加载 agent，Mockito 的 inline mock maker 依赖 ByteBuddy agent 触发了警告。这只是警告不影响测试运行。临时消除可以在 `maven-surefire-plugin` 加 `-XX:+EnableDynamicAgentLoading`，长期方案等 Mockito 后续版本适配。

## Swagger 文档自动生成

SpringDoc OpenAPI 自动扫描 Controller 类，根据路径、方法、参数类型、返回值类型生成 API 文档页面。中文描述需要通过 `@Operation(summary = "...", description = "...")` 手动添加，不加也能生成页面，只是没有说明文字。
