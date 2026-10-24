# Solana 新盘开盘狙击 · 准备清单（Solana 版）

> 来源：照搬 `token-launch-snipe-prep`（BSC/Base 版）的框架，改成 Solana 的写法。
> 代码在 `/workspace/sol-sniper/src`（lib / detect / snipe / create_pool），配置在 `configs/*.json`。
> **铁律**：
> - 全部参数写在配置里，代码里不写死；
> - 报价币从池子读，不默认是 SOL；
> - 主网发送默认关闭；
> - 私钥只放 `keys/*.json`（权限 600），任何地方都不打印。

---

## 0. 第一步：判断开盘类型（决定后面走哪套流程）

| 类型 | 典型 | 在链上是什么样 | 用哪套流程 |
|---|---|---|---|
| **定时开盘（timestamp）** | Raydium CPMM `open_time`；AMM v4 `open_time`；Meteora DAMM v2 / DLMM / DAMM v1 `activation_type=1` | 池子先建好，到时间之前 swap 会 revert（CPMM 会拒绝；Meteora 报 6003 PoolDisabled） | **本清单完整流程**：提前签名，按链上 Clock 到点单发 |
| **定 slot 开盘（slot）** | Meteora `activation_type=0` | 同上，只是 gate 按 slot 算 | 本清单完整流程，轮询 slot，`lead_slots` 设 0 或 1 |
| **管理员手动开关** | Meteora 有权限控制的池（TRUMP 用的就是这种）；DLMM 的 `creatorPoolOnOffControl` | 没有确定的时间点，管理员发交易把池子打开 | 监听开关交易或池子状态变化，抢跟随它的下一笔（只能抢 shred/gRPC 上的速度） |
| **即时开盘、按曲线成交** | pump.fun；Raydium LaunchLab（letsbonk / bonk.fun）；Meteora DBC（Believe、Bags 等） | 建币那笔交易一落块就能买（creator 常把 create 和 buy 打进同一个 Jito bundle） | **不走定时流程**，改成"监听建币 → 在同块或下一块跟买"；同一个 bundle 里的买单抢不过 |
| **毕业迁移** | pump → PumpSwap（无需权限，任何人都能触发）；LaunchLab → CPMM（有权限的 migrate 钱包）；DBC → DAMM v1/v2 | 迁移那笔交易里建池加流动性，建好立即能交易 | 监听迁移交易，抢下一笔 |
| **Orca Whirlpool** | `initializePool` 里没有开盘时间字段，流动性要另外一笔交易加 | 池子一有流动性就能交易 | 监听"第一次加流动性"再买，不要按初始化时的储备量算仓位（初始化时储备是 0） |

## 1. 开盘配置字段（`configs/*.json`）

| 字段 | 说明 |
|---|---|
| `cluster / rpc_url / ws_url / send_rpcs[]` | 主 RPC 加额外的广播 RPC（同一个签名，多路一起发） |
| `mainnet_send_enabled=false` · `mainnet_ack=""` | **三重开关**：主网要同时满足 `mainnet_send_enabled=true`、`mainnet_ack=I_UNDERSTAND_REAL_FUNDS`、环境变量 `SOL_SNIPER_MAINNET=1`。是不是主网按 genesis hash 判断，不看配置里写的名字 |
| `dex / program / pool / target_mint` | 程序 ID、池子地址、目标代币 mint |
| `expected_quote_mint` | 只用来**核对**。真正的报价币从池子的 tokenA/tokenB 读出来；对不上就退出（除非设了 `allow_quote_mismatch`） |
| `gate.type` | `auto_from_pool`：从池子读出 activation_type（slot 或 ts）和 activation_point。**目前只实现了 Meteora DAMM v2 的适配器**；Raydium CPMM（`open_time`）和 DLMM（`activation_point`）的适配器还没写，是下一步工作 |
| `gate.lead_slots / poll_ms / ts_source` | 提前几个 slot 发；轮询间隔；时间戳 gate 用链上 Clock（`chain_clock`），**不要用本地时钟**（原因见第 5 节的实测结果） |
| `wallet` | 用 `keys/` 下的哪个钱包 |
| `buy.amount_in_ui / max_slippage_bps / min_out_raw` | 买入数量；minOut 默认用开盘那一刻的报价（已经算进反狙击手续费）再扣滑点；也可以直接写死 |
| `buy.cu_limit / cu_price_micro_lamports` | priority fee = CU limit × CU price ÷ 1e6 lamports。按申请的 CU limit 收费，不是按实际用量，所以 limit 不要设得太大 |
| `buy.jito.{enabled,tip_lamports,block_engine_url}` | 只在主网有用。最低 tip 是 1000 lamports；tip 账户每次从 8 个里随机选一个；单笔 `sendTransaction` 官方建议 priority fee 和 tip 按 70/30 分 |
| `buy.max_retries_not_open` | 如果因为"还没开盘"（6003）失败了，用新的 blockhash 重新签名再发几次（同一个签名不能重发） |
| `buy.reduce_factor` | 检测结果是 REDUCE（降仓）时按这个比例缩小仓位 |
| `presign.before_slots` | 开盘前几个 slot 签名。blockhash 大约 150 slot（约 60 秒）后过期，所以要在过期窗口内签；如果要更早签，改用 durable nonce |
| `preopen_probe` | 开盘前先模拟一次，再在链上真发一笔小额买单，预期失败，用来证明 gate 确实生效 |
| `detect.max_buy_fee_bps / abort_on[]` | 开盘那一刻手续费超过上限就放弃；命中哪些检测项就放弃 |
| `exit.*` | 见第 4 节 |

