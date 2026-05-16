# 知光平台后端（zhiguang_be）

知光平台是一个面向知识获取、知识分享与智能问答的社区型应用。本仓库为知光平台后端服务，基于 **Java 21 + Spring Boot 3.2.4** 构建，围绕用户认证、内容发布、点赞收藏、关注关系、Feed 流、搜索推荐、对象存储、AI 摘要和 RAG 知识问答等核心能力进行设计。

项目目标不是只实现简单的 CRUD，而是尝试以真实业务系统的标准设计一个具备高并发、高可用、可扩展和 AI 能力的知识社区后端。

---

## 1. 项目定位

知光平台主要面向以下场景：

- 用户发布知识文章、学习笔记、Markdown 文档和多媒体内容。
- 用户对内容进行点赞、收藏、评论、关注等互动。
- 首页通过 Feed 流展示个性化内容。
- 通过 Elasticsearch 实现内容搜索和联想建议。
- 通过 AI 生成文章摘要，提升内容消费效率。
- 通过 RAG 知识问答，让用户围绕单篇文章或知识库进行智能问答。

---

## 2. 技术栈

| 分类 | 技术 |
|---|---|
| 语言 | Java 21 |
| Web 框架 | Spring Boot 3.2.4、Spring MVC |
| 安全认证 | Spring Security、OAuth2 Resource Server、JWT 双 Token |
| AI 能力 | Spring AI、DeepSeek、OpenAI-compatible API、RAG |
| 数据访问 | MyBatis、JDBC |
| 数据库 | MySQL |
| 缓存 | Redis、Caffeine |
| 消息队列 | Kafka |
| 数据同步 | Canal |
| 搜索引擎 | Elasticsearch Java API Client |
| 对象存储 | 阿里云 OSS |
| 分布式能力 | Redisson、Lua 脚本、Outbox、binlog 订阅 |
| 工程工具 | Maven、Lombok、Actuator、JUnit、Mockito |

---

## 3. 主要功能模块

### 3.1 用户认证系统

认证系统采用 **Spring Security + JWT 双 Token** 方案，核心设计包括：

- Access Token：短期有效，用于接口访问认证。
- Refresh Token：长期有效，用于刷新 Access Token。
- JWT 使用 RS256 非对称签名。
- Redis 存储 Refresh Token 白名单，支持服务端主动撤销。
- 支持手机号/邮箱验证码登录、第三方登录绑定等扩展场景。
- 通过 Spring Security Resource Server 完成 Bearer Token 校验。

设计目标：

```text
短期访问令牌保证接口访问性能
长期刷新令牌保证用户体验
Redis 白名单支持主动失效和会话控制
RS256 提升 Token 签名安全性
```

---

### 3.2 内容发布系统

发布系统支持知识文章、图片、视频、Markdown 文档等内容发布。

核心设计：

- 图片、视频、Markdown 等静态资源存储到阿里云 OSS。
- 后端生成 OSS 预签名上传地址，前端直传，减少后端带宽压力。
- 支持渐进式发布流程，降低发布过程中的失败影响。
- 接入 DeepSeek / Spring AI 生成文章摘要。
- 后续可扩展审核、草稿、版本管理和定时发布能力。

---

### 3.3 点赞与收藏系统

点赞/收藏属于高频写场景，项目采用异步化与幂等设计：

- 使用 Redis / 位图结构进行快速判重和幂等控制。
- 使用 Kafka 异步写入，降低数据库写压力。
- 支持写聚合，减少高并发下的数据库更新次数。
- 读取异常或缓存缺失时，支持按需重建。
- Kafka 可作为灾难恢复和事件回放的兜底手段。

设计目标：

```text
高并发写入不直接压垮数据库
用户重复点赞不会产生重复业务影响
缓存异常时可以通过底层数据重建
消息系统支持最终一致性和灾难恢复
```

---

### 3.4 用户关系系统

关注/取关系统采用事件驱动设计。

核心设计：

