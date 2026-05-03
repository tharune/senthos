# Senthos (SCBC Hackathon 2026)

> **Senthos** (SCBC Hackathon 2026): Polymarket-backed structured products on Solana devnet—NLP-filtered curated markets, bundle correlation and CVaR gates before persistence, Next.js 16 + Express + Supabase, Anchor (`senthos_vault`, `senthos_ppn`, `senthos_lending`), and the trained bundle under `senthos-correlation-deliverables/` as read by the TypeScript scorer. Full route and program list: `ARCHITECTURE.md` in the application codebase.

---

## Table of contents

1. [Overview](#overview)
2. [Design principles](#design-principles)
3. [Architecture](#architecture)
4. [Runtime topology](#runtime-topology)
5. [Correlation model and artifacts](#correlation-model-and-artifacts)
6. [Runtime integration (TypeScript)](#runtime-integration-typescript)
7. [On-chain programs](#on-chain-programs)
8. [Backend and data plane](#backend-and-data-plane)
9. [Frontend](#frontend)
10. [Configuration and secrets](#configuration-and-secrets)
11. [Testing and verification](#testing-and-verification)
12. [Technical stack](#technical-stack)
13. [Cloud infrastructure (Akash Network)](#cloud-infrastructure-akash-network)
14. [Repository layout](#repository-layout)
15. [Quick start](#quick-start)
16. [Reference documents](#reference-documents)

---

## Overview

Combined hackathon tree: Polymarket → structured products (baskets, tranches, PPN, lending surfaces), five-stage NLP + quality filter on `GET /api/markets/curated`, bundle correlation and CVaR gates on `POST /api/bundles`. Stack: Next.js 16, Express, Supabase, Anchor on devnet, ML artifacts in `senthos-correlation-deliverables/`. Below: boundaries between offline training (Python) and request-path scoring (TypeScript), plus Akash lease numbers for the GPU training run.

---

## Design principles

- Curated markets: `GET /api/markets/curated` is the single ingress for products that consume listings (`MARKET_FILTER.md`).
- Correlation gate: `POST /api/bundles` applies weight optimization and tail-risk checks against audited envelopes (`MODEL_INTEGRATION.md`).
- Training vs serving: trained model in the `.tar.zst` under `senthos-correlation-deliverables/`; request path uses the TypeScript heuristic in `correlation.ts` (see `MODEL_INTEGRATION.md`), no Python at runtime.
- State placement: native baskets and PPN state live on-chain; tranche and lending mechanics are implemented at the service layer within hackathon scope, with an on-chain roadmap (`PRODUCTS.md`).

---

## Architecture

Five layers:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│  Client: Next.js 16 (App Router), wallet adapter, Solana JSON-RPC             │
├──────────────────────────────────────────────────────────────────────────────┤
│  API: Express — markets, bundles, hedge, PPN, lending, ML manifest, chain   │
├──────────────────────────────────────────────────────────────────────────────┤
│  Chain: Anchor — senthos: vault, PPN, lending (devnet)                        │
├──────────────────────────────────────────────────────────────────────────────┤
│  Data: Supabase; Polymarket Gamma; Anthropic API (filter pipeline)          │
├──────────────────────────────────────────────────────────────────────────────┤
│  ML: senthos-correlation-deliverables — audit, checksums, training bundle     │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## Runtime topology

Standard local development uses three listening ports:

| Process | Port | Role |
|---------|------|------|
| Next.js (`npm run dev`) | 3000 | Public site `/` and authenticated `/app/*` |
| Express (`npm run dev` in `backend/`) | 3001 | REST API, Polymarket access, bundle creation, Solana transaction assembly |
| Monitor (same Node process as backend) | 3002 | Process monitor JSON (`/data`) |

Scheduled price refresh runs inside the backend (`ARCHITECTURE.md`, §3).

---

## Correlation model and artifacts

Objective: binary classification on pairs of prediction-market contracts: whether absolute return correlation reaches at least 0.6 (`abs_corr_target >= 0.6`), with decision threshold 0.70.

Reported metrics (`senthos-correlation-deliverables/model_metrics_step12.json`):

| Quantity | Value |
|----------|-------|
| Labeled rows | 410,777 |
| Train / validation split | 328,623 / 82,154 |
| Engineered features | 20 |
| Precision | 0.9432 |
| Recall | 0.8831 |
| ROC-AUC | 0.9326 |
| Regression RMSE | 0.0836 |

Walk-forward summary (`final_summary_step19.json`): mean lift +4.82% versus a naive baseline; p = 3.10 × 10⁻⁵; audit checklist completed.

Monte Carlo (`monte_carlo_step14.json`): 50-leg basket, 20,000 paths, 7-day horizon, σ_daily = 4%; VaR and CVaR recorded for the audited risk envelope.

Deliverables directory (`senthos-correlation-deliverables/`): step-level metrics (JSON/Markdown), `SHA256SUMS.txt`, manifests, and `senthos-correlation-production-20260417T071458Z.tar.zst` (Python training bundle). Paths embedded in the archive reference `/root/senthos-correlation/...` on the training host.

---

## Runtime integration (TypeScript)

Implementation: `backend/src/services/correlation.ts`.

| Export | Role |
|--------|------|
| `loadArtifacts()` | Lazy, memoized load of audit JSON; exposes thresholds, Monte Carlo envelope, walk-forward flags |
| `scoreLegPair(a, b)` | Noisy-OR combination of text, tag, and temporal similarity signals (`MODEL_INTEGRATION.md`, §3.2) |
| `optimizeWeights`, `assessBasketRisk`, `getModelManifest` | Basket weights, CVaR ratio gate, manifest for `/api/ml/*` |

The HTTP tier does not invoke Python during request handling.

---

## On-chain programs

| Program | Function |
|---------|----------|
| `senthos_vault` | Native basket vault: deposit, redeem, resolve/finalize, fees |
| `senthos_ppn` | Principal-protected notes; mock yield adapter |
| `senthos_lending` | Lending pool: supply, borrow, repay, withdraw |

Program IDs, PDAs, and instruction matrices: `ONCHAIN.md`, `ARCHITECTURE.md`, and `programs/*`.

---

## Backend and data plane

- Framework: Express 4, TypeScript, Zod, `@coral-xyz/anchor`, Supabase client, `express-rate-limit`, `node-cron`.
- HTTP surface: `backend/src/routes/` — bundles, markets, hedge, PPN, lending, alerts, diagnostics, on-chain status, ML manifest. Enumeration: `ARCHITECTURE.md`, §§5–6.
- Services: `backend/src/services/` — correlation, NLP/market filter, pricing, Polymarket adapters, Solana bridge.

---

## Frontend

- Stack: Next.js 16.2.4, React 19, Turbopack (dev), Tailwind CSS v4.
- Wallet: `@solana/wallet-adapter-*`; shell under `app/app/`.
- Routes: marketing root `app/page.tsx`; product routes under `app/app/*` (`ARCHITECTURE.md`, §5).

---

## Configuration and secrets

- Root: `.env.local` (templates: `.env.local.testnet.example`, devnet variant). Cluster switching: `SWITCH-CLUSTER.command`.
- Backend: `backend/.env` from `.env.testnet.example` or `.env.devnet.example` — Supabase, RPC endpoints, program IDs, optional `DISABLE_RATE_LIMIT`, Anthropic credentials for the filter stages.
- Variable index: `CREDENTIALS.md`.

---

## Testing and verification

- Programs: `tests/` — Anchor integration (Mocha/Chai).
- API / NLP: `backend/src/scripts/` — e.g. `test-api.ts`, `nlp-probe.ts` (26 checks per `ARCHITECTURE.md`).
- Liveness: `GET /api/health`, `GET /api/onchain/status`, `GET http://localhost:3002/data`.

---

## Technical stack

| Layer | Technology |
|-------|------------|
| Frontend | Next.js 16.2.4, React 19.2.4, TypeScript 5, Tailwind 4 |
| Backend | Express 4.21, TypeScript 5.9, tsx |
| Chain | `@solana/web3.js`, `@coral-xyz/anchor` 0.30.1, SPL Token |
| Programs | Rust, Anchor (`senthos_vault`, `senthos_ppn`, `senthos_lending`) |
| Persistence / feeds | Supabase JS v2, Polymarket Gamma API |
| NLP | Anthropic SDK |
| ML (offline) | Python, scikit-learn, LightGBM |
| ML (online) | TypeScript correlation service (heuristic + frozen audit metrics) |
| Training compute | Akash Network — GPU lease for correlation pipeline |

---

## Cloud infrastructure (Akash Network)

Correlation training (features, LightGBM/sklearn on 410,777 labeled rows per audit JSON, walk-forward, Monte Carlo) ran on [Akash Network](https://akash.network): SDL v2 lease, settlement in AKT.

Training used only the leased GPU node; repo artifacts under `senthos-correlation-deliverables/` were built on that host (`/root/senthos-correlation/...` paths inside the tarball).

### Deployment (training phase)

| Phase | Nodes | Workload | Wall-clock duration |
|-------|-------|----------|---------------------|
| Training | 1 × GPU | Python correlation pipeline and audit scripts | ~9.5 h |

### Resource profile (hackathon lease)

Declared resources for the hackathon lease (confirm against your saved SDL / provider console).

| Resource | Declared value |
|----------|----------------|
| CPU | 32.0 `cpu.units` (32 threads under SDL v2 resource semantics) |
| Memory | 100 GB |
| Storage | 400 GiB |
| GPU | 1× NVIDIA H100 (`vendor.nvidia.model: h100`) |
| Replicas | 1 |

### Cost

| Line item | USD | Basis |
|-----------|-----|--------|
| GPU training lease | 34 | Lease-related AKT for this workload, converted to U.S. dollars at hackathon close-out (rounded). Wallet or block explorer history is the reconciliation source. |

Settlement in AKT via the Akash marketplace. The amount covers the ~9.5 h training window; it excludes unrelated leases, local hardware, and third-party API egress.

### SDL excerpt

Same shape as the training deployment (adjust `image`, env, bids to match your lease).

```yaml
---
version: "2.0"

services:
  senthos-train:
    image: your-registry/senthos-correlation-train:latest
    env:
      - RUN_ID=senthos-2026-correlation
    expose:
      - port: 8080
        as: 80
        to:
          - global: false

profiles:
  compute:
    trainer:
      resources:
        cpu:
          units: 32.0
        memory:
          size: 100Gi
        storage:
          - size: 400Gi
        gpu:
          units: 1
          attributes:
            vendor:
              nvidia:
                - model: h100
  placement:
    akash:
      pricing:
        trainer:
          denom: uakt
          amount: 10000

deployment:
  senthos-train:
    akash:
      profile: trainer
      count: 1
```

Pin images by digest in production; keep keys out of SDL; `amount` is a bid cap — settled rate is whatever the provider accepted.

---

## Repository layout

High-level mirror of the combined tree (paths relative to that codebase):

```
app/                       # Next.js App Router
backend/src/               # Express routes, services, Solana, IDL
programs/                  # senthos_vault, senthos_ppn, senthos_lending
senthos-correlation-deliverables/
scripts/  migrations/  tests/
ARCHITECTURE.md, MODEL_INTEGRATION.md, MARKET_FILTER.md, …
```

---

## Quick start

From the application repository root (clone/runbook internal to your team):

```bash
cp .env.local.testnet.example .env.local
cp backend/.env.testnet.example backend/.env
npm install && (cd backend && npm install)
# Terminal 1: (cd backend && npm run dev)
# Terminal 2: npm run dev
```

Devnet: use `SWITCH-CLUSTER.command` and the env templates documented beside it in that tree.

---

## Reference documents

| File | Contents |
|------|----------|
| `ARCHITECTURE.md` | Topology, routes, mathematics, change recipes |
| `MODEL_INTEGRATION.md` | Model contract and HTTP behavior |
| `MARKET_FILTER.md` | NLP filter stages |
| `PRODUCTS.md` | Product taxonomy and roadmap |
| `ONCHAIN.md`, `ONCHAIN_DESIGN.md` | Program design |
| `CREDENTIALS.md` | Environment variables |
| `AUDIT.md`, `SECURITY.md` | Security notes |
| `senthos-correlation-deliverables/README.md` | Artifact inventory |