## 2. 资金准备（SOL 和 USDC 两种情况）

- **报价币是 SOL**：swap 时现场把 SOL 包成 WSOL（转账加 `syncNative`），交易结束时关掉 WSOL 账户，租金退回。余额要够：买入金额 + WSOL ATA 租金 0.00203928 SOL（会退回）+ 目标代币 ATA 租金约 0.002 SOL（不会退，Token-2022 带扩展时更高）+ 每个签名 5000 lamports + priority fee + Jito tip。
- **报价币是 USDC 或其他代币**（例如 TRUMP、MET 开盘用的都是 USDC 池）：要提前在热钱包里准备好 USDC 的 ATA 和余额；另外还要留 SOL 付手续费和租金。
- **开盘前把目标代币的 ATA 建好**（`createAssociatedTokenAccountIdempotent`），这样买入那笔交易里不用再建账户，指令更少、CU 更低。
- 热钱包只放这一单要用的资金；用完把钱转回主钱包，放进复盘清单。

## 3. 检测项（`src/detect.js`，只读不改链上状态）

| 检测项 | 结论 |
|---|---|
| mint authority 还在（可以增发） | REDUCE，或人工判断 |
| freeze authority 还在（可以冻结你的账户） | **ABORT**（默认） |
| Token-2022 TransferFee（转账税）的 bps 和上限；fee config authority 能改税率 | 税率大于 0 时降仓；税率可改要警惕 |
| TransferHook（每次转账都调用一个外部程序，可以做黑名单或卖出限制） | **ABORT**（有人轮换靓号钱包绕过 hook 黑名单，见第 3 部分的复盘） |
| PermanentDelegate（项目方可以直接转走或销毁你的币） | **ABORT** |
| NonTransferable / DefaultAccountState=Frozen / Pausable | **ABORT** |
| 池子状态：activation_type、activation_point、pool_status、alpha vault | 有 alpha vault 说明开盘前已经有人按开盘价买过了 |
| 开盘那一刻的反狙击手续费（fee scheduler 的起始费率；DBC/Believe 的"开头费率极高"） | 用开盘那一刻和费率衰减完之后各报一次价比较；超过 `max_buy_fee_bps` 就 ABORT，或者等费率降下来再买 |
| 流动性和价格 | 决定仓位大小和 minOut |
| 买卖模拟 | 开盘前买入应该失败；开盘后可以先 `simulateTransaction` 模拟一次卖出，确认能卖（防貔貅） |

## 4. 止盈止损策略 A–F 怎么套到 Solana 上

| BSC/Base 版 | Solana 版 |
|---|---|
| A 同一块里分几片卖，最后一片卖真实余额 | 一笔交易里放多个 swap 指令，或者一个 Jito bundle（最多 5 笔，原子执行）；最后一片必须读 `getTokenAccountBalance` 拿真实余额 |
| B 目标价阶梯，带 minOut | `exit.mode=targets`，比如 `targets:[{mult:1.3,bps:3000},…]`。价格从池子的 sqrtPrice 读，每片都用 getQuote 算 minOut。**本地网已经实测** |
| C 分批、轮换热钱包 | 多个 `keys/hotN.json`。Solana 没有 nonce 冲突的问题，但每个钱包都要单独准备 ATA 和 SOL |
| D 移动回撤（trail_arm_mult） | 用 `exit.trail.{arm_mult,drawdown_bps}`：价格先涨到入场价的 arm_mult 倍，之后从最高点回撤超过 drawdown_bps 就清仓 |
| E 多个子合约或子地址 | 只有当代币限制单个地址持仓时才用（例如 Token-2022 加 hook 做的"最多持有 1.5%"），用多个钱包分开买 |
| F 止损/熔断 | `exit.stop_loss_mult`；连续失败 N 次就停 |
| keep_bps | 留一点底仓（默认 0） |
| 开盘时不走聚合器（Kyber 留残的教训） | 开盘时直接调用池子的程序（本脚本直接调 cp-amm 的 swap），不走 Jupiter；等路由稳定后再考虑用 Jupiter 卖 |

