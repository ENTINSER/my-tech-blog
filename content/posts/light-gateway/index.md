---
title: 'Light Gateway：一个 Go 轻量级 API 网关的设计与实现'
date: 2026-07-19T10:10:00+08:00
draft: false
tags: ['Go', 'API Gateway', 'Gin', 'Redis', 'Prometheus', 'Microservices', 'Load Balancing', 'Rate Limiting']
categories: ['工程实践']
series: ['后端系统构建']
description: '从零构建一个 Go 轻量级 API 网关，不依赖现成网关框架，亲手实现路由转发、加权轮询负载均衡、自实现令牌桶限流、Redis 响应缓存、上游健康检查、Prometheus 可观测性与配置热更新，并完成本地端到端验证。'
---

## 摘要

本文记录了一次从零构建轻量级 API 网关的完整过程。该网关基于 Go + Gin 实现，涵盖路由转发、加权轮询负载均衡、自实现令牌桶限流、Redis 响应缓存、上游健康检查、Prometheus 可观测性与配置热更新等核心能力。项目不依赖现成网关框架，所有模块均手写，以便于深入理解网关层的工作原理。

本地验证结果表明：加权轮询能按 `weight` 比例在健康上游间分发流量；GET 请求首次访问产生 `X-Cache: MISS`，后续访问命中缓存并返回 `X-Cache: HIT`；`/healthz`、`/livez`、`/readyz` 探针与 `/metrics` 指标端点均正常可用；修改 `config.yaml` 后，Viper 能在毫秒级触发配置重载。

| 能力 | 实现方式 | 验证结果 |
|---|---|---|
| 路由转发 | `net/http/httputil` + Gin | ✅ 正常转发 |
| 负载均衡 | 加权轮询 + 健康过滤 | ✅ 按 weight 比例分发 |
| 限流 | 自实现令牌桶（全局 + per-key） | ✅ 支持热更新 |
| 响应缓存 | Redis + MD5 键 | ✅ MISS/HIT 正常 |
| 健康检查 | TCP 探活 | ✅ 不健康节点自动摘除 |
| 可观测性 | Prometheus `/metrics` | ✅ 指标可抓取 |
| 配置热更新 | Viper + fsnotify | ✅ 修改后即时生效 |

