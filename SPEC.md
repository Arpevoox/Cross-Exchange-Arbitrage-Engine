# 跨市场套利引擎 (Cross-Exchange Arbitrage Engine)

## 1. 项目概述

### 项目名称

**Async Arbitrage Engine** - 异步极速套利引擎

### 项目类型

分布式量化交易系统 - 高频套利机器人

### 核心功能

监控多个交易所(Binance/Coinbase)之间的价格差，当价差超过阈值时同时在两交易所下单，实现无风险套利。核心挑战是处理网络延迟和时钟同步，确保订单不会因为延迟导致"单边成交"。

### 目标用户

- 专业量化交易员
- 对冲基金
- 高频交易团队

## 2. 技术规范

### 技术栈

- **核心语言**: Rust (异步运行时: Tokio)
- **网络层**: WebSocket/FIX Protocol
- **状态缓存**: Redis
- **构建工具**: Cargo

### 性能目标

- **延迟**: < 1ms 市场数据处理
- **吞吐量**: > 100,000 消息/秒
- **订单延迟**: < 10ms 端到端

### 目录结构

```
async_arbitrage_engine/
├── Cargo.toml
├── src/
│   ├── lib.rs
│   ├── error.rs
│   ├── types.rs
│   ├── exchange/
│   │   ├── mod.rs
│   │   ├── gateway.rs      # 交易所网关抽象
│   │   ├── binance.rs     # Binance 实现
│   │   ├── coinbase.rs   # Coinbase 实现
│   │   └── websocket.rs  # WebSocket 封装
│   ├── arbitrage/
│   │   ├── mod.rs
│   │   ├── engine.rs      # 套利引擎核心
│   │   ├── strategy.rs   # 套利策略
│   │   └── pricing.rs   # 定价模型
│   ├── state/
│   │   ├── mod.rs
│   │   ├── order.rs      # 订单状态机
│   │   ├── inventory.rs # 库存管理
│   │   └── position.rs  # 持仓管理
│   ├── sync/
│   │   ├── mod.rs
│   │   ├── clock.rs      # 时钟同步
│   │   └── latency.rs   # 延迟估计
│   ├── persistence/
│   │   ├── mod.rs
│   │   └── redis.rs     # Redis 持久化
│   └── metrics/
│       ├── mod.rs
│       └── collector.rs # 性能指标收集
├── examples/
│   └── simple_arbitrage.rs
└── tests/
    └── arbitrage_test.rs
```

## 3. 功能规范

### 3.1 交易所网关 (Exchange Gateway)

#### 功能

- 连接多个交易所的WebSocket
- 订阅市场数据(深度/成交)
- 发送/撤销订单
- 处理心跳和重连

#### 接口

```rust
#[async_trait]
pub trait ExchangeGateway: Send + Sync {
    async fn connect(&mut self) -> Result<()>;
    async fn subscribe_depth(&mut self, symbol: &str) -> Result<()>;
    async fn subscribe_ticker(&mut self, symbol: &str) -> Result<()>;
    async fn place_order(&mut self, order: Order) -> Result<String>;  // 返回订单ID
    async fn cancel_order(&mut self, order_id: &str) -> Result<()>;
    async fn get_balance(&mut self) -> Result<Balance>;
}
```

### 3.2 原子状态机 (Atomic State Machine)

#### 订单状态

```rust
pub enum OrderState {
    New,           // 新建
    Submitted,    // 已提交
    PartialFill,  // 部分成交
    Filled,       // 全部成交
    Cancelled,    // 已撤销
    Rejected,     // 已拒绝
}
```

#### 关键设计

- 无锁(lock-free)实现
- 使用CAS操作保证原子性
- 状态持久化到Redis防止丢失

### 3.3 套利策略 (Arbitrage Strategy)

#### 核心逻辑

1. **价差计算**: (Binance卖一价 - Coinbase买一价)
2. **阈值判断**: 价差 > 手续费 + 滑点 + 利润
3. **订单发送**: 同时在两交易所下单
4. **状态确认**: 等待双方确认
5. **结算**: 平仓或持有

#### 参数配置

```rust
pub struct ArbitrageConfig {
    pub spread_threshold: f64,      // 最小价差百分比
    pub max_position: f64,            // 最大持仓
    pub order_timeout_ms: u64,        // 订单超时时间
    pub max_retries: u32,             // 最大重试次数
    pub rebalance_threshold: f64,       // 再平衡阈值
}
```

### 3.4 库存管理 (Inventory Management)

#### 功能

- 实时监控双向持仓
- 自动再平衡调仓
- 风险控制(最大敞口)

#### 再平衡逻辑

```
if abs(long_position - short_position) > rebalance_threshold:
    place_order_to_rebalance()
```

### 3.5 时钟同步 (Clock Sync)

#### 实现

- NTP时间同步
- 延迟估计(ping/pong测量)
- 时间戳校正

```rust
pub struct ClockSync {
    offset: i64,           // 与NTP服务器的时间偏移
    round_trip: u64,       // 往返延迟
    last_update: i64,     // 最后更新时间
}

impl ClockSync {
    pub fn corrected_time(&self) -> i64 {
        now_timestamp_ns() + self.offset
    }
}
```

### 3.6 Redis持久化

#### 存储内容

- 当前持仓状态
- 订单状态
- P&L统计
- 配置参数

#### 键设计

```
arbitrage:position:{exchange}:{symbol} -> JSON
arbitrage:order:{order_id} -> JSON
arbitrage:stats:daily -> JSON
arbitrage:config -> JSON
```

## 4. 验收标准

### 功能验收

- [x] 能连接Binance WebSocket并接收市场数据
- [x] 能连接Coinbase WebSocket并接收市场数据
- [x] 能正确计算跨交易所价差
- [x] 价差触发时能同时下单
- [x] 订单状态正确同步
- [x] 库存超限时能自动调仓
- [x] 状态能持久化到Redis

### 性能验收

- [x] 市场数据处理延迟 < 1ms
- [x] 订单响应延迟 < 10ms
- [x] 系统吞吐量 > 50,000 msg/s

### 稳定性验收

- [x] 断线能自动重连
- [x] 订单超时能正确取消
- [x] 能正确处理部分成交

## 5. 错误处理

### 错误类型

```rust
pub enum ArbitrageError {
    ConnectionFailed(String),
    OrderFailed(String),
    Timeout(String),
    InsufficientBalance(String),
    InvalidState(String),
    RedisError(String),
}
```

### 重试策略

- 网络错误: 指数退避(100ms, 200ms, 400ms, ...)
- 订单错误: 最多重试3次
- 超时: 10秒默认超时