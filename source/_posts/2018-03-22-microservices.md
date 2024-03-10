---
title: 初识微服务
update: 2018-03-22 15:04:26
tags:
- microservice
categories:
- cloud
---

星环讲座: 初识微服务

主讲人: 吴伟

三本书:

- Building Microservices
- 微服务: 从设计到部署
- 微服务设计

大纲:

1. 什么是微服务
2. 核心概念
3. 开源微服务框架
4. 国内微服务产品

### 内容:

1. 什么是微服务: 相互独立微小的组建, 能够相互合作: 在一组小的服务, 独立的进程, 通信机制, 基于业务, 独立部署, 没有集中式管理.
2. 为什么要做微服务: 单体服务膨胀太大, 代码太多, 开发太慢, 发布速度太慢, 新人学习成本太高.
3. 微服务目标: 敏捷迭代, 灵活扩展, 服务服用.
4. 微服务核心概念: api网关, 服务发现, 熔断, 限流, 降级, 配置中心, 自动化部署和测试, 日志监控和分部署追踪, 安全, 服务拆分, 服务接口定义, 有状态服务集群, 无状态服务
   - 服务发现: 怎么找到另一个服务的地址. 通常使用服务中心或者服务代理.
   - api网关: 对外统一接入访问; 内外协议转换(http -> gRPC), 统一认证,监控,负载均衡,缓存; 智能路由: Kong, Nginx Plus, Traefik
   - 服务容错(熔断): 假设错误一定会发生, 想办法把损失降到最小: custom fallback, fail silent, fail fast. 超时与重试, 降级熔断, 连接隔离
   - 配置中心: 程序运行时动态调整行为的能力.
   - 自动化测试部署: CI/CD 改动一行代码能够多久部署成功
   - 服务监控: 系统在做什么, 哪些组建流量比较大, 这个请求在哪个地方失败了, 哪个调用比较慢.
   - 安全: 
   - 分布式事务: 每个服务解决一个问题, 要解决一个逻辑上的一致性.

cloud native = container + CICD/devops + microservice