# Work Receipt v0.1

**An open data format and signing scheme for representing the outcome of a
unit of work performed by an AI agent on behalf of (or for) another party.**

A work receipt is a cryptographically signed JSON document that captures:

- who performed the work (a verified AI agent, anchored to a human owner)
- who received the work (an agent, a human, or a service)
- the cryptographic hash of the deliverable
- the settlement (if any) — payment rail, amount, currency, transaction
  reference
- the outcome — `delivered`, `disputed`, `rejected`, `refunded`
- the privacy mode under which the receipt is published — `public`,
  `private` (default), or `zk`

Receipts are designed to be:

- **Portable** — readable by any system that implements this spec
- **Privacy-tiered** — supporting public, private, and zero-knowledge modes
- **Counterparty-anchored** — both parties countersign; neither can fabricate
  unilaterally
- **Verifiable offline** — signature verification requires only the public key
  of each party
- **Versionable** — explicit `version` field; forward-compatible extensibility

The receipt is intentionally **payment-rail-agnostic**. It does not assume
Stripe, x402, ERC-8004, Skyfire, or any other rail. It can reference
settlement events from any of them.

## Status

- **Version:** 0.1 wire-format · v0.6 spec revision (May 2026)
- **Issued:** 4 May 2026
- **Last revision:** 26 May 2026 — v0.6 spec adds `environment`, `issuedByHumanId`, replay link, and the anomaly event taxonomy (all backward-compatible with v0.5 receivers)
- **License (spec text):** [CC BY-SA 4.0](./LICENSE-SPEC)
- **License (reference SDKs):** [Apache-2.0](./LICENSE-CODE)
- **Donation target:** Linux Foundation Agentic AI Foundation, working-group submission ready (see [`LF-EMAIL-DRAFT.md`](./LF-EMAIL-DRAFT.md))

This is a **draft**. Expect additive changes inside v0. Wire format will be
frozen at v1.0; all v0.5+ additions are receiver-tolerant for the v1.0
freeze.

### What's new in v0.6 (May 2026)

- **§8.2.2 `environment` field** — `production` / `staging` / `dev`. Lets producers separate test traffic from real work. Verifiers SHOULD exclude non-production receipts from aggregate trust calculations.
- **§8.2.2 `issuedByHumanId` field** — UUID of the human (resolved from API key) who triggered issuance. Distinct from `buyer.ownerHumanId` — needed for self-issued receipts and multi-human org agents where attribution accuracy matters.
- **§8.2.2 `extensions.replay.url`** — optional URL pointing at a tool-call payload bundle (args + result, capped 10KB per direction). Debug-surface convenience, NOT part of the cryptographic anchor.
- **§8.2.3 Anomaly event taxonomy** — five canonical anomaly kinds (`failure_rate`, `dispute_burst`, `cost_spike`, `trust_score_drop`, `cap_warning`) for out-of-band webhook delivery.
- See [`CONFORMANCE.md` §J / §K / §L](./CONFORMANCE.md) for test vectors.

### What was new in v0.5 (May 2026)

- **§3.4 Runtime fields** (model / tokens / duration / cost) — captures what the agent actually used, in micropound precision, without ever capturing prompt or output content. Zero-content by construction.
- **§3.7 Pacts** — pre-hoc behavioural commitments the seller signs as part of the draft. Pacts are citable as breach evidence in disputes and may carry explicit slash amounts.
- **§3.8 `project` sub-object** — promoted to first-class field in v0.5.6; extended in v0.5.9 with a `provider` slot for cross-LLM continuity (Claude / ChatGPT / Cursor).
- **§3.8.1 Self-issued signatures (v0.5.7)** — server-cosigned receipts where `signatures.buyer` and `signatures.seller` carry the sentinel `"self-issued"` and the cryptographic anchor is the issuer's Ed25519 signature.

## Reference implementations

All bumped to **0.6.0** in lockstep (26 May 2026):

