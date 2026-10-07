# Endpoints reference

Verified against a live call on 2026-04-13 using a free-tier key tagged `appName: "insumer-skill"`. All field names, types, and nesting match the response the API actually returned. When in doubt, trust a fresh live call over this document — the spec at <https://insumermodel.com/openapi.yaml> is the source of truth.

## `POST /v1/attest`

Verify 1–10 on-chain conditions for a wallet. Returns ECDSA-signed booleans.

### Request

```http
POST https://api.insumermodel.com/v1/attest
Content-Type: application/json
X-API-Key: insr_live_...
```

```json
{
  "wallet": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "format": "jwt",
  "conditions": [
    {
      "type": "token_balance",
      "contractAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
      "chainId": 8453,
      "threshold": "100",
      "label": "USDC on Base >= 100"
    }
  ]
}
```

**Wallet fields** (at least one required, depending on chain):

| Field | Format | Used for |
|---|---|---|
| `wallet` | `0x...` 40 hex chars | All 31 EVM chains |
| `solanaWallet` | base58 | Solana conditions |
| `xrplWallet` | r-address 25–35 chars | XRPL conditions |
| `bitcoinWallet` | P2PKH / P2SH / bech32 / Taproot | Bitcoin conditions |
| `tronWallet` | T-address, base58, 34 chars | Tron conditions |
| `stellarWallet` | G-address, 56 chars | Stellar conditions |
| `suiWallet` | `0x` + 64 hex chars | Sui conditions |

**Optional flags**:

| Flag | Value | Effect |
|---|---|---|
| `format` | `"jwt"` | Adds a ready-to-verify ES256 JWT to the response (no extra credit cost) |
| `proof` | `"merkle"` | Adds EIP-1186 Merkle proofs on 27 of 31 EVM chains: storage proofs for `token_balance` conditions, account proofs (`subject: "account_code"`, with `codeHash` as the proven value) for `account_code` conditions, revocation-slot proofs for `erc7710_delegation`. Not available on ZKsync Era (324), Sei (1329), Viction (88) or XDC Network (50), nor on any non-EVM chain. Costs 2 credits instead of 1. Note: Merkle mode reveals the raw balance, standard mode never does. |

**Condition types**: `token_balance`, `nft_ownership` (33 of 37 chains: EVM + Solana + XRPL), `eas_attestation` (Ethereum, Optimism, Polygon, Base, Arbitrum), `farcaster_id`, `evm_view_call` (single-address-argument view function returning bool; needs `selector`, EVM chains only), `ratio_to_amount`, `ratio_to_supply`, `erc8004_agent` (Base; needs `agentId`), `erc7710_delegation` (Base; needs `delegationManager`, `expectedDelegator`, `delegation`; max 3 per call, 5-minute attestation expiry), and `account_code` (EVM chains only; needs `expect`: `"none"` for a plain key account with no code, `"eip7702"` for an EIP-7702 delegation designator, `"contract"` for any other code; optional `delegate`, only with `"eip7702"`, met only when the designator points at that address; a non-EVM `chainId` or a `delegate` with another `expect` is a `400`. The result is the boolean `met`; the code and the delegation target are never returned. Signed `evaluatedCondition`: `{"type":"account_code","chainId":8453,"expect":"eip7702","operator":"code_state"}` plus `delegate`, lowercase, when supplied. Real example: wallet `0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045` with `{"type":"account_code","chainId":8453,"expect":"eip7702"}` returns `met: true` and `conditionHash` `0x6c5752bfbfcfd6ba36c9cda6c74df567f0e0414da6b7a3176061ba734aeadc46`). Ten types in all.

**Max conditions per call**: 10.

### Condition shapes

