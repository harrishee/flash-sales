# Flash Sales

## 项目介绍

本项目是一个高并发抢购系统，旨在模拟实际场景中的商品抢购活动。抢购作为一种高度竞争的销售模式，涉及到大量用户瞬间的请求，对库存资源进行争夺。在这种情况下，容易出现数据库压力过大、系统崩溃或数据不一致等问题

### 核心特点

- `高并发读`：支持大量用户同时访问系统，实时获取秒杀商品信息
- `高并发写`：处理瞬间涌入的秒杀请求，保证资源的合理分配
- `库存争夺`：模拟商品秒杀活动，用户竞争有限库存资源

### 难点与挑战

1. `数据库压力`：大量瞬时请求容易导致数据库负载过重，进而影响系统稳定性和性能
2. `数据一致性`：并发请求可能导致数据不一致，如库存超卖等情况
3. `竞态条件`：多个请求同时竞争有限的库存资源，需要确保资源分配的原子性和正确性
4. `防刷策略`：防止恶意用户通过刷单等手段影响正常秒杀活动

### 解决方案

1. `缓存优化`：使用 Redis 缓存热门商品信息，减轻数据库压力
2. `队列削峰`：使用消息队列对瞬时请求进行削峰平稳处理，防止数据库崩溃
3. `分布式锁`：引入分布式锁机制，保证秒杀操作的原子性，避免竞态条件
4. `限流措施`：实施限流策略，限制每个用户的请求频率，防止恶意刷单

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
2. `静态资源的处理`：将所有请求路径映射到 classpath:/static/ 目录下，以确保前端所需的静态资源能够被正确加载和展示

<br>

## 项目发展

### 超售问题

#### 1. feat: basic sell

测试接口：`/sale/processSaleNoLock`，脚本：`fs-basic-sell`，完成基本的下单功能，但是存在的问题：

1. QPS 并不高 ~1100，未进行各类优化。每次请求都会去数据库查询库存，然后再去更新库存，数据库压力非常大
2. 高并发时出现超卖，库存数量为负数，订单数超出

在高并发环境下，多个请求同时读取了相同的库存数量，然后都尝试将库存减少至零，并创建了订单，从而出现了订单数超出的情况

#### 2. feat: sql optimistic lock

在之前的基础上，我引入了 SQL 乐观锁机制来解决高并发时的超卖问题：`即在更新库存和锁定库存的同时，检查 available_stock 是否大于 0`。如果条件成立，执行更新操作；否则，不进行任何操作，从而有效地防止了超卖现象的发生

当多个请求尝试同时执行这个 SQL 更新语句时，只有一个请求能够成功地执行更新操作，而其他请求则因为条件不满足而无法执行更新

存在的问题：

1. 乐观锁通常需要在更新数据时进行额外的比较和验证，以确定数据是否被其他操作修改过。这可能增加数据库操作的负担，可能导致性能下降
2. 每个抢购请求都需要去数据库 检验库存，然后再去更新库存，导致数据库的压力过大

这种方法旨在通过利用乐观锁机制确保数据一致性，从而防止超卖现象的发生。然而，需要注意潜在的性能影响以及额外验证步骤可能引入的数据库负担。可能需要进一步优化，以在数据完整性和系统性能之间取得平衡

#### 3. feat: Redis Lua Script

为了解决 数据库乐观锁 方案中的问题，我引入了 Redis Lua 脚本来实现库存的原子性更新。通过 Redis 的高速内存读写，我决定缓存库存信息，并通过 Lua 脚本来确保库存的原子性更新

1. 缓存库存信息，大部分数据读取请求都被 Redis 给挡住了，保护了 mysql
2. 检查 Redis 库存 和 扣减 Redis 库存 是两个操作。通过 Lua 脚本把这两步操作合并成一个整体，保证了原子性
3. 哪怕 Redis 侧放行，可以创建订单了，到 mysql 的时候还是会再检查一次

```
用户 -> Lua 读 Redis 库存并减扣 -> 扣减失败 -> 抢购结束
                            \
                             -> 扣减成功 -> 锁定数据库库存，创建订单 -> 付款 -> 减少数据库库存
```

首先通过 Redis 预热，将商品库存数量加载到 Redis 中

#### 4. feat: process order by mq

瞬间流量冲击下，需要进行订单流量的削峰填谷。利用 MQ 流量削峰，做异步处理从而减少数据库的顺时负载