- 主表记录用户关注关系。
- 粉丝列表、关注计数、缓存列表作为派生数据源。
- 在同一事务中写入关注表和 Outbox 表。
- Canal 订阅 MySQL binlog 中的 Outbox 变更。
- 将变更事件投递到 Kafka。
- 消费端异步更新计数、缓存和列表等派生数据。

设计模式：

```text
本地事务 + Outbox + Canal + Kafka + 最终一致性
```

优点：

- 避免业务事务中直接操作多个外部系统。
- 减少跨系统强一致带来的复杂度。
- 通过消息重试和补偿保证最终一致。

---

### 3.5 计数系统

计数系统用于维护文章点赞数、收藏数、用户关注数、粉丝数等数据。

核心设计：

- Redis 作为高性能计数存储。
- Lua 脚本保证计数更新原子性。
- 支持采样一致性校验。
- 支持异常情况下的自愈重建。
- 可结合 Kafka 聚合写入减少数据库压力。

适用场景：

```text
文章点赞数
文章收藏数
用户关注数
用户粉丝数
Feed 互动计数
```

---

### 3.6 Feed 流系统

Feed 流用于首页内容分发。

缓存设计：

```text
本地 Caffeine 缓存
  ↓
Redis 页面缓存
  ↓
Redis 片段缓存
  ↓
数据库 / 搜索引擎回源
```

核心能力：

- 多级缓存降低数据库压力。
- 热点探测识别高频访问内容。
- 热点内容延长缓存时间。
- TTL 随机抖动防止缓存雪崩。
- single-flight 单飞机制避免同一页面并发回源风暴。
- 支持缓存一致性策略，避免内容更新后展示过期数据。

---

### 3.7 搜索系统

搜索系统基于 Elasticsearch 构建。

主要能力：

- 关键词全文检索。
- 标签过滤。
- search_after 游标分页，避免深分页性能问题。
- function_score 融合 BM25 相关性与点赞、收藏等业务权重。
- completion suggester 实现低延迟前缀联想。
- 后续可扩展向量检索和混合检索。

排序思路：

```text
文本相关性 BM25
+ 内容热度
+ 作者权重
+ 发布时间衰减
+ 用户行为反馈
```

---

### 3.8 AI 摘要系统

内容发布后，可调用 AI 模型生成文章摘要。

流程：

```text
用户发布文章
  ↓
提取标题、正文和 Markdown 内容
  ↓
构造摘要 Prompt
  ↓
调用 DeepSeek / Spring AI
  ↓
生成文章摘要
  ↓
保存摘要结果
```

价值：

- 降低用户阅读成本。
- 提高长文内容消费效率。
- 为搜索、推荐和 RAG 问答提供结构化内容。

---

### 3.9 RAG 知识问答系统

RAG 问答系统用于支持用户围绕文章进行智能问答。

核心流程：

```text
用户提问
  ↓
检查文章是否已完成索引
  ↓
读取文章内容
  ↓
文本切分与清洗
  ↓
Embedding 向量化
  ↓
向量检索召回相关片段
  ↓
构造 Prompt
  ↓
调用大模型流式生成
  ↓
返回答案和引用上下文
```

工程设计：

- 文章发布后预索引，减少首次提问等待时间。
- 基于内容 hash / etag 判断是否需要重新索引。
- 删除旧版本切片后写入新版本切片，避免多版本污染。
- 支持向量检索与 Elasticsearch 存储。
- 通过合理分块和 overlap 提升召回质量。
- 后续可加入 rerank、query rewrite 和评估集。

---

## 4. 系统架构概览

```text
前端 React / Vite
  ↓
Spring Boot API 服务
  ↓
认证鉴权层 Spring Security + JWT
  ↓
业务模块
  ├── 用户认证
  ├── 内容发布
  ├── 点赞收藏
  ├── 关注关系
  ├── Feed 流
  ├── 搜索联想
  ├── AI 摘要
  └── RAG 问答
  ↓
基础设施
  ├── MySQL
  ├── Redis
  ├── Kafka
  ├── Canal
  ├── Elasticsearch
  ├── 阿里云 OSS
  └── DeepSeek / Spring AI
```