项目源码已开源：[github.com/ENTINSER/light-gateway](https://github.com/ENTINSER/light-gateway)。

## 1. 引言

API 网关是微服务架构中承上启下的关键层。它对外暴露统一入口，对内完成路由、负载均衡、限流、缓存、认证、可观测等横切能力。日常工作中，Nginx、Kong、Traefik 等成熟产品能覆盖绝大多数需求，但若只停留在“配置参数”层面，很难真正理解请求从进来到出去的完整链路。

为了补齐这一认知缺口，本文决定从零手写一个最小可用的 API 网关，命名为 **Light Gateway**。目标不是替代成熟产品，而是通过亲手实现，深入理解以下问题：

- 反向代理如何转发请求并保留原始语义？
- 加权轮询负载均衡如何在并发环境下安全地选择上游？
- 令牌桶限流如何保证线程安全并支持热更新？
- 响应缓存应该放在哪一层、如何生成缓存键？
- 健康检查如何动态影响流量？
- 配置文件变更后，如何在不重启进程的情况下生效？

## 2. 设计目标

Light Gateway 的核心设计目标包括：

1. **功能完整**：覆盖路由转发、负载均衡、限流、缓存、健康检查、可观测性、配置热更新。
2. **不依赖现成网关框架**：使用 Go 标准库与少量基础设施库（Gin、Redis 客户端、Viper、Prometheus）手写核心逻辑。
3. **可观测**：暴露 Prometheus 指标，便于监控流量、延迟、限流拒绝与上游健康状态。
4. **可部署**：提供 `Dockerfile` 与 `docker-compose.yml`，支持本地一键启动。
5. **可扩展**：代码按模块拆分，后续可较容易地加入认证、熔断、重试、TLS 等能力。

## 3. 系统架构

项目采用分层模块化设计：

```text
light-gateway/
├── main.go                  # 程序入口、依赖组装、优雅关闭
├── config.yaml              # 运行时配置
├── config/config.go         # 配置加载与热更新
├── proxy/proxy.go           # 反向代理 + 加权轮询
├── limiter/token_bucket.go  # 令牌桶限流
├── cache/redis_cache.go     # Redis 响应缓存
├── health/health.go         # TCP 健康检查 + 探针
├── metrics/metrics.go       # Prometheus 指标
├── middleware/middleware.go # 中间件链组装
├── Dockerfile
├── docker-compose.yml
└── README.md
```

请求处理链路如下：

```text
客户端请求
   │
   ▼
[metrics 记录]
   │
   ▼
/healthz /livez /readyz /metrics ──→ 直接响应
   │
   ▼
[RateLimit] ──→ 全局 / per-key 令牌桶
   │
   ▼
[Cache] ──→ 仅 GET，命中则直接返回
   │
   ▼
[Proxy] ──→ 选择健康上游，反向代理
   │
   ▼
上游服务
```

## 4. 核心实现

### 4.1 路由转发与负载均衡

使用 `net/http/httputil.NewSingleHostReverseProxy` 完成请求转发。为了实现加权轮询，将每个上游的 `weight` 展开为索引表：

```go
for _, u := range upstreams {
    for i := 0; i < u.Weight; i++ {
        p.weights = append(p.weights, len(p.names))
    }
    p.names = append(p.names, u.Name)
}
```

轮询时使用 `atomic.Uint64` 计数器，保证并发安全；选择过程中跳过不健康的上游。若加权表中全部不健康，则退化为简单轮询健康列表。

### 4.2 令牌桶限流

自实现 `TokenBucket`，支持全局共享桶、按 `X-API-Key` 分桶（懒创建 + 过期清理）以及运行期热更新 `rate` 与 `burst`：

```go
func (tb *TokenBucket) Allow(n int) bool {
    tb.mu.Lock()
    defer tb.mu.Unlock()

    now := time.Now()
    tb.tokens += now.Sub(tb.lastUpdate).Seconds() * tb.rate
    if tb.tokens > tb.burst { tb.tokens = tb.burst }
    tb.lastUpdate = now

    if tb.tokens >= float64(n) {
        tb.tokens -= float64(n)
        return true
    }
    return false
}
```

### 4.3 Redis 响应缓存

仅对 `GET` 请求启用缓存。缓存键格式为：

```text
lg:cache:<md5(path?query)>
```

命中缓存时直接写回响应头与体，并附加 `X-Cache: HIT`。客户端可通过 `Cache-Control: no-cache` 绕过缓存。

### 4.4 健康检查与探针

后台协程每 10 秒对所有上游执行一次 TCP 拨号探测，维护 `healthy` 状态表，并同步更新 Prometheus 的 `upstream_health_status` 指标。

| 端点 | 含义 | 返回 |
|---|---|---|
| `/healthz` | 网关存活 | `200 {"status":"ok"}` |
| `/livez` | 网关运行中 | `200 {"status":"alive"}` |
| `/readyz` | 所有上游健康 | `200/503` |

### 4.5 配置热更新

使用 Viper 加载 `config.yaml`，并通过 fsnotify 监听文件变更。变更触发后更新限流器参数、上游列表与健康检查列表、缓存 TTL。若新配置解析失败，则保留旧配置，避免网关因错误配置宕机。

### 4.6 可观测性

通过 Prometheus 暴露以下指标：

| 指标名 | 类型 | 说明 |
|---|---|---|
| `http_requests_total` | Counter | 按 path/method/status 统计 |
| `http_request_duration_seconds` | Histogram | 按 path/method 统计延迟 |
| `rate_limit_rejects_total` | Counter | 按 type/api_key 统计限流拒绝 |
| `upstream_health_status` | Gauge | 上游健康状态 |

## 5. 本地验证

测试环境：macOS、Go 1.21+、Redis 7、两个 Go 测试服务分别监听 `9001` 与 `9002`。

### 5.1 负载均衡

配置 `service-a:9001 weight=1`、`service-b:9002 weight=2`，携带 `Cache-Control: no-cache` 请求 6 次：

```bash
for i in {1..6}; do
  curl -s -H "Cache-Control: no-cache" http://localhost:8080/
done
```

结果：

```text
hello from port 9001
hello from port 9002
hello from port 9002
hello from port 9001
hello from port 9002
hello from port 9002
```

流量按 `1:2` 比例循环分发。

### 5.2 响应缓存

```bash
curl -s -D - -o /dev/null http://localhost:8080/test?a=1 | grep X-Cache
# X-Cache: MISS

curl -s -D - -o /dev/null http://localhost:8080/test?a=1 | grep X-Cache
# X-Cache: HIT
```

### 5.3 探针与指标

```bash
curl -s http://localhost:8080/readyz
# {"status":"ready"}

curl -s http://localhost:8080/metrics | grep upstream_health_status
# upstream_health_status{upstream="service-a"} 1
# upstream_health_status{upstream="service-b"} 1
```

### 5.4 配置热更新

修改 `config.yaml` 中的 `rate_limit.global.rate` 并保存后，日志输出：

```text
[config] 配置文件已更新: /Users/mingrun/light-gateway/config.yaml
[main] 配置热更新生效
```

限流参数即时生效，无需重启。

## 6. 遇到的问题与解决

### 问题 1：Gin 通配路由与 `/metrics` 冲突

最初使用 `r.Any("/*path", ...)` 做兜底，导致 `/metrics`、`/healthz` 也被代理中间件捕获。

**解决**：改用 `r.NoRoute(...)`，只对未匹配路径走业务中间件链。

### 问题 2：缓存干扰负载均衡测试

初次测试轮询时，所有请求都返回同一上游，一度怀疑轮询逻辑有 bug。

**解决**：后来发现第一次 GET 已被缓存，后续请求直接命中缓存。加 `Cache-Control: no-cache` 后验证轮询正常。

### 问题 3：Python 测试上游偶发 EOF

使用 `python3 -m http.server` 做上游时，Go 反向代理偶尔报 EOF。

**解决**：这是 Python 测试服务器的连接关闭行为问题，不影响网关核心逻辑。本地验证改用 Go 服务后稳定。

## 7. 总结与展望

通过手写 Light Gateway，我对 API 网关层的工作机制有了更系统的理解。项目中所有核心模块都尽量保持最小可用，代码结构清晰，便于后续扩展。

下一步计划加入的能力：

- HTTPS/TLS 终止
- 路径前缀重写
- 熔断与重试
- JWT / API Key 认证
- 基于 Redis 的分布式限流

---

**项目地址**：[github.com/ENTINSER/light-gateway](https://github.com/ENTINSER/light-gateway)
