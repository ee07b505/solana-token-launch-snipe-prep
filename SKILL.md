---
name: solana-token-launch-snipe-prep
description: >-
  Use when preparing to snipe a Solana token launch (Raydium / Meteora /
  pump.fun / DBC): identify open mechanism, detect risks, configure buy/exit,
  dry-run on devnet before any mainnet send.
---
# Solana 新盘开盘狙击准备（链：Solana）

**适用链：Solana only。** Base / BSC 请用 `token-launch-snipe-prep`。

原则：参数全进 launch config；报价币从池子读，不默认 SOL；主网发送默认关闭（三重开关）；私钥只存 600 文件，永不打印。不承诺抢到第一。

## 0. 先识别开盘机制（决定流程）

| 类型 | 典型 | 流程 |
|---|---|---|
| 定时 timestamp | Raydium CPMM/AMM v4 `open_time`；Meteora DAMM v2/DLMM `activation_type=1` | 预签 → 读链上 Clock → 到点单发 |
| 定时 slot | Meteora `activation_type=0` | 预签 → 轮询 slot → `lead_slots` 0/1 单发 |
| 手动开关 | 有权限的 DLMM / set_pool_status | 监听开启交易，跟下一笔 |
| 即时/曲线 | pump.fun、LaunchLab、Meteora DBC | 监听建币/建池 → 跟买（同 bundle 抢不过项目方） |
| 毕业迁移 | pump→PumpSwap、LaunchLab→CPMM、DBC→DAMM | 监听 migrate → 抢下一笔 |
| Orca | 无 open_time | 监听第一次加流动性 |

识别结果写入 config 的 `gate`。识别不出或模拟不一致：不上报前不下单。

## 1. 基础设施（Solana 特有）

- Slot ≈ 400ms；SWQoS/staked RPC 决定能否送到 leader，priority fee 决定队列排序，两者都要。
- Priority fee 按申请的 CU limit 计费，不是实际用量。
- Jito tip 最低 1000 lamports，仅 Jito leader 有效；bundle ≤5 笔。
- 收数据：Yellowstone gRPC。**Jito ShredStream 已于 2026-09-05 停服**，改用 DoubleZero Edge 等。
- 时间戳 gate **必须读链上 Clock**，勿用本机时钟。

## 2. 检测（只读）→ OK / 减仓 / 放弃

必查 Token-2022：mint/freeze authority、TransferFee、TransferHook、PermanentDelegate、NonTransferable、DefaultAccountState Frozen、Pausable。

池子：activation、pool_status、Alpha Vault、开盘瞬间反狙击费（fee scheduler）。费率过高可等衰减，未必抢 +0 slot。

开盘前买入应失败；开盘后 simulate 卖出确认能卖。

## 3. 资金

- SOL 池：本金 + WSOL/ATA 租金 + fee + tip。
- USDC 等：报价币 ATA 提前备好 + 少量 SOL 付费。
- 开盘前预建目标 mint ATA；热钱包只放当次资金。

## 4. 发单

1. T−30min：detect、余额、ATA、RPC 延迟、leader schedule（是否 Jito）。
2. T−10s：冻结配置。
3. 开盘前约 30 slot：按开盘瞬间报价算 minOut → 构建（ComputeBudget + swap + 可选 tip）→ 签名缓存（blockhash ~60s 过期；更早用 durable nonce）。
4. 探针：预期 6003 / not open。
5. 到点：`skipPreflight`、`maxRetries=0`，同签名多入口广播；确认前可每 2s 重发。

主网三重开关：`mainnet_send_enabled` + `mainnet_ack=I_UNDERSTAND_REAL_FUNDS` + env `SOL_SNIPER_MAINNET=1`；是否主网按 genesis hash 判断。

## 5. 止盈止损（写进 `exit`，默认 dry_run）

| 策略 | Solana 做法 |
|---|---|
| A 同块多刀 | 多 swap 指令或 Jito bundle；最后一刀读真实余额 |
| B 目标价阶梯 | `targets[{mult,bps}]`，每档 minOut |
| C 多热钱包轮流 | 多 keys，各备 ATA/SOL |
| D 峰值回撤 | `trail.{arm_mult,drawdown_bps}` |
| E 多地址 | 仅单地址限额/税时 |
| F 止损熔断 | `stop_loss_mult` + 连续失败停 |

开盘波动期直连池程序，不走聚合器。

## 6. 收尾与复盘

卖完归集、关空 ATA 收租、主网开关改回 false。复盘：开盘后第几个 slot、谁在前、实际 fee/tip、滑点 vs 反狙击费、各档成交价。

## 7. 实现入口（参考仓库 sol-sniper）

- 定时：`snipe.js` / `snipe_dlmm.js` / `snipe_raydium_cpmm.js`
- 跟单：`snipe_follow.js` / `snipe_dbc.js` / `snipe_pump.js`
- 检测：`detect.js` / `detect_mint.js`；费率最优：`fee_optimal.js`
- 共享：`exit.js`、`tx_send.js`、`rpc_retry.js`

Devnet 验证过再谈主网；主网先极小额全流程。
