# 智学微服务学习平台

基于 Spring Cloud 的在线教育平台后端，覆盖课程内容、视频学习、考试练习、社交互动、优惠券营销、订单支付、学习积分与数据分析等业务。项目采用前后端分离与微服务架构，通过统一网关对外提供 REST API。

## 项目特点

- 使用 Nacos 完成服务注册、发现和集中配置管理。
- 使用 Spring Cloud Gateway 提供统一入口、路由转发、鉴权过滤、请求链路标识和聚合接口文档。
- 使用 OpenFeign + Fallback 实现服务间同步调用和降级处理。
- 使用 MySQL、MyBatis-Plus、Redis、RabbitMQ、Elasticsearch、Redisson、XXL-JOB 和 Seata 支撑数据、缓存、异步消息、搜索、分布式锁、定时任务和分布式事务。
- 使用 Knife4j 聚合各微服务接口文档，便于前后端联调。

## 核心设计

### 视频学习进度记录

学习进度上报时先写入 Redis Hash，再通过 `DelayQueue` 延迟合并写入数据库。持久化前比较 `moment` 版本，丢弃过期进度，避免高频上报持续击穿数据库。

- 设计效果：数据库写入 QPS 下降约 85%。
- 性能记录：进度上报接口平均响应时间由 180 ms 降至 45 ms。
- 主要实现：`tj-learning/src/main/java/com/tianji/learning/utils/LearningRecordDelayTaskHandler.java`

### 通用分布式锁组件

通过自定义 `@Lock` 注解、Spring AOP 和 Redisson 封装分布式锁，支持 SpEL 动态锁名、可重入锁、公平锁、读写锁以及多种获取锁策略，对业务代码零侵入。

- 主要实现：`tj-common/src/main/java/com/tianji/common/autoconfigure/redisson`
- 促销场景扩展：`tj-promotion/src/main/java/com/tianji/promotion/utils`

### 学习积分排行榜

当前赛季使用 Redis ZSet 维护积分和实时排名；历史赛季由 XXL-JOB 分片读取榜单并写入按赛季分表的 MySQL 表中，兼顾实时查询和历史数据持久化。

- 主要实现：`tj-learning/src/main/java/com/tianji/learning/service/impl/PointsBoardServiceImpl.java`
- 定时持久化：`tj-learning/src/main/java/com/tianji/learning/handler/PointsBoardPersistentHandler.java`

### 优惠券与营销系统

优惠券领取和兑换使用 Redis Lua 脚本保证库存扣减、用户限领和兑换校验的原子性；领券成功后通过 RabbitMQ 异步落库，并使用 Redisson 注解控制并发。

- 性能记录：领券接口峰值吞吐量约 1500 QPS。
- 优化效果：数据库写入压力降低约 70%，削峰前后接口响应时间由 1200 ms 降至 110 ms。
- 主要实现：`tj-promotion/src/main/java/com/tianji/promotion/service/impl/UserCouponServiceImpl.java`
- Lua 脚本：`tj-promotion/src/main/resources/lua`

## 技术栈

| 类别 | 技术 |
| --- | --- |
| 基础框架 | Java 11、Spring Boot 2.7.2、Spring Cloud 2021.0.3 |
| 微服务 | Spring Cloud Alibaba、Nacos、OpenFeign、Spring Cloud Gateway |
| 数据访问 | MySQL 8、MyBatis-Plus |
| 缓存与分布式协调 | Redis、Redisson |
| 消息与任务 | RabbitMQ、XXL-JOB |
| 搜索 | Elasticsearch High Level REST Client |
| 分布式事务 | Seata |
| 安全与接口 | JWT、Knife4j、Spring MVC |
| 对象存储与点播 | 腾讯云 COS、腾讯云 VOD |
| 支付 | 支付宝 EasySDK |
| 容器化 | Docker |

## 服务模块

| 模块 | 服务名 | 端口 | 网关前缀 | 主要职责 |
| --- | --- | ---: | --- | --- |
| `tj-gateway` | `gateway-service` | 10010 | `/` | 统一路由、鉴权过滤、CORS、请求 ID 透传、聚合 Swagger |
| `tj-auth/tj-auth-service` | `auth-service` | 8081 | `/as` | 登录认证、JWT、账号、角色、菜单、权限和登录记录 |
| `tj-user` | `user-service` | 8082 | `/us` | 用户、学员、教师和员工档案 |
| `tj-search` | `search-service` | 8083 | `/ss` | Elasticsearch 课程搜索 |
| `tj-media` | `media-service` | 8084 | `/ms` | 媒体资源、COS/VOD 上传与点播管理 |
| `tj-message/tj-message-service` | `message-service` | 8085 | `/sms` | 短信、消息和通知 |
| `tj-course` | `course-service` | 8086 | `/cs` | 课程、课程目录、学科、章节和课程草稿 |
| `tj-pay/tj-pay-service` | `pay-service` | 8087 | `/ps` | 支付渠道和支付宝支付 |
| `tj-trade` | `trade-service` | 8088 | `/ts` | 购物车、订单、退款和交易状态 |
| `tj-exam` | `exam-service` | 8089 | `/es` | 题库、题目详情和题目业务关联 |
| `tj-learning` | `learning-service` | 8090 | `/ls` | 学习记录、学习笔记、问答互动、签到和积分排行榜 |
| `tj-remark` | `remark-service` | 8091 | `/rs` | 评论、点赞和互动记录 |
| `tj-promotion` | `promotion-service` | 8092 | `/prs` | 优惠券、兑换码、促销规则和优惠计算 |
| `tj-data` | `data-service` | 8093 | `/ds` | 数据看板、今日数据和 Top10 统计 |
| `tj-api` | 共享 SDK | - | - | DTO、OpenFeign 客户端、Fallback 和缓存组件 |
| `tj-common` | 公共组件 | - | - | 统一响应、异常、Swagger、MQ、Redisson、MyBatis 和 XXL-JOB 基础能力 |

