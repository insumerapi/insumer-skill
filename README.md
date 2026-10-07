# insumer-skill

**Wallet auth for Claude Code.** Add condition-based access to any project — token balances, NFT ownership, on-chain eligibility, pre-transaction trust — in whatever language you're writing. One install, one free API key, signature-verifying code on the first try.

> OAuth proves who the user is. Wallet auth proves what the wallet holds. Boolean, not balance.

> **Not in Claude Code?** This repo is Claude-Code-specific. For Cursor, GitHub Copilot, OpenAI Codex, Gemini CLI, JetBrains Junie, Sourcegraph Amp, Block Goose, and 25+ other IDEs supporting the [agentskills.io](https://agentskills.io) open standard, use [`insumer-agent-skills`](https://github.com/insumerapi/insumer-agent-skills) — same wallet auth primitive, multi-agent install via `npx skills add insumerapi/insumer-agent-skills`.

## What this skill does

When you're working in Claude Code and you ask Claude to "add wallet verification," "gate this endpoint by USDC balance," "check if a wallet holds a specific NFT," or "add a pre-transaction trust check," this skill gives Claude the canonical shapes, verification recipes, and hard-coded reference values for [InsumerAPI](https://insumermodel.com) so the code Claude emits works on the first try.

Concretely, the skill contains:

- `SKILL.md` — the instructions Claude reads when the skill activates
- `reference/endpoints.md` — full request/response shapes for `/v1/attest` and `/v1/trust`, verified against a live call
- `examples/gate-express.ts` — Express middleware that gates an endpoint by USDC balance on Base, with offline JWKS verification via `jose`
- `forbidden.md`: hard stops (things the skill must never emit: inline API keys, unverified responses, raw balance leaks, a guessed `decimals` value, Cloud Functions URLs)

## Install

```bash
git clone https://github.com/insumerapi/insumer-skill.git ~/.claude/skills/insumer
```

The Smithery listing does not currently serve installs, so install from GitHub as above.

Claude Code picks up any skill one level inside `~/.claude/skills/`, which is where this clone puts `SKILL.md`. Restart Claude Code after copying and the skill will be available on the next conversation where you mention wallet auth, token gating, condition-based access, or any of the trigger phrases in `SKILL.md`.

## First use

1. Install the skill (above).
2. Generate a free API key — 10 starter credits + 100 `/v1/attest` calls per day, no signup:

   ```bash
   curl -s -X POST https://api.insumermodel.com/v1/keys/create \
     -H "Content-Type: application/json" \
     -d '{"email":"you@example.com","appName":"insumer-skill","tier":"free"}'
   ```

3. Put the returned key in `.env` as `INSUMER_API_KEY`.
4. In Claude Code, describe what you want: *"add wallet verification to my /premium endpoint so it only lets through wallets with at least 100 USDC on Base."*

Claude will use the skill to produce correct, signature-verifying code.

## The primitive

InsumerAPI is a wallet auth primitive: read → evaluate → sign.

1. **Read**: the API reads blockchain state across 37 chains (31 EVM, 27 of them with Merkle proofs, plus Solana, XRPL, Bitcoin, Tron, Stellar, Sui).
2. **Evaluate**: it evaluates your conditions (token balance threshold, NFT ownership, delegated authority, EAS attestation) against that state.
3. **Sign**: it returns a boolean — pass / fail — signed with ES256 and wrapped in an ES256 JWT with a `kid` that any party can resolve through a public JWKS at `https://insumermodel.com/.well-known/jwks.json`.

The signed boolean is counterparty-portable. Agent A can hand it to Agent B, who can verify it against the JWKS without ever calling the API. There are no secrets to rotate, no identity broker, no static credentials.

**Boolean, not balance**: standard mode returns only the pass/fail result. The wallet's actual holdings never leave the verification layer. Merkle mode is available for callers who need the raw balance for client-side proof reconstruction — it costs double and is opt-in.

**Agents pay for their own access**: there are two crypto-native paths, both no-human-in-the-loop.

- **Cold start** (no key yet): the agent sends USDC, USDT, or BTC to the platform wallet and calls `POST /v1/keys/buy` with the transaction hash. **No email needed** — the sender wallet from the transaction becomes the key's identity. One key per sender wallet.
- **Top-up** (existing key, low credits): the agent sends crypto and calls `POST /v1/credits/buy` with the transaction hash. The key keeps its identity, history, and integrations; credits just increment. Sender must match the wallet registered to the key.

Platform wallets (publicly listed at [insumermodel.com/pricing](https://insumermodel.com/pricing/)):

- **EVM**: `0xAd982CB19aCCa2923Df8F687C0614a7700255a23` (these seven EVM chains only: Ethereum, Base, Polygon, Arbitrum, Optimism, BNB Chain, Avalanche)
- **Solana**: `6a1mLjefhvSJX1sEX8PTnionbE9DqoYjU6F6bNkT4Ydr`
- **Bitcoin**: `bc1qg7qnerdhlmdn899zemtez5tcx2a2snc0dt9dt0` (1 confirmation, market-rate USD conversion)

Volume discounts: $5–$99 → $0.04/call, $100–$499 → $0.03/call (25% off), $500+ → $0.02/call (50% off).

The top-up path is the only continuous-identity upgrade — same key, history preserved. No Stripe, no human approval, no subscription renewal. This is what makes the "agent pays for its own access" loop real.

## Endpoints (the two the skill uses)

- `POST /v1/attest` — 1–10 custom conditions, per-condition booleans, one overall `pass`. 1 credit.
- `POST /v1/trust`: curated profile of 155 base checks across 27 chains in 10 dimensions (stablecoins, governance, NFTs, staking, institutional stablecoins, tokenized treasuries, stablecoin deposits, wrapped bitcoin, names, account), up to 176 checks across 29 chains in 14 dimensions with the optional wallets. Every check is a presence check. 3 credits.

Full shapes in `reference/endpoints.md`.

## Try the example end-to-end (2 minutes)

The `examples/gate-express.ts` file is a working Express server. Verify the full flow — API call, JWKS fetch, offline signature verification, gating — in a scratch dir:

```bash
mkdir /tmp/insumer-skill-try && cd /tmp/insumer-skill-try
npm init -y >/dev/null
npm install express jose tsx typescript @types/express @types/node >/dev/null
curl -sO https://raw.githubusercontent.com/insumerapi/insumer-skill/main/examples/gate-express.ts

# Get a free key (if you haven't already)
curl -s -X POST https://api.insumermodel.com/v1/keys/create \
  -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","appName":"insumer-skill","tier":"free"}'
# → copy the "key" field

# Start the server
INSUMER_API_KEY=insr_live_... npx tsx gate-express.ts
```

In another terminal:

```bash
# Fail path: a wallet without 100 USDC on Base → 403 with signed attestation metadata, no raw balance
curl -sS "http://localhost:3000/premium?wallet=0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045"

# Success path (lower MIN_USDC in gate-express.ts to 1 for a quick check with a whale):
curl -sS "http://localhost:3000/premium?wallet=0x28C6c06298d514Db089934071355E5743bf21d60"

# Validation path: 400 before any API call
curl -sS "http://localhost:3000/premium?wallet=notawallet"
```

The 403 response includes `attestationId`, `blockNumber`, and `blockTimestamp` — enough for a downstream auditor to re-verify — but never the raw balance. That is boolean-not-balance in practice.

## Related tools

- [`insumer-agent-skills`](https://github.com/insumerapi/insumer-agent-skills) — same wallet auth primitive packaged for the [agentskills.io](https://agentskills.io) open standard. Installable in Cursor, Copilot, Codex, Gemini CLI, JetBrains Junie, Block Goose, Sourcegraph Amp, Letta, Roo Code, and 25+ other agentskills-compatible IDEs via `npx skills add insumerapi/insumer-agent-skills`. Use this if you want the same wallet-auth behavior in any agent that isn't Claude Code.
- [`mcp-server-insumer`](https://github.com/insumerapi/mcp-server-insumer) — MCP server for runtime agent access to the same API. Install this if you want an agent to *call* InsumerAPI at runtime; install `insumer-skill` if you want Claude Code to help you *write* code that calls it.
- [`eliza-plugin-insumer`](https://www.npmjs.com/package/@insumermodel/plugin-eliza) — ElizaOS plugin for the same API.
- [`insumer-verify`](https://github.com/insumerapi/insumer-verify) — standalone offline verification library, on [npm](https://www.npmjs.com/package/insumer-verify) for Node and on [PyPI](https://pypi.org/project/insumer-verify/) for Python under the same name; same checks, same 27 published test vectors.

## See also

For multi-issuer trust envelope builders: this skill ships the `wallet_state` category of the trust-evidence-format as defined in [agent-governance-vocabulary](https://github.com/aeoess/agent-governance-vocabulary). For the full multi-issuer envelope pattern (10 issuers, 12 dimensions, JWKS-verifiable offline), see [insumer-examples #1](https://github.com/insumerapi/insumer-examples/issues/1).

## License

MIT.
