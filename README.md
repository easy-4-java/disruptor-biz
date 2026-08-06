# disruptor-biz

[English](./README.md) | [简体中文](./README.zh-CN.md)

[![License](https://img.shields.io/badge/license-Apache%202.0-green)

> Business-oriented event types for the LMAX Disruptor async framework: a routing
> event base class and a high-concurrency cached clock — the foundation for async
> event push / processing in Disruptor-based applications.

## Table of Contents

- [1. Project Overview](#1-project-overview)
- [2. Features & Status](#2-features--status)
- [3. Requirements & Compatibility](#3-requirements--compatibility)
- [4. Architecture & Modules](#4-architecture--modules)
- [5. Installation](#5-installation)
- [6. Quick Start](#6-quick-start)
- [7. Configuration](#7-configuration)
- [8. Core Usage / API](#8-core-usage--api)
- [9. Testing & Build](#9-testing--build)
- [10. Versioning & Branches](#10-versioning--branches)
- [11. Contributing & License](#11-contributing--license)

## 1. Project Overview

`disruptor-biz` is the business layer of the easy4j Disruptor integration: it
provides the event type and the clock utility used by Disruptor-based async event
push / processing applications.

- **`DisruptorEvent`** — the abstract base type for events exchanged through a
  Disruptor ring buffer. Every event captures the creation timestamp and carries a
  `routeExpression` used to route it to the right handler chain.
- **`SystemClock`** — a cached-clock utility (`SystemClock.now()`) that avoids the
  overhead of repeated `System.currentTimeMillis()` calls under high concurrency.

The event-routing model follows Ant-style rule expressions such as
`/Event-DC-Output/TagA-Output/** = inDbPostHandler`: an event whose route matches
`Event-DC-Output` + `TagA-Output` + any key is dispatched to `inDbPostHandler`;
`/Event-DC-Output/TagB-Output/** = smsPostHandler` routes a different tag to the
SMS handler. This chain-of-responsibility style enables classified async
processing — exactly what a message-queue consumer needs when it must handle many
message types with different implementations.

> Note: the full handler-chain implementation (rules, `HandlerChain`, dispatcher)
> lives in the companion module `disruptor-extension`; this module provides the
> event contract (`routeExpression`) those chains dispatch on.

What it is **not**:

- Not a full event-processing framework — the chain / dispatcher machinery is in
  `disruptor-extension`.
- Not the LMAX Disruptor core itself — `com.lmax:disruptor` is a regular dependency.

Typical scenarios:

| Scenario | What you use |
| :--- | :--- |
| Define a domain event for a Disruptor ring buffer | Extend `DisruptorEvent` |
| Attach a routing expression to an event | `event.setRouteExpression("/Event-DC-Output/TagA-Output/**")` |
| High-concurrency timestamps in event producers | `SystemClock.now()` / `SystemClock.nowDate()` |

## 2. Features & Status

| Capability | Status | Notes |
| :--- | :--- | :--- |
| `DisruptorEvent` base type | Stable | Abstract `EventObject`; creation timestamp + `routeExpression` |
| Cached system clock | Stable | `SystemClock.now()` / `nowDate()`; daemon-thread scheduled refresh, 1 ms period |

## 3. Requirements & Compatibility

| Requirement | Version / Notes |
| :--- | :--- |
| JDK | 21+ |
| Maven | 3.0+ (enforced; Maven Wrapper `./mvnw` included) |
| LMAX Disruptor | `com.lmax:disruptor` (managed by this pom) |

Version lines:

| Branch | JDK | Version |
| :--- | :--- | :--- |
| `feature/1.0.x` | 8 | `1.0.x.*` |
| `feature/2.0.x` | 17 | `2.0.x.*` |
| `feature/3.0.x` | 21 | `3.0.x.*` |

## 4. Architecture & Modules

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

Single-module Maven project (`packaging: jar`). No child modules.

| Artifact | Responsibility |
| :--- | :--- |
| `io.github.easy4j:disruptor-biz` | Event base type + cached clock |

Key classes:

| Class | Responsibility |
| :--- | :--- |
| `com.lmax.disruptor.biz.event.DisruptorEvent` | Abstract event base: `getTimestamp()`, `getRouteExpression()` / `setRouteExpression(...)` |
| `com.lmax.disruptor.biz.util.SystemClock` | Cached clock: `now()`, `nowDate()` |

## 5. Installation

The project is **not yet published to Maven Central**. Snapshots/releases are
distributed through the Aliyun Maven repository and GitHub Releases.

Maven:

```xml
<dependency>
    <groupId>io.github.easy4j</groupId>
    <artifactId>disruptor-biz</artifactId>
    <version>3.0.x.x.20260630-SNAPSHOT</version>
</dependency>
```

Gradle:

```groovy
implementation 'io.github.easy4j:disruptor-biz:3.0.x.x.20260630-SNAPSHOT'
```

## 6. Quick Start

```java
import com.lmax.disruptor.biz.event.DisruptorEvent;

public class OrderEvent extends DisruptorEvent {

    public OrderEvent(Object source) {
        super(source);
    }
}

// producer side
OrderEvent event = new OrderEvent("order-1001");
event.setRouteExpression("/Order-Created/New-Order/**");
System.out.println("created at: " + event.getTimestamp());
System.out.println("route     : " + event.getRouteExpression());
```

Expected result: the event carries the creation timestamp (cached-clock based)
and the routing expression that later determines which handler chain processes it.

## 7. Configuration

The library has no configuration file or property prefix. The only tunable is the
clock refresh period, fixed at construction time inside `SystemClock` (1 ms).
Events are configured per instance via `setRouteExpression(...)`.

## 8. Core Usage / API

### 8.1 Domain event with routing

```java
// define once
public class MessageEvent extends DisruptorEvent {
    public MessageEvent(Object source) { super(source); }
}

// per message: attach the routing rule
MessageEvent evt = new MessageEvent(source);
evt.setRouteExpression("/Event-DC-Output/TagA-Output/**");
```

### 8.2 Cached clock

```java
long ts = SystemClock.now();        // cached millis (cheap under high load)
String date = SystemClock.nowDate(); // cached SQL timestamp string
```

## 9. Testing & Build

```bash
./mvnw clean verify
```

- The build is configured with the JaCoCo Maven plugin (report + `check` goal with a
  90% line-coverage rule bound to the `verify` phase; `haltOnFailure=false`).
- **Assumption**: the 1.0.x branch currently checks in no test sources under
  `src/test`; coverage thresholds are therefore enforced only when tests exist.
- No CI workflow files are present under `.github/` in this worktree.

## 10. Versioning & Branches

| Branch | JDK | Version | Notes |
| :--- | :--- | :--- | :--- |
| `feature/1.0.x` | 8 | `1.0.x.*` | Current branch, JDK 8 baseline, maintained |
| `feature/2.0.x` | 17 | `2.0.x.*` | JDK 17 line |
| `feature/3.0.x` | 21 | `3.0.x.*` | JDK 21 line |

Maintenance policy: the `1.0.x` line receives bug fixes and compatibility updates
for the JDK 8 baseline. New features targeting newer JDKs land on the `2.0.x` /
`3.0.x` lines. Releases are published to the Aliyun Maven repository and as
GitHub Releases; the project is not yet published to Maven Central.

## 11. Contributing & License

Contributions are welcome — please open issues or pull requests on GitHub.

Licensed under the [Apache License, Version 2.0](LICENSE).