**Token balance (ERC-20, SPL, XRP, XRPL trust line, Bitcoin)**:
```json
{
  "type": "token_balance",
  "contractAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
  "chainId": 8453,
  "threshold": "100",
  "label": "USDC on Base >= 100"
}
```
- `threshold` is a **decimal string** in token/display units (e.g. `"100"`, not `100`). Keys created from 2026-06-10 sign with `kid: insumer-attest-v2`, which requires the string form to preserve full precision and rejects a JSON number with a `400`. (Older `insumer-attest-v1` keys accept either; a string is safe on both.)
- `decimals` is optional. Leave it out: the token's own decimals are always read from the chain. If sent it is only a cross-check, and a value that differs from the token's own decimals is rejected with a `400`.
- For native tokens (ETH, SOL, XRP, BTC, TRX, XLM), set `contractAddress: "native"`. `"native"` is for `token_balance` and `ratio_to_amount` only.
- On Sui, `contractAddress` is always a coin type (`address::module::Name`). Native SUI is `"0x2::sui::SUI"`; the string `"native"` is a `400` there.
- For XRPL trust line tokens, add `currency: "RLUSD"` (or similar currency code). Currency codes are case-sensitive and matched exactly as issued: send the code exactly as the issuer created it (`USD` and `usd` are different currencies). XRP is the native coin, not an issued currency, so it is not accepted as the `currency` of a trust line condition: use `contractAddress: "native"` for XRP.

**NFT ownership (ERC-721 style on EVM, XRPL NFToken)**:
```json
{
  "type": "nft_ownership",
  "contractAddress": "0xBC4CA0EdA7647A8aB7C2061c2E118A18a936f13D",
  "chainId": 1,
  "label": "Bored Ape holder"
}
```
- `contractAddress` is the NFT contract (0x + 40 hex on EVM). `"native"` with `nft_ownership` is a `400`; use `token_balance` to check the native coin.

**EAS attestation (via compliance template)**:
```json
{
  "type": "eas_attestation",
  "template": "coinbase_verified_account",
  "label": "Coinbase KYC verified"
}
```

**EAS attestation (raw)**:
```json
{
  "type": "eas_attestation",
  "schemaId": "0xf8b05c79f090979bf4a80270aba232dff11a10d9ca55c4f88de95317970f0de9",
  "attester": "0x357458739F90461b99789350868CD7CF330Dd7EE",
  "indexer": "0x2c7eE1E5f416dfF40054c27A62f7B357C4E8619C",
  "chainId": 8453,
  "label": "Coinbase Verified Account"
}
```

**Account code state (EVM only)**:
```json
{
  "type": "account_code",
  "chainId": 8453,
  "expect": "eip7702",
  "label": "EIP-7702 delegation on Base"
}
```
- `expect` is required: `"none"` (no code, a plain key account), `"eip7702"` (the EIP-7702 delegation designator), or `"contract"` (any other code). The three states are exclusive on a chain.
- `delegate` (optional, an EVM address) is accepted only with `expect: "eip7702"` and is met only when the designator points at it. It is echoed, lowercase, inside the signed `evaluatedCondition`.
- The wallet above (`0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045`) returns `met: true` for this condition; it carries an EIP-7702 delegation designator on Base, Ethereum and Optimism.
- In proof mode the result carries an EIP-1186 account proof, `subject: "account_code"`, with `blockNumber`, `nonce`, `balance`, `storageHash`, `codeHash`, `accountProof`. `codeHash` is the proven value: keccak256 of empty code (`0xc5d2460186f7233c927e7db2dcc703c0e500b653ca82273b7bfad8045d85a470`) means no code; keccak256 of `0xef0100` followed by the 20-byte target means an EIP-7702 delegation to that target (checkable only with the `delegate` the verifier supplies); anything else means contract code. An account absent from the state trie may report all zeros, which also means no code. Proof subjects are three: `account_balance`, `account_code`, `delegation_revocation`, plus the subject-less ERC-20 slot proof.

### Response (live capture, 2026-04-13 — a v1 key)

This is a real recorded response, kept because the `sig`, `jwt` and `conditionHash` in it are
genuine bytes. It predates the 2026-06-10 cutover, so it shows the **v1** shape. On any key created
since then the same call returns `kid: "insumer-attest-v2"`, the `evaluatedCondition.threshold` as a
canonical decimal string, and **no `decimals` field**. The hashing and signing algorithms are
unchanged; the preimage differs.

