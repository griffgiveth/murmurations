# Murmurations

A token-gated **governance signaling** app. A community runs
"murmurs" (ballots); eligible wallets allocate points across the
voting options ("directions") on a murmur, and every allocation is recorded as a
wallet-signed, publicly verifiable vote. No funds are ever custodied or
moved — the app only collects and tallies signed ballots.

Production: <https://murmur.thedao.fund/>

---

## How it works (the 60-second version)

1. An **admin** creates a *round* (a "murmur"): a question, a set of options
   ("directions"), a voting mode, a point **budget**, an eligibility token,
   and open/close times.
2. A voter connects a wallet. The app checks whether that wallet is
   **eligible** for the round (holds the round's required badge/token).
3. The voter allocates points across the directions, up to the round budget.
   In **quadratic** mode the cost of putting *n* points on one direction is
   *n²*, so spreading support is cheaper than concentrating it.
4. The voter signs an **EIP-712 ballot** with their wallet. The signed ballot
   is the vote — the server stores it and anyone can re-verify the signature.
5. The live **tally** is derived from the stored ballots. Re-signing replaces
   a voter's previous ballot.

There is no on-chain transaction to vote and nothing is spent — a ballot is a
signed message, not a transaction.

---

## Architecture

```
Browser (React + wagmi/viem + RainbowKit)
   │   signs EIP-712 ballots / admin actions / option submissions
   ▼
/api  (Fastify, Node)  ──►  Postgres   (proposals + ballots)
   │   verifies every signature server-side
   │   reads token/badge balances over RPC to check eligibility
   ▼
Ethereum / L2 RPC (read-only: badge balanceOf, contract reads)

Optional on-chain: a TallyCommit contract can anchor a finalized tally;
ETHSecurity badge ERC-721s are the eligibility gate.
```

- **Frontend** — a single-page React app. Almost all UI lives in
  [`src/app.jsx`](src/app.jsx). Wallet connection is wagmi + RainbowKit;
  signing and contract reads use viem. Eligibility helpers are in
  [`src/eligibility.ts`](src/eligibility.ts); the wallet gate and
  role/badge derivation are in [`src/main.tsx`](src/main.tsx).
- **Backend** — a Fastify server in [`server/api.mjs`](server/api.mjs) that
  serves the public read API and verifies all signed writes. Storage is
  Postgres via [`server/db.mjs`](server/db.mjs).
- **Contracts** — in [`contracts/`](contracts/): an ERC-721 badge and a
  tally-commit contract. The eligibility badges are deployed ERC-721s; the
  app only ever *reads* `balanceOf`.

---

## Security model (start here for a review)

This app holds no funds, but it does make **trust decisions** off signatures
and token balances. The parts worth scrutiny:

- **Signed ballots (EIP-712).** A user's vote is an EIP-712 `Ballot` signed by the
  voter. The domain is `murmurations` (v1). The server recovers the signer
  and stores `{ballot, signature, signedAt}`; the public ballots endpoint
  lets anyone re-verify. See `BALLOT_TYPES` / `castVote` in
  [`src/votingApi.ts`](src/votingApi.ts) and the `/vote` handler in
  `server/api.mjs`.
- **Eligibility is enforced on BOTH sides.** The client decides whether to
  *show* the voting UI, but the **server independently re-checks** eligibility
  (and round open/close times) before accepting a ballot — the client gate is
  convenience, not security. Per-round eligibility (`canVoteInRound`) is the
  single source of truth on the client; the server reads badge `balanceOf`
  over RPC.
- **Two badge identities.** A round can require the public ETHSecurity badge
  or the incognito badge. A wallet holding the wrong one is shown exactly
  which address to connect; it cannot vote on a round it isn't eligible for.
- **Signed admin & submission actions.** Creating/deleting proposals,
  submitting a direction, and deleting a direction are all EIP-712-signed and
  verified server-side. Option submission carries the submitter's address;
  option deletion is authorized for an **admin or the option's original
  submitter** (signature-verified, so it can't be spoofed).
- **Soft-delete preserves signatures.** Removing a direction marks it
  `deleted` rather than erasing it, so already-cast ballots stay
  signature-valid; the tally and budget math skip deleted options, and voters
  recover the points they had on them.
- **Budget integrity.** Allocations are clamped to the round budget on both
  client and server; duplicate directions are rejected
  (case/whitespace-insensitive) so the same option can't be created twice.

Known trust assumptions: admins are an allowlist; eligibility trusts the
configured badge contracts; RPC reads trust the configured providers.

---

## Repository map

| Path | What it is |
| --- | --- |
| `src/app.jsx` | The entire SPA UI (rounds list, voting screen, direction detail, submit, admin). Large on purpose — it's the canonical UI source. |
| `src/main.tsx` | App entry: wallet provider, the wallet gate, role/badge derivation. |
| `src/votingApi.ts` | Client API wrapper + all EIP-712 type definitions (the signing schema). |
| `src/eligibility.ts` | Token registry + eligibility helpers. |
| `server/api.mjs` | Fastify API: proposals, options, ballots, signature verification, eligibility. |
| `server/db.mjs` | Postgres storage layer. |
| `contracts/` | Badge ERC-721 + tally-commit Solidity. |
| `scripts/` | Deploy/mint/migration utilities. |
| `tests/` | Vitest suite — `tests/server` (API) and `tests/client` (UI + helpers). |
| `public/` | Static assets. |
| `API.md` | Public read-API reference. |
| `TESTING.md` | How the test suite is structured and run. |
| `DEPLOYMENT_GHCR.md` | How production builds/deploys (GHCR images → VPS). |

> Note: `source-jsx/` and `build-app.mjs` are a legacy build path. The live
> app is built directly from `src/` with `vite build`; edit `src/app.jsx`,
> not `source-jsx/`.

---

## Running it locally

Requirements: Node 20+, pnpm, Docker (for Postgres).

```bash
pnpm install
pnpm db:up        # starts Postgres in Docker
pnpm server       # API on http://127.0.0.1:7101
pnpm dev          # web on http://127.0.0.1:7100 (proxies /api → 7101)
```

Open <http://127.0.0.1:7100>. Configuration is via `.env` (see
[`.env.example`](.env.example)) — `DATABASE_URL`, RPC URLs, and the admin
allowlist.

### Tests

```bash
pnpm test            # full suite (server + client)
pnpm test:coverage   # with coverage
```

See [TESTING.md](TESTING.md) for structure and mocking notes.

---

## Naming: `thedaolog` vs `murmurations`

The product is **Murmurations**. The name `thedaolog` survives only where a
rename would be an infra migration rather than a cosmetic change, and is left
intentionally:

- the **Postgres database name** (`thedaolog`) — renaming it would break the
  connection to live data;
- the **GHCR image names** and the **deploy workflow** (`.github/workflows/`) —
  deploy-critical, and changed only as part of a coordinated infra update.