---

## 5. 项目目录结构

实际目录可能随着开发继续调整，当前建议结构如下：

```text
src/main/java/com/tongji/
  ZhiGuangApplication.java          # Spring Boot 启动类

  config/                           # 配置类
    SecurityConfig.java
    RedisConfig.java
    OssConfig.java
    KafkaConfig.java
    ElasticsearchConfig.java

  controller/                       # HTTP 接口层
    AuthController.java
    UserController.java
    PostController.java
    LikeController.java
    FollowController.java
    FeedController.java
    SearchController.java
    AiController.java

  service/                          # 业务服务层
    AuthService.java
    UserService.java
    PostService.java
    LikeService.java
    FollowService.java
    FeedService.java
    SearchService.java
    RagService.java

  mapper/                           # MyBatis Mapper
    UserMapper.java
    PostMapper.java
    LikeMapper.java
    FollowMapper.java

  domain/                           # 实体 / DTO / VO
    entity/
    dto/
    vo/

  security/                         # 认证与鉴权
    JwtService.java
    TokenService.java
    CurrentUserHolder.java

  cache/                            # 缓存相关
    FeedCacheService.java
    CounterCacheService.java

  mq/                               # Kafka / Canal / Outbox
    producer/
    consumer/
    outbox/

  ai/                               # AI 摘要与 RAG
    summary/
    rag/
    embedding/

  common/                           # 通用响应、异常、工具类
    Result.java
    BaseException.java
    GlobalExceptionHandler.java
```

---

## 6. 本地开发环境

### 6.1 环境要求

| 工具 | 版本建议 |
|---|---|
| JDK | 21 |
| Maven | 3.9+ |
| MySQL | 8.0+ |
| Redis | 6.0+ |
| Kafka | 3.x |
| Elasticsearch | 8.x / 9.x 需与客户端兼容 |
| Docker | 可选 |
| IDE | IntelliJ IDEA / VS Code |

---

### 6.2 克隆项目

```bash
git clone https://github.com/Gardenia-zx/zhiguang_be.git
cd zhiguang_be
```

---

### 6.3 安装依赖

```bash
mvn clean install
```

如果依赖下载失败，可以尝试：

```bash
mvn clean install -U
```

---

### 6.4 配置 application.yml

当前仓库中的 `application.yml` 需要根据本地环境补充配置。示例：

```yaml
server:
  port: 8080

spring:
  application:
    name: zhiguang

  datasource:
    driver-class-name: com.mysql.cj.jdbc.Driver
    url: jdbc:mysql://127.0.0.1:3306/zhiguang?useUnicode=true&characterEncoding=utf8&serverTimezone=Asia/Shanghai
    username: root
    password: your_password

  data:
    redis:
      host: 127.0.0.1
      port: 6379
      password:
      database: 0

  kafka:
    bootstrap-servers: 127.0.0.1:9092

  ai:
    openai:
      api-key: your_api_key
      base-url: https://api.deepseek.com
      chat:
        options:
          model: deepseek-chat

mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.tongji.domain.entity

aliyun:
  oss:
    endpoint: your_endpoint
    access-key-id: your_access_key_id
    access-key-secret: your_access_key_secret
    bucket-name: your_bucket_name

elasticsearch:
  host: 127.0.0.1
  port: 9200
```

注意：

```text
不要把真实数据库密码、OSS 密钥、API Key 提交到 GitHub。
```

---

### 6.5 启动项目

方式一：IDE 启动

```text
运行 com.tongji.ZhiGuangApplication
```

方式二：Maven 启动

```bash
mvn spring-boot:run
```

方式三：打包运行

```bash
mvn clean package -DskipTests
java -jar target/zhiguang-1.0-SNAPSHOT.jar
```

---

## 7. 核心接口示例

以下接口仅作为 README 说明示例，具体路径以实际 Controller 为准。

### 7.1 用户登录

```http
POST /api/auth/login
Content-Type: application/json
```

```json
{
  "account": "user@example.com",
  "password": "123456"
}
```