## 5. 预热、延迟测量和发送

1. **T−30 分钟**：
   - 跑 `detect.js`，确认 gate、费率、报价币；
   - 检查余额，建好 ATA；
   - 测 RPC 延迟，确认 slot 跟得上，用 `getSlot` 和 gRPC 的 slot 流对比；
   - 看 leader schedule（`getLeaderSchedule`），确认开盘那几个 slot 是谁出块，是不是 Jito 验证者（不是 Jito 验证者的话 tip 没有用）。
2. **T−10 秒冻结配置**，不再改任何参数。
3. **开盘前 30 个 slot 左右**：拉最新池子状态 → 按开盘那一刻报价算 minOut → 构建交易（ComputeBudget limit 和 price + swap + 可选的 Jito tip）→ 签名，在内存里保存好序列化后的交易。
4. **开盘前的探针**：模拟买入，确认返回 6003 或"还没开盘"的错误；也可以在链上真发一笔，留下签名作为证据。
5. **到点**：
   - **slot gate**：轮询 `getSlot(processed)`，到达 `AP − lead_slots` 就发。实测：`lead_slots=0` 落在开盘后第 1 个 slot；`lead_slots=1` 正好落在开盘那个 slot。
   - **时间戳 gate**：必须读链上 `Clock.unix_timestamp`（SYSVAR_CLOCK，processed 级别）。实测本地时钟比链上快 1–2 秒，按本地时钟发会因 6003 失败。
   - 用 `skipPreflight` 和 `maxRetries=0`，把同一个签名同时发给多个 RPC 或发送通道（staked/SWQoS RPC、Jito、0slot、Nozomi 等）。确认之前每 2 秒重发一次。
6. **基础设施**（主网要不要花钱买）：
   - 收消息：Yellowstone gRPC（processed 级别，执行后的数据）；更早的原始 shred 数据流（**Jito ShredStream 已于 2026-09-05 停止服务**，官方推荐迁移到 DoubleZero Edge）。
   - 发交易：staked RPC 或 SWQoS（leader 约 80% 的接收带宽留给有质押的连接）+ priority fee（决定在 leader 队列里的排序）+ Jito tip（只在 Jito 验证者当 leader 时有效）。

## 6. 收尾和复盘

- `logs/txlog.jsonl` 记录每一笔交易；`logs/run_<id>.json` 记录每一次运行，内容包括：检测结果、计划（amountIn、报价、minOut、开盘时手续费）、每笔交易的签名、explorer 链接、落在哪个 slot、开盘后第几个 slot 或第几秒、盈亏（SOL）、剩余代币数量。
- 复盘要回答这几个问题：
  - 落在开盘后第几个 slot；
  - 被谁抢在前面（看同一个 slot 里排在你前面的交易）；
  - 实际付的手续费和 tip；
  - 滑点和反狙击费各占多少；
  - 卖出每一片的成交价。
- 收尾：把热钱包余额转回主钱包；关掉空的代币账户回收租金（`closeAccount`）；把 `mainnet_send_enabled` 改回 false。


## 7. 开盘模式 → 策略速查（2026-10-24 补）

| 你看到的开盘 | 用哪条路径 |
|--------------|------------|
| 已知 `activation_point` / `openTime` | `src/snipe.js` 预签到点发；时间戳读链上 Clock |
| 建池即开 / pump 迁移 / 手动“突然建池” | `src/snipe_follow.js` 发现账户后立即买；记 slot 延迟 |
| 高反狙击费 scheduler | 用 quote 算最优 slot，未必抢 +0（见 REPORT §5.9.3） |
| Raydium CPMM diget | `create_raydium_cpmm.js` + `snipe_raydium_cpmm.js` |
| Token-2022 税/hook/freeze | `detect.js` ABORT，不狙击 |

完整分类与 X/文档链接：`research/launch_modes_cn.md`。


## 7.1 手法补全入口（2026-10-24）

| 模式 | 命令 |
|------|------|
| DLMM 定时 | `node src/create_dlmm.js configs/modes/dlmm-timed.json` → `node src/snipe_dlmm.js …` |
| DBC 即开 | `node src/create_dbc.js configs/modes/dbc-instant.json` → `node src/snipe_dbc.js …` |
| pump 监听/模拟 | `node src/snipe_pump.js configs/modes/pump-follow.json --listen 30` |
| mint 检测 | `node src/detect_mint.js <mint>` |
| 费率最优估算 | `node src/fee_optimal.js configs/modes/fee-early.json` |

共享：`src/exit.js`（A/B/D/F）、`src/tx_send.js`（多入口/Jito 字段默认关）、`src/rpc_retry.js`（限流退避）。开源吸取见 `research/github_review.md` 与 REPORT §5.10.2。
