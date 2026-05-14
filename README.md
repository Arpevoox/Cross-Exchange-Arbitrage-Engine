# Async Arbitrage Engine

跨市场异步极速套利引擎 - Cross-Exchange Arbitrage Engine

[![Rust](https://img.shields.io/badge/Rust-1.70+-blue?style=flat-square)](https://www.rust-lang.org/)
[![License](https://img.shields.io/badge/License-MIT-green?style=flat-square)](LICENSE)

## 项目简介

一个高性能的跨交易所套利机器人,监控 Binance 和 Coinbase 等交易所之间的价格差,当价差超过阈值时同时在两交易所下单,实现无风险套利。

核心挑战是处理网络延迟和时钟同步,确保订单不会因为延迟导致"单边成交"。

## 核心特性

- **异步IO优化**: 基于 Tokio 运行时的高并发网络处理
- **原子状态机**: 无锁订单状态管理,防止单边成交
- **动态对冲**: 库存风险管理与自动调仓
- **时钟同步**: NTP时间同步 + 延迟估计
- **Redis持久化**: 订单/持仓状态缓存
- **性能指标**: 延迟/吞吐量实时监控

## 技术架构

```
async_arbitrage_engine/
├── src/
│   ├── exchange/          # 交易所网关 (Binance/Coinbase)
│   ├── arbitrage/         # 套利引擎核心
│   ├── state/            # 状态管理 (订单/持仓/库存)
│   ├── sync/             # 时钟同步 & 延迟估计
│   ├── persistence/       # Redis持久化
│   └── metrics/          # 性能指标收集
```

## 性能目标

| 指标 | 目标 |
|------|------|
| 市场数据处理延迟 | < 1ms |
| 订单响应延迟 | < 10ms |
| 系统吞吐量 | > 50,000 msg/s |

## 快速开始

### 安装

```bash
git clone https://github.com/your-repo/async-arbitrage-engine.git
cd async-arbitrage-engine
cargo build --release
```

### 配置

创建 `config.json` 配置文件:

```json
{
    "exchange_a": {
        "name": "binance",
        "ws_url": "wss://stream.binance.com:9443/ws",
        "api_key": "your-api-key",
        "secret_key": "your-secret-key"
    },
    "exchange_b": {
        "name": "coinbase",
        "ws_url": "wss://ws-feed.exchange.coinbase.com",
        "api_key": "your-api-key",
        "secret_key": "your-secret-key"
    },
    "arbitrage": {
        "spread_threshold": 0.001,
        "max_position": 10000,
        "order_timeout_ms": 5000,
        "rebalance_threshold": 100
    },
    "redis": {
        "url": "redis://127.0.0.1:6379"
    }
}
```

### 运行

```bash
# 生产环境
cargo run --release --example simple_arbitrage

# 开发测试
cargo run --example simple_arbitrage
```

## API 使用

### 创建套利引擎

```rust
use async_arbitrage_engine::{
    arbitrage::ArbitrageEngine,
    exchange::{binance::BinanceGateway, coinbase::CoinbaseGateway},
    types::ArbitrageConfig,
};

let config = ArbitrageConfig::default();
let exchange_a = Box::new(BinanceGateway::new());
let exchange_b = Box::new(CoinbaseGateway::new());
let mut engine = ArbitrageEngine::new(config, exchange_a, exchange_b);
```

### 处理市场数据

```rust
use async_arbitrage_engine::types::{Depth, PriceLevel};

// 更新深度数据
let depth = Depth {
    symbol: "BTCUSDT".to_string(),
    bids: vec![PriceLevel::new(50000.0, 1.0)],
    asks: vec![PriceLevel::new(50100.0, 1.0)],
    timestamp: now_timestamp_ns(),
};

engine.on_market_update("binance", depth).await?;
```

### 获取持仓

```rust
use async_arbitrage_engine::types::PositionSummary;

let summary = engine.get_position_summary();
println!("净持仓: {}", summary.net());
```

## 核心概念

### 1. ���差计算

```
spread = exchange_b.bid_price - exchange_a.ask_price
spread_percent = spread / ask_price * 100%
```

### 2. 订单状态机

```
New → Submitted → PartialFill → Filled
                ↓
            Cancelled/Rejected
```

### 3. 库存再平衡

当多头/空头持仓差超过阈值时,自动触发调仓:

```
if abs(long - short) > rebalance_threshold:
    place_order_to_rebalance()
```

### 4. 时钟同步

```rust
let corrected_time = local_time + clock_offset;
let latency = exchange_timestamp - corrected_time;
```

## 测试

```bash
# 运行所有测试
cargo test

# 运行特定模块测试
cargo test --lib state

# 运行文档测试
cargo test --doc
```

## 监控指标

### Prometheus 指标

```
# HELP arbitrage_orders_submitted Total orders submitted
# TYPE arbitrage_orders_submitted counter
arbitrage_orders_submitted 1000

# HELP arbitrage_latency_avg Average latency in microseconds
# TYPE arbitrage_latency_avg gauge
arbitrage_latency_avg 500.5
```

### 日志输出

```bash
# 设置日志级别
RUST_LOG=info cargo run
```

## 安全注意事项

1. **API密钥管理**: 生产环境使用环境变量或密钥管理服务
2. **订单超时**: 设置合理的超时时间防止单边成交
3. **仓位控制**: 严格控制最大持仓和单笔订单量
4. **熔断机制**: 连续失败自动暂停交易
5. **审计日志**: 记录所有交易操作

## 依赖项

| 依赖 | 版本 | 用途 |
|------|------|------|
| tokio | 1.35 | 异步运行时 |
| tungstenite | 0.21 | WebSocket |
| redis | 0.25 | 状态缓存 |
| parking_lot | 0.12 | 原子操作 |
| serde | 1.0 | 序列化 |

## 许可证

MIT License - 查看 [LICENSE](LICENSE) 文件

## 贡献者

欢迎提交 Issue 和 Pull Request!

## 参考资料

- [Binance WebSocket API](https://developers.binance.com/docs/simple-earn/history/get-earn-flexible-position-history)
- [Coinbase WebSocket API](https://docs.cloud.coinbase.com/ws-api/docs)
- [Rust Tokio 文档](https://tokio.rs/)