---

### 7.2 刷新 Token

```http
POST /api/auth/refresh
Content-Type: application/json
```

```json
{
  "refreshToken": "xxx"
}
```

---

### 7.3 发布文章

```http
POST /api/posts
Authorization: Bearer access_token
Content-Type: application/json
```

```json
{
  "title": "Redis 缓存设计总结",
  "content": "文章正文...",
  "tags": ["Redis", "后端", "缓存"]
}
```

---

### 7.4 点赞内容

```http
POST /api/posts/{postId}/like
Authorization: Bearer access_token
```

---

### 7.5 关注用户

```http
POST /api/users/{targetUserId}/follow
Authorization: Bearer access_token
```

---

### 7.6 搜索内容

```http
GET /api/search?keyword=Redis&pageSize=20
```

---

### 7.7 AI 文章摘要

```http
POST /api/ai/summary
Authorization: Bearer access_token
Content-Type: application/json
```

```json
{
  "postId": "123456"
}
```

---

### 7.8 RAG 知识问答

```http
POST /api/ai/rag/chat
Authorization: Bearer access_token
Content-Type: application/json
```

```json
{
  "postId": "123456",
  "question": "这篇文章主要讲了什么？"
}
```

---

## 8. 关键设计说明

### 8.1 JWT 双 Token 认证设计

```text
用户登录
  ↓
校验账号密码 / 验证码 / 第三方身份
  ↓
生成 access_token 和 refresh_token
  ↓
access_token 返回给前端访问接口
  ↓
refresh_token 写入 Redis 白名单
  ↓
access_token 过期后使用 refresh_token 换取新 access_token
```

优点：

- Access Token 生命周期短，降低泄露风险。
- Refresh Token 可被服务端撤销。
- Redis 白名单支持强制下线、退出登录和风控封禁。
- 适合前后端分离系统。

---

### 8.2 Outbox + Canal + Kafka 最终一致性

关注、点赞、计数等场景需要同时更新多个数据源，如果全部放到一个同步事务里，会增加系统耦合和失败风险。

本项目采用：

```text
业务表更新
  ↓
同事务写 Outbox 表
  ↓
Canal 订阅 binlog
  ↓
投递 Kafka
  ↓
消费者异步更新缓存、计数、列表等派生数据
```

优点：

- 本地事务保证业务表和 Outbox 表一致。
- Kafka 解耦下游消费。
- 消费失败可以重试。
- 支持最终一致性。

---

### 8.3 Feed 多级缓存设计

```text
用户请求 Feed
  ↓
Caffeine 本地缓存
  ↓
Redis 页面缓存
  ↓
Redis 片段缓存
  ↓
数据库 / Elasticsearch 回源
```

优化点：

- 热点页面走本地缓存，减少 Redis 压力。
- Redis 页面缓存减少重复分页查询。
- 片段缓存支持局部更新。
- TTL 随机化防止缓存雪崩。
- single-flight 防止并发回源风暴。

---

### 8.4 搜索排序设计

搜索不仅依赖文本相关性，还需要结合业务因素。

排序公式可以抽象为：

```text
最终得分 = BM25 文本相关性
        + 点赞/收藏/评论等热度权重
        + 发布时间衰减
        + 作者质量权重
```

使用 Elasticsearch 的 `function_score` 可以把文本相关性和业务权重融合起来。

---

### 8.5 RAG 问答设计

RAG 的核心不是让大模型凭空回答，而是先从知识库中召回相关内容，再让模型基于上下文回答。

```text
问题
  ↓
向量检索相关文章片段
  ↓
构造 Prompt
  ↓
LLM 基于片段回答
  ↓
返回答案和来源
```

后续可继续优化：

```text
query rewrite
hybrid search
rerank
多路召回
答案引用
RAG 评估集
```

---

## 9. 数据库设计建议

核心表可包括：

```text
user                    用户表
user_auth               用户认证信息表
post                    内容/文章表
post_media              内容媒体资源表
post_like               点赞表
post_favorite           收藏表
follow_relation         关注关系表
comment                 评论表
outbox_event            事务消息表
rag_document            RAG 文档表
rag_chunk               RAG 切片表
search_index_log        搜索索引同步日志表
```

