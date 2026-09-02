# Monad Volume Bot

CLI toolkit for cycling Nad.fun trades across many wallets on [Monad](https://monad.xyz) (chain ID `143`).

Generate accounts, fund them from a master wallet, track balances, and refund MON. Interactive menu. TypeScript. ethers v6.

> **Limited release.** Parallel buy and sell implementations (`executeBuys`, `executeSells`) are **not included** in this repository. Stubs remain so the volume loop still compiles. For the full version, [contact the developer](#contact).

[![License: MIT](https://img.shields.io/badge/license-MIT-0ea5e9.svg)](LICENSE)
[![TypeScript](https://img.shields.io/badge/TypeScript-5.3-3178C6.svg)](https://www.typescriptlang.org/)
[![Node.js](https://img.shields.io/badge/Node.js-20+-339933.svg)](https://nodejs.org/)
[![ethers](https://img.shields.io/badge/ethers-v6-627EEA.svg)](https://docs.ethers.org/v6/)
[![Chain](https://img.shields.io/badge/Monad-143-836EF9.svg)](https://rpc3.monad.xyz)

---

## Status

| Module | Location | State |
|---|---|---|
| Wallet generation | `WalletService.generateAccounts` | Included |
| Balances | `WalletService.viewBalances` | Included |
| Fund from master | `WalletService.fundAccounts` | Included |
| Token discovery | `WalletService.getTokensInWallet` | Included |
| Single-wallet sell helper | `WalletService.sellTokenFromWallet` | Included |
| Refund (sell holdings + sweep MON) | `WalletService.refundToMaster` | Included |
| Volume loop shell | `WalletService.runVolumeBot` | Included (calls stubs) |
| **Batch buy** | `WalletService.executeBuys` | **Removed — comments only** |
| **Batch sell** | `WalletService.executeSells` | **Removed — comments only** |
| Token approve (unused) | `WalletService.approveTokens` | **Stub — never implemented** |

Selecting **Run Volume Bot** still starts the infinite cycle. Each buy/sell phase logs a warning and returns an empty result. No Nad.fun `buy()` / `sell()` is sent from those methods.

**Need the full volume bot?** Contact the developer. Do not expect this checkout to print wash volume on a token.

---

## What this checkout does

1. Create up to 100 wallets (`ethers.Wallet.createRandom`).
2. Push the same MON amount to every wallet (`BUY_AMOUNT` + `gasFee` buffer).
3. Read live balances from `https://rpc3.monad.xyz`.
4. On refund: scan recent `Transfer` logs, sell detected tokens, then sweep leftover MON to the master wallet.
5. Volume loop: prompt for a token, then `buy → delay → sell → repeat` — **buy/sell bodies are not shipped**.

Wallets persist to `src/wallets/wallets.json`. Every overwrite writes a timestamped copy under `src/wallets/backups/`.

---

## Requirements

- Node.js 20+
- npm or yarn
- A Monad master wallet with MON for gas and distribution

---

## Install

```bash
cd monad-volume-bot
npm install
```

Create a `.env` in the project root:

```env
MASTER_PRIVATE_KEY=your_private_key_here
BUY_AMOUNT=1
```

`src/config.ts` also exposes:

| Option | Meaning | Default |
|---|---|---|
| `rpcUrl` | Monad RPC | `https://rpc3.monad.xyz` |
| `chainId` | Monad | `143` |
| `buyAmount` | MON spent per wallet per buy | `BUY_AMOUNT` or `1` |
| `gasFee` | Extra MON per wallet when funding | `0.01` |
| `delaySeconds` | Pause between buy phase and sell phase | `3` |
| `explorerUrl` | Tx links | `https://monad.socialscan.io/tx` |

Never commit a live private key. Prefer `.env` over hardcoding.

---

## Run

```bash
npm run dev      # ts-node, src/index.ts
npm start        # tsc → node dist/index.js
npm run build    # compile only
```

On launch the CLI shows the master address and MON balance.

---

## Menu

```
Generate Accounts
View Balances
Fund Accounts
Run Volume Bot              ← loop runs; buy/sell execution is a placeholder
Refund to Master
Exit Application
```

Typical included flow: generate → fund → (volume cycles in the full build) → refund.

---

## Project layout

```
src/
├── index.ts                      CLI shell, menu, dispatch
├── config.ts                     RPC, env, buy amount, delay
├── types.ts                      WalletInfo / WalletsData
├── services/
│   └── wallet.service.ts         Engine (lifecycle + volume loop)
└── utils/
    ├── storage.ts                wallets.json + backups
    └── logger.ts                 CLI formatting
```

| File | Role |
|---|---|
| `src/index.ts` | `MonadVolumeBot` — welcome, menu, `handleAction` |
| `src/config.ts` | dotenv + `rpcUrl`, `buyAmount`, `delaySeconds` |
| `src/types.ts` | Wallet records (no per-wallet buy amount) |
| `src/services/wallet.service.ts` | All chain I/O |
| `src/utils/storage.ts` | Save / load / backup |
| `src/utils/logger.ts` | Colored log helpers |

---

## Missing code (this release)

The following methods in `src/services/wallet.service.ts` are **intentionally incomplete**.

### `executeBuys`

Comments describe the intended path. No quotes, no `buy()` calls, no parallel send. Returns `[]`.

Intended behavior (full version):

- For each generated wallet, spend `buyAmount` MON
- Quote via Nad.fun Lens (`getAmountOut`, buy)
- Apply slippage on `amountOutMin`
- Call `buy()` on the router Lens returns
- Run all wallets in parallel and return per-wallet success / fail + tx hash

### `executeSells`

Comments describe the intended path. No balances, no approve, no `sell()` calls. Returns `[]`.

Intended behavior (full version):

- Read each wallet’s ERC-20 balance for the target token
- Quote via Nad.fun Lens (`getAmountOut`, sell)
- Apply slippage on `amountOutMin`
- Approve the router, then `sell()` the full balance
- Run all wallets in parallel and return per-wallet success / fail + tx hash

### Volume loop

`runVolumeBot` still owns the outer `while (true)`: buy phase → `delaySeconds` → sell phase → 2s pause → next cycle. Without `executeBuys` / `executeSells`, that loop does not trade.

These stubs are not a substitute for the production bot. **Contact the developer for the full version.**

---

## Stack

| Piece | Detail |
|---|---|
| Runtime | Node.js 20+ |
| Language | TypeScript 5.3 (strict, CommonJS, ES2020) |
| Chain | Monad (`143`) via `https://rpc3.monad.xyz` |
| Library | ethers `^6.9.0` |
| CLI | inquirer, chalk, ora, boxen, cli-table3 |
| Config | dotenv (`MASTER_PRIVATE_KEY`, `BUY_AMOUNT`) |
| Venue | Nad.fun Lens + bonding-curve / DEX routers |

---

## Security

- `src/wallets/wallets.json` holds **plaintext private keys**. Restrict filesystem access. Do not push it.
- Backups under `src/wallets/backups/` contain the same secrets.
- Keep `MASTER_PRIVATE_KEY` in `.env`, never in git.
- This tool signs real transactions on fund, refund, and (in the full build) every volume cycle. Confirm amounts before you start.

---

## Contact

**Author:** Soulcrancerdev

This repository is a **limited / demo build**. `executeBuys` and `executeSells` are not shipped.

If you need the **full version** (infinite Nad.fun buy/sell cycles with the production execution path):

**Contact the developer.** Request the complete volume bot — do not attempt to reconstruct those modules from the comment stubs.

---

## License

MIT
