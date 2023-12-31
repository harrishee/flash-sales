# Flash Sales

## 项目介绍

本项目是一个高并发抢购系统，旨在模拟实际场景中的商品抢购活动。抢购作为一种高度竞争的销售模式，涉及到大量用户瞬间的请求，对库存资源进行争夺。在这种情况下，容易出现数据库压力过大或数据不一致等问题

### 核心特点

- `高并发读`：支持大量用户同时访问系统，实时获取商品信息
- `高并发写`：处理瞬间涌入的抢购请求，保证资源的合理分配
- `库存争夺`：模拟商品抢购活动，用户竞争有限库存资源

### 难点与挑战

1. `数据库压力`：大量瞬时请求容易导致数据库负载过重，进而影响系统稳定性和性能
2. `数据一致性`：并发请求可能导致数据不一致，如库存超卖等情况
3. `防刷策略`：防止恶意用户通过刷单等手段影响正常抢购活动

### 解决方案

1. `缓存优化`：使用 Redis 缓存热门商品信息，减轻数据库压力
2. `队列削峰`：使用消息队列对瞬时请求进行削峰平稳处理，防止数据库崩溃
3. `分布式锁`：引入分布式锁机制，保证抢购操作的原子性，避免竞态条件
4. `限流措施`：实施限流策略，限制每个用户的请求频率，防止恶意刷单

<br>

## 项目发展

### 1. 超售问题

#### 1.1 feat: basic sell
测试接口：`/sale/processSaleNoLock`，脚本：`fs-basic-sell`

完成基本的下单功能，但是存在的问题：

1. QPS 并不高 ~1100，未进行各类优化。每次请求都会去数据库查询库存，然后再去更新库存，数据库压力非常大
2. 在高并发环境下，多个请求同时读取了相同的库存数量，然后都减库存并创建了订单，从而出现了超卖的情况

#### 1.2 feat: sql optimistic lock
测试接口：`/sale/processSaleOptimisticLock`，脚本：`fs-lock-sell`

在之前的基础上，引入了 SQL 乐观锁机制来解决高并发时的超卖问题：`即在更新库存和锁定库存的同时，检查 available_stock 是否大于 0`。
当多个请求尝试同时执行这个 SQL 更新语句时，只有一个请求能够成功地执行更新操作，而其他请求则因为条件不满足而无法执行更新

但是，还是存在问题：

1. 乐观锁通常需要在更新数据时进行额外的比较和验证，以确定数据是否被其他操作修改过。这可能增加数据库操作的负担，可能导致性能下降
2. 每个抢购请求还是需要去数据库检验库存，然后再去更新库存，仍然导致数据库的压力过大

这种方法旨在通过利用乐观锁机制确保数据一致性，但是并没有解决数据库压力过大的问题

#### 1.3 feat: Redis Lua Script
测试接口：`/sale/processSaleCache`，脚本：`fs-cache-sell`

为了解决 数据库乐观锁 方案中的问题，引入了有高速内存读写的 Redis 作为缓存和 Lua 脚本来实现在缓存中库存的原子性更新

1. 缓存库存信息，大部分数据读取请求都被 Redis 给挡住了，保护了 mysql
2. 检查 Redis 库存 和 扣减 Redis 库存 是两个操作。通过 Lua 脚本把这两步操作合并成一个整体，保证了原子性
3. 哪怕 Redis 侧放行，可以创建订单了，到 mysql 的时候还是会再检查一次

```
用户 -> Lua 读 Redis 库存并减扣 -> 扣减失败 -> 抢购结束
                            \
                             -> 扣减成功 -> 锁定数据库库存，创建订单 -> 付款 -> 减少数据库库存
```

首先通过 Redis 预热，将商品库存数量加载到 Redis 中。然后，每次抢购请求到来时，首先检查 Redis 中的库存数量，如果库存不足，则直接返回抢购失败；
否则，执行 Lua 脚本，将 Redis 中的库存数量减 1，并将减 1 后的库存数量返回。如果减 1 后的库存数量小于 0，则表示抢购失败，直接返回；否则，可以进行抢购成功的后续操作

