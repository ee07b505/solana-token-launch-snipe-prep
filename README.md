# Solana Token Launch Snipe Prep

**Chain: Solana only.** For Base/BSC see the sibling Base/BSC sniper skill.

This repo publishes the reusable skill and research notes used by Solana区块链专家 for timed and follow-on launch sniping on Solana (Raydium / Meteora / pump.fun / DBC).

## Contents
- `SKILL.md` — skill recipe (load as `solana-token-launch-snipe-prep`)
- `docs/SOL_SNIPE_PREP_CN.md` — full Chinese checklist
- `docs/REPORT_CN.md` — research + Devnet practice report
- `research/` — launch modes survey, GitHub OSS review

## Hard rules
- All params in config; never hardcode.
- Quote mint read from pool (do not assume SOL).
- Mainnet send off by default (triple switch).
- No private keys in this repo.

## Modes → strategy
| Mode | Strategy |
|------|----------|
| Timed `open_time` / `activation_point` | Presign → chain clock/slot → single send |
| Manual open | Watch status flip → follow next tx |
| Instant (pump / LaunchLab / DBC) | Watch create → follow (cannot beat creator bundle) |
| High anti-snipe fee | Quote optimal entry; +0 slot not always best |

## License
Docs and skill: MIT for reuse of the checklist text. Do not run unreviewed third-party sniper bots with real keys.
