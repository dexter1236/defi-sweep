# DeFi Sweeper

AI agent that scans any wallet across 6 EVM chains for stale DeFi positions, scores them with a heuristic + LLM, and executes **one-click (or one-signature batch) cleanup** that logs every sweep on-chain on **X Layer**.

Built for the **OKX Build X Hackathon 2026** — X Layer Arena track.

---

## Live X Layer deployment

| | |
|---|---|
| **Contract** | `SweeperRegistry` |
| **Address** | [`0xb8a5a48c334eb2fa0836eaa676a35bf19247a46b`](https://www.okx.com/web3/explorer/xlayer-test/address/0xb8a5a48c334eb2fa0836eaa676a35bf19247a46b) |
| **Network** | X Layer Testnet (chain ID **1952**) |
| **Deploy tx** | [`0x257cce16…fbcabda`](https://www.okx.com/web3/explorer/xlayer-test/tx/0x257cce16284b0166257cbaac858d5039618e5621c47f9066ebff6c1cdfbcabda) |
| **Agent wallet** | [`0x8bf616b27f0450ae258806f40915f8fa6e844220`](https://www.okx.com/web3/explorer/xlayer-test/address/0x8bf616b27f0450ae258806f40915f8fa6e844220) |
| **Source** | [`contracts/SweeperRegistry.sol`](./contracts/SweeperRegistry.sol) |

Every user sweep triggers an on-chain `PositionSwept` event. The contract tracks `totalSweeps`, `totalValueRecoveredScaled`, and per-user counters — all queryable on-chain.

### Contract API

```solidity
logSweep(platform, chain, valueUsdScaled, healthScore, destination)
batchLogSweep(platforms[], chains[], values[], scores[], destinations[])
totalSweeps() → uint256
totalValueRecoveredScaled() → uint256
userSweepCount(address) → uint256
```

---

## Why this hits hackathon criteria

| Criterion | How we hit it |
|---|---|
| **X Layer Arena — "at least one part deployed on X Layer"** | `SweeperRegistry` live on X Layer Testnet. Every user sweep and every **batch sweep** writes a verifiable `PositionSwept` event. |
| **Meaningful development during hackathon** | 13+ Claude Code sessions, git history, Figma-matched frontend, live contract, batch transaction, AI integration. |
| **Onchain OS API usage** (Most Active Agent special prize) | Scan API calls `/api/v6/defi/user/asset/platform/list` + `.../platform/detail` with HMAC-SHA256, plus `onchainos` CLI subprocess fallback that transparently takes over when HMAC auth fails. Every scan touches OKX. |
| **Agentic claim backed by data** | Heuristic scoring runs server-side, then Grok / Gemini / Groq (auto-detected) rewrites every position's `aiSummary` + explanations with LLM-quality reasoning. |

---

## What it does

1. **Connect wallet** — auto-prompts to switch to X Layer Testnet (1952). No other chains offered.
2. **Scan** — server calls OKX v6 DeFi endpoints for the connected (or spectated) address across ETH / BSC / Polygon / Arbitrum / Base / X Layer.
3. **Score** — heuristic 0–100 health based on dust threshold, protocol age, rewards neglect, liquidation risk. LLM post-pass produces the final `aiSummary` + 2–4 concrete explanations.
4. **Sweep** — the user can clean up:
   - **One position** → signature required before any animation runs; rejection cancels cleanly.
   - **All stale/warning positions** → single `batchLogSweep` tx signs everything at once (~4× cheaper gas).
5. **Log on-chain** — the wallet signs an X Layer tx that fires `PositionSwept` events for every position. After success, the source row is removed from the table and `move` recommendations spawn a new healthy X Layer position.

### UX affordances

- **Spectator mode** — paste any EVM address (e.g. Vitalik's) to pull its real DeFi positions from OKX; still sign the sweep log from your own wallet. Zero-cost demo.
- **Use demo data** — 20 diverse positions across 6 chains and 4 health bands for offline testing.
- **Dust warning** — DEAD positions (score < 20) show a red warning before confirm: "gas may exceed recovery; consider Sweep all instead."
- **Post-sweep table update** — instant row removal + toast with recovered $ and per-session `totalSaved` counter (persisted to `localStorage`).
- **4-segment risk bar** — dead / stale / warning / healthy, with counts.

---

## Tech stack

| Layer | Tools |
|---|---|
| Framework | Next.js 16 (App Router, Turbopack) · React 19 · TypeScript |
| Styling | TailwindCSS v4 · Red Hat Display / Text · JetBrains Mono |
| Wallet | wagmi v3 · viem · RainbowKit (locked to chain 1952) |
| Data | OKX Web3 v6 REST (HMAC-SHA256) · `onchainos` CLI fallback · TanStack React Query |
| AI | Grok (xAI), Gemini, or Groq (Llama 3.3 70B) — OpenAI-compatible, auto-detect |
| Contract | Solidity 0.8.24 · compiled with `solc` · deployed via `viem` |
| Icons | `@web3icons/react` + `@web3icons/common` metadata (1842 tokens, resolved by contract address) |
| Chain | X Layer Testnet (chain 1952) |

---

## Routes

| Path | Purpose |
|---|---|
| `/` | Marketing landing page — all CTAs route to `/app` |
| `/app` | Gateway: shows a blurred dashboard + connect card when disconnected, full dashboard when connected |
| `/dashboard` | Legacy alias; redirects unconnected users to `/app` |
| `/position/[id]` | Position detail view with sweep flow |
| `/api/scan` | OKX scan → scoring → optional LLM enhancement |
| `/api/sweep` | Builds the withdraw/swap/deposit tx sequence |
| `/sitemap.xml` | Auto-generated |
| `/robots.txt` | Auto-generated — disallows `/api/` and `/position/` |

---

## Getting started

### 1. Install

```bash
npm install
```

### 2. Environment — `.env.local`

```bash
# OKX Web3 API (HMAC) — https://web3.okx.com/build/dev-portal
OKX_PROJECT_ID=...
OKX_API_KEY=...
OKX_SECRET_KEY=...
OKX_PASSPHRASE=...

# WalletConnect — https://cloud.walletconnect.com
NEXT_PUBLIC_WALLETCONNECT_PROJECT_ID=...

# AI enhancer (optional — heuristic scoring always runs first; pick one)
XAI_API_KEY=      # https://console.x.ai (paid)
GEMINI_API_KEY=   # https://aistudio.google.com/apikey (free, personal Gmail)
GROQ_API_KEY=     # https://console.groq.com (free, no card)

# Contract deployment (only if redeploying)
DEPLOYER_PRIVATE_KEY=0x...
# NETWORK=mainnet   # default: testnet

# Demo mode — skip OKX calls, auto-load 20 demo positions (for video recording)
NEXT_PUBLIC_DEMO_MODE=false

# Site URL for sitemap / robots / OG tags
NEXT_PUBLIC_APP_URL=http://localhost:3000
```

### 3. Run

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000), click **Launch app**, connect a wallet.

### 4. Redeploy contract (optional)

```bash
node scripts/deploy-sweeper-registry.mjs                     # X Layer Testnet
NETWORK=mainnet node scripts/deploy-sweeper-registry.mjs     # X Layer Mainnet
```

Writes address + ABI to `src/contracts/sweeper-registry.json` — the frontend imports it.

---

## Testing paths

| Path | Real OKX data | Real on-chain log | Cost |
|---|---|---|---|
| Wallet dropdown → **Use demo data** | ❌ mock | ✅ (~0.0001 OKB testnet) | free |
| Wallet dropdown → **Spectator mode** → paste address | ✅ real positions for any wallet | ✅ | free |
| Connect your own mainnet wallet | ✅ your positions | ✅ | free if wallet has OKB testnet |
| Create a tiny mainnet position (e.g. $5 Aave on Base) | ✅ | ✅ | ~$5 one-time |

Testnet OKB faucet: [https://web3.okx.com/xlayer/faucet](https://web3.okx.com/xlayer/faucet) — 0.2 OKB/day.

---

## Project structure

```
contracts/
└── SweeperRegistry.sol              # 80 LOC: logSweep + batchLogSweep
scripts/
└── deploy-sweeper-registry.mjs      # solc compile + viem deploy
src/
├── app/
│   ├── page.tsx                     # Landing
│   ├── app/                         # Gateway + connected dashboard
│   ├── dashboard/                   # Dashboard implementation
│   ├── position/[id]/               # Position detail
│   ├── api/scan/                    # OKX scan + scoring + AI enhance
│   ├── api/sweep/                   # Build cleanup tx sequence
│   ├── sitemap.ts · robots.ts
│   └── layout.tsx                   # Root metadata + providers
├── components/
│   ├── header.tsx                   # X Layer pill + wallet dropdown
│   ├── wallet-dropdown.tsx          # Demo · Spectator · Disconnect actions
│   ├── position-detail-sheet.tsx    # Sweep flow with signature-before-sweep + logSweep() wiring
│   ├── web3-icons.tsx               # TokenIcon / NetworkIcon / TokenPair with address→metadata resolution
│   └── sweep-steps.tsx
├── contracts/
│   └── sweeper-registry.json        # deployed address + ABI (generated)
├── lib/
│   ├── okx-api.ts                   # HMAC v6 client
│   ├── onchainos-cli.ts             # Subprocess fallback for scan
│   ├── scoring.ts                   # Heuristic 0–100 health engine
│   ├── ai-enhance.ts                # Grok / Gemini / Groq post-processor
│   └── types.ts
└── providers/web3-provider.tsx      # wagmi + RainbowKit, X Layer Testnet only
```

---

## Scripts

| Command | Does |
|---|---|
| `npm run dev` | Turbopack dev server on :3000 |
| `npm run build` | Production build |
| `npm run start` | Start production server |
| `npm run lint` | ESLint |
| `node scripts/deploy-sweeper-registry.mjs` | Compile + deploy contract |

---

## License

MIT.
