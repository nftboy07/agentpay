# AgentPay

**An AI agent with a wallet, a budget, and rules — it negotiates for data, pays in USDC on Arc, and proves every purchase onchain.**

University Blockchain Conference Hackathon 2026 · Track: **Agentic Payments and AI** · Chain award: **Best use of Arc** · Solo builder: Jacku P

Live demo: https://nftboy07.github.io/agentpay/ · Devpost: *(link added at submission)*

> Example onchain payment (Day-1 spike, Arc Testnet):
> [`0xd34072c2e0c98531ee5808ea587052ee63266984a5228c989ab7a52787360b4b`](https://testnet.arcscan.app/tx/d34072c2e0c98531ee5808ea587052ee63266984a5228c989ab7a52787360b4b)
> — 0.01 USDC agent → seller, receipt status 1, block 72129605. Faucet drip that funded it:
> [`0x0ce9edf7ed33346c5bd034781ce57d52f4b7a4bc5093b570ecc8d39f7405f56c`](https://testnet.arcscan.app/tx/0x0ce9edf7ed33346c5bd034781ce57d52f4b7a4bc5093b570ecc8d39f7405f56c) (20 test USDC, Circle faucet).

## What it is

You give an AI agent a job ("build me a morning market briefing") and a small USDC budget. The agent
decomposes the job into data it must buy, collects quotes from priced micro-services, **negotiates**
(comparing quotes against its budget and policy, counter-offering once, walking away from bad prices), and
**settles each accepted micro-payment onchain in USDC on Arc testnet**, verifying the receipt before using
the data. Every step is visible on a live dashboard: the budget draining, the guardrails holding, each
decision the agent made and why, and every transaction linked to the block explorer.

## Architecture

One static HTML file, no backend, no build step, hosted on GitHub Pages. Everything a judge needs to
verify is either on that page or on the block explorer.

```
index.html (GitHub Pages)
├── User panel — task input, budget, per-payment cap, approval threshold, allowlist, kill switch
├── Agent engine (in-browser JS)
│   ├── decompose task → purchasable items
│   ├── collect quotes from allowlisted sellers
│   ├── NEGOTIATE (target price → one counter-offer at midpoint → accept / walk away)
│   ├── POLICY CHECK (budget / cap / allowlist / approval threshold)
│   ├── PAY (ERC-20 USDC transfer) → VERIFY (receipt + balance delta) → deliver
│   └── log every step in plain language
├── Seller services (in-page mocks) — market data · headlines/sentiment · AI summary
│   └── delivery gated on a verified onchain payment receipt (x402-style, demo scale)
├── Agent wallet (in-browser, ethers.js Wallet, session-only key) → seller wallet
└── Live dashboard — balances, budget gauge, decision log, tx feed with Arcscan links,
    assembled briefing, run history, read-only address tracker
                    │
                    ▼
         Arc Testnet (chain 5042002)
    USDC 0x3600…0000 · RPC rpc.testnet.arc.io
```

### Decision loop (the track requires this to be published)

```
Task → decompose into needed items
  → for each item: collect quotes from allowlisted sellers
  → NEGOTIATE: accept if ≤ target price; else counter-offer once at midpoint;
               accept a counter at/below cap, else walk away and try the next seller
  → POLICY CHECK: budget remaining, per-payment cap, allowlist, approval threshold
  → if above approval threshold: pause and ask the human (Approve / Reject)
  → PAY: ERC-20 USDC transfer on Arc
  → VERIFY: receipt status + balance delta
  → deliver item (delivery is gated on payment) → log everything
```

### Safety limits (all enforced before money moves — user-editable in the UI)

| Limit | Default | Behaviour |
|---|---|---|
| Total budget | 1.00 USDC | Hard stop — the agent never exceeds it; an item that would breach it is skipped |
| Per-payment cap | 0.10 USDC | Any quote above it is refused outright, no negotiation |
| Seller allowlist | all 3 on | Toggled-off sellers' quotes are refused even when cheapest |
| Human-approval threshold | 0.05 USDC | Payments above it pause for an Approve / Reject click |
| Kill switch | — | Halts the agent mid-run; state stays inspectable |

## Chain: Arc Testnet — and the decimals trap

- **Chain ID:** 5042002 · **RPC:** `https://rpc.testnet.arc.io` · **Explorer:** https://testnet.arcscan.app
- **USDC ERC-20 interface:** `0x3600000000000000000000000000000000000000`, **6 decimals**
- **Faucet:** https://faucet.circle.com (Arc Testnet → USDC; 20 test USDC per drip, ~2 h cadence)
- Arc was chosen because USDC is its native gas token: one asset is the budget, the fee, and the payment,
  so balances and costs stay simple and predictable. One 20-USDC drip funds hundreds of demo payments
  (a 0.01-USDC payment cost 0.001479 USDC in gas in the Day-1 spike).

⚠️ **The decimals trap:** on Arc, USDC's *native* balance is denominated at 18 decimals while its
*ERC-20 interface* is 6 decimals, over the same underlying balance. AgentPay uses **only the ERC-20
interface** (`balanceOf` / `transfer` on `0x3600…0000`, 6 dp) for every read and write, and never
displays a raw native balance. One convention, everywhere, no mixed units.

### Fallback chain (tested, not needed)

If Arc had been unreachable on Day 1, the same code ports to **Base Sepolia** by swapping the one `CHAIN`
config object in `index.html`: chain 84532, RPC `https://sepolia.base.org`,
USDC `0x036CbD53842c5426634e7929541eC2318f3dCF7e` (6 dp), gas = Sepolia ETH from the Coinbase CDP faucet.
Arc RPC + Circle faucet were both verified reachable from India on 13 Nov 2026, so the fallback was not used.

## Setup / run it

No install, no build, no keys, no secrets.

1. Open `index.html` directly in a browser, or serve the folder (`python3 -m http.server`) and open it.
   (Served over http(s) is recommended so `sessionStorage` behaves normally; `file://` also works.)
2. The page generates an agent wallet + a seller wallet in-page and connects to the Arc testnet RPC.
3. Fund the agent wallet from the Circle faucet link shown on the page (Arc Testnet → USDC).
   A 20-USDC drip funds the whole demo week.
4. Press **Run the agent**. Watch quotes arrive, the premium seller get countered, the approval pause
   fire above the threshold, payments settle onchain, and the briefing assemble from paid-for data.
5. Try the guardrails: lower the per-payment cap and re-run (premium quote refused outright);
   toggle a seller off the allowlist (its cheapest quote is skipped); hit the kill switch mid-run.

To use an already-funded testnet wallet instead of a fresh one, paste its private key in the
"import a funded testnet agent key" field. The key lives in **session storage only** and never leaves the page.

## Honest framing

- **Sellers are simulated** priced services with fixed price personalities. **Settlement is real**: every
  accepted purchase is an ERC-20 USDC `transfer` on Arc testnet, receipt-verified, linked to Arcscan.
  Nothing about the money is faked; the demo's "data" is the simulated part, and the README says so.
- **Testnet only.** The agent wallet key is generated in-page, kept in session storage only, never sent
  anywhere, never persisted to disk, never committed. Never send mainnet funds to it.
- No server, no database, no deployed custom contract (the rules explicitly say using existing contracts
  counts — the USDC system contract is the contract used), no LLM dependency. The "AI" is the agent's
  decompose / negotiate / policy behaviour: deterministic and demo-proof.

## Built with

JavaScript · ethers.js 6 · Arc Testnet · USDC (Circle) · Circle Faucet · GitHub Pages · HTML/CSS (single-file web app)

## Submission

- **Project:** AgentPay — Agentic Payments and AI (main track), Best use of Arc (chain award), no bounties
- **Demo video:** *(2–3 min, YouTube — recorded 19 Nov, link added here at submission)*
- **Deadline:** Fri 20 Nov 2026, 23:30 IST (12:00 PM CST) — internal target 20:00 IST
- **Note:** remote submission is allowed; per the published rules, prizes require in-person presence at the
  Austin closing ceremony (21 Nov). This entry is built with no prize expectation.

## License

MIT — see `LICENSE`.
