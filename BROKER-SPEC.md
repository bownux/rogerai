# RogerAI Broker - Open Specification (v0 draft)

A **broker** is the only public component. It is **open-source and self-hostable**: anyone can
run one. Providers and users are pure clients that dial out to a broker they trust, and can
**switch brokers at any time** (`rogerai --broker <url>` / `rogerai config set broker <url>`).
This makes RogerAI a federation of brokers, not a single gatekeeper - the protocol is the
product, no broker can lock anyone in.

## What a broker MUST provide (the contract)
A conforming broker implements these over HTTPS (control) + WSS (data tunnel):

### Agent (provider) plane
- `WSS /agent` - provider dials out, authenticates (Ed25519 node key challenge), and holds the
  connection. Carries: `register`, `heartbeat` (≤10s; live availability + load), and inbound
  relayed request streams. Provider advertises `ModelOffer[]` (model, price_in, price_out, ctx,
  max_concurrency, availability/slots). Price changes are rate-limited (§trust).
- The provider exposes **only `/v1/*`** to the broker; the broker never gets shell/file/host.

### Consumer (user) plane - OpenAI-compatible
- `POST /v1/chat/completions` (+ `/completions`, `/embeddings`) - auth by API key; honors
  per-key + per-request constraints (price/tps/latency/region) and routes (see openapi.yaml);
  streams (SSE) responses; returns lineage + cost headers.
- `GET /v1/models` - the union of routable offers for this key.
- `GET /discover` - ranked offers with measured TPS bands, price, region, reputation (powers the
  CLI/site search).
- `GET /v1/usage` / `GET /balance` - receipts + wallet.

### Identity, wallet, payouts
- API keys (scoped: model_allow, price/tps caps, budget). Device-grant login for humans.
- Prepaid credit wallet (Stripe top-ups); per-token debit; **batched Stripe Connect payouts** to
  providers above a threshold; append-only, event-sourced ledger reconciled to the receipt chain.

### Trust & metering (non-negotiable for "conforming")
- **Co-signed, hash-chained UsageReceipts** (node-sig + broker-sig) as the billing truth.
- **Independent token re-count** + **lineage attestation** (TOPLOC/logprob) per VERIFICATION.md.
- **Measured TPS/latency**, price-locked-per-request, price-change rate-limit, bonded
  stake/escrow + reputation, dispute handling.
- **Centralized content pre-filter** (text-only launch; mandatory CSAM screening before dispatch;
  stateless/no-log inference) per the market-research guardrail.

## Trust model & broker reputation
- Each broker publishes its policy (fee %, payout terms, content rules, jurisdiction,
  verification level) at `GET /.well-known/rogerai-broker.json`. The CLI shows it before you join.
- Providers/users pick brokers they trust; a provider may register with **multiple brokers** at
  once (more demand); a user may keep **multiple brokers** configured and let the CLI pick the
  cheapest match across them (cross-broker meta-routing - later).
- Open metrics: uptime, dispute rate, payout reliability → community trust, not a monopoly.

## Switching brokers (the freedom guarantee)
```
rogerai config set broker https://broker.rogerai.net    # the default we run
rogerai config set broker https://friends-broker.example # or a friend's
rogerai brokers add <url> ; rogerai brokers list         # keep several
```
Keys/wallets are per-broker (each broker is its own ledger). Reputation is portable via the
node's Ed25519 identity + signed receipt history a node can present to a new broker.

## Reference implementation
`cmd/rogerai-broker` (this repo) is the reference broker. The **spec is normative**; the impl is
one conforming instance. P0 ships register/discover/relay/wallet/co-signed-receipts (HTTP); P1.5
adds the WSS tunnel; P1/P2 add Stripe, lineage verification, content filter, persistence,
federation metadata. Versioned: brokers advertise `rogerai-broker/0.x` capabilities.