- **TypeScript** — [`@genzagentsio/receipts@0.6.0`](https://www.npmjs.com/package/@genzagentsio/receipts) (npm)
- **Python** — [`genzagents@0.6.0`](https://pypi.org/project/genzagents/) (PyPI)
- **MCP server** — [`@genzagentsio/mcp-server@0.6.8`](https://www.npmjs.com/package/@genzagentsio/mcp-server) — includes `check_duplicates` tool, session replay (`GENZAGENTS_REPLAY=on`), auto-receipt modes
- **LangChain wrapper** — [`@genzagentsio/langchain@0.6.0`](https://www.npmjs.com/package/@genzagentsio/langchain)
- **CrewAI wrapper** — [`@genzagentsio/crewai@0.6.0`](https://www.npmjs.com/package/@genzagentsio/crewai)
- **AutoGen wrapper** — [`@genzagentsio/autogen@0.6.0`](https://www.npmjs.com/package/@genzagentsio/autogen)

The reference SDKs implement the full v0.1 wire format including BLS12-381
aggregate ZK proofs and Ed25519 signing over RFC 8785 (JCS) canonicalised
receipt bodies, plus all v0.6 additive fields.

**Live production deployment:** [genzagents.com](https://genzagents.com) — 22 db tables, 30+ routes, 17 MCP tools, signed compliance evidence packs for SOC 2 / ISO 42001 / EU AI Act / EU CRA, semantic hybrid search, 5-category anomaly watcher, audit log with SOC 2 CC6.1 conformance, SSO + domain-verified auto-membership.

## Quick start (TypeScript)

```ts
import {
  ReceiptBuilder,
  countersignReceipt,
  verifyReceiptSignatures,
  generateKeyPair,
  hashDeliverable,
  publicKeyToBase64,
} from '@genzagentsio/receipts'

const buyer  = generateKeyPair()
const seller = generateKeyPair()

const draft = new ReceiptBuilder()
  .buyer({  type: 'agent', id: 'did:genz:' + publicKeyToBase64(buyer.publicKey) })
  .seller({ type: 'agent', id: 'did:genz:' + publicKeyToBase64(seller.publicKey),
            ownerHumanId: 'human_01HF...' })
  .task({ category: 'code-review',
          deliverableHash: hashDeliverable('PR #42 review notes') })
  .settlement({ amount: '50.00', currency: 'GBP', rail: 'stripe', txRef: 'pi_3O...' })
  .privacy('private')
  // Pacts (added in v0.5) — pre-hoc commitments the seller signs:
  .pact({
    id: 'sla1',
    kind: 'response-sla',
    predicate: 'first review reply within 60 minutes',
    deadline: '2026-05-09T12:00:00Z',
  })
  .pact({
    id: 'cap1',
    kind: 'cost-cap',
    predicate: 'total runtime cost ≤ £5',
    slashMicrogbp: 5_000_000,
  })
  .buildDraft(buyer.privateKey)

const receipt = countersignReceipt(draft, seller.privateKey, 'delivered')

const result = verifyReceiptSignatures(receipt, buyer.publicKey, seller.publicKey)
console.log('valid:', result.valid)  // true
```

## Quick start (Python)

```python
from genzagents import (
    ReceiptBuilder,
    countersign_receipt,
    verify_receipt_signatures,
    generate_keypair,
    hash_deliverable,
    public_key_to_base64,
)

buyer = generate_keypair()
seller = generate_keypair()

draft = (
    ReceiptBuilder(seller_did=f"did:genz:{public_key_to_base64(seller.public_key)}",
                   seller_owner_human_id="human_01HF...")
    .with_buyer(buyer_id=f"did:genz:{public_key_to_base64(buyer.public_key)}")
    .with_task(category="code-review",
               deliverable_hash=hash_deliverable("PR #42 review notes"))
    .with_settlement(amount="50.00", currency="GBP", rail="stripe", tx_ref="pi_3O...")
    .with_privacy("private")
    .build(buyer_private_key=buyer.private_key)
)

receipt = countersign_receipt(draft, seller.private_key)
result = verify_receipt_signatures(receipt, buyer.public_key, seller.public_key)
assert result.valid
```

## Spec sections at a glance

| § | Topic | Notes |
|---|---|---|
| §3 | JSON schema | Top-level structure + sub-objects |
| §4 | Privacy modes | `public` / `private` (default) / `zk` |
| §5 | Signing | Ed25519 over JCS-canonicalised body |
| §6 | Lifecycle | `draft → delivered → (disputed → resolved)` |
| §7 | Disputes | Tier-1 LLM judge, Tier-2 human, Tier-3 panel |
| §8 | Versioning | `version` field; forward-compatible extensions |
| §9 | MCP tools | Standard tool surface for IDE integrations |
| §10 | SDK API | Reference shapes for TS + Python |
| §11 | Examples | Public, private, ZK receipts |
| §12 | Security | Threats and mitigations |
| §13 | Implementation guidance | Minimum viable + reference impls |
| §3.7 | **Pacts** | Pre-hoc behavioural commitments + slash mechanics |
| §3.4 | **Runtime fields** | Model / tokens / duration / cost (zero-content) |
| §3.8 | **Project** | First-class chat / session context tag (v0.5.6) |
| §3.8.1 | **Self-issued sigs** | Server-cosigned receipts (v0.5.7) |
| §8.2.2 | **Environment** (NEW v0.6) | Production / staging / dev separation |
| §8.2.2 | **issuedByHumanId** (NEW v0.6) | Per-receipt human attribution |
| §8.2.3 | **Anomaly events** (NEW v0.6) | 5-kind webhook taxonomy |

The full spec text is in [`SPEC.md`](./SPEC.md).

## Why this exists

The agent commerce stack — Anthropic's Claude, OpenAI's agent SDK, Stripe
Agentic, Visa Agent, Mastercard Agentic, Coinbase x402 — has shipped real
infrastructure in 2026 for AI agents to transact on behalf of humans. What's
*missing* is a verifiable record of work done. Anthropic's own April 2026
"Project Deal" report stated the gap directly:

> "The policy and legal frameworks around AI models that transact on our
> behalf simply don't exist yet."

A standard receipt format gives every regulated buyer (banks, insurers,
healthcare, government) a single answer to:

- Did this AI agent actually do the work it claims it did?
- Who is the legally accountable human behind the agent?
- What's the dispute outcome history?
- Can I verify all of the above without exposing my internal task content?

That's what this spec addresses.

## Contributing

This repository tracks the open spec and reference SDKs. The maintainers are
GenZAgents Ltd (in formation, UK). Co-author credits will be added in v0.2 for
the Moltbook authors who first proposed the receipt-as-primitive thesis (see
acknowledgements in `SPEC.md` §14).

Issues and PRs welcome. Spec changes follow this process:

1. Open an issue describing the change
2. If accepted, a PR against `SPEC.md` with the diff
3. PRs touching reference SDKs must include test coverage
4. Wire-format changes require bumping the `version` field

Code (reference SDKs) is Apache-2.0; spec text is CC BY-SA 4.0.

## Acknowledgements

- u/Rahcd — the Moltbook post that named the receipt-format-as-primitive idea
- u/Arha_AGIRAILS — articulated the synchronous-vs-async payment gap
- u/relayzero — surfaced the "intent receipts" framing
- u/theagora — production-tested verification system that exposed the
  discovery-liability problem
- Anthropic Project Deal final report (April 2026)
- OWASP MCP Top 10 v0.1 (April 2026) — security considerations input

Co-author credits will be finalised during design-partner alpha with explicit
consent.
