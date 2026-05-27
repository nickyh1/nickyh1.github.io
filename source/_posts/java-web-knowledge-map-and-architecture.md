---
title: Java Web 项目知识地图与架构风格选型
date: 2026-05-28
tags:
  - Java
  - Spring Boot
  - 架构设计
  - DDD
  - 微服务
categories:
  - 知识笔记
---

## 典型四层职责

Java Web 项目最常见的分层方式是四层结构，每一层有明确的职责边界。

**展现层**（Controller）负责接收 HTTP 请求、参数校验、返回 JSON 或 HTML，技术上对应 Spring MVC。**应用层**（Service）负责编排业务用例、管理事务边界、调用其他服务，用 `@Transactional` 控制原子性。**领域层**（Entity + 领域方法）承载纯业务规则和状态机，是业务逻辑的核心。**基础设施层**（Repository、Redis DAO）负责读写外部系统，包括数据库、缓存和消息队列。

这套分层对应 Layered Architecture + MVC 模式：Controller 只管 Web 交互，Service 只管业务编排，Repository 只管存储，各司其职。

## 项目里默认会用到的交叉能力

除了四层本身，一个正经的 Spring Boot 项目还会用到一系列"背景板"能力。

**依赖注入（IoC）** 把对象生命周期交给 Spring 容器管理，通过 `@Component`、`@Autowired` 或构造器注入实现解耦。**Bean 配置与环境隔离**通过 `application.yml` 和 `@Profile` 让本地、测试、生产用不同配置。**数据校验**用 `@Valid`、`@NotNull` 等注解保证入参合法。**全局异常处理**通过 `@ControllerAdvice` 把 Java 异常统一转成 API 响应格式。**日志与链路追踪**用 Logback + Slf4j + MDC 排查线上问题。**安全认证**用 Spring Security 或 JWT 实现登录鉴权和接口权限控制。**事务管理**用 `@Transactional(propagation = ...)` 保证多表操作的原子性。

## 常用基础设施

关系型数据库（MySQL + MyBatis/JPA）提供事务强一致和复杂查询能力。缓存（Redis / Caffeine）用于热数据读写加速和削峰填谷。消息队列（RabbitMQ / Kafka）实现系统解耦和异步处理。对象存储（MinIO / 阿里 OSS）处理文件和图片。配置中心与注册中心（Nacos / Consul）用于分布式服务治理。监控告警（Prometheus + Grafana / ELK）覆盖性能指标和日志分析。

## 开发流水线

一条最小可行的开发流水线包括：Maven/Gradle 管理依赖与打包，Git 做版本控制（feature 分支 → PR → main），JUnit 5 + Mockito 做单元测试覆盖 Service 和领域模型，SpotBugs/SonarLint 做静态检查，Dockerfile 实现容器化，GitHub Actions 或 Jenkins 实现 CI/CD 自动测试、构建镜像和部署，`application-prod.yml` + 环境变量做环境配置，Spring Actuator 暴露 `/health` 和 metrics 做运行时观测。

## DDD 分模块 + Layered 分层

当领域复杂度高、团队规模达到 5-6 人以上、需要并行开发时，通常会采用 DDD 分模块的方式组织代码。

具体做法是：先用 Domain-Driven Design 把系统切成「订单」「库存」「支付」等 Bounded Context，形成独立的 package 或 module；每个模块内部再按 Controller → Service → Domain/Repository → Infra 分层；公共设施（消息总线、权限）单独放 shared/infra 模块，只向上游暴露接口。

这种方式的好处是既保留了单体部署的简单运维，又让代码耦合降到子域内。后期若要拆微服务，可以按模块整体迁移。

## 常见架构风格对比

**Monolithic Layered Architecture** 是传统三层单体，快速启动适合小团队，但后期演进成本高。**Modulith（模块化单体）** 是单体进程加明确模块边界的过渡形态，先规整模块再择机拆微服务。**Hexagonal / Clean / Onion Architecture** 把核心域模型放中心，外围通过 Port/Adapter 接第三方，测试友好且技术栈可替换。**Microservices Architecture** 每个子域独立部署，支持独立扩缩和技术多样，但 DevOps 与观测成本高。**Event-Driven Architecture** 通过事件（RabbitMQ/Kafka）松耦合，天然异步但调试链路和一致性更复杂。**CQRS + Event Sourcing** 读写模型分离，适合大量读、少量写或审计型系统。**Microkernel Architecture** 核心稳定外围可热插拔，适合产品化平台。**Serverless Architecture** 运维极简但有冷启动延迟和厂商绑定问题。

## 如何选型

选型的核心是先评估业务。复杂领域加多团队适合 DDD 模块化，可演进至微服务。读多写少且需要追溯适合 CQRS + Event Sourcing。功能插件化产品适合 Microkernel。突发流量且不想运维服务器适合 Serverless 或事件驱动。

分阶段演进是常见策略。0 到 1 阶段用单体或模块化单体快速上线，1 到 N 阶段流量或团队扩大时再将关键模块拆成微服务，或将同步调用切成事件驱动。

治理工具链方面，微服务需要 API 网关、链路观测、CI/CD 和灰度发布；事件驱动需要关注事务一致性（Outbox、幂等）和死信队列监控；Serverless 需要关注冷启动、函数粒度和状态持久策略。

## 渐进学习路径

从 HTTP 与 REST 基础开始（方法、状态码、JSON、Postman 调试），到 Spring MVC 和 Spring Boot（自动装配、Starter），再到数据访问（JDBC → MyBatis 或 JPA，事务注解），然后学习分层与领域模型（Value Object / DTO / Entity 的区别），接着掌握缓存 + MQ + 分布式锁（Redis 计数、RabbitMQ 异步），再到测试与运维（单元/集成测试、日志、监控、Docker），最后进入架构演进（DDD 分模块、微服务拆分、事件驱动）。

没有银弹，合适的就是最好的。先让边界清晰、依赖单向、测试可控，再逐步演进到更分布式的形态。
