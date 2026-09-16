---
title: "从单体到分布式：Spring Boot 微服务架构的渐进式落地指南"
date: "2026-09-16"
updatedAt: "2026-09-16T11:28:01.614Z"
author: "Admin"
summary: "简单介绍spring boot的微服务实现"
tags: "微服务,spring boot,分布式,Spring Cloud"
---

# 从单体到分布式：Spring Boot 微服务架构的渐进式落地指南

## 结论先行

微服务架构的本质不是“拆得越细越好”，而是**用可接受的复杂度换取独立部署与弹性伸缩的能力**。Spring Boot 让单服务开发变简单，Spring Cloud Alibaba（Nacos + Gateway + OpenFeign + Seata）让服务之间的协作变得有章可循。本文按“**建服务 → 能通信 → 管配置 → 控入口 → 保数据**”的脉络，给出每步的代码与决策依据。

***

## 一、先跑通一个：Spring Boot 微服务最小单元

微服务的第一步不是引入一堆 Spring Cloud 依赖，而是让一个服务能独立启动、独立暴露 API。

```java
@RestController
@RequestMapping("/api/user")
public class UserController {
    
    @GetMapping("/info")
    public Map<String, String> getUser() {
        Map<String, String> user = new HashMap<>();
        user.put("id", "1001");
        user.put("name", "Echo_Wish");
        return user;
    }
}
```

这个简单 Controller 揭示了微服务的底线：**服务有清晰的边界（用户信息）、能独立运行、通过 HTTP 对外提供能力**。如果连这个边界都画不清楚，后续的注册中心、网关、分布式事务只会让混乱加倍。

用 Docker 让它脱离主机环境：

```dockerfile
FROM openjdk:17-jdk-slim
COPY target/user-service.jar /app/user-service.jar
EXPOSE 8080
ENTRYPOINT ["java", "-jar", "/app/user-service.jar"]
```

到此，微服务的**物理形态**就具备了：可复制、可部署、可迁移的容器化服务。

***

## 二、让服务互相说话：Nacos 注册中心 + Feign 声明式调用

服务拆分后的第一个现实问题是：订单服务怎么知道用户服务在哪台机器、哪个端口上？如果写成 `http://192.168.1.100:8080`，那每次用户服务扩容或迁移，订单服务都要改代码重新部署。

### 2.1 Nacos 服务注册与发现

Nacos 承担的角色是**服务名到实例列表的映射**。每个微服务启动时把“我在哪”告诉 Nacos，调用方只需问 Nacos“用户服务有哪些实例”。

引入依赖并启用：

```xml
<dependency>
    <groupId>com.alibaba.cloud</groupId>
    <artifactId>spring-cloud-starter-alibaba-nacos-discovery</artifactId>
</dependency>
```

```yaml
spring:
  application:
    name: service-order
  cloud:
    nacos:
      server-addr: localhost:8848
```

启动类加注解：

```java
@SpringBootApplication
@EnableDiscoveryClient
public class OrderApplication { ... }
```

启动后，Nacos 控制台会看到 `service-order` 和 `service-user` 的服务列表。此时通过 `DiscoveryClient` 可以编程式获取实例：

```java
@Autowired
private DiscoveryClient discoveryClient;

List<ServiceInstance> instances = discoveryClient.getInstances("service-user");
```

但日常开发不建议直接用 `DiscoveryClient`——URL 拼接、负载均衡、序列化都要手写。

### 2.2 Feign：声明式服务调用

Feign 让你用“写接口”的方式完成 HTTP 调用，底层自动从 Nacos 获取实例并做负载均衡。

```java
@FeignClient("service-user")
public interface UserClient {
    @GetMapping("/api/user/info")
    Map<String, String> getUserInfo();
}
```

订单服务中直接注入使用：

```java
@RestController
@RequestMapping("/api/order")
public class OrderController {
    
    @Autowired
    private UserClient userClient;
    
    @GetMapping("/detail")
    public Map<String, Object> getOrder() {
        Map<String, String> userInfo = userClient.getUserInfo();
        Map<String, Object> result = new HashMap<>();
        result.put("orderId", "A001");
        result.put("userInfo", userInfo);
        return result;
    }
}
```

对比直接使用 `RestTemplate`，Feign 的优势在于：**URL 不再散落在业务代码中，参数拼装由声明式接口管理，更换调用地址只需改配置**。

Feign 的几个生产级配置值得关注：

| 配置项              | 推荐值         | 原因                |
| ---------------- | ----------- | ----------------- |
| `connectTimeout` | 5000ms      | 避免网络抖动导致长时间阻塞     |
| `readTimeout`    | 5000ms      | 业务处理超时兜底          |
| 日志级别             | `BASIC`（生产） | 记录请求方法、URL、状态码、耗时 |

***

## 三、配置集中管理：Nacos Config

当服务实例从个位数涨到几十个时，“改一个配置、重启全部服务”就变成了一场运维灾难。Nacos Config 解决的问题是：**配置与代码分离，修改配置后服务无需重启即可生效**。

### 3.1 关键文件：bootstrap.yaml

Spring Boot 启动时，`bootstrap.yaml` 的加载优先级**高于** `application.yaml`。Nacos Config 利用这一点，先在 `bootstrap.yaml` 中配置“去哪里拉远程配置”，拉到后再与本地配置合并。

