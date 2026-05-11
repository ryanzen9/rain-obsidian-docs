# Design: NestJS 整合 RabbitMQ 实现消息队列 — 文章扩充

## 定位

个人学习笔记，全面深度教程。从零开始覆盖 RabbitMQ 核心概念、Express 原生实战、NestJS 整合、死信处理、最佳实践。

## 文章结构

1. **什么是 RabbitMQ** — 同步 vs 异步对比、消息队列定位、RabbitMQ 简介、AMQP 协议、优缺点、适用场景
2. **RabbitMQ 核心概念** — Exchange/Queue/Binding/Routing Key/Channel/Connection/Virtual Host，含 Mermaid 架构图
3. **Express 原生实战** — 用 amqplib 演示生产者/消费者，理解底层机制
4. **NestJS 整合 RabbitMQ** — @golevelup/nestjs-rabbitmq 装饰器驱动开发，完整业务示例
5. **进阶：死信队列与错误处理** — DLX、TTL、重试策略、消息确认机制
6. **最佳实践与常见问题** — 连接管理、消息持久化、幂等性、监控运维

## 约束

- 图表使用 Mermaid
- 代码示例使用 @golevelup/nestjs-rabbitmq
- 保留 Express 前置示例，形成递进
- 风格与现有笔记一致：`##` 标题、无 frontmatter、无嵌套过多层级、段落叙述为主
- 专业术语有详细说明
- 内容有对应来源引用
