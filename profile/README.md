# Cambium Protocol

**A transparent, liquid, and verifiable carbon credit market built on Stellar / Soroban.**

Cambium (n.) — the thin layer of growth tissue beneath a tree's bark, responsible for producing new wood and bark each year. It is the literal biological mechanism of carbon fixation. We borrow the name because it describes what this protocol tries to do for carbon markets: create a new, healthy growth layer underneath a system that has grown opaque, illiquid, and hard to trust.

---

## Why this exists

Voluntary and compliance carbon markets today share a common set of structural problems:

- **Double counting** — the same credit can be claimed by more than one buyer because registries don't share a single source of truth.
- **Opacity** — it's hard to trace a credit back to its project, vintage, methodology, and retirement status.
- **Illiquidity** — large lot sizes, OTC deal-making, and slow settlement shut out smaller buyers and keep price discovery weak.
- **Verification (MRV) trust gap** — "did this emissions reduction actually happen?" is still the single biggest source of scandal in the space.
- **No fractionalization** — you generally can't buy 0.1 tons of carbon reduction; you buy in large batches.

Cambium Protocol addresses the **ledger-layer** problems (double counting, opacity, illiquidity, fractionalization) with Soroban smart contracts, and narrows — though does not fully solve — the **measurement-layer** problem (MRV trust) with an oracle + zero-knowledge proof pipeline that lets project data be independently verifiable without always being fully public.

We are explicit about the limits of this approach: **a blockchain cannot verify that a tree was actually planted.** It can only guarantee that once a claim is recorded, it can't be duplicated, silently altered, or double-spent, and that the proof behind the claim is checkable. Where possible, we aim to make claims falsifiable rather than merely "trust us."

---

## Architecture at a glance

```
                                   ┌─────────────────────┐
                                   │      web-app          │
                                   │  (Next.js frontend)    │
                                   └──────────┬───────────┘
                                              │ uses
                                   ┌──────────▼───────────┐
                                   │       sdk-js           │
                                   │ (TS client library)    │
                                   └──────────┬───────────┘
                                              │ calls
                     ┌────────────────────────▼────────────────────────┐
                     │                  contracts                       │
                     │   Registry · CreditToken · Marketplace ·         │
                     │   Retirement · ZK Verifier                       │
                     │            (Soroban / Rust / WASM)               │
                     └───────────▲───────────────────────▲──────────────┘
                                 │ submits proofs          │ verifies proofs
                     ┌───────────┴───────────┐  ┌──────────┴──────────┐
                     │      oracle-node        │  │     zk-circuits      │
                     │ MRV ingestion + signing  │  │  Groth16 / PLONK      │
                     │        service           │  │  circuit definitions  │
                     └─────────────────────────┘  └───────────────────────┘
```

Data flows roughly like this: a project's real-world monitoring data (satellite imagery, IoT sensors, third-party audit reports) is ingested by `oracle-node`, which runs it against methodology logic and produces a zero-knowledge proof using circuits defined in `zk-circuits`. That proof — not the raw data — is submitted on-chain to `contracts`, where it authorizes minting of new credits in the `Registry`/`CreditToken` contracts. Buyers and sellers trade those credits through the `Marketplace` contract, and `Retirement` provides a public, immutable record when a credit is permanently consumed as an offset. `sdk-js` and `web-app` are the developer- and end-user-facing surfaces on top of all of this.

---

## Repositories

| Repo | Purpose | Status |
|---|---|---|
| [`contracts`](../contracts) | Core Soroban smart contracts: credit registry, tokenization, marketplace, retirement, ZK proof verification | **Testnet deployed** — all 5 contracts live; registry, credit-token, marketplace, retirement verified end-to-end |
| [`zk-circuits`](../zk-circuits) | Zero-knowledge circuits for private, verifiable MRV proofs | **Partial** — `reduction_threshold` circuit working (Groth16, ~180k constraints); `group_membership` and `range_proof` deferred |
| [`oracle-node`](../oracle-node) | Off-chain service that ingests MRV data and generates/submits proofs | **Working** — single-signer manual-audit pipeline mints real credits on testnet via Groth16 proof |
| [`sdk-js`](../sdk-js) | TypeScript SDK for wallets, dapps, and integrators | **Working** — registry, credits, marketplace, retirement modules verified against testnet; limit orders and shielded retirement deferred |
| [`web-app`](../web-app) | Public marketplace frontend | **Working** — project explorer, trade page, wallet connection on testnet; portfolio dashboard and public ledger pages deferred |

Each repo has its own detailed README with setup instructions, architecture notes, and contribution guidelines.

---

## Design principles

1. **Public claims, private strategy.** Environmental claims (project methodology, retirement events, aggregate proof validity) are public by default. Trade sizes and buyer identity can be private using ZK, but *what was reduced and whether it was retired* should generally not be.
2. **No silent trust assumptions.** Anywhere the protocol relies on an off-chain actor (an oracle signer, an auditor, a registry bridge), that trust assumption is documented, not hidden.
3. **Fractional by default.** Credits are divisible; there is no reason a small business or individual should be locked out of a 1,000-ton minimum lot.
4. **Boring cryptography where possible.** We prefer well-audited, widely used primitives (Groth16 over BN254, standard SEP-41 token interfaces) over novel cryptography, because carbon market credibility is already fragile.

---

## Getting started

Contracts are already deployed on [Stellar testnet](https://developers.stellar.org/docs/networks). See [`contracts/README.md`](../contracts#deploying) for canonical addresses. To run the full stack:

1. Clone `sdk-js`, `npm install`, and point it at the testnet contract addresses (see the README's Quickstart).
2. Clone `web-app`, `pnpm install`, copy `.env.example` to `.env.local`, and `pnpm run dev`.
3. Connect a testnet wallet (Freighter) and browse projects, trade, or retire credits.
4. To exercise the oracle → proof → mint pipeline, clone `oracle-node` and `zk-circuits` and follow their READMEs.

Each repo's README has copy-pasteable setup steps.

---

## Status

Testnet-deployed core loop working end to end: oracle mint → trade → retire → visible on public ledger.

### What works today

- **Full credit lifecycle on testnet:** oracle ingests MRV data → ZK proof generated → credits minted on-chain → traded via AMM marketplace → retired → retirement visible on public ledger.
- **Wallet-connected frontend:** connect Freighter, browse projects, get quotes, execute swaps against real testnet contracts.
- **One ZK circuit working:** `reduction_threshold` generates and verifies Groth16 proofs (~180k constraints).
- **SDK verified against testnet:** typed TypeScript client for all core contract interactions.

### What's still open

- Limit order book (place, cancel, fill) in marketplace
- Shielded retirement flow (requires `group_membership` ZK circuit + multi-contributor ceremony)
- Second and third ZK circuits (`group_membership`, `range_proof`)
- Multi-oracle federation and satellite/IoT connector support in `oracle-node`
- Portfolio dashboard and public retirement ledger pages in `web-app`
- Independent security audit before mainnet

Contract interfaces, proof formats, and SDK APIs may change without notice until we tag a `v1.0.0` release across the org. Do not use these contracts to custody real value without an independent security audit.

## License

Unless otherwise noted in an individual repo, all Cambium Protocol repositories are released under the [Apache License 2.0](https://www.apache.org/licenses/LICENSE-2.0).

## Community & contributing

- Issues and discussions happen per-repo — file issues in the relevant repository.
- See each repo's `CONTRIBUTING.md` for setup and PR guidelines.
- For security-sensitive reports, do not open a public issue — see the `SECURITY.md` in the `contracts` repo.