后续可根据模块继续拆分。

---

## 10. 项目亮点

1. **安全认证体系完整**：基于 Spring Security + JWT 双 Token + Redis 白名单实现可撤销的无状态认证。
2. **高并发互动设计**：点赞、关注、计数等模块结合 Redis、Kafka、Lua、Outbox、Canal，实现高性能与最终一致性。
3. **Feed 缓存体系**：采用 Caffeine + Redis 多级缓存，结合热点检测、TTL 抖动和 single-flight 降低回源压力。
4. **搜索能力完善**：基于 Elasticsearch 实现关键词检索、标签过滤、深分页优化和联想建议。
5. **AI 能力融合业务**：通过 Spring AI 接入 DeepSeek，支持文章摘要与 RAG 知识问答。
6. **工程化意识较强**：围绕幂等、缓存一致性、异步处理、可扩展性和高可用设计核心模块。

---

## 11. 后续规划

### 11.1 完善基础业务

- 评论系统
- 私信系统
- 内容举报
- 内容审核
- 用户等级体系
- 管理后台

### 11.2 完善 AI 能力

- RAG 问答评估体系
- 多路召回 + rerank
- 文章自动标签生成
- 用户学习路线推荐
- 内容质量评分

### 11.3 完善工程能力

- 接入 Prometheus + Grafana
- 接入链路追踪
- Kafka 消费监控
- Redis 热点监控
- Elasticsearch 慢查询分析
- CI/CD 自动化部署

### 11.4 完善生产部署

- Docker Compose 本地编排
- Nginx 网关
- HTTPS 配置
- 灰度发布
- 容器化部署

---

## 12. 开发规范建议

### 12.1 分层规范

```text
Controller：只负责参数接收和响应返回
Service：负责业务逻辑编排
Repository / Mapper：负责数据访问
DTO：前端请求参数
VO：前端响应对象
Entity：数据库实体
Config：配置类
Common：通用结果、异常、工具类
```

### 12.2 异常规范

建议统一响应格式：

```json
{
  "success": false,
  "code": "AUTH_TOKEN_EXPIRED",
  "message": "登录已过期，请重新登录",
  "data": null
}
```

### 12.3 日志规范

建议记录：

```text
traceId
userId
接口路径
请求耗时
异常堆栈
关键业务 ID
```

### 12.4 安全规范

请不要提交：

```text
数据库密码
Redis 密码
OSS AccessKey
DeepSeek / OpenAI API Key
JWT 私钥
真实用户隐私数据
```

---

## 13. 常见问题

### 13.1 Maven 依赖下载失败

可以尝试：

```bash
mvn clean install -U
```

或者检查本地 Maven 镜像源配置。

---

### 13.2 启动时报数据库连接失败

检查：

```text
MySQL 是否启动
数据库是否创建
用户名密码是否正确
application.yml 中连接地址是否正确
```

---

### 13.3 Redis 连接失败

检查：

```text
Redis 是否启动
端口是否为 6379
密码是否配置正确
防火墙是否拦截
```

---

### 13.4 Elasticsearch 连接失败

检查：

```text
Elasticsearch 是否启动
版本是否与客户端兼容
端口是否为 9200
是否启用了安全认证
```

---

### 13.5 AI 调用失败

检查：

```text
API Key 是否正确
base-url 是否正确
模型名称是否正确
网络是否能访问模型服务
```

---

## 14. 项目总结

知光平台后端是一个综合性的知识社区后端项目，覆盖了认证、内容、互动、关系、Feed、搜索、AI 摘要和 RAG 问答等多个模块。项目在设计上重点关注高并发、缓存一致性、异步解耦、最终一致性、搜索体验和 AI 工程化落地。

该项目适合作为 Java 后端综合实践项目，也适合作为面试中展示系统设计、缓存设计、消息队列、搜索系统、AI 应用工程化能力的核心项目。
