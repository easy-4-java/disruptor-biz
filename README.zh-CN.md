# disruptor-biz

[English](./README.md) | [简体中文](./README.zh-CN.md)

[![License](https://img.shields.io/badge/license-Apache%202.0-green)

> 面向业务的 LMAX Disruptor 事件类型：带路由表达式的异步事件基类与高并发缓存时钟
> ——Disruptor 应用异步事件推送 / 处理的基础。

## 目录

- [1. 项目概述](#1-项目概述)
- [2. 功能与状态](#2-功能与状态)
- [3. 环境要求与兼容性](#3-环境要求与兼容性)
- [4. 架构与模块](#4-架构与模块)
- [5. 安装](#5-安装)
- [6. 快速开始](#6-快速开始)
- [7. 配置](#7-配置)
- [8. 核心用法 / API](#8-核心用法--api)
- [9. 测试与构建](#9-测试与构建)
- [10. 版本与分支](#10-版本与分支)
- [11. 贡献与许可](#11-贡献与许可)

## 1. 项目概述

`disruptor-biz` 是 easy4j Disruptor 集成的业务层：提供基于 Disruptor 异步并发框架
的异步事件推送 / 处理所需的事件类型与时钟工具。

- **`DisruptorEvent`** — 通过 Disruptor ring buffer 交换的事件的抽象基类。每个事件
  记录创建时间戳，并携带用于路由到对应处理链的 `routeExpression`。
- **`SystemClock`** — 缓存时钟工具（`SystemClock.now()`），避免高并发下反复调用
  `System.currentTimeMillis()` 的开销。

事件路由采用 Ant 风格规则表达式，如 `/Event-DC-Output/TagA-Output/** = inDbPostHandler`：
路由匹配 `Event-DC-Output` + `TagA-Output` + 任意 key 的事件交由 `inDbPostHandler`
处理；`/Event-DC-Output/TagB-Output/** = smsPostHandler` 则把另一 Tag 的事件路由给
短信处理器。这种责任链机制非常适合消息队列消费端：需要快速消费各类消息、且每类
处理实现各不相同的场景，正是需要事件的分类异步处理。

> 说明：完整的处理链实现（规则、`HandlerChain`、分发器）位于姊妹模块
> `disruptor-extension`；本模块提供这些链所分发的路由契约（`routeExpression`）。

它不是：

- 完整的事件处理框架——链 / 分发器机制在 `disruptor-extension`。
- LMAX Disruptor 核心本身——`com.lmax:disruptor` 是常规依赖。

典型场景：

| 场景 | 使用内容 |
| :--- | :--- |
| 为 Disruptor ring buffer 定义领域事件 | 继承 `DisruptorEvent` |
| 为事件附加路由表达式 | `event.setRouteExpression("/Event-DC-Output/TagA-Output/**")` |
| 事件生产者中的高并发时间戳 | `SystemClock.now()` / `SystemClock.nowDate()` |

## 2. 功能与状态

| 能力 | 状态 | 说明 |
| :--- | :--- | :--- |
| `DisruptorEvent` 基类 | 稳定 | 抽象 `EventObject`；创建时间戳 + `routeExpression` |
| 缓存系统时钟 | 稳定 | `SystemClock.now()` / `nowDate()`；守护线程定时刷新，周期 1 ms |

## 3. 环境要求与兼容性

| 要求 | 版本 / 说明 |
| :--- | :--- |
| JDK | 21+ |
| Maven | 3.0+（enforcer 强制；项目内置 Maven Wrapper `./mvnw`） |
| LMAX Disruptor | `com.lmax:disruptor`（由本 pom 管理） |

版本线：

| 分支 | JDK | 版本 |
| :--- | :--- | :--- |
| `feature/1.0.x` | 8 | `1.0.x.*` |
| `feature/2.0.x` | 17 | `2.0.x.*` |
| `feature/3.0.x` | 21 | `3.0.x.*` |

## 4. 架构与模块

```text
+---------------------+   +---------------------------------------+
| Business producer   |   | disruptor-biz                        |
| (domain event)      |-->|  DisruptorEvent (abstract)           |
|                     |   |    - timestamp (SystemClock.now())   |
|                     |   |    - routeExpression /Event/Tag/**   |
|                     |   |  SystemClock (cached, 1 ms period)   |
+---------------------+   +------------------+--------------------+
                                             |
                                             v
                     +-------------------------------------------+
                     | LMAX Disruptor ring buffer (async push &  |
                     | processing)                              |
                     +------------------+------------------------+
                                             |
                                             v
                     +-------------------------------------------+
                     | disruptor-extension handler chains route  |
                     | /Event/Tag/** to handlers                 |
                     +-------------------------------------------+
```

单模块 Maven 工程（`packaging: jar`），无子模块。

| 构件 | 职责 |
| :--- | :--- |
| `io.github.easy4j:disruptor-biz` | 事件基类 + 缓存时钟 |

关键类：

| 类 | 职责 |
| :--- | :--- |
| `com.lmax.disruptor.biz.event.DisruptorEvent` | 抽象事件基类：`getTimestamp()`、`getRouteExpression()` / `setRouteExpression(...)` |
| `com.lmax.disruptor.biz.util.SystemClock` | 缓存时钟：`now()`、`nowDate()` |

## 5. 安装

项目**尚未发布到 Maven Central**。快照 / 发布版本通过阿里云 Maven 仓库与 GitHub
Releases 分发。

Maven：

```xml
<dependency>
    <groupId>io.github.easy4j</groupId>
    <artifactId>disruptor-biz</artifactId>
    <version>3.0.x.x.20260630-SNAPSHOT</version>
</dependency>
```

Gradle：

```groovy
implementation 'io.github.easy4j:disruptor-biz:3.0.x.x.20260630-SNAPSHOT'
```

## 6. 快速开始

```java
import com.lmax.disruptor.biz.event.DisruptorEvent;

public class OrderEvent extends DisruptorEvent {

    public OrderEvent(Object source) {
        super(source);
    }
}

// 生产者侧
OrderEvent event = new OrderEvent("order-1001");
event.setRouteExpression("/Order-Created/New-Order/**");
System.out.println("created at: " + event.getTimestamp());
System.out.println("route     : " + event.getRouteExpression());
```

预期结果：事件携带创建时间戳（基于缓存时钟）与决定后续由哪条处理链消费的路由表达式。

## 7. 配置

本库没有配置文件或属性前缀。唯一可调项是时钟刷新周期，在 `SystemClock` 内部构造
时固定（1 ms）。事件按实例通过 `setRouteExpression(...)` 配置。

## 8. 核心用法 / API

### 8.1 带路由的领域事件

```java
// 定义一次
public class MessageEvent extends DisruptorEvent {
    public MessageEvent(Object source) { super(source); }
}

// 每条消息附加路由规则
MessageEvent evt = new MessageEvent(source);
evt.setRouteExpression("/Event-DC-Output/TagA-Output/**");
```

### 8.2 缓存时钟

```java
long ts = SystemClock.now();         // 缓存毫秒数（高负载下开销低）
String date = SystemClock.nowDate(); // 缓存的 SQL 时间戳字符串
```

## 9. 测试与构建

```bash
./mvnw clean verify
```

- 构建配置了 JaCoCo Maven 插件（报告 + 绑定在 `verify` 阶段的 `check` 目标，
  行覆盖率规则为 90%；`haltOnFailure=false`）。
- **假设**：1.0.x 分支当前 `src/test` 下未提交测试源码；覆盖率门禁仅在存在测试时生效。
- 本 worktree 的 `.github/` 下无 CI 工作流文件。

## 10. 版本与分支

| 分支 | JDK | 版本 | 说明 |
| :--- | :--- | :--- | :--- |
| `feature/1.0.x` | 8 | `1.0.x.*` | 当前分支，JDK 8 基线，维护中 |
| `feature/2.0.x` | 17 | `2.0.x.*` | JDK 17 版本线 |
| `feature/3.0.x` | 21 | `3.0.x.*` | JDK 21 版本线 |

维护策略：`1.0.x` 版本线接收针对 JDK 8 基线的缺陷修复与兼容性更新；面向新 JDK 的
新特性在 `2.0.x` / `3.0.x` 版本线开发。发布物通过阿里云 Maven 仓库与 GitHub
Releases 分发；项目尚未发布到 Maven Central。

## 11. 贡献与许可

欢迎通过 GitHub Issue 或 Pull Request 参与贡献。

本项目基于 [Apache License, Version 2.0](LICENSE) 许可。