```json
{
  "ok": true,
  "data": {
    "attestation": {
      "id": "ATST-85ADCD1399EF9C2A",
      "pass": false,
      "results": [
        {
          "condition": 0,
          "label": "USDC on Base >= 100",
          "type": "token_balance",
          "chainId": 8453,
          "met": false,
          "evaluatedCondition": {
            "type": "token_balance",
            "chainId": 8453,
            "contractAddress": "0x833589fCD6eDb6E08f4c7C32D4f71b54bdA02913",
            "operator": "gte",
            "threshold": 100,
            "decimals": 6
          },
          "conditionHash": "0x60c2b6298d381185a85767ab332390eda5e9993be93bdeb2c002776bf7464c22",
          "blockNumber": "0x2a93c39",
          "blockTimestamp": "2026-04-13T11:36:53.000Z"
        }
      ],
      "passCount": 0,
      "failCount": 1,
      "attestedAt": "2026-04-13T11:36:55.032Z",
      "expiresAt": "2026-04-13T12:06:55.032Z"
    },
    "sig": "SGyO7nOrviIF/QQMp9lWPhtnIuraV4vrcVJyF0Z1ZWxsOSmXJmDNMzNZEb6KAk96T62iuTqzgr0eYkD+3Xg4YQ==",
    "kid": "insumer-attest-v1",
    "jwt": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCIsImtpZCI6Imluc3VtZXItYXR0ZXN0LXYxIn0.eyJwYXNzIjpmYWxzZSwiY29uZGl0aW9uSGFzaCI6WyIweDYwYzJiNjI5OGQzODExODVhODU3NjdhYjMzMjM5MGVkYTVlOTk5M2JlOTNiZGViMmMwMDI3NzZiZjc0NjRjMjIiXSwiYmxvY2tOdW1iZXIiOiIweDJhOTNjMzkiLCJibG9ja1RpbWVzdGFtcCI6IjIwMjYtMDQtMTNUMTE6MzY6NTMuMDAwWiIsInJlc3VsdHMiOlsuLi5dLCJpc3MiOiJodHRwczovL2FwaS5pbnN1bWVybW9kZWwuY29tIiwic3ViIjoiMHhkOGRBNkJGMjY5NjRhRjlEN2VFZDllMDNFNTM0MTVEMzdhQTk2MDQ1IiwianRpIjoiQVRTVC04NUFEQ0QxMzk5RUY5QzJBIiwiaWF0IjoxNzc2MDgwMjE1LCJleHAiOjE3NzYwODIwMTV9.6GSiVkD74qhLPg-Asnb-_9Gcl0G9waTKvfLQZguFx8_HXgGzEUxwiXDTQgeGEWzd46Qk-LsARA6S9zcn22JI7Q"
  },
  "meta": {
    "version": "1.0",
    "timestamp": "2026-04-13T11:36:55.194Z",
    "creditsRemaining": 9,
    "creditsCharged": 1
  }
}
```

### JWT claims (when `format: "jwt"`)

Standard claims:
- `iss`: `"https://api.insumermodel.com"`
- `sub`: the wallet a condition in the request evaluated (with conditions across chain families, the first in the order EVM, Solana, XRPL, Bitcoin, Tron, Stellar, Sui)
- `jti`: the attestation ID (e.g. `"ATST-85ADCD1399EF9C2A"`)
- `iat`: issued-at, Unix seconds
- `exp`: expires-at, Unix seconds (`iat + 1800`, 30 minutes; `iat + 300` when the request includes an `erc7710_delegation` condition)

Custom claims:
- `pass`: overall boolean (true only if all conditions met)
- `conditionHash`: array of per-condition SHA-256 hashes (one per condition in the request)
- `blockNumber`: hex block number the conditions were evaluated against
- `blockTimestamp`: ISO 8601 timestamp of that block
- `results`: full per-condition results array (same shape as the top-level `data.attestation.results`)

Header: `{"alg":"ES256","typ":"JWT","kid":"<the signing key ID>"}` — `insumer-attest-v2` on a key issued today, `insumer-attest-v1` on a pre-cutover key (as in the captured example above). Read the `kid` from the header rather than assuming it.