#### 4. feat: process order by mq
测试接口：`/sale/processSaleCacheMq`，脚本：`fs-cache-mq-sell`

瞬间流量冲击下，需要进行订单流量的削峰填谷。利用 MQ 流量削峰，做异步处理从而减少数据库的瞬时负载。
本次更新了创建订单后通过 MQ 发送同步消息来处理创建订单，异步消息来扣减/回滚最终库存

这里采用延时消息。原本 createOrder 在订单创建后立刻发送订单创建消息：
```java
messageSender.sendMessage("new_order", JSON.toJSONString(order));
```

后面再加一个发送订单付款状态校验的延时消息，延时根据 delayTimeLevel 决定：
```java
messageSender.sendDelayMessage("pay_check", JSON.toJSONString(order), delayTimeLevel);
```

目前新订单请求仍然是发送的实时消息。后续可以考虑将其改为延时消息，但这样用户抢购后需等待一段时间才能得到响应，用户体验不好

还有，扣减库存可以发生在：

A. 创建订单时扣减
- 优点 1：逻辑清晰、简单
- 优点 2：符合业务需求，防止超卖情况出现
- 缺陷 1：如果用户未支付或支付失败，怎么处理?
- 缺陷 2：未支付订单会占用有效库存，其他用户无法购买

B. 支付时扣减
- 优点 1：能够避免用户下单不支付，占用库存情况
- 缺陷 1：会出现订单创建成功，但是支付时发现没有库存了

C. 采用：创建订单时锁定，支付时扣减，分段状态

1 创建订单时锁定库存 createOrder：
`boolean lockStockResult = activityService.lockStock(activityId);`

2 付款成功扣减库存 PayDoneListener：
`activityMapper.deductLockById(order.getActivityId());`

3 订单关闭冻结库存回补 PayCheckListener：
`activityMapper.revertStockById(orderNoInfo.getActivityId());`

#### 5. feat: in-memory marking
使用了一个名为 `EmptyStockMap` 的映射来在内存中标记活动的库存是否为空。这样可以避免不必要地访问 Redis 数据库来检查库存状态，进一步提升性能

#### 6. feat: distributed lock

分布式锁：
- 进来一个线程先占位，当别的线程进来操作时，发现已经有人占位了，就会放弃或者稍后再试 线程操作执行完成后，需要调用del指令释放位子
- 为了防止业务执行过程中抛异常或者挂机导致del指定没法调用形成死锁，可以添加超时时间

但是这样如果业务非常耗时会紊乱，解决方案：
- 尽量避免在获取锁之后，执行耗时操作
- 将锁的 value 设置为一个随机字符串，每次释放锁的时候，都去比较随机字符串是否一致，如果一致，再去释放，否则不释放
- 释放锁时要去 1.查看所得 value，2.比较 value 是否正确，3.释放锁 总共三个步骤，这三个步骤不具备原子性
- 所以，可以使用 Lua 脚本来确保这三个步骤的原子性

### 2. 页面优化

#### 2.1 feat: Page Caching
Introducing Redis as a page cache solution, reducing database queries and accelerating page loading, QPS ~3700

Technologies Used:
1. Thymeleaf template engine and Spring MVC configuration are employed for page rendering and caching.
2. Redis is chosen as the caching solution, utilizing RedisTemplate for cache read and write operations.

How it Works:
1. It first checks if the HTML content for "activityAll" is cached in Redis. If found, it serves as a cache hit, and the cached content is retrieved and returned.
2. If the content is not cached, the method generates the HTML content, caches it in Redis for future requests, and returns it to the user, acting as a cache miss.

Performance Improvement:
1. Redis' fast in-memory read/write capabilities significantly accelerate page loading.
2. By setting a cache expiration time of 60 seconds, data remains fresh while minimizing database queries.

Limitations and Solutions:
1. Transmitting entire pages to the frontend can lead to large data transfers. To mitigate this, consider only transmitting essential data.
2. AJAX-based data retrieval instead of modelandview-based page rendering.