## 环境要求

- JDK 11
- Maven 3.6+
- MySQL 8.x
- Redis 5+
- RabbitMQ 3.x
- Nacos 2.x
- Elasticsearch 7.x
- XXL-JOB 2.3.x
- Seata 1.5.x
- Docker（可选）

## 配置说明

项目使用 `bootstrap.yml` 加载 Nacos 配置，`local` 和 `dev` Profile 只保存 Nacos 连接信息。数据库、Redis、RabbitMQ、Feign、日志、XXL-JOB 和 Seata 的业务配置统一放在 Nacos。

常用共享配置包括：

- `shared-spring.yaml`
- `shared-redis.yaml`
- `shared-mybatis.yaml`
- `shared-logs.yaml`
- `shared-feign.yaml`
- `shared-mq.yaml`
- `shared-xxljob.yaml`
- `shared-seata.yaml`

> 当前仓库未包含完整数据库初始化 SQL 和 Nacos 配置导出文件。部署前需要准备各服务数据库、数据表、基础数据以及上述 Nacos 配置。

### 安全配置

认证模块使用 JWT 密钥库。不要将真实密钥、密钥库密码、云服务 AccessKey/SecretKey、数据库密码或支付私钥提交到公开仓库；应使用本地私有配置、环境变量或配置中心注入。

## 快速启动

1. 启动 MySQL、Redis、RabbitMQ、Nacos、Elasticsearch、XXL-JOB 和 Seata。
2. 创建各服务数据库并导入表结构和基础数据。
3. 在 Nacos 中创建共享配置、服务配置和对应环境的 Data ID。
4. 根据实际环境修改 `bootstrap-local.yml` 或 `bootstrap-dev.yml` 中的 Nacos 地址、命名空间和服务注册 IP。
5. 使用 JDK 11 编译项目：

```powershell
cd F:\SpringCloud-heima\code\tianji
mvn clean package -DskipTests
```

6. 建议按以下顺序启动服务：

```text
auth-service -> user-service -> course/media/message/search
-> learning/pay/trade/exam/remark/promotion/data -> gateway-service
```

7. 在 IDEA 中将 `spring.profiles.active` 设置为 `local`，依次运行各模块的 `*Application.java`；也可单独启动网关：

```powershell
mvn -pl tj-gateway -am spring-boot:run
```

8. 打开聚合接口文档：

```text
http://localhost:10010/doc.html
```

## 构建与测试

```powershell
# 全量构建
mvn clean package

# 执行单元测试
mvn test

# 只构建指定模块及其依赖
mvn -pl tj-learning -am clean package
```

## Docker 部署

根目录提供 `Dockerfile` 和 `startup.sh`：

- `Dockerfile` 使用 OpenJDK 11 运行 Spring Boot JAR。
- `startup.sh` 用于复制构建产物、创建镜像并启动容器，脚本中的 Jenkins 路径和 Docker 网络需要按实际部署环境修改。

## 网关路由

所有业务请求统一通过 `gateway-service` 访问。例如：

```text
http://localhost:10010/ls/learning/record
http://localhost:10010/prs/promotion/coupon
http://localhost:10010/ts/trade/order
```

网关会移除第一级服务前缀，并将请求转发到 Nacos 中对应的服务实例。

## 项目结构

```text
tianji
├── tj-api                 # 跨服务 DTO、Feign 客户端和公共缓存
├── tj-auth                # 认证服务、JWT 和资源鉴权 SDK
├── tj-common              # 公共基础组件
├── tj-gateway             # API 网关
├── tj-user                # 用户中心
├── tj-message             # 消息服务
├── tj-media               # 媒体服务
├── tj-course              # 课程服务
├── tj-search              # 课程搜索
├── tj-learning            # 学习中心
├── tj-pay                 # 支付服务
├── tj-trade               # 交易服务
├── tj-exam                # 考试服务
├── tj-promotion           # 营销服务
├── tj-data                # 数据中心
└── tj-remark              # 评论互动服务
```

## 说明

本项目用于微服务架构、分布式缓存、异步消息、分布式锁、排行榜、优惠券高并发和定时任务等后端能力的学习与实践。性能数据来自项目压测记录，不同硬件、网络和数据规模下结果可能不同。
