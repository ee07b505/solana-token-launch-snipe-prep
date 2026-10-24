# Solana 新盘开盘狙击 · 调研与实践报告

日期：2026-10-23（UTC+8）。目录：`/workspace/sol-sniper`。
方法沿用「区块链链上专家」做 Base/BSC 狙击时的 skill：`token-launch-snipe-prep`。
配套清单：`SOL_SNIPE_PREP_CN.md`；GitHub 审查记录：`research/github_review.md`。

---

## 一、开盘机制调研（哪些平台会定时开盘）

| 平台 | 开盘方式 | 有没有定时 gate | 说明 |
|---|---|---|---|
| pump.fun 内盘 | 即时开盘、按曲线成交 | 无 | create 一落块就能买。creator 常把 create 和 buy 打进同一个 Jito bundle，所以在建币那一块里买入的多半是项目方自己（Pine Analytics 统计：超过 50% 的代币在建币那一块就被买入） |
| PumpSwap（pump 毕业后） | 即时开盘 | 无 | 内盘卖完约 793M 枚代币（约 85 SOL）后，任何人都可以触发迁移，费用 0.015 SOL；迁移那笔交易里建池，建好立即能交易 |
| Raydium LaunchLab / letsbonk（bonk.fun） | 即时开盘、按曲线成交 → 迁移到 CPMM | 内盘无 | 85 SOL 毕业（LaunchLab 模式下 creator 可以自定，最少 24 SOL）；自 2026-08-17 起毕业后统一迁移到 CPMM |
| **Raydium CPMM** | **定时（open_time）** | 有，时间戳 | 程序用 `max(open_time, block_ts+1)`，开盘前 swap 会被拒绝。devnet 上的程序 ID 是 DRaycpLY18Lh… |
| **Raydium AMM v4** | **定时（open_time）** | 有，时间戳 | `initialize2` 里带 open_time，可以设成未来时间；这是经典的狙击触发信号 |
| **Meteora DAMM v2（cp-amm）** | **定时（activation_point）** | 有，slot 或时间戳 | activation_type=0 按 slot，=1 按时间戳。开盘前 swap 报 6003 PoolDisabled。支持手续费调度器（线性或指数衰减）做反狙击；可以挂 Alpha Vault。MET 代币 TGE 用的是 DAMM v2 的 USDC 池 |
| **Meteora DLMM** | **定时（activation_point）或管理员手动开关** | 有 | 有 `pre_activation_duration`（Alpha Vault 在开盘前可以先买的时间窗口）。TRUMP（2025-01）用的就是有权限控制的 DLMM 池，开盘前有人尝试买入，交易失败 |
| Meteora DBC（Believe、Bags 等） | 即时开盘、按曲线成交 → 迁移到 DAMM | 内盘无 | Believe 的做法：市值到 $10 万毕业；为了防狙击，"开头费率极高，之后逐步降到 2%" |
| Orca Whirlpool | 即时开盘 | 无 | initializePool 里没有开盘时间字段，流动性要另一笔交易加。要监听第一次加流动性 |

**结论**：
- **定时开盘**：CPMM、AMM v4（open_time）；Meteora DAMM v2 和 DLMM（activation_point）。
- **即时或手动**：pump.fun、PumpSwap、LaunchLab/letsbonk、DBC/Believe、Orca，以及 Meteora 有权限控制的池（管理员手动开）。
- 定时开盘是本方案的主战场。即时开盘那一类比的是监听和发送速度，以及能不能挤进同一个 bundle；而建币那一块里的买入，基本被项目方自己的 bundle 占掉了。