```yaml
# bootstrap.yaml
spring:
  application:
    name: service-user
  profiles:
    active: dev
  cloud:
    nacos:
      server-addr: localhost:8848
      config:
        file-extension: yaml
```

这段配置会让应用自动去 Nacos 拉取 `service-user-dev.yaml` 这个 Data ID 的配置。你只需要在 Nacos 控制台创建同名配置，内容会**覆盖**本地 `application.yaml` 中的对应项。

### 3.2 动态刷新的代价

`@RefreshScope` 注解能让 Bean 在不重启的情况下获取最新配置值。但需要注意：动态刷新并非“零成本”——它通过创建代理对象实现，对高频调用的 Bean 有轻微性能影响。因此建议**只对真正需要热更新的配置使用**，如开关类、限流阈值类配置。

***

## 四、统一入口：Spring Cloud Gateway

微服务对外暴露时，如果让前端记住“用户请求走 8081、订单请求走 8082”，那网关就没有存在的必要了。Gateway 的核心价值是：**所有外部请求统一入口，按路径路由，并在转发前完成鉴权**。

### 4.1 基础路由配置

```yaml
spring:
  cloud:
    gateway:
      routes:
        - id: user-route
          uri: lb://service-user
          predicates:
            - Path=/api/user/**
        - id: order-route
          uri: lb://service-order
          predicates:
            - Path=/api/order/**
```

`lb://` 前缀告诉 Gateway：“不要写死 IP，从 Nacos 获取实例列表并做负载均衡”。

### 4.2 网关鉴权：为什么必须在转发前做

假设有 10 个微服务，如果每个服务都写一套 JWT 校验代码，结果就是：**密钥散落在 10 个地方，改一次算法要改 10 个服务，任何一个服务遗漏都会留下安全漏洞**。

在 Gateway 中实现全局过滤器，统一拦截并校验 Token：

```java
@Component
public class AuthGlobalFilter implements GlobalFilter, Ordered {
    
    @Override
    public Mono<Void> filter(ServerWebExchange exchange, GatewayFilterChain chain) {
        String token = exchange.getRequest().getHeaders().getFirst("Authorization");
        // 校验逻辑...
        if (invalid) {
            exchange.getResponse().setStatusCode(HttpStatus.UNAUTHORIZED);
            return exchange.getResponse().setComplete();
        }
        return chain.filter(exchange);
    }
    
    @Override
    public int getOrder() {
        return -100; // 在 NettyRoutingFilter 之前执行
    }
}
```

Gateway 的过滤器链分为 `pre` 和 `post` 两个阶段：`pre` 逻辑在请求转发到微服务之前执行，`post` 逻辑在响应返回时执行。鉴权逻辑必须放在 `pre` 阶段。

***

## 五、分布式事务：Seata 的 AT 模式

当订单服务调用库存服务扣减库存、再调用支付服务完成支付时，如果支付失败，库存必须回滚——这就是分布式事务要解决的问题。

### 5.1 Seata 的角色模型

Seata 的架构中三个核心角色：

* **TC（Transaction Coordinator）**：独立部署的协调器，维护全局事务状态
* **TM（Transaction Manager）**：业务发起方，定义全局事务边界（`@GlobalTransactional`）
* **RM（Resource Manager）**：每个参与事务的微服务，管理自己的分支事务

### 5.2 AT 模式的核心设计：一阶段提交 + 二阶段补偿

AT 模式与 XA 的最大区别在于：**AT 在第一阶段就完成本地事务提交**，而不是持有锁等待协调。

具体流程：

**第一阶段**：RM 执行业务 SQL 时，Seata 自动拦截并记录数据修改前后的快照到 `undo_log` 表。业务 SQL 与日志记录在同一个本地事务中提交。

**第二阶段**：如果全局事务成功，TC 通知各 RM 异步清理 `undo_log`；如果失败，RM 根据 `undo_log` 中的快照生成反向 SQL 进行回滚。

这意味着 AT 模式以**短暂的数据不一致窗口**换取了显著高于 XA 的并发性能。它适合**对最终一致性可接受**的场景，如电商下单、库存扣减等。

使用方式极其简洁：

```java
@GlobalTransactional
public void createOrder(OrderDTO order) {
    orderMapper.insert(order);       // 本地事务
    inventoryClient.deduct(order);   // Feign 调用，自动纳入全局事务
    paymentClient.pay(order);        // 同上
}
```

`undo_log` 表是 AT 模式的必需基础设施，每个参与事务的数据库都需要创建。

***

## 六、落地建议：什么阶段用什么工具

最后给出一个务实的判断框架：

**服务数量 \< 3 且团队 \< 5 人**：不要引入微服务。单体 + 模块化足够，强行拆分只会增加部署和调试成本。

**需要独立部署与弹性伸缩**：从 Nacos 注册中心 + Feign 开始，这两个是微服务化的“最小可行组合”。

**服务数量 > 5 且配置管理开始痛苦**：引入 Nacos Config。当“改配置要重启多个服务”成为日常困扰时，配置中心的价值自然显现。

**需要统一鉴权与对外屏蔽内部结构**：引入 Gateway。网关的核心价值不在于路由（Nginx 也能做），而在于**在 Java 层实现可编程的鉴权与过滤**。

**跨库事务失败频发**：评估是否真的需要分布式事务。很多“分布式事务”问题通过合理的业务设计（如异步最终一致性 + 对账补偿）可以避免，而不必引入 Seata 的复杂度。确需强一致时，优先考虑 AT 模式。