Verify the JWT with any standard library pointed at `https://insumermodel.com/.well-known/jwks.json`.

### Key response fields

| Field | Meaning |
|---|---|
| `ok` | Top-level success flag |
| `data.attestation.pass` | True only if ALL conditions met |
| `data.attestation.results[].met` | Per-condition boolean |
| `data.attestation.results[].evaluatedCondition` | Canonical form of the condition that was actually evaluated (may differ from your input where the API normalized it, for example the operator it applied) |
| `data.attestation.results[].conditionHash` | SHA-256 of `evaluatedCondition`, prefixed `0x`. Callers can recompute to verify the condition wasn't tampered with. |
| `data.attestation.results[].blockNumber` | Hex block number (all 31 EVM chains; Solana uses `slot`, XRPL uses `ledgerIndex`). API returns 503 if the anchor can't be captured rather than signing a partial result. |
| `data.attestation.results[].blockTimestamp` | ISO 8601 timestamp of the evaluation block |
| `data.sig` | Base64 P1363 ES256 signature (88 chars). The preimage is selected by `data.kid`: `insumer-attest-v2` signs `"insumer.attestation.v2" + "\n" + canonical_json({v:2,id,pass,results,attestedAt})` (keys sorted recursively); `insumer-attest-v1` signs the bare `JSON.stringify({id,pass,results,attestedAt})`. `expiresAt` is outside the preimage. |
| `data.pqSig` / `data.pqKid` | Post-quantum companion (ML-DSA-65, FIPS 204) over the post-quantum domain tag plus the same classical preimage; `pqKid` is `insumer-attest-pq1`, an RFC 9964 `AKP` entry in the same JWKS. Additive beside `sig`/`kid`. |
| `data.kid` | Signing key ID — look up in JWKS |
| `data.jwt` | Present only when `format: "jwt"` requested; ES256 JWT with claims listed above |
| `meta.creditsRemaining` | Credit balance after this call |
| `meta.creditsCharged` | `1` standard, `2` with `proof: "merkle"` |

### Error responses