本方案选择将 RocketMQ 作为消息队列系统，其出色的高吞吐量和消息可靠性使其非常适合处理大规模消息流。
在此方法中，新订单请求的处理与即时响应解耦，创建了一个缓冲区，用于吸收突发的流量，并将处理负载分散到一段时间内：

1. `消息队列设置`： 将 RocketMQ 集成到体系结构中，允许将新订单请求发布到 new_order 主题的队列中
2. `消费者实现`： 当用户发起订单请求时，与订单相关的信息将被推送到 new_order 队列。然后，消费者会提取这些消息并执行创建订单和库存操作

目前新订单请求仍然是发送的实时消息。后续可以考虑将其改为延时消息，用户提交订单后，先返回订单创建中排队中

后续的订单支付校验就可以通过延时消息来完成


#### 5. feat: delayed order handling

处理超时任务，第一种方案是`定时轮询`，但是：1. 时效性差、2. 容易挤压、3. 效率低

所以这里采用`延时消息`的方案来处理超时任务，因为：1. 性能可靠、2. 时效性好、3. 低消耗

原本 createOrder 在订单创建后后发送订单创建消息：
```java
messageSender.sendMessage("new_order", JSON.toJSONString(order));
```

这个基础上在后面再加一个发送订单付款状态校验的延时消息，延时根据 delayTimeLevel 决定：
```java
messageSender.sendDelayMessage("pay_check", JSON.toJSONString(order), delayTimeLevel);
```

本次主要是一个功能性的实现，异步处理来关闭超时未支付的订单，保证了数据一致性


#### 6. feat: limit user purchase

限流策略：限制每个用户的购买数量，防止恶意刷单

```java
void addLimitMember(long activityId, String userId);

boolean isInLimitMember(long activityId, String userId);

void removeLimitMember(Long activityId, String userId);
```

通过 Redis 来进行记录，设一个 key 为 activity_limited_user，value 为一个 set，里面存放的是用户的 id。
每次请求过来，先去 Redis 里面查一下，如果有就说明已经购买过了，没有就执行之前抢购的逻辑，抢购成功后，再把用户 id 放到 Redis 里面去

#### 7. feat: segmented stock states

扣减库存可以发生在：

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

`boolean lockStockResult = activityMapper.addLockAndDeductAvaById(order.getActivityId());`

2.1 付款成功扣减库存 payOrder：

`messageSender.sendMessage("pay_done", JSON.toJSONString(order));`

2.2 付款成功扣减库存 PayDoneListener：

`activityMapper.deductLockById(order.getActivityId());`

3 订单关闭冻结库存回补 PayCheckListener：

`activityMapper.addAvaAndDeductLockById(orderNoInfo.getActivityId());`

#### 7. tmp

分布式锁：
- 进来一个线程先占位，当别的线程进来操作时，发现已经有人占位了，就会放弃或者稍后再试 线程操作执行完成后，需要调用del指令释放位子
- 为了防止业务执行过程中抛异常或者挂机导致del指定没法调用形成死锁，可以添加超时时间

但是这样如果业务非常耗时会紊乱，解决方案：
- 尽量避免在获取锁之后，执行耗时操作
- 将锁的 value 设置为一个随机字符串，每次释放锁的时候，都去比较随机字符串是否一致，如果一致，再去释放，否则不释放
- 释放锁时要去 1.查看所得 value，2.比较 value 是否正确，3.释放锁 总共三个步骤，这三个步骤不具备原子性
- 所以，可以使用 Lua 脚本来确保这三个步骤的原子性

Lua脚本优势：
- 使用方便，Redis 内置了对 Lua 脚本的支持
- Lua 脚本可以在 Rdis 服务端原子的执行多个 Redis 命令
- 由于网络在很大程度上会影响到 Redis 性能，使用 Lua 脚本可以让多个命令一次执行，可以有效解决网络给 Redis 带来的性能问题

使用Lua脚本思路：
- 提前在 Redis 服务端写好 Lua 脚本，然后在 java 客户端去调用脚本
- 可以在 java 客户端写 Lua 脚本，写好之后，去执行。需要执行时，每次将脚本发送到 Redis 上去执行

### 页面优化

#### 1. feat: Page Caching

通过 Thymeleaf 模板引擎和 Spring MVC 配置，实现了以下页面缓存优化：

1. 商品列表页缓存：对商品列表页面进行缓存到 Redis，有效减少数据库查询次数，提高访问速度。缓存时限设置为 60 秒，确保数据及时更新
2. 商品详情页缓存：类似的对商品详情页面进行缓存，后续会使用静态页面

