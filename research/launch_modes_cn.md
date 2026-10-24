# Solana 常见开盘模式调研（公开资料 + X）

> 时间：2026-10-24（Asia/Shanghai）。仅公开读；不含未审计第三方可执行代码。

## 1. 定时开盘（presign-at-gate）

| 模式 | 机制 | 典型反狙击 | 狙击策略 | 近期案例/出处 |
|------|------|-----------|----------|--------------|
| Raydium CPMM / AMM v4 `open_time` / `startTime` | 池状态里写未来 unix 时间，未到不可 swap | 可叠加高初始费；CPMM createPool 参数 `startTime` | **提前签名到点单发**；开盘前探针应失败 | Raydium SDK v2 `CreateCpmmPoolParam.startTime`；文档/API `api-v3*.raydium.io` |
| Meteora DAMM v2 `activation_point` | slot **或** timestamp；`activation_type` 区分 | Fee scheduler（时间指数衰减）、可选 Alpha Vault | **提前签名到点单发**；读链上 Clock 防本地时钟漂移 | [Meteora DAMM](https://docs.meteora.ag/)；本仓已验证 slot/ts gate |
| Meteora DLMM `activation_point` | 同族 slot/ts | bin 数组+动态费 | 同 DAMM；另需处理 bin 数组账户 | docs.meteora.ag DLMM |
| Alpha Vault 预售→开盘 | 先募资进 vault，到点再注入池 | 募资期不可在池买；开盘瞬间流动性到账 | 监听 vault→pool 迁移/激活，或对 activation 预签 | Meteora Alpha Vault docs；X 上项目方常用“反狙击金库”叙事 |

## 2. 即时 / 建池即开（listen-create-and-follow）

| 模式 | 机制 | 狙击策略 | 案例/出处 |
|------|------|----------|-----------|
| pump.fun bonding curve | 曲线内买卖，满额 migrate → PumpSwap | **监听 create/迁移事件跟单**；无法可靠“到点预签” | [pump.fun](https://pump.fun)；程序 `6EF8rrect…`；X 大量 migrate 讨论 |
| PumpSwap 迁移盘 | bonding 毕业后进 AMM | 监听 migrate tx，同 slot/下一 slot 买 | X: PumpSwap graduate 相关帖；程序 `pAMMBay…` |
| Raydium LaunchLab / letsbonk | 发射台完成后再挂 CPMM/池 | 监听 LaunchLab 完成事件 | Raydium LaunchLab 文档；devnet 程序 `DRay6fNd…` |
| Meteora DBC（Believe 等） | 动态 bonding / 配置池，常近似即开 | 监听 DBC create；注意反狙击费曲线 | docs + X Believe/DBC；程序 `dbcij3…` |
| Orca Whirlpool | 建池后通常立即可交易（无统一 open_time） | 跟单监听；或项目方延迟加流动性≈手动 | Orca 文档 |

## 3. 手动开盘（权限位 / 运营开关）

| 模式 | 机制 | 狙击策略 | 案例 |
|------|------|----------|------|
| 管理员 `set_pool_status` / `UpdatePoolStatus` | 池存在但 status=disabled，运营改 status 才可买 | 预构建交易，**监听 status 变更**后立刻发；开盘前买应失败 | TRUMP 类叙事；Meteora `set_pool_status` **需 operator**（非 creator）；Raydium `UpdatePoolStatus` **需 program admin** |
| 延迟建池/延迟加流动性 | mint 已发、池未建或 LP=0 | 监听 create_pool / add_liquidity | 常见“先喊单后上池” |
| 冻权限/暂停扩展 | Token-2022 pausable / freeze | detect 应 ABORT；不可当作可狙击盘 | 安全检测清单 |

## 4. 其他变种（近期 X/文档常见）

| 变种 | 要点 | 策略影响 |
|------|------|----------|
| 白名单 / allowlist | 仅名单可买前 N 秒 | 无资格则放弃；有资格走预签 |
| 限购 / per-wallet cap | 单地址上限 | 多钱包拆单（合规前提下）或改策略 |
| Token-2022 税 / transfer hook / permanent delegate | 买到无法卖或额外扣税 | **detect ABORT** |
| 分阶段解锁 / 阶梯费 | fee scheduler、rate limiter | 对比“开盘即买 vs 晚几 slot”——高 cliff 时晚进可能到手更多 |
| 预售 → 开盘（Alpha Vault / IDO） | 两阶段 | 阶段1参与募资；阶段2按激活点狙击 |

## 5. 策略映射（本套代码）

- **提前签名到点单发**：`src/snipe.js` + Meteora DAMM activation；适用定时开盘、已知 AP。
- **监听建池跟单**：`src/snipe_follow.js`；适用即时开盘 / 手动“一键建池即开”。
- **探针**：`preopen_probe` 开盘前链上买应失败，证明 gate。
- **主网默认关闭**；私钥仅 `keys/` mode 600。

## 6. 参考链接（公开）

- Meteora docs: https://docs.meteora.ag/
- Raydium docs / API: https://docs.raydium.io/ ; https://api-v3-devnet.raydium.io/main/cpmm-config
- Pump.fun: https://pump.fun/
- Solana Explorer diget: https://explorer.solana.com/?cluster=devnet
- X 公开检索（本轮用 x 命名空间）：关键词 `Meteora activation_point`、`Alpha Vault`、`PumpSwap migrate`、`Raydium LaunchLab`、`anti sniper fee`、`letsbonk`（具体帖子 URL 写入 REPORT 实测章节）


## 7. 本轮 X 公开帖（抽样）

- pump.fun→PumpSwap: https://x.com/bitecong/status/2113700460385444235
- Raydium LaunchLab live: https://x.com/sbitbot/status/2113633042581356924
- Launchpad 量级（PumpSwap/curve/DBC/LaunchLab）: https://x.com/ArtemPadinggton/status/2113574666669523141
- Meteora DBC slot0 被扫: https://x.com/edycutjong/status/2106581843499954490
- Anti-sniper 50%→1%: https://x.com/knowballfun/status/2106113055138480166
- DLMM Pro anti-sniper: https://x.com/byJamesMarston/status/2107390460469911580
- Docs: https://docs.meteora.ag/core-products/damm-v2/fees/time-scheduler