| Status | Meaning |
|---|---|
| `400` | Validation error (malformed wallet or contract address, unknown chain, missing required field, a `decimals` value that differs from the token's own) |
| `401` | Missing or invalid `X-API-Key` |
| `402` | Insufficient credits — top up or upgrade tier |
| `503` | Error code `rpc_failure`: a read did not complete. No attestation signed, no credits charged. It is never a `false`. Retry after a short delay. |

All errors follow the `ErrorEnvelope` shape:
```json
{
  "ok": false,
  "error": {
    "code": "string",
    "message": "string"
  }
}
```

---

## `POST /v1/trust`

Curated wallet trust profile: 155 base checks across 27 chains in 10 dimensions, up to 176 checks across 29 chains in 14 dimensions with the optional wallets. Every check is a presence check (held or not held, never a balance; present or not present for the `account` dimension). Returns a signed profile with per-check booleans and an overall summary.

### Request

```http
POST https://api.insumermodel.com/v1/trust
Content-Type: application/json
X-API-Key: insr_live_...
```

```json
{
  "wallet": "0xd8dA6BF26964aF9D7eEd9e03E53415D37aA96045",
  "solanaWallet": "...",
  "xrplWallet": "...",
  "bitcoinWallet": "...",
  "tronWallet": "...",
  "stellarWallet": "...",
  "suiWallet": "..."
}
```

- `wallet` is required (EVM).
- `solanaWallet`, `xrplWallet`, `bitcoinWallet`, `tronWallet` are optional; each switches on its own dimension. `stellarWallet` and `suiWallet` are optional and add no dimension; they let the Stellar and Sui rows inside the base dimensions evaluate.
- Optional `proof: "merkle"` costs 6 credits instead of 3: EIP-1186 storage proofs on EVM token rows. Rows whose balance is computed rather than stored (Aave aTokens, BUIDL) are declined at once with a reason, as are NFT, name and non-EVM rows; `account` rows carry `proof.available: false` with a reason pointing at `/v1/attest`; the premium is refunded whenever no proof is delivered.

### Response shape

Abbreviated; the ids and counts are from a real profile of `0x1601843c5E9bC251A3272907010AFa41Fa18347E`, a contract on all five `account` chains:

```json
{
  "ok": true,
  "data": {
    "trust": {
      "id": "TRST-81224",
      "wallet": "0x1601843c5E9bC251A3272907010AFa41Fa18347E",
      "conditionSetVersion": "2026-10-08",
      "dimensions": {
        "stablecoins": {
          "checks": [ { "label": "...", "met": true, "chainId": 1, "..." } ],
          "passCount": 6,
          "failCount": 46,
          "notEvaluatedCount": 0,
          "total": 52
        },
        "governance": { "checks": [ ... ], "passCount": 0, "failCount": 8, "notEvaluatedCount": 0, "total": 8 },
        "nfts":       { "checks": [ ... ], "passCount": 0, "failCount": 3, "notEvaluatedCount": 0, "total": 3 },
        "staking":    { "checks": [ ... ], "passCount": 0, "failCount": 5, "notEvaluatedCount": 0, "total": 5 },
        "institutional_stablecoins": { "checks": [ ... ], "passCount": 0, "failCount": 2, "notEvaluatedCount": 6, "total": 8 },
        "tokenized_treasuries": { "checks": [ ... ], "passCount": 0, "failCount": 15, "notEvaluatedCount": 1, "total": 16 },
        "stablecoin_deposits":  { "checks": [ ... ], "passCount": 5, "failCount": 34, "notEvaluatedCount": 0, "total": 39 },
        "wrapped_bitcoin":      { "checks": [ ... ], "passCount": 0, "failCount": 12, "notEvaluatedCount": 0, "total": 12 },
        "names":                { "checks": [ ... ], "passCount": 0, "failCount": 2, "notEvaluatedCount": 0, "total": 2 },
        "account": {
          "checks": [
            { "label": "Contract code on Ethereum", "chainId": 1, "met": true, "evaluatedCondition": { "type": "account_code", "chainId": 1, "expect": "contract", "operator": "code_state" }, "conditionHash": "0xfd7b6aa42eb012184fa54d9d5d99c6ab18481e0ce33ec0ec0e9d28d37b54ccbc", "blockNumber": "0x18eea5a", "blockTimestamp": "2026-10-07T21:52:59.000Z" },
            { "label": "EIP-7702 delegation on Ethereum", "chainId": 1, "met": false, "evaluatedCondition": { "type": "account_code", "chainId": 1, "expect": "eip7702", "operator": "code_state" }, "conditionHash": "0xdef6fadcef95f59f4621fa2bf788e6be0ffc0492dba22038999b8cd757adf18b", "blockNumber": "0x18eea5a", "blockTimestamp": "2026-10-07T21:52:59.000Z" },
            "..."
          ],
          "passCount": 5, "failCount": 5, "notEvaluatedCount": 0, "total": 10
        },
        "solana":     { "checks": [ ... ], "...": "only present when solanaWallet provided" },
        "xrpl":       { "checks": [ ... ], "...": "only present when xrplWallet provided" }
      },
      "summary": {
        "totalChecks": 155,
        "totalPassed": 16,
        "totalFailed": 132,
        "totalNotEvaluated": 7,
        "dimensionsWithActivity": 3,
        "dimensionsChecked": 10
      },
      "profiledAt": "2026-10-07T21:53:04.795Z",
      "expiresAt": "2026-10-07T22:23:04.795Z"
    },
    "sig": "base64 P1363 signature over trust object",
    "kid": "insumer-trust-v2",
    "pqSig": "base64 ML-DSA-65 companion signature",
    "pqKid": "insumer-trust-pq1"
  },
  "meta": {
    "creditsRemaining": 850,
    "creditsCharged": 3,
    "version": "1.0",
    "timestamp": "2026-10-07T21:53:05.000Z"
  }
}
```

### Dimensions

- **stablecoins**: USDC, USDT, OUSD, PYUSD, USDG, USD1, RLUSD, USDS, DAI and EURC across 23 EVM chains (52 checks)
- **governance**: UNI, AAVE, ENS, LDO, SKY and COMP on Ethereum, ARB on Arbitrum, OP on Optimism (8 checks)
- **nfts**: BAYC, Pudgy Penguins and Wrapped CryptoPunks on Ethereum (3 checks)
- **staking**: stETH, rETH, cbETH, wstETH and weETH on Ethereum (5 checks)
- **institutional_stablecoins**: EURCV, USDCV, USDC and BENJI across Ethereum, Solana, XRPL, Stellar and Sui (8 checks, always present; the Solana, XRPL, Stellar and Sui entries carry `evaluated: false` unless the matching wallet is supplied)
- **tokenized_treasuries**: BUIDL, USYC, OUSG, USTB and USDY (16 checks, always present; the USDY on Sui row carries `evaluated: false` unless `suiWallet` is supplied)
- **stablecoin_deposits**: Aave v3 aUSDC/aUSDT, sUSDS, sDAI and the listed Morpho USDC vaults (39 checks)
- **wrapped_bitcoin**: cbBTC, WBTC and tBTC (12 checks)
- **names**: ENS .eth names on Ethereum, Basenames on Base (2 checks)
- **account**: contract code or EIP-7702 delegation present on Ethereum, Base, Arbitrum, Optimism, Polygon (10 checks; two per chain, "Contract code on X" met when bytecode other than the delegation designator is at the wallet address, "EIP-7702 delegation on X" met when the designator is there; exclusive per chain, a plain key reads false on both; which contract is never named)
- **solana**: USDC, EURC, OUSD, PYUSD, USD1, USDG, USDS, BUIDL, USDY, WBTC, cbBTC, tBTC, JitoSOL, mSOL on Solana (14 checks, only when `solanaWallet` provided)
- **xrpl**: RLUSD, USDC, OUSG on XRPL (3 checks, only when `xrplWallet` provided)
- **bitcoin**: native BTC (1 check, only when `bitcoinWallet` provided)
- **tron**: USDT, USD1, WBTC on Tron (3 checks, only when `tronWallet` provided)

Base profile is 155 checks across 27 chains in 10 dimensions. With optional Solana + XRPL + Bitcoin + Tron wallets it reaches up to 176 checks across 29 chains in 14 dimensions. The dimensions come back in a fixed order: the base dimensions as listed above (`stablecoins` through `names`, then `account`), then whichever of `solana`, `xrpl`, `bitcoin`, `tron` were switched on, in that order; the order is the same for every wallet in a batch. `conditionSetVersion` is a dated set id (currently `"2026-10-08"`), the same on every key version, signed with the profile; it names the check list that was run. Readers log it and must not reject on it.

### Credits

- Standard: 3 credits
- With `proof: "merkle"`: 6 credits

---

## JWKS verification

Fetch the public key once and cache it. Every library that supports ES256 / P-256 JWTs understands the JWKS at <https://insumermodel.com/.well-known/jwks.json>.

The JWKS contains an array of keys. **Match on the `kid` from the response you are verifying.** It
publishes five entries over two keys: three IDs over the same P-256 key, `insumer-attest-v2` (attest),
`insumer-trust-v2` (trust), `insumer-attest-v1` (pre-cutover keys, and the commerce discount path),
followed by two RFC 9964 `AKP` entries, `insumer-attest-pq1` and `insumer-trust-pq1`, for the
ML-DSA-65 companion key. Do not pin one and do not take `keys[0]` — the set holds keys of two
types, and position is not a contract. Fail closed when a `kid` does not resolve.

See `../examples/gate-express.ts` for a live-verified Node/TypeScript example using `jose`'s `createRemoteJWKSet`.

## Chain IDs

- **EVM**: standard chain IDs (1 for Ethereum, 8453 for Base, 137 for Polygon, 42161 for Arbitrum, 10 for Optimism, etc.)
- **Solana**: `"solana"`
- **XRPL**: `"xrpl"`
- **Bitcoin**: `"bitcoin"`
- **Tron**: `"tron"`
- **Stellar**: `"stellar"`
- **Sui**: `"sui"`

For the authoritative list of supported chains, GET <https://insumermodel.com/openapi.yaml> and inspect the `ChainId` schema.