### Solana 基础设施要点
- **出块和 leader**：一个 slot 约 400ms；leader schedule 提前一个 epoch 就能查到（`getLeaderSchedule`）。只有当前 leader 跑的是 Jito-Solana 客户端时，Jito tip 才有用。
- **priority fee**：计算方式是 CU limit × CU price（micro-lamports）÷ 1e6；**按申请的 CU limit 收费**。leader 排序看的是 (fee 奖励)/(cost) [solana.com fee-structure](https://solana.com/docs/core/fees/fee-structure)。
- **Jito**：
  - bundle 最多 5 笔交易，原子执行；最低 tip 1000 lamports；tip 账户从 8 个里随机选一个，不要用地址查找表（ALT）引用 tip 账户；
  - 单笔 `sendTransaction` 时，官方建议 priority fee 和 tip 按 70/30 分；
  - 自 2024 年起 Jito 关闭了公开 mempool，夹子机器人转去了私有通道。
  - 来源：[docs.jito.wtf](https://docs.jito.wtf/lowlatencytxnsend/)、[X: milkblue1](https://x.com/milkblue1/status/1893619182958170178)
- **SWQoS 和 staked RPC**：leader 大约 80% 的接收带宽留给有质押的连接。SWQoS 决定交易**能不能送到 leader**，priority fee 决定送到之后**在 leader 队列里的排序**，两者缺一不可。来源：[solana.com SWQoS](https://solana.com/docs/defi/stake-weighted-qos)、[Helius](https://www.helius.dev/blog/stake-weighted-quality-of-service-everything-you-need-to-know)
- **收数据**：
  - Yellowstone gRPC（Geyser，processed 级别，交易执行后的数据）[rpcpool/yellowstone-grpc](https://github.com/rpcpool/yellowstone-grpc)；
  - **Jito ShredStream 已于 2026-09-05 停止服务**，官方建议迁移到 DoubleZero Edge [docs.jito.wtf/lowlatencytxnfeed](https://docs.jito.wtf/lowlatencytxnfeed/)。shred 流比 gRPC 早几到十几毫秒，但给的是还没执行的原始数据。
- **Token-2022 检测项**：
  - TransferFee（转账税，以及谁有权改税率）；
  - TransferHook（可以做黑名单或单地址持仓上限，例如有项目用 Token-2022 限制每个地址最多持有 1.5%：[X: Autonoma1st](https://x.com/Autonoma1st/status/2087246158619942936)）；
  - PermanentDelegate、NonTransferable、DefaultAccountState=Frozen、Pausable、freeze authority、mint authority。
  - 这些都已经在 `src/detect.js` 里实现。
- **Meteora 文档**：
  - [Alpha Vault](https://docs.meteora.ag/helper-products/alpha-vault/what-is-alpha-vault)
  - [DLMM Launch Pool](https://docs.meteora.ag/invent/launch-pools/dlmm-launch-pool)
  - [launch.meteora.ag（反狙击套件，费率最高可以设到 99%）](https://launch.meteora.ag/)
  - [DLMM 更新日志：activation_type 和 activation_point](https://github.com/MeteoraAg/dlmm-sdk/blob/main/CHANGELOG.md)
- **毕业门槛**：[memefees 各平台毕业门槛对比](https://memefees.com/learn/memecoin-graduation)、[Believe FAQ](https://believe.app/faqs)、[bonk.fun / LaunchLab](https://solfoundry.io/launchpads/bonk-fun)

### 2025–2026 案例
- **TRUMP（2025-01，Meteora DLMM，USDC 池）**：
  - 有交易员在开盘前几小时尝试买入，因为池子还没被管理员打开，交易失败 [Decrypt](https://decrypt.co/306872/how-100-million-trump-meme-coin-whale-inside-scoop)；
  - 开盘那一块，地址 6QSc2 付了 357 SOL（约 $7.8 万）的 Jito tip，买入 597 万枚 TRUMP [Arkham](https://info.arkm.com/research/trump-sniper-buying-5m-libra)。
- **MET TGE（2025-10，DAMM v2 USDC 池）**：[X: rodionadov](https://x.com/rodionadov/status/1978475833644757400)
- **Meteora 反狙击工具**：[X: Hashbergers](https://x.com/Hashbergers/status/1924541848250466469)

## 二、准备清单
见 `SOL_SNIPE_PREP_CN.md`，包含：
- 开盘类型判断表；
- 配置字段（主网三重开关）；
- SOL 和 USDC 两种资金准备，以及租金、ATA、WSOL；
- 检测项和 OK / REDUCE / ABORT 判定；
- 策略 A–F 在 Solana 上的写法；
- T−30 分钟预热、T−10 秒冻结、提前签名、到点按 gate 发送；
- 收尾和复盘。

## 三、真实狙击复盘（X 和网页）
| 来源 | 要点 |
|---|---|
| [Pine Analytics 长文](https://pineanalytics.substack.com/p/exit-liquidity-machines) / [X](https://x.com/PineAnalytics/status/1914301091337953734) / [BeInCrypto](https://beincrypto.com/pump-fun-meme-coin-snipers-systematic-problem/) | pump.fun 上超过 50% 的代币在建币那一块就被买入；大量狙击钱包的启动资金来自部署者 → 在建币那一块里跟买，大多是在给项目方接盘 |
| [DeFade_ $TRADER](https://x.com/DeFade_/status/2070813534275985803) / [$BULLARC](https://x.com/DeFade_/status/2076713266127024163) / [$ATTENTION](https://x.com/DeFade_/status/2056833292800823737) | 用 Jito bundle 取证：同一个 bundle 里 8 个钱包拿走 81.49% 的筹码；另一个币 86%；另一个 20% → 买前要查捆绑持仓 |
| [Decrypt TRUMP](https://decrypt.co/306872/how-100-million-trump-meme-coin-whale-inside-scoop) / [Arkham](https://info.arkm.com/research/trump-sniper-buying-5m-libra) | 有人从链上管理员的异常操作提前发现池子；开盘前买入失败；开盘那一块靠高额 tip 抢到 |
| [_onchain](https://x.com/_onchain/status/2106406109888266503) | 有人轮换靓号钱包，绕过 TransferHook 的黑名单 |
| [LineageMemes](https://x.com/LineageMemes/status/2031057315663605893)、[LegendaryLibr](https://x.com/LegendaryLibr/status/1923780991476256912)、[truceefun](https://x.com/truceefun/status/2109699345159446584)、[em1lyx587](https://x.com/em1lyx587/status/1989037662783910064)、[btc99ltc](https://x.com/btc99ltc/status/2042161947127476393)、[0xfakeSpike](https://x.com/0xfakeSpike/status/1937450840027267382) | 其他狙击和反狙击的实际经历（链接已保留，详情见各推文） |

**复盘结论**：
1. 即时开盘类的建币那一块，基本是项目方自己的 bundle 先买 → 不要盲目跟。
2. 定时开盘类比的是到点的精度（slot 级）、发送通道（SWQoS/Jito）和开盘那一刻的费率。
3. 反狙击手续费会把狙击利润吃掉，必须先报价再决定。

## 四、GitHub 开源代码评估
详见 `research/github_review.md`。要点：
- ⚠️ **MortarStallionMarina/Solana-PumpFun-Sniper-V2（65★）是窃密木马**：偷钱包扩展数据、Telegram 会话、.env 文件，通过 Telegram Bot 外发。
- ⚠️ **mortdeus/solana-copy-sniper-mev-trading-bot（4502★）高风险**：依赖 `web3-fb`，npm 上不存在，指向第三方 GitHub release 的 tarball；另有非官方的 `pump-swap-core-v1`。
- **推荐参考**（只读代码、学写法）：
  - chainstacklabs/pumpfun-bonkfun-bot（Apache-2.0）：pump.fun、letsbonk 的流程；
  - 0xfnzero/solana-streamer（MIT）：gRPC 和 shred 事件解析；
  - outsmartchad/onchain-dex-cli（MIT）：多 DEX 适配和多通道发送。
- 没有任何开源项目实现"activation_point / open_time → 提前签名 → 到点单发"这一套，所以用我们自己的脚本。

## 五、测试网实践

### 5.1 环境和钱包
- 已安装 Solana CLI（validator、`solana`）、Node，以及 `@meteora-ag/cp-amm-sdk`、`@solana/web3.js`、`@solana/spl-token`。
- 钱包：`keys/deployer.json`（公钥 7U4fiWX2YkKQFqE1unyxogSesLsB4vnfv8dcDoGf4Ly4）和 `keys/sniper.json`（公钥 5bXC5sJ5PXgvmNhNaNd8YXzr43Qo77WmMYfmvp8fXXVN），权限都是 600，私钥从没打印过。`loadKey` 遇到权限过宽的文件会拒绝加载。

### 5.2 devnet 领水（先失败，后由用户手动领到）
- `requestAirdrop` 和后台重试脚本前后共请求 **96 次，全部返回 429**（"airdrop limit / faucet has run dry"），两个钱包的 devnet 余额都是 0。
- testnet 水龙头也被限流；而且 testnet 上没有部署 Meteora cp-amm 程序。
- faucet.solana.com 需要在浏览器里用 GitHub 登录，这一步我做不了。
- **更新（15:23 UTC+8）**：用户用 faucet.solana.com 手动领水，部署钱包 devnet 余额实测 **10 SOL**（不是 5）。公开 devnet 重跑结果见 5.6。
- **当时的替代方案**：在本机起了一个 `solana-test-validator`，从 devnet **克隆了真实的 Meteora cp-amm 程序**（cpamdpZC…，和主网是同一个 ID）和 Raydium CPMM 程序。合约字节码和 devnet/主网一致，**但这是本地网，不是公开的 devnet**。下面的签名只能在本机 RPC（http://127.0.0.1:8899）上查到，链接用的是 explorer 的 custom 模式，只有能访问这个 box 上 8899 端口的浏览器才能打开。

### 5.3 测试流程
1. `src/create_pool.js`（只用于测试，遇到主网会拒绝执行）：
   - 新建一个 6 位小数的测试代币，用 Meteora `createCustomPool` 建一个 代币/WSOL 的 DAMM v2 池；
   - 开盘方式用 slot 或时间戳的 activation_point；
   - 反狙击手续费按指数衰减：1500→100 bps，分 10 期、共 50 个 slot。
2. `src/detect.js`：读代币属性、Token-2022 扩展、gate、开盘那一刻的费率 → 给出 OK / REDUCE / ABORT。
3. `src/snipe.js`：
   - 检测，并核对报价币；
   - 预建 ATA，检查余额；
   - 开盘前探针：在链上发一笔，预期失败；
   - 提前签名，minOut 按开盘那一刻报价算；
   - 到点只发一笔，带 CU price 和 minOut；
   - 按配置卖出（ladder，或目标价、移动回撤、止损），最后一笔卖出链上真实余额；
   - 每一笔都记录签名、链接、落在哪个 slot 和相对开盘的偏移，写入 `logs/`。
4. 主网三重开关一直是关的：`mainnet_send_enabled=false`、没有 ack、没有设环境变量；而且程序按 genesis hash 识别主网。

### 5.4 结果（本地网，2026-10-23 约 12:04–12:11 UTC+8）

#### Run1 slot gate, lead_slots=0, ladder 分片卖出
池子 `743vftPoqarzAmd6atrfPSkpHFwLhv8sAyJvSSPvXZT` · AP=slot 472 · 盈亏 −0.01196 SOL（亏损主要来自开盘那一刻 15% 的反狙击手续费）

| 步骤 | 落块 slot（相对开盘） | 签名 |
|---|---|---|
| 建池 | — | [`2EVBJvwxuC…`](https://explorer.solana.com/tx/2EVBJvwxuC3eEDjwJQrq1gD7DuE845tiXLBjTZCa1FTynQD3ueNMWcypGTBsn4d4TKM99P1j5Mqcju4yKMDeKQAU?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| 开盘前探针（预期失败）✅失败 | 332 (−140) | [`ozbFuMrbgA…`](https://explorer.solana.com/tx/ozbFuMrbgAsYmfUBFjiUb9fY5aN5sTnSjnfUNSgMmpDmMq5VN1QGS6KdDdz8s2YAdAhHpaXX6uFeNiy9A7mXbkQ?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| BUY 买入 | 473 (+1) | [`2TS6iyuF9u…`](https://explorer.solana.com/tx/2TS6iyuF9ufG4oKEopQRfK9W3B8DszhZTTe1YPjg81se9U1V1Wk2qqXLSGD3R9Gekft1G2Wk4BZYhZaZ2dZu4Rww?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| ladder 第 1 片 30% | 477 (+5) | [`2CQd3t62sg…`](https://explorer.solana.com/tx/2CQd3t62sgsDPKyVJ4m3d4DYWSjYA1egwF81KsJ48yEsBmm3FJAY4YDDh4yC4nKmHkKm7z9CLv3TSNSHk7HsC1Yr?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| ladder 第 2 片 30% | 480 (+8) | [`FkLac8ENU5…`](https://explorer.solana.com/tx/FkLac8ENU5oZES8t1uEtAAVuBaR2CdCDSZgHUumNu62sQcxdidQZgt9fp1F4qco3eCe4SGGj6s2GExdJ4ggEWbk?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| 最后一笔卖出真实余额，卖完剩 0 | 483 (+11) | [`ZGC9YiTDfc…`](https://explorer.solana.com/tx/ZGC9YiTDfcg9RpSY7h4JrCfqrgoQ8wp9JK8bvxXKJvYAnDAu213qFSKvEKYZoWGTfiE5XjgrVcG4phZdXKPcCbv?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |

#### Run2 slot gate, lead_slots=1, 目标价 B + 移动回撤 D + 止损 F
池子 `EE57xDcM6QbJU4WKqEbKuKFCeS6tk87QpnHGKzE2PL8b` · AP=slot 670 · 盈亏 +0.01322 SOL（**是人为造出来的**：测试助手让 deployer 自己买了 0.6 SOL 把价格拉上去，只是为了验证止盈逻辑，不代表真实盈利）

| 步骤 | 落块 slot（相对开盘） | 签名 |
|---|---|---|
| 开盘前探针（预期失败）✅失败 | 525 (−145) | [`6Dc89MeD2g…`](https://explorer.solana.com/tx/6Dc89MeD2gUmmw2E4B3AFb3F7k9WSC7u9BrLikWkfxGo41rD1cxezi7BRWNcSxEQUe5nbXqRecgpDD4abXUyKBG?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| BUY 买入 | 670 (+0) 正好落在开盘那个 slot | [`5HSzhra5e6…`](https://explorer.solana.com/tx/5HSzhra5e6sUKXRfeDkV4QfzezFCNEFfXSJofRAf7itiUPK4Q9vZQWmga2k9CGAiktQ77Fu2Ej9cmWtsBcvhtn2h?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| 测试助手拉盘（只在测试网允许） | 671 (+1) | [`45wQdLQiV5…`](https://explorer.solana.com/tx/45wQdLQiV5ZhBR9gQY8CBGp4p5G2QwhaywgKz9LnE43BNbw9LUX3g47NZdh7rH6vsbPatRfTfRDibezANJNmgRug?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| 目标 ×1.3 触发（当时价格 ×1.356），卖出 30% | 675 (+5) | [`5D2oz5b5cY…`](https://explorer.solana.com/tx/5D2oz5b5cYfCX5nuh4nr81SwcSRPSe3HWofWF6E4p8sfMzkGV2PAndzD7CdThZucysAWfQzVp2nLn3SCkhH37D8q?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| ×1.6 没到，超时后卖出全部真实余额 | 824 (+154) | [`4rETgeMxkZ…`](https://explorer.solana.com/tx/4rETgeMxkZVPdBWH3zEWiNwiHgsQqrCSYVzXpVj2TeNAuhMxMfizQMakgaVAr1ed647RjrooA9aGKcL7Z7jDnn6H?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |

#### Run3 时间戳 gate，用**本地时钟**判断开盘（失败案例）
池子 `DTNcsEfzja68t5uZxT2xQdunpyumoyyoYtRCDcFfJCSS` · AP ts=1792728525 · 盈亏 没有成交

| 步骤 | 落块 slot（相对开盘） | 签名 |
|---|---|---|
| 开盘前探针（预期失败）✅失败 | 847 | [`4HgYcHwFVL…`](https://explorer.solana.com/tx/4HgYcHwFVLFpiYrXfhJxgSFqYWp49V65xVqPF847No3mhD7gYWkDmGeqC3wSnRQWN8ecMxHz1kTqPAeEe9Yx4fuY?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| BUY ❌ 6003 PoolDisabled（本地时钟比链上快 1–2 秒，发早了） | 982（链上时间还没到 AP） | [`3ogcqaBHjP…`](https://explorer.solana.com/tx/3ogcqaBHjPoD2ZxZTn7Yo3ELgSnysmMRSRRxHaEKxFs8NMSqtQDigqApNZ3sDbritr85S2wvmJqTnkMXbYgXY9cT?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |

#### Run4 时间戳 gate，改用**链上 Clock** 判断开盘（修复后）
池子 `B5QPA76mLMufDghkrHoTLYzNMZUVgiRD7v4oFzuBxjKH` · AP ts=1792728598 · 盈亏 −0.01393 SOL（主要是反狙击手续费）

| 步骤 | 落块 slot（相对开盘） | 签名 |
|---|---|---|
| 开盘前探针（预期失败）✅失败 | 1021 (−58s) | [`3vPbeHn1Zv…`](https://explorer.solana.com/tx/3vPbeHn1ZvS7E7q6kb84JadKsENrirVFSZJCHrXvtciDgu11qtPNLp6ziKERHEGNghrZfoP49DpvjEceEqsHrRdY?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| BUY 买入 | 1159 (+0s) | [`2QtL1wLLS7…`](https://explorer.solana.com/tx/2QtL1wLLS7RASXwKEKvWErshQZy91BQjkYEYMa7YUSXHnKra48kxFCfJbpcYZwxejMupzRmyY2yEyRM94ucAnhVN?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| ladder 第 1 片 | 1163 (+2s) | [`5tXGJqBfKY…`](https://explorer.solana.com/tx/5tXGJqBfKYHVGTjPESTjYXXNubRP273FBdJ5UFRtFmKYTUUdqoLtZW5X5uqvLn7BcMXrFpfp9ZJ77mZiFRC2nny9?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| ladder 第 2 片 | 1166 (+3s) | [`3qgvSQzUaZ…`](https://explorer.solana.com/tx/3qgvSQzUaZjtxTAHFM8saDetsP6c9CXiNS9FFWzD5yktfbwyCAJ3grJHaEmtdkcXBFHPg5SLLtvafyHjk2So86zK?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |
| 最后一笔卖出真实余额 | 1169 (+4s) | [`3PdjK9Le2Z…`](https://explorer.solana.com/tx/3PdjK9Le2ZVnHshYEQWcANVFGti1qCbsTityc4UbbKNvs9BZxcfr81heQ16bKfS2v3P6bYZGTwY4ukMbgeWfzizs?cluster=custom&customUrl=http%3A%2F%2F127.0.0.1%3A8899) |

### 5.5 实测得到的教训
1. **slot gate**：当前 slot 等于 AP 时才发，会落在 **+1**；提前 1 个 slot 发（`lead_slots=1`）正好落在 **+0**。主网上网络延迟更大，lead_slots 要按测到的延迟来调，代价是可能落在开盘前一个 slot 而失败（所以要配合重试）。
2. **时间戳 gate 必须看链上 Clock**：本地时钟比链上 `Clock.unix_timestamp` 快 1–2 秒，Run3 因此报 6003 失败；Run4 改成读链上 Clock 后落在 +0s。
3. **失败后不能用同一个签名重发**：必须换新 blockhash 重新签名（`max_retries_not_open`）。
4. **反狙击费的威力**：开盘那一刻 15% 的手续费，直接让 Run1 和 Run4 在价格不动的情况下亏了约 24–28%（来回各付一次手续费加滑点）。开盘那一刻的费率必须先报价再决定买不买。
5. 最后一笔按真实余额卖出，每次都把代币清到 0。

### 5.6 公开 devnet 重跑（2026-10-23 约 15:23–15:32 UTC+8）

**环境**
- 程序：真实的 Meteora DAMM v2 devnet 程序 `cpamdpZCGKUy5JxQXB4dcpGPiikHawvSWAd6mEn1sGG`。
- RPC：`api.devnet.solana.com` 在这台 box 上 TLS 握手失败（curl exit 35，域名被本地代理解析成 198.18.0.1，强制 IPv4、TLS1.2 都试过，仍然失败）。Ankr 和 Helius 要 key，drpc 免费版不支持 devnet，publicnode 返回空。最后改用 **OnFinality 公共 devnet RPC** `https://solana-devnet.api.onfinality.io/public`，按 genesis hash 校验过确实是 devnet。这个 RPC 没有 websocket，所以加了 `confirm_by_polling`；它还频繁返回 429，所以确认轮询放慢到 400ms，gate 轮询 250ms。
- 资金：部署钱包转 1 SOL 给狙击钱包，[2yZaQuQxdwPYAapovELcYpvGpEYx2z3RZC7QthrbZbngKo97Fa4N8rJxu3f8Sm3144dfrGV35yQz9rdMc6SoZQsy](https://explorer.solana.com/tx/2yZaQuQxdwPYAapovELcYpvGpEYx2z3RZC7QthrbZbngKo97Fa4N8rJxu3f8Sm3144dfrGV35yQz9rdMc6SoZQsy?cluster=devnet)。
- 省钱参数：每笔买入 0.02 SOL，每个池子注入 0.3 SOL 流动性，Run2 拉盘 0.1 SOL。其余配置和 localnet 相同：开盘前 150 slot 或 60 秒建池，反狙击费 1500→100 bps。
- 执行顺序：Run1（优先组 A）→ Run4（优先组 B）→ Run2 → Run3。资金充足，**4 组全部跑完**。

#### Run1 slot gate, lead_slots=0, ladder 分片卖出（优先组 A）
池子 `6HNgEEBBbUeRbnsgwrz5BYkv44xbMhM4EEXTpK1cHCAV` · AP=slot 514170149 · 狙击钱包盈亏 −0.001997 SOL · 卖完代币剩 0

| 步骤 | 结果 | 落块 slot（相对开盘） | 完整签名 / 链接 |
|---|---|---|---|
| 建池（deployer） | ✅ | — | [5yD4DPt3TWJ5voktGrdTZir5ampD6dDWbo5wfF1e5fhx7wGwHEZSuaiTQFkGbNp71JyC3kfruUUpemYcXWzvjAm5](https://explorer.solana.com/tx/5yD4DPt3TWJ5voktGrdTZir5ampD6dDWbo5wfF1e5fhx7wGwHEZSuaiTQFkGbNp71JyC3kfruUUpemYcXWzvjAm5?cluster=devnet) |
| prep_create_target_ata | ✅ | — (—) | [5AWAuYGf6p5yhpiqcpkDrWXE34oYbbgEN7vWaFqwezCuyh5uK43VwsrN4qLZvUBvfH9DGeaXZNBdYJoRdXaLhArY](https://explorer.solana.com/tx/5AWAuYGf6p5yhpiqcpkDrWXE34oYbbgEN7vWaFqwezCuyh5uK43VwsrN4qLZvUBvfH9DGeaXZNBdYJoRdXaLhArY?cluster=devnet) |
| preopen_probe_buy(expect_fail) | ✅失败（6003 PoolDisabled，符合预期），发送时点位 514170083 | 514170100 (-49 slot) | [CfENdU1acA1HX1wpEFN91Wc9p7VbcAw55qhB3VbFrV3Lr4dcX92npkvdehG5sgdG5ykFTZmZj1UrJ9NVxkJXzaf](https://explorer.solana.com/tx/CfENdU1acA1HX1wpEFN91Wc9p7VbcAw55qhB3VbFrV3Lr4dcX92npkvdehG5sgdG5ykFTZmZj1UrJ9NVxkJXzaf?cluster=devnet) |
| BUY | ✅，发送时点位 514170153 | 514170162 (+13 slot) | [59qbmNNdB4WLxfiS8kvRXgVVp89eDAeUVqH8YzXuztgoooEeavTo9p1TvaKf1Kwu4ya57RbzkdnXdsvQeRaNvKpX](https://explorer.solana.com/tx/59qbmNNdB4WLxfiS8kvRXgVVp89eDAeUVqH8YzXuztgoooEeavTo9p1TvaKf1Kwu4ya57RbzkdnXdsvQeRaNvKpX?cluster=devnet) |
| SELL_ladder_1_3000bps | ✅ | 514170194 (+45 slot) | [5HWHDmMHGPscbBsLYU2z7SF2fL6UGj5QNKUiLPZnNn4MbxwW8uHrFv9bBsJCGBQczRxtGLTeBUZwLNvuhNznbCVV](https://explorer.solana.com/tx/5HWHDmMHGPscbBsLYU2z7SF2fL6UGj5QNKUiLPZnNn4MbxwW8uHrFv9bBsJCGBQczRxtGLTeBUZwLNvuhNznbCVV?cluster=devnet) |
| SELL_ladder_2_3000bps | ✅ | 514170220 (+71 slot) | [3D9maGBL971jZT5KFznC7sgW3CvNCBr49Y8JbJqQ4RqYMi94XtQij1mvA23Jz61bFZtg3jL5AZEueGYPqzMC9Ne3](https://explorer.solana.com/tx/3D9maGBL971jZT5KFznC7sgW3CvNCBr49Y8JbJqQ4RqYMi94XtQij1mvA23Jz61bFZtg3jL5AZEueGYPqzMC9Ne3?cluster=devnet) |
| SELL_final_real_balance(keep_0bps) | ✅ | 514170258 (+109 slot) | [4Kp4Vt6HMomdLJg8GVWDsX1sDzMFAvVmgoRVGKzMUsmizUkVAU9oeArFUgqXRNddZtxNnQ8hQbsNua2MsMJ9WsV6](https://explorer.solana.com/tx/4Kp4Vt6HMomdLJg8GVWDsX1sDzMFAvVmgoRVGKzMUsmizUkVAU9oeArFUgqXRNddZtxNnQ8hQbsNua2MsMJ9WsV6?cluster=devnet) |

#### Run4 时间戳 gate，读**链上 Clock**（优先组 B）
池子 `TBUmT7NdRKZi3sC75i64svUSNfGkQ2mAJpd9mpQV2Tj` · AP ts=1792740435（15:27:15 UTC+8） · 狙击钱包盈亏 −0.004109 SOL · 卖完代币剩 0

| 步骤 | 结果 | 落块 slot（相对开盘） | 完整签名 / 链接 |
|---|---|---|---|
| 建池（deployer） | ✅ | — | [3NbcKWKdtmcnFjCXeDx1Bfr2dC6ckccnvhYbMAraVk5wTa5ZT8m8u3fn8adbS7aNRK5rz8XvtVGLEbr7PXGpfM34](https://explorer.solana.com/tx/3NbcKWKdtmcnFjCXeDx1Bfr2dC6ckccnvhYbMAraVk5wTa5ZT8m8u3fn8adbS7aNRK5rz8XvtVGLEbr7PXGpfM34?cluster=devnet) |
| prep_create_target_ata | ✅ | — (—) | [3jb7CZ5jR3twcd4eSb74kBL6iQLgbna1JynFrXGFPzRhKxby5g1WaFqMjU8EWVuqRcpfTq7LBb3tnTWpGxGQVnwA](https://explorer.solana.com/tx/3jb7CZ5jR3twcd4eSb74kBL6iQLgbna1JynFrXGFPzRhKxby5g1WaFqMjU8EWVuqRcpfTq7LBb3tnTWpGxGQVnwA?cluster=devnet) |
| preopen_probe_buy(expect_fail) | ✅失败（6003 PoolDisabled，符合预期），发送时点位 1792740392 | 514170452 (-39s) | [2aYvGKBoCD9UQNfuUwoB3EXWB8CwaHnUu2L9kriEjRKP8xsmurWc3XL7Ua4Lra2fEixjb67qzRmRQ675T4LqBv2i](https://explorer.solana.com/tx/2aYvGKBoCD9UQNfuUwoB3EXWB8CwaHnUu2L9kriEjRKP8xsmurWc3XL7Ua4Lra2fEixjb67qzRmRQ675T4LqBv2i?cluster=devnet) |
| BUY | ✅，发送时点位 1792740436 | 514170623 (+2s) | [5SJ8XcMsDyg39AAEGqgF3ZF35qLGJVYFJrFixuzCoWLaqmqjT5FJgFuFtLBzWeNuGcewnPXQZLEoLq7W5wcmNU8z](https://explorer.solana.com/tx/5SJ8XcMsDyg39AAEGqgF3ZF35qLGJVYFJrFixuzCoWLaqmqjT5FJgFuFtLBzWeNuGcewnPXQZLEoLq7W5wcmNU8z?cluster=devnet) |
| SELL_ladder_1_3000bps | ✅ | 514170657 (+10s) | [4SJnZr4XZBLe27A36qPHSVsfNNo4mgWqLcPuNF2JwgHUoNcNrJCsHFEiJJunJ7BXNVJkSjKQsU37RwF9wR2kHJSe](https://explorer.solana.com/tx/4SJnZr4XZBLe27A36qPHSVsfNNo4mgWqLcPuNF2JwgHUoNcNrJCsHFEiJJunJ7BXNVJkSjKQsU37RwF9wR2kHJSe?cluster=devnet) |
| SELL_ladder_2_3000bps | ✅ | 514170689 (+18s) | [2HsU7BSjbGmquCZGp8t3tnn2CcyQrtU6Y821dUG1N7AAHZZwpqcdBHDniLWvLtwqVGLnuHddtFQixRfR3KnZcL6e](https://explorer.solana.com/tx/2HsU7BSjbGmquCZGp8t3tnn2CcyQrtU6Y821dUG1N7AAHZZwpqcdBHDniLWvLtwqVGLnuHddtFQixRfR3KnZcL6e?cluster=devnet) |
| SELL_final_real_balance(keep_0bps) | ✅ | 514170728 (+27s) | [46je6zPuiZbaG8FmBT9dJWpAMEjz4FibxG9svhbGD383cuij5fbCkzf4ENgzmcGtXJHGuoftPmtgjAuA24r8j72K](https://explorer.solana.com/tx/46je6zPuiZbaG8FmBT9dJWpAMEjz4FibxG9svhbGD383cuij5fbCkzf4ENgzmcGtXJHGuoftPmtgjAuA24r8j72K?cluster=devnet) |

#### Run2 slot gate, lead_slots=1, 目标价 + 移动回撤 + 止损
池子 `8BYbrKeME9aV6HWcHJ4X3Df9CQ3AG9zqxS39SS66qAsX` · AP=slot 514170955 · 狙击钱包盈亏 +0.009212 SOL（**人为造出来的**：deployer 拉盘 0.1 SOL 只为验证止盈逻辑） · 卖完代币剩 0

| 步骤 | 结果 | 落块 slot（相对开盘） | 完整签名 / 链接 |
|---|---|---|---|
| 建池（deployer） | ✅ | — | [3TmSQQqGhknwXsYWkYQTjchsSsd6hgMbw6EYdwYW5rGnQgsbQi9yiwuik7L93ZJASV2iazKdkm2d1rkfFhDF1gmu](https://explorer.solana.com/tx/3TmSQQqGhknwXsYWkYQTjchsSsd6hgMbw6EYdwYW5rGnQgsbQi9yiwuik7L93ZJASV2iazKdkm2d1rkfFhDF1gmu?cluster=devnet) |
| prep_create_target_ata | ✅ | — (—) | [5UihswvzhJQoRcFdHi33GVGVKTEy9VCEu89T3bRokrQdG6L2PZXZD68YSf1raVXBcVJWRwkb6JvK5ThMrLwz8jij](https://explorer.solana.com/tx/5UihswvzhJQoRcFdHi33GVGVKTEy9VCEu89T3bRokrQdG6L2PZXZD68YSf1raVXBcVJWRwkb6JvK5ThMrLwz8jij?cluster=devnet) |
| preopen_probe_buy(expect_fail) | ✅失败（6003 PoolDisabled，符合预期），发送时点位 514170875 | 514170891 (-64 slot) | [g36rqSv7pdRS7tShpi2fa9at7nYoKW5o5iF5FdgAgKF9E8AqoVqLFQiVULkcw7sTo9w5uscTm2u2FUVpSitKMSg](https://explorer.solana.com/tx/g36rqSv7pdRS7tShpi2fa9at7nYoKW5o5iF5FdgAgKF9E8AqoVqLFQiVULkcw7sTo9w5uscTm2u2FUVpSitKMSg?cluster=devnet) |
| BUY | ✅，发送时点位 514170961 | 514170963 (+8 slot) | [64Vvr8p5VnJviXje64WrNAdGN1zq4DhozBPTh7Ax1GcJXMeJ1caTQ57G2dkAvvm3zDBsiMAp7yphuAgeqHyBxHRY](https://explorer.solana.com/tx/64Vvr8p5VnJviXje64WrNAdGN1zq4DhozBPTh7Ax1GcJXMeJ1caTQ57G2dkAvvm3zDBsiMAp7yphuAgeqHyBxHRY?cluster=devnet) |
| test_helper_deployer_pump_buy | ✅ | 514170984 (+29 slot) | [54fEsYBMertREay9Y4SbrPkW9sv2vM9WmSqCbMf3XxHBty1rp9CyiZFMSrnk784yi1fiUrPNTxxQnjmVS9n8JJTB](https://explorer.solana.com/tx/54fEsYBMertREay9Y4SbrPkW9sv2vM9WmSqCbMf3XxHBty1rp9CyiZFMSrnk784yi1fiUrPNTxxQnjmVS9n8JJTB?cluster=devnet) |
| SELL_target_x1.3_3000bps(mark_x1.591) | ✅ | 514171030 (+75 slot) | [4rveRqaCJBJo88FiMwr2QsZHZGe6toS627pmQYUrwLo5JhftHuXRKQy86w4SyUqLLcEqsKENY1VYSZLF2M792xQh](https://explorer.solana.com/tx/4rveRqaCJBJo88FiMwr2QsZHZGe6toS627pmQYUrwLo5JhftHuXRKQy86w4SyUqLLcEqsKENY1VYSZLF2M792xQh?cluster=devnet) |
| SELL_final_real_balance(keep_0bps) | ✅ | 514171177 (+222 slot) | [5azzuNKcEZpAF2KDiw9Z3T6BC7k39nt2LkttwJeuB6mxSDPAxqXPJjDScRuQAiDkCdxpvhW6TvyG4ZosQnT7sA7z](https://explorer.solana.com/tx/5azzuNKcEZpAF2KDiw9Z3T6BC7k39nt2LkttwJeuB6mxSDPAxqXPJjDScRuQAiDkCdxpvhW6TvyG4ZosQnT7sA7z?cluster=devnet) |

#### Run3 时间戳 gate，用**本地时钟**（localnet 上的失败案例）
池子 `8WahT1RTyMJJjmaoPAQnD98SYDye5HJcDAJh4Abw7u6d` · AP ts=1792740646（15:30:46 UTC+8） · 狙击钱包盈亏 −0.004439 SOL · 卖完代币剩 0

| 步骤 | 结果 | 落块 slot（相对开盘） | 完整签名 / 链接 |
|---|---|---|---|
| 建池（deployer） | ✅ | — | [4QHG3mtgBsJzKyVfYwpmVoEQgKmABLKkPdLEPjCh6mC4irVBQhtDw7YAXKQ4KBXVc2oDrpZodiHMAuMKtE1xYQts](https://explorer.solana.com/tx/4QHG3mtgBsJzKyVfYwpmVoEQgKmABLKkPdLEPjCh6mC4irVBQhtDw7YAXKQ4KBXVc2oDrpZodiHMAuMKtE1xYQts?cluster=devnet) |
| prep_create_target_ata | ✅ | — (—) | [5BJeaaEiP9CWJHQUfrigVQ3egg1YCzbYq7oVac5oqm2wanWLsn6QuQG2ABn5XjNrWqRQ9PenRDuwpsMvom3wLqjq](https://explorer.solana.com/tx/5BJeaaEiP9CWJHQUfrigVQ3egg1YCzbYq7oVac5oqm2wanWLsn6QuQG2ABn5XjNrWqRQ9PenRDuwpsMvom3wLqjq?cluster=devnet) |
| preopen_probe_buy(expect_fail) | ✅失败（6003 PoolDisabled，符合预期），发送时点位 1792740602 | 514171341 (-39s) | [3hRc3eYezLSKTCSHeM1Gdmq8WGL5MBzGgog13yCbCo87fmTBPuWcB7cL5ugRLxhS1NmDJrER1UyBhGPsaCnysqxg](https://explorer.solana.com/tx/3hRc3eYezLSKTCSHeM1Gdmq8WGL5MBzGgog13yCbCo87fmTBPuWcB7cL5ugRLxhS1NmDJrER1UyBhGPsaCnysqxg?cluster=devnet) |
| BUY | ✅，发送时点位 1792740646 | 514171506 (+0s) | [5rhTMnpbtdMqYj6UJrDm3MyiW5p683VianSXkvAETM46saj3CNxUqvHAMjP7KXAEuiaBVxrU1wq4hD5siyRwGVkU](https://explorer.solana.com/tx/5rhTMnpbtdMqYj6UJrDm3MyiW5p683VianSXkvAETM46saj3CNxUqvHAMjP7KXAEuiaBVxrU1wq4hD5siyRwGVkU?cluster=devnet) |
| SELL_ladder_1_3000bps | ✅ | 514171532 (+6s) | [3mHoM4233tmrtpfsYfTZCee3MAjHiTkx1ei2skewrqj3RebmWdNaNWSmGJG1QzY593syP3ryGG3gELHv2WtWRWu3](https://explorer.solana.com/tx/3mHoM4233tmrtpfsYfTZCee3MAjHiTkx1ei2skewrqj3RebmWdNaNWSmGJG1QzY593syP3ryGG3gELHv2WtWRWu3?cluster=devnet) |
| SELL_ladder_2_3000bps | ✅ | 514171560 (+13s) | [49nq774vZhDtV79xCPdopkaoRwoC6CHE6x8z4XhGY2BPgQKUUNkFFy8DWeojhTs2hm8t2snvGhFQWScP23EGTPkQ](https://explorer.solana.com/tx/49nq774vZhDtV79xCPdopkaoRwoC6CHE6x8z4XhGY2BPgQKUUNkFFy8DWeojhTs2hm8t2snvGhFQWScP23EGTPkQ?cluster=devnet) |
| SELL_final_real_balance(keep_0bps) | ✅ | 514171593 (+21s) | [5rn6uXu2igpWQRAfk7ysh85wvefDQeD5WybAass6CofEx1yZHeyxjVAdxiiTmZPiUHpZkP4Q4JoYAWwAs3PwNB4B](https://explorer.solana.com/tx/5rn6uXu2igpWQRAfk7ysh85wvefDQeD5WybAass6CofEx1yZHeyxjVAdxiiTmZPiUHpZkP4Q4JoYAWwAs3PwNB4B?cluster=devnet) |

**收尾**
- 关闭狙击钱包的 4 个空 token 账户回收租金：[3PLhAj1js5v6tmAqRkrtbXukWsT4UbJihxiMHFDL586fn8pny2ep62y3wpgFmExtjnYDv5Uorkb9k4EJPS1LT2Zd](https://explorer.solana.com/tx/3PLhAj1js5v6tmAqRkrtbXukWsT4UbJihxiMHFDL586fn8pny2ep62y3wpgFmExtjnYDv5Uorkb9k4EJPS1LT2Zd?cluster=devnet)
- 狙击钱包全部余额 0.998637424 SOL 归集回部署钱包：[SR5769zqJu4pKPewEqxRD4F1nsV9fjNB8iAQT6a6YNNFhub4CXeM5FCdPiWrNGySU16s49mDfJDRDaASZnm8AQP](https://explorer.solana.com/tx/SR5769zqJu4pKPewEqxRD4F1nsV9fjNB8iAQT6a6YNNFhub4CXeM5FCdPiWrNGySU16s49mDfJDRDaASZnm8AQP?cluster=devnet)
- 最终余额：部署钱包 **8.620778664 SOL**，狙击钱包 **0 SOL**。总共用掉约 1.379 SOL，其中约 1.2 SOL 是 4 个测试池里锁着的流动性（LP 仓位还在部署钱包名下，没写撤池脚本，所以没撤），其余是租金、手续费，以及 4 轮狙击的净盈亏（合计约 −0.0013 SOL）。

### 5.7 公开 devnet 和 localnet 的差异（实测）
| 场景 | localnet 买入落点 | devnet 买入落点 | 原因 |
|---|---|---|---|
| Run1 slot, lead 0 | +1 slot | **+13 slot** | gate 发现开盘时已经是 +4 slot（公共 RPC 的 getSlot 每隔一次就吃到 429，web3.js 退避约 1.6s）。另外旧代码在 gate 之后、发送之前还查了一次余额，又多一次 RPC 往返。发出后又过了 9 个 slot（约 2.6s）才落块。 |
| Run2 slot, lead 1 | +0 slot | **+8 slot** | 已修掉“gate 之后查余额”。但 gate 发现时已经是 +6，发出后 2 个 slot 落块。瓶颈主要是**发现**慢，不是传播慢。 |
| Run4 ts，链上 Clock | +0s | **+2s** | 读到链上时间 ≥AP 时已经是 +1s，落块时 block_time 是 +2s。 |
| Run3 ts，本地时钟 | ❌ 6003（发早了） | **✅ +0s 成交** | 实测 box 本地时钟比 devnet 链上 `unix_timestamp` 快约 0.3–1.4s，而 `unix_timestamp` 是整数秒、向下取整。localnet 的验证节点时钟漂移更大，所以那边会失败。devnet 这次能成功，一半是运气，一半是公共 RPC 延迟正好抵消了时钟偏差。**结论不变：按时间戳开盘仍然应该读链上 Clock，或者用链上时间加上实测偏移。** |

其他数据：
- RPC 往返时间：没有被限流时约 40ms，被 429 限流时 1.6–1.8s，两种情况交替出现。
- 确认时间：localnet 约 0.5s，devnet 1.3–2.6s。

### 5.8 devnet 实测得到的新教训
1. **决定落点的是“发现开盘的速度”，不是 lead_slots**。用限流的公共 RPC 轮询 getSlot，发现开盘时就已经晚了 4–7 个 slot。主网要拿到 +0/+1，必须用自己的或付费的 RPC，或者用 gRPC/slot 订阅来推送 slot，按 slot 节奏做预测，并在开盘前就把交易发出去，比如用 Jito bundle 或者在边界上多发几笔。
2. **gate 触发之后不能再有任何 RPC 往返**。已经修正：余额等信息都放到 gate 之前读。
3. **晚进场在这次测试里反而更便宜**：Run1 在 +13 slot 才进场，反狙击费已经衰减，拿到的代币比开盘那一刻的报价多约 7%，亏损也最小。说明在有反狙击费的池子里，“抢第一个 slot”不一定划算，应该按费率曲线算最优的进场 slot。
4. 开盘前探针在公开网络上同样被拒（6003 PoolDisabled），证明 gate 判断正确。4 组的最后一笔都按真实余额卖出，卖完代币都是 0。
5. 公共 RPC 不稳定：官方的 devnet 入口在这台机器上连不上，第三方免费入口也会限流。真盘前至少要准备 2 个付费 RPC，再加独立的发送通道。


## 5.9 开盘模式调研 + Devnet 多模式模拟（2026-10-24 09:04 UTC+8）

详细分类与链接见 `research/launch_modes_cn.md`；原始结果 `logs/modes/RESULTS.json`。

### 5.9.1 模式分类（策略映射）

| 大类 | 代表机制 | 本套策略 | 近期公开例（X/文档） |
|------|----------|----------|---------------------|
| 定时开盘 | Raydium CPMM `openTime`/`startTime`；Meteora DAMM v2/DLMM `activation_point`（slot\|ts）；可选 Alpha Vault + fee scheduler | **提前签名到点单发**（`snipe.js`） | [DAMM fee configs](https://docs.meteora.ag/developer-guides/damm-v2/pool-fee-configs)；[Time scheduler](https://docs.meteora.ag/core-products/damm-v2/fees/time-scheduler)；[Alpha Vault](https://meteoraag.medium.com/alpha-vault-new-anti-bot-mechanism-to-level-the-playing-field-for-genuine-supporters-at-token-9cf800a41b5c) |
| 即时/建池即开 | pump.fun 曲线→PumpSwap；LaunchLab/letsbonk；Meteora DBC；Orca | **监听建池/迁移跟单**（`snipe_follow.js`） | X: [pump→PumpSwap graduate](https://x.com/bitecong/status/2113700460385444235)；[LaunchLab](https://x.com/sbitbot/status/2113633042581356924)；[launchpad 量级](https://x.com/ArtemPadinggton/status/2113574666669523141)；[DBC slot0 被扫](https://x.com/edycutjong/status/2106581843499954490) |
| 手动开盘 | `set_pool_status` / `UpdatePoolStatus`（需协议 operator/admin，非 creator）；或延迟建池 | 远 AP/关盘探针应失败；监听 status 或 create | X: [50%→1% anti-sniper](https://x.com/knowballfun/status/2106113055138480166)；Orca captcha 权限讨论 [帖](https://x.com/mobigorillafam/status/2107852300110737686) |
| 变种 | 白名单、限购、Token-2022 税/hook、分阶段、Alpha Vault 预售 | detect ABORT 危险扩展；高 cliff 算最优进场 | [DLMM Pro anti-sniper](https://x.com/byJamesMarston/status/2107390460469911580) |

### 5.9.2 RPC 可用性（本轮）

| RPC | 结果 |
|-----|------|
| `api.devnet.solana.com` | TLS/连接失败 |
| OnFinality public devnet | **唯一可用**（频繁 429） |
| Helius/Ankr/Alchemy demo/Omniatech | 401 / Unauthorized / 429 / 521 |

### 5.9.3 各模式模拟结果（均为 devnet；主网开关关闭）

钱包：部署 `7U4fi…4Ly4`；狙击 `5bXC5…XXVN`（测完已归集，余额 0）。起始部署约 8.62 SOL → 结束后约 **6.62 SOL**（差额主要锁在各测试池 LP + Raydium 建池费 0.15 SOL + 租金/手续费）。

| 模式 | 成功? | 池 / 要点 | 关键 tx（explorer `?cluster=devnet`） |
|------|-------|-----------|--------------------------------------|
| 手动关盘（远 AP） | ✅ 买失败=预期 | pool `7AVNo…MQRX`；探针+买均 **6003 PoolDisabled**；`set_pool_status` 以 creator 无法构建（需 operator） | 建池 [3rq4mB…](https://explorer.solana.com/tx/3rq4mBdUp2QQdqydRYsbTB4QYDJVBHtEyL9VnrtrLr78NYSH5L9HPtYKfAKZRLNHCszTWFrwBU8u4PcvzEnN2ea5?cluster=devnet)；探针失败 [4vwL6j…](https://explorer.solana.com/tx/4vwL6j2RwD7FBeyG8h9agAjjktCTspGcubpngTB9dKmPiNZVt2qJR5vNkJafExJjPjJeMJ3Umk9SwWYA6wkAXnoi?cluster=devnet)；买失败 [1zLr3N…](https://explorer.solana.com/tx/1zLr3Nwa4SLPqbv4szRPBQVzVzRRAbvYLRR8WEaCFTqgYDaoLzWNPtAPNGopTNXRFotCp2eZmecTJDp2W4jbEf5?cluster=devnet) |
| 手动开盘（近 AP≈45 slot） | ✅ | pool `6DThj…c6H4`；BUY +172 slot（RPC 慢）；卖出清仓 | 建池 [A5MvYb…](https://explorer.solana.com/tx/A5MvYbtg5w7Ks3NgRC2FSbJf6JzDRzz22c8zpJ35Zv5AdpH6MsN2cLGUPP7sfX42DuyuRJV41FbwHQqy7baKZko?cluster=devnet)；BUY [M32SSE…](https://explorer.solana.com/tx/M32SSEAvWw8S6cSQBRXLjNbUDkV4SYqWRZbAYPHuNsNBj8oHYTZbbSo5ZyW2FmN2JUEgLBwJKCnVUNTA9E2Y97B?cluster=devnet) |
| DAMM slot 短测 | ✅ | pool `AJhH6…5M5g`；BUY **+56** slot | 建池 [5iYRen…](https://explorer.solana.com/tx/5iYRenkFnar6HjcTMCGNCL87G4hK8baVGdiz9vLsaxR8N5pGJ3LBpzVsy3tmwmWgq1Q3z9uD1HyxSoaaG14GVKN7?cluster=devnet)；BUY [d8PwrA…](https://explorer.solana.com/tx/d8PwrAEGQZaMdQr1VCpxyhRgkc6n1xH6vZ9UwJJ4seBwHtxopVK6xyYCB6QURVJDcCGQSh9jBSygcqoCF4wo6ki?cluster=devnet) |
| DAMM timestamp 短测 | ✅ | pool `EBXSv…2gum`；探针失败后买入成功 | 建池 [4dZumh…](https://explorer.solana.com/tx/4dZumhoLcWcs5JG7kQSp9zNJHaATDyCQsv1Xwp4HgTNW5DKd5RkTAb8i2SLeQ45J4kSuBopk4x8Q1STZiVQPN7Mg?cluster=devnet)；探针 [edD3DG…](https://explorer.solana.com/tx/edD3DGznbHx2PYgKFgnbrMqYi9MRcBkKLpZZL5Jq83ehTzEWS7mCcVLWado2NhWsbUkn5vAm1Lm6cWTGDtQxuLA?cluster=devnet)；BUY [3P58hP…](https://explorer.solana.com/tx/3P58hP5tN8GMnquBjaH21CEPhVuWsAR4L2vnQHJ1LXEcEFP5LeDw9BZA8xbZiPdwBTwXjfRGbkYDFNn7rJb4ThEt?cluster=devnet) |
| 高反狙击费 早买 | ✅ | cliff 70%→1%/50slot；报价开盘到手 **13.27T**（fee 0.014 SOL/0.02） | BUY +70 [4xymwh…](https://explorer.solana.com/tx/4xymwh2XWASiVuj4naHogaTVpFFa2FzvWhD8QJfjxZ4K6LHp8ekyf495Q4G5k7MosvufMj6QU9mcCYFXqSzLexZu?cluster=devnet) |
| 高反狙击费 晚买（+45） | ✅ | 延迟后重建签名；报价 **41.28T**（fee≈0.0002 SOL）；实成交与早买同为 41.28T（二者都因 429 落到费已衰减区间） | BUY +85 [38AbxT…](https://explorer.solana.com/tx/38AbxTo6wUNmkkMAJRbC4YGMpjEWDjbZNEbo53Y1AApdbDdyrhuiKDUPfiVqQWHsdiAZohatxzVHP1gWe5ALAZNB?cluster=devnet) |
| 即时跟单 | ✅ | 发现→买 **43 slot**；开盘后约 +22 | 建池 [57CUUE…](https://explorer.solana.com/tx/57CUUE8CZXp464JiaUn4A2rGUPpRaz51tY5n7dB6ueePiJ5iuPWgLbHPbAnnoCQQttCMPL62cMnhDyanL7CXRkio?cluster=devnet)；FOLLOW_BUY [31kBHM…](https://explorer.solana.com/tx/31kBHM9u21DmNHPAr1jhY5bJLo5L9pFfBDaM6qYrHfFjaJH39kYrW6VV6j9mie7W2VshAughSKVN4rnJQ7EUYRu?cluster=devnet) |
| Raydium CPMM `openTime` | ✅ | devnet 程序 `DRaycpLY…` 可执行；建池费 0.15 SOL；开盘前 Custom **6000**；开盘后 +8s 买成并卖清 | 建池 [4ioAj6…](https://explorer.solana.com/tx/4ioAj6HBsCBQUeLQuQxzNva5nz1Tk5BKDsTDyfLgMmNkvrJX7wYXp4igP2MgbqYMW5yVzrF6cQMNsFSV6Y8Pe7e8?cluster=devnet)；预开失败 [4Png17…](https://explorer.solana.com/tx/4Png17PHgR4JWZRPuSoxtfXG2kRxLa8YFLieH7NiU2zTxcCTiD2J7MWfpR1gWtvfeyUBCrn86sNR5R516DeaRgci?cluster=devnet)；BUY [2prAh1…](https://explorer.solana.com/tx/2prAh1wkP31UNU8n46Su7k5hD3wrfTHmwkNASdq6e3JJ9YVEyn4rcP5cB8AJKdvsXq8fWkLBDucruSBJobM6rFEj?cluster=devnet) |

**反狙击费结论（报价层）**：同 0.02 SOL，开盘瞬间理论到手约 13.3T，延迟衰减后约 41.3T（约 **3.1×**）——验证“晚进场不一定更差”。本轮实际两笔都落到衰减后区间，故成交量相同；要对比真实首 slot 成交需低延迟 RPC。

**手动开盘诚实说明**：公开 devnet 上 creator **不能**调用 Meteora `set_pool_status`（需 operator）。本轮用「远 activation=关 / 近 activation=开」模拟门禁效果；真·管理员翻 status 需协议方密钥。

### 5.9.4 本轮 SOL

- 部署起始 ≈8.6208 → 结束 ≈**6.6217**（狙击归集后）；狙击 2.0 注入后测完扫回 0。
- 消耗约 **2.0 SOL** 量级：多池各 ~0.22 SOL LP + Raydium 0.12 LP + 0.15 建池费 + mint/ATA 租金 + 交易费；LP 仍锁在池内，可后续撤池回收。




## 5.10 手法补全 + 开源吸取（2026-10-24 Asia/Shanghai UTC+8）

目标：把开盘狙击手法补到可配置可用，并吸收许可证清晰的开源优点（只读审查，不跑未审查代码处理私钥）。

### 5.10.1 新增能力与 diget 验证

| 能力 | diget 验证 | 关键 tx / 说明 |
|------|------------|----------------|
| Meteora DLMM `activation_point` 定时买+卖清 | ✅ | 建池 [2XVKnF…](https://explorer.solana.com/tx/2XVKnFbcdNBfUH7oJdtvQqS4rWFC2UXRmfY7mG4VqnrH7x7v6JtZHEcfsUUzVDtCDhe3CbfnycH7FApg7D5xDe5a?cluster=devnet)；BUY [4h5Dzy…](https://explorer.solana.com/tx/4h5DzyhFyp81DRVA53ncKST2bD7pKvAa1XrVbmCLgg76ZEnE3biJkDNocuJfovVLr8osGRWohYysyc4Qx93BKLUL?cluster=devnet)；SELL ladder [3dAoGR…](https://explorer.solana.com/tx/3dAoGRdn8PAwL7pV49nVb821izFdS5LRbXuHfit4qgngE1gzw6tuEF22x3BRaNxsUqozGyDkSTjsMx8DWn2DtHp5?cluster=devnet)；SELL final [3r7tNG…](https://explorer.solana.com/tx/3r7tNGn6HoNtMRZWUvezqVKQsXh5NEkUD9TGjRfyB1zeqTqshMKP1ZHmbXT5nRpouaD2Jssf4ZEj5M6aPHvaoJwG?cluster=devnet)。注意：`activeId` 勿过负（需 bitmap extension）；勿与 SDK 重复叠 ComputeBudget |
| Meteora DBC 建池即开跟单 | ✅ | 建池 [5jYcZz…](https://explorer.solana.com/tx/5jYcZziNp4MjR7cWZQiW3dtC5tWU7Gce4JarX3uKQzzyzPf2biZV9hJPrVXB3MbKXp8N4z9zkbQctJg6azj88NaB?cluster=devnet)；BUY [XiCd1n…](https://explorer.solana.com/tx/XiCd1n3AQzr8JFfsSR7Jz1Z5XCSV9e7Kes5ydhGdJiRwh2ZvbFDrXokEHSrUaoRyp7EV8XQppWyRowFXFiPxQ8S?cluster=devnet)；SELL [4XHa8q…](https://explorer.solana.com/tx/4XHa8qzbecnZPhtUN8PNzeFJWX5dbbYzCiPhbPJoYX7oSSxmTj4EzJjqdgjairXLNGX1kspqT3oMBq9i7YjywQym?cluster=devnet) / [3YiWwf…](https://explorer.solana.com/tx/3YiWwfJefkGzfMwSY9dDnE7U657UYDRR6eAuuXKhUoV2KttJhAXutXJCydYYoveJqkRKKkoe9UfeG6an4axQUSHR?cluster=devnet)。`swapQuote2` 需传 `getPool()` 包装 `{poolState}` |
| pump.fun / PumpSwap | 🟡 部分 | 程序 diget 可执行；`--listen` 捕获 15 条近期签名；IDL 构建 buy + `simulateTransaction`（无 bonding-curve 时预期失败）。**限制**：无完整 diget 建币/元数据流水线，未做链上买入 |
| exit A/B/D/F 统一 | ✅ 配置+代码 | `src/exit.js`；`snipe.js` / DLMM / DBC 已接入 |
| 反狙击费最优进场估算 | ✅ 本地 | `src/fee_optimal.js`（DAMM quote 曲线 vs delay） |
| 多入口广播 + Jito tip 字段 | ✅ 接口 | `src/tx_send.js`；主网 tip 默认 `enabled:false` |
| Token-2022 detect_mint | ✅ | 样例 mint `818TRpay…` → verdict **OK** |
| RPC token-bucket + 日志脱敏 | ✅ | `rpc_retry.js` / `lib.redactSecrets` |

原始 JSON：`logs/modes2/RESULTS.json`。

### 5.10.2 开源吸取（详见 `research/github_review.md`）

- **chainstacklabs/pumpfun-bonkfun-bot (Apache-2.0)**：token-bucket、secret redaction、IDL/cookbook 拆分 → 已落入限流与脱敏；pump 只读 IDL。
- **0xfnzero/solana-streamer (MIT)**：订阅/解析分层 → 架构预留 + diget 无 WS 时用签名轮询降级。
- **outsmartchad/onchain-dex-cli (MIT)**：landing provider 拆分 → `broadcastMulti` / Jito 字段。
- **不抄**：无 LICENSE 营销仓、Telegram 外发密钥、`eval`/混淆、git+http 依赖、硬编码 swap 方向、默认开主网 Jito。

### 5.10.3 脚本入口

```
node src/create_dlmm.js configs/modes/dlmm-timed.json
node src/snipe_dlmm.js  configs/modes/dlmm-timed.json
node src/create_dbc.js  configs/modes/dbc-instant.json
node src/snipe_dbc.js   configs/modes/dbc-instant.json
node src/snipe_pump.js  configs/modes/pump-follow.json [--listen 30]
node src/detect_mint.js <mint>
node src/fee_optimal.js configs/modes/fee-early.json
```

主网三重开关仍关闭；私钥不打印。


## 六、阻塞点
- ~~devnet 领水~~：已解决（用户手动领了 10 SOL），公开 devnet 交易见 5.6。
- `api.devnet.solana.com` 在这台 box 上 TLS 失败，只能用限流的 OnFinality 公共 RPC。这直接导致 slot gate 落点偏晚（+8 到 +13）。
- 主网实盘需要的东西都还没有：staked RPC 或 SWQoS、Yellowstone gRPC、Jito 或其他发送通道的账号，都要付费。
- 适配器：DAMM v2 / Raydium CPMM / follow / DLMM / DBC diget 已验证；pump.fun 为监听→构建→模拟（限制见 §5.10）。
- 下载核查 `web3-fb` tarball 的操作被安全审核拦截，没有重试，所以该依赖按"未核实、高风险"处理。

## 七、下一步建议
1. ✅ devnet 多模式模拟见 §5.9。下一步：付费/不限流 devnet RPC 或 gRPC，把 slot gate 压到 +0/+1；补撤池脚本回收各测试池锁定的 SOL。
2. 确定目标项目：开盘方式、DEX、报价币、开盘时间、Alpha Vault / 反狙击费（对照 §5.9 分类表）。
3. 确定预算：单笔仓位、反狙击费上限、tip / priority fee；高 cliff 池用费率曲线算最优进场 slot。
4. 采购基础设施：staked RPC / SWQoS + Yellowstone gRPC + Jito；shred 用 DoubleZero Edge。
5. ✅ DLMM / DBC / pump 骨架已补（§5.10）。下一步：pump 真实 diget 建币或主网小额；gRPC 订阅落地；撤池回收测试 LP。
6. 上主网前小金额演练，再开三重开关。
