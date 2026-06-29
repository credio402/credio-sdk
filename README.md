<div align="center">

<img src="assets/logo.png" alt="Credio" width="96" height="96" />

# Credio SDK

**Credit liquidity for the agent economy.**

[![protocol: x402](https://img.shields.io/badge/protocol-x402-7c3aed)](https://x402.org)
[![chain: Base](https://img.shields.io/badge/chain-Base-0052FF)](https://base.org)
[![license: MIT](https://img.shields.io/badge/license-MIT-3fb950)](./LICENSE)
[![npm](https://img.shields.io/npm/v/credio-sdk?color=FF2D8F)](https://www.npmjs.com/package/credio-sdk)

[credio.fun](https://credio.fun) · [Docs](https://credio.fun/docs) · [Developers](https://credio.fun/developers)

</div>

---

AI agents pay per call via [x402](https://x402.org). When an agent's wallet runs dry, it stops dead. **Credio** is the credit layer that keeps it running — when your agent can't afford a paid x402 resource, Credio fronts the USDC, settles the real x402 payment on Base, and hands over the resource. Your agent repays later to unlock a higher limit.

```
Agent hits 402 → Credio fronts USDC → Resource delivered → Agent repays → Limit grows
```

| | |
|---|---|
| **Non-custodial** | Credio never holds your keys. Ever. |
| **Real x402** | Genuine on-chain USDC settlement on Base. |
| **Gasless repayment** | Repay over x402 with no ETH required. |
| **No API keys** | Auth is your Base wallet address. Zero config. |
| **Verified on-chain** | Every payment checked on-chain before debt changes. |
| **Zero dependencies** | Tiny SDK. Drop in and go. |

---

## Install

```bash
npm install credio-sdk
```

Requires Node 18+. Zero runtime dependencies.

---

## Quickstart

```ts
import { CredioClient } from "credio-sdk"

const credio = new CredioClient({ baseUrl: "https://credio.fun" })
const WALLET = "0xYOUR_BASE_WALLET_ADDRESS"

// Pay for an x402-protected resource with credit.
// Credio reads the 402, fronts the USDC, settles on Base,
// and returns the resource — all in one call.
const res = await credio.payForResource({
  agentWalletAddress: WALLET,
  resourceUrl: "https://api.example.com/premium",
  agentMetadata: { agentName: "My Agent" },
})

if (res.success) {
  console.log(res.resource?.body)               // the paid content
  console.log(res.settlementTx)                 // on-chain proof on Base
  console.log(res.agentStatus?.currentUsdcDebt) // what you now owe
}
```

---

## Automatic Fallback (recommended)

A credit line is only useful if it kicks in on its own. `withCredioFallback` wraps your agent's fetch so it pays from its **own wallet first**, then falls back to Credio credit **automatically** when funds run out. No manual branching.

```ts
import { createX402Client } from "x402-base/client"
import { withCredioFallback } from "credio-sdk"

// Your agent's own x402 client
const client = createX402Client({ wallet, network: "base", rpcUrl })
const ownFetch = client.fetch.bind(client)

// Wrap once. Every request just works.
const fetch = withCredioFallback(ownFetch, { agentWalletAddress: WALLET })

const res = await fetch("https://api.example.com/premium")
// → paid from own funds if available, otherwise Credio credit
```

On the credit path the response includes:
- `x-credio-paid: credit`
- `x-credio-settlement: <on-chain tx hash>`

---

## Repayment

Repaying clears your debt and unlocks a higher credit limit. Two options:

### Gasless over x402 (recommended)

Your agent pays the repay invoice as an x402 request — no ETH needed, the facilitator covers gas.

```ts
import { createX402Client } from "x402-base/client"

const client = createX402Client({ wallet, network: "base", rpcUrl })
const res = await client.fetch(credio.repayInvoiceUrl(WALLET))

const body = await res.json()
// body.repay  → { success, clearedUsdc, remainingDebt }
// body.status → updated tier
```

### Manual USDC transfer

Sign and broadcast a USDC transfer to the treasury yourself, then report the hash.

```ts
const { treasuryAddress } = await credio.checkDebt(WALLET)
const sig = await sendUsdcToTreasury(treasuryAddress, debt) // your transfer
await credio.repay(WALLET, sig) // Credio verifies on-chain, clears debt
```

---

## How It Works

```
                ┌────── pays provider directly (USDC, x402 on Base) ───────┐
                │                                                           ▼
  Agent ──▶ Credio ──▶ reads 402 ──▶ credit check ──▶ settle on-chain ──▶ Provider
    ▲           │                                                           │
    └─ resource ┘◀─────────────── returns the paid resource ───────────────┘

  Later:  Agent ──▶ repays Credio over x402 (gasless) ──▶ debt cleared on Base
```

Credit is **earned, not given** — a new agent starts with a small limit that grows automatically as it demonstrates consistent repayments. Decisions are instant and fully trustless.

---

## API Reference

| Method | Description |
|---|---|
| `payForResource({ agentWalletAddress, resourceUrl, agentMetadata? })` | Pay an x402 resource with credit. Returns the resource + updated agent status. |
| `withCredioFallback(innerFetch, opts)` | Wrap a fetch to auto-fallback to Credio credit on 402. |
| `repayInvoiceUrl(wallet)` | URL of the x402 repay invoice — pay gaslessly with an x402 client. |
| `repay(wallet, txSignature)` | Manual repayment: report a USDC→treasury transfer (verified on-chain). |
| `checkCredit(wallet, amount?)` | Check available credit for a given amount. |
| `checkDebt(wallet)` | Outstanding debt + treasury address. |
| `getStatus(wallet)` | Agent status: tier, debt, credit limit, credibility score. |
| `register(wallet, name?)` | Register an agent (auto-called on first credit request). |
| `treasuryAddress()` | The Credio treasury address on Base. |

Full reference → [credio.fun/developers](https://credio.fun/developers)

---

## Credit Tiers

| Tier | Limit | Requirement |
|---|---|---|
| **Seed** | $5 USDC | New agent, no history |
| **Builder** | $25 USDC | 3 successful repayments |
| **Operator** | $100 USDC | 10 repayments, low default rate |
| **Fleet** | $500+ USDC | Sustained track record |

Limits grow automatically — no applications, no approvals, no humans in the loop.

---

## License

MIT © Credio · [credio.fun](https://credio.fun)
