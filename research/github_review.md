# GitHub 开源狙击器/监听器 静态审查（2026-10-23 UTC+8）

方法：仓库只做浅克隆，放在 `research/gh/`，只做静态 grep，**没有执行任何第三方代码，也没有安装依赖**。
- grep 的内容：外发端点（http/fetch/axios/telegram/webhook）、私钥读取方式、eval/base64 混淆、从 git 或 tarball 安装的依赖。
- npm 依赖只查 registry 的元数据，或者下载 tarball 只看不运行。
- 表中 "命中文件数" 是按关键词 grep 出来的粗略数字，代表"代码里提到过"，不代表功能完整可用。
- 推送时间已换算成 UTC+8。

| 仓库 | ★ | 最近推送(UTC+8) | 语言 | License | Jito | gRPC/Geyser | Shred | Meteora activation_point | CPMM/AMM open_time | 风险结论 |
|---|---|---|---|---|---|---|---|---|---|---|
| **MortarStallionMarina/Solana-PumpFun-Sniper-V2** | 65 | 2026-07-08 | Python | 无 | 假的（宣传噱头） | – | – | – | – | **恶意软件**。界面显示"Jito MEV bypass"，暗地里用 base64 混淆的代码偷浏览器 MetaMask/Phantom 扩展数据、Telegram tdata、桌面/文档/下载目录里的 .env/config.json/secret.json，打包后通过 Telegram Bot sendDocument 发出去。**严禁运行** |
| mortdeus/solana-copy-sniper-mev-trading-bot | 4502 | 2026-09-21 | Rust/Node | BSD-3 | 有 | Yellowstone | 有（Rust 版） | 无 | 少量 | **高风险**。Node 版依赖 `web3-fb`：npm 上不存在，lockfile 指向第三方 GitHub release 的 tgz（crypto2001813/solana-sdk），名字也和 web3 无关，是典型投毒路径。另外还有 `pump-swap-core-v1`（个人发布的"非官方 PumpSwap SDK"）、占位包 `fs`/`require`、`@degenfrends/solana-rugchecker`；私钥从环境变量 PRIVATE_KEY 读取。星数像刷出来的。`web3-fb` 的 tgz 因为安全审核被拦截，没有下载核查，所以按"未核实、高风险"处理 |
| coffellas-cto/Solana-Copy-Trading-Bot | 412 | 2026-04-17 | Rust | 无 | 有 | README 里说有 | README 里说有 | 无 | 无 | 中风险。README 以营销和联系 Telegram 为主；git 依赖里有 raydium 官方仓库和 wisarmy 的 jito-json-rpc-client；没有许可证 |
| bigmacman1129/solana-sniper-trading-mev-bot | 191 | 2026-08-25 | TS | MS-PL | 有 | 无 | 无 | 无 | Raydium v4 | 中风险。telegram 只出现在元数据过滤里；Jupiter 那段 base64 是正常的交易解码。功能比较老（AMM v4） |
| m8s-lab/solana-sniping-bot | 95 | 2025-06-13 | TS | 无 | 有 | 少量 | 无 | 无 | 有 | 中风险。已经一年多没更新；没有许可证 |
| Solana-Vibe-Station/grpc-sniper | 69 | 2024-09-10 | TS | 无 | 少量 | 有 | 无 | 无 | 无 | 太旧，只适合当 gRPC 订阅示例看 |
| **chainstacklabs/pumpfun-bonkfun-bot** | 1035 | 2026-10-20 | Python | Apache-2.0 | 少量 | Geyser 等多种监听 | 有 | 无 | 无 | **低风险，推荐参考**。厂商官方维护；覆盖 pump.fun 和 letsbonk(LaunchLab)，Token-2022 有处理；日志模块主动脱敏私钥；外发只连 RPC，没有可疑端点 |
| **0xfnzero/solana-streamer**（含 sol-parser-sdk） | 187 | 2026-10-22 | Rust | MIT | 有 | Yellowstone | 有 | 能解析 | 能解析 CPMM/AMM v4 | **低风险，推荐参考监听和解析部分**。它只是监听库，不碰私钥；能解析 pump/PumpSwap/LaunchLab/CPMM/AMM v4/DAMM v2 等事件 |
| **outsmartchad/onchain-dex-cli** | 581 | 2026-03-07 | TS | MIT | 有 | 有 | 无 | 有少量 | – | **低到中风险，推荐参考多通道发送部分**。支持 18 个 DEX，含 DAMM v2/DLMM/DBC/Orca；发送通道有 Jito/0slot/Nozomi/bloXroute/Node1 等；私钥只从本地 .env 或 600 权限文件读。要注意：社区包 `pumpdotfun-sdk` 需要锁定版本并审查；发送提供商的端点要自己核对 |
| Shyft-to/solana-defi | 354 | 2026-10-07 | Rust/TS | 无 | – | 有 | – | 有部分示例 | 有部分示例 | 只是 gRPC 解析示例集合，不涉及私钥；没有许可证，不能直接拿去商用 |
| rpcpool/yellowstone-grpc | 1015 | 2026-10-22 | Rust | AGPL-3.0（以 API 返回为准） | – | 官方 | – | – | – | 官方 Geyser 插件和客户端 |