#### 2.2 feat: Static Page for Commodity Detail
把前端页面放在静态资源目录下，后端只负责提供数据

### 3. 接口优化

#### 3.1 feat: limit user purchase
限流策略：限制每个用户的购买数量，防止恶意刷单

```java
void addLimitMember(long activityId, String userId);

boolean isInLimitMember(long activityId, String userId);

void removeLimitMember(Long activityId, String userId);
```

通过 Redis 来进行记录，设一个 key 为 activity_limited_user，value 为一个 set，里面存放的是用户的 id。
每次请求过来，先去 Redis 里面查一下，如果有就说明已经购买过了，没有就执行之前抢购的逻辑，抢购成功后，再把用户 id 放到 Redis 里面去

#### 3.2 feat: sentinel flow control
限流策略：通过 Sentinel 实现限流，防止不断重复访问接口造成数据库压力过大

#### 3.3 feat: hide sale api
如果抢购接口地址对外公开，恶意攻击者可以利用自动化工具发送大量请求，尝试抢购大量商品，从而导致系统负载过高，甚至瘫痪。通过隐藏接口地址，可以减少恶意攻击和刷单的可能性

#### 3.4 feat: verify code
验证码可以有效防止恶意攻击者通过自动化工具进行刷单。在抢购前，需要先输入验证码，才能进行抢购操作

但是感觉用户体验一般，就先不加了

<br>

## 项目配置类

### 1. Redis 连接池选择
在项目中，考虑性能、可靠性以及适用性等方面的因素，我选择了 Lettuce 作为 Redis 连接池的实现：
1. `性能`： Lettuce 在性能方面通常比 Jedis 更好。Lettuce 是基于 Netty 库构建的，它使用异步、非阻塞的方式处理连接和命令操作，因此在高并发场景下可以更有效地利用资源，提供更好的性能
2. `线程模型`： Jedis 使用阻塞的同步方式处理连接和操作，这可能在高并发时导致线程阻塞，从而影响性能。Lettuce 利用了 Netty 的事件驱动模型，避免了线程阻塞，使得每个连接可以处理多个操作，有助于更好地处理并发请求
3. `连接管理`： Lettuce 提供了更灵活的连接管理和连接池配置选项，可以更精细地控制连接的创建、复用和销毁等
4. `响应模式`： Lettuce 支持同步、异步和响应式的操作模式，这使得它适用于不同的编程风格和需求

### 2. Spring MVC 配置
在 Spring MVC 的配置方面，针对本项目进行了以下设置：
1. `添加自定义参数解析器`：UserArgumentResolver，通过从 Cookie 中获取 ticket，并通过该标识获取对应的用户，未来可能换成 interceptor
2. `静态资源的处理`：将所有请求路径映射到 classpath:/static/ 目录下，数据库图片路径直接选 /img/xxx.jpg

### 3. MQ 选择
采用了 RocketMQ 作为消息队列：
1. RocketMQ 数据吞吐量比起 RabbitMQ 更高，架构设计是基于分布式消息存储和多副本同步复制机制的，可以实现快速、高可靠性的消息传输
2. RocketMQ 使用了 Netty 作为网络通信框架，可以高效地处理大量的网络连接请求和数据传输；而 RabbitMQ 使用的 Erlang VM 作为底层支持，相对于 Netty 的性能要低一些

<br>

## 一些思考

### 1. 如何处理超时任务？
处理超时任务有两种主流方案：
1. `定时轮询`：1. 时效性差、2. 容易挤压、3. 效率低
2. `延时消息`：1. 性能可靠、2. 时效性好、3. 低消耗

1 定时轮询：通过定时任务实现轮询

- 时效性差：如果每分钟轮询一次，那么订单取消的最大误差就有一分钟
- 容易挤压：如果一分钟之内有大量数据，但是一分钟内没处理完，那么下一分钟的就会顺延
- 效率低：需要遍历资源，不适合拥有大量数据的项目

2 延时消息：通过延时消息实现延时触发

- 性能可靠：异常退出不丢数据
- 时效性好：时间精度高，秒级
- 低消耗：不需要遍历，系统资源低消耗