选择 Redis 作为缓存，利用 RedisTemplate 进行缓存的读取和写入操作。通过 Redis 的高速内存读写，可以有效加速页面的加载。页面缓存优化后
QPS ~3700

1. 在 Controller 中，使用 @RequestMapping 注解指定 produces = "text/html;charset=utf-8"，将渲染后的页面以 HTML
   格式存储在 Redis 中
2. 设置适当的缓存时限，根据业务需要进行调整，以确保数据及时更新

缺陷和解决方案：

这时候传输给前端的是整个页面，数据量其实是非常大的。需要把页面给拆分出来，因为很多内容其实是不需要传输的。需要传输的其实也就是那几个需要经常变动的数据。
所以我们可以把页面和我们专门的数据拆分出来：

1. 拆分出来的页面给前端，利用浏览器的缓存做一下缓存，减少数据量的传输
2. 现在都是前后端分离，并不是通过 modelandview 的方式来渲染页面，而是通过 ajax 的方式来获取数据，然后前端自己渲染页面

#### 2. feat: Static Page for Commodity Detail

把前端页面放在静态资源目录下，后端只负责提供数据


### 接口优化

思路:减少数据库访问

1. 系统初始化，把商品库存数量加载到 Redis
2. 收到请求，Redis 预减库存。库存不足，直接返回。否则进入第3步
3. 请求入队，立即返回排队中
4. 请求出队，生成订单，减少库存
5. 客户端轮询，是否秒杀成功

#### 1. 接口优化 v1

1. `使用内存标记减少 Redis 访问`：seckillController 中引入了一个 `EmptyStockMap（Map<Long, Boolean>）` 来在内存中标记商品的库存状态。这样当库存为空时，可以通过内存中的标记立即返回，而无需频繁地访问 Redis，从而减轻了服务器的负担
2. `预减库存`：通过使用 Redis 的原子减法操作（decrement），来在内存中减少库存。这样可以避免并发下的超卖问题。如果减库存后库存小于0，会将内存标记设置为true，并将库存还原，表示该商品已售罄。
3. `消息队列异步处理请求`：通过MQSender将秒杀请求异步发送到消息队列中，而不是立即执行秒杀操作。这样可以将请求削峰，提高系统的并发处理能力。在第一阶段处理了库存的逻辑，第二阶段在消息队列中逐个处理秒杀请求。
4. `系统初始化预热`：在系统初始化时将商品库存数量加载到 Redis 中。这样可以避免系统刚启动时因为大量请求导致的 Redis 访问峰值，提前将库存数据加载到内存中

#### 2. 接口优化 v2

`消息队列`：引入了消息队列，可以异步处理来削峰，但目前未处理

但是 Redis 的库存有问题，因为原因在于 Redis 没有做到原子性。可以用锁去解决

### 安全优化

#### 1. 秒杀接口地址隐藏

`防止恶意攻击和刷单`： 如果秒杀接口地址对外公开，恶意攻击者可以利用自动化工具发送大量请求，尝试秒杀大量商品，从而导致系统负载过高，甚至瘫痪。通过隐藏接口地址，可以减少恶意攻击和刷单的可能性

还可以添加 `验证码` 和 `接口限流` 来进一步提升，目前先不实现，不好测试

<br>

## 一些思考

### 如何处理超时任务？

1. 定时轮询：通过定时任务实现轮询

- 时效性差：如果每分钟轮询一次，那么订单取消的最大误差就有一分钟
- 容易挤压：如果一分钟之内有大量数据，但是一分钟内没处理完，那么下一分钟的就会顺延
- 效率低：需要遍历资源，不适合拥有大量数据的项目

2. 延时消息：通过延时消息实现延时触发

- 性能可靠：异常退出不丢数据
- 时效性好：时间精度高，秒级
- 低消耗：不需要遍历，系统资源低消耗



## Review

#### rabbitmq 的 4 种模式：

1. Direct Exchange：直连交换机，根据消息携带的路由键将消息投递给对应队列
2. Fanout Exchange：扇形交换机，将消息投递给所有绑定到当前交换机的队列
3. Topic Exchange：主题交换机，根据消息携带的路由键将消息投递给符合匹配规则的队列
4. Headers Exchange：首部交换机，根据消息携带的首部信息进行匹配，匹配成功则投递给对应队列