## 推荐参考（只读代码、学写法，不拿真实私钥运行）
1. **chainstacklabs/pumpfun-bonkfun-bot**：参考即时开盘类（pump.fun、letsbonk）的监听 → 买入 → 卖出流程。
2. **0xfnzero/solana-streamer**：参考 gRPC/Shred 事件解析，用来监听建池、迁移、开关池。
3. **outsmartchad/onchain-dex-cli**：参考多 DEX 适配器，以及多通道（SWQoS/Jito/0slot）同时发送。

**结论**：没有一个开源项目实现了"读 Meteora activation_point 或 CPMM open_time → 提前签名 → 按链上时钟到点单发"这套定时开盘流程。这部分用我们自己的 `src/snipe.js`。

## 安全规则
- 任何第三方代码都只在没有真实密钥的沙箱里用测试钱包运行；先审查依赖：不能有 git/tarball 来源的包，不能有 postinstall 脚本，名字要和官方包核对。
- 宣传"MEV bypass"、"0 block"并让你私信 Telegram 的仓库，按诈骗或恶意软件处理。

---

## 手法补全阶段吸收清单（2026-10-24 Asia/Shanghai UTC+8）

方法不变：浅克隆静态读，**不执行第三方代码、不把私钥交给未审查运行时**。

### 对照仓库（许可证清晰）

| 仓库 | 许可 | 可借鉴 | 已落到本仓库 |
|------|------|--------|--------------|
| chainstacklabs/pumpfun-bonkfun-bot | Apache-2.0 | Token-bucket 限流；日志 secret redaction；买卖/监听 cookbook 与 IDL 拆分；dry-run 思路 | `src/rpc_retry.js` makeBucket；`lib.redactSecrets/safeLog`；`snipe_pump.js` 只读 IDL discriminator |
| 0xfnzero/solana-streamer | MIT | gRPC/订阅与事件解析分层；DEX 无关的流式入口 | 架构预留：`landing_endpoints` + 后续 gRPC adapter 位；pump 用 getSignatures 轮询作无 WS 降级 |
| outsmartchad/onchain-dex-cli | MIT | Landing provider 接口（RPC/Jito 拆分）；交易构建与发送分离；重试退避 | `src/tx_send.js` broadcastMulti + maybeAddJitoTip（主网默认关） |

### 额外扫描（不作为依赖）

| 仓库 | 许可 | 结论 |
|------|------|------|
| Solana-Vibe-Station/grpc-sniper | **无 LICENSE**（已 archive） | 仅作 gRPC 订阅 PoC 结构参考，**不抄代码、不装依赖** |
| Shyft-to/solana-defi | **无 LICENSE** | 示例向；含 Jito bundle 样例路径，只读结构，不并入运行时 |
| crypdev007/pumpfun-sniper 等近期高星无许可营销仓 | 无清晰开源许可 / 营销文案重 | **不吸收**；避免后门与密钥窃取模式 |

### 明确不抄的危险模式

1. Telegram / webhook 外发私钥或助记词  
2. `eval` / 混淆 base64 载荷 / 从 git+http 或 tarball 拉未钉版本依赖  
3. 硬编码 `swapYtoX=true`（池子方向反了会买错边）— DLMM 路径改为按 `tokenX/target` 动态判断  
4. 把第三方 bot 当 npm 依赖直接 `require` 跑真实密钥  
5. 主网 Jito tip / block engine 默认开启  

### 本仓库落地文件

- `src/exit.js` — exit A/B/D/F 配置化统一  
- `src/fee_optimal.js` — DAMM 反狙击费 vs 延迟进场估算  
- `src/detect_mint.js` — Token-2022/权限 → OK/REDUCE/ABORT  
- `src/tx_send.js` — 多入口广播 + Jito tip 字段预留  
- `src/rpc_retry.js` — 429 退避 + token-bucket  
- `src/create_dlmm.js` / `src/snipe_dlmm.js`  
- `src/create_dbc.js` / `src/snipe_dbc.js`  
- `src/snipe_pump.js` — 监听→构建→模拟（devnet 限制见脚本头）

