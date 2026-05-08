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

- **Version:** 0.1 (Draft)
- **Issued:** 4 May 2026
- **Last revision:** 8 May 2026 (added §3.7 Pacts — pre-hoc behavioural commitments)
- **License (spec text):** [CC BY-SA 4.0](./LICENSE-SPEC)
- **License (reference SDKs):** [Apache-2.0](./LICENSE-CODE)
- **Donation target:** Linux Foundation Agentic AI Foundation, working-group
  submission pending Q3 2026 (after v1.0 lands with three+ independent
  implementations in the wild).

This is a **draft**. Expect breaking changes inside v0. Wire format will be
frozen at v1.0.

### What's new in v0.1 (May 2026 revisions)

- **§3.4 Runtime fields** (model / tokens / duration / cost) — captures what
  the agent actually used, in micropound precision, without ever capturing
  prompt or output content. Zero-content by construction.
- **§3.7 Pacts** — pre-hoc behavioural commitments the seller signs as part
  of the draft. Pacts are citable as breach evidence in disputes and may
  carry explicit slash amounts. Inspired by procurement contract patterns.
- **§3.8 Lifecycle extensions** — `agent_transferred`, `previous_dids[]` to
  support agent ownership rotation while preserving receipt continuity.

## Reference implementations

- **TypeScript** — [`@genzagentsio/receipts`](https://www.npmjs.com/package/@genzagentsio/receipts) (npm)
- **Python** — [`genzagents`](https://pypi.org/project/genzagents/) (PyPI)
- **MCP server** — [`@genzagents/mcp-server`](https://www.npmjs.com/package/@genzagents/mcp-server) (npm)

The reference SDKs implement the full v0.1 spec including BLS12-381 aggregate
ZK proofs and Ed25519 signing over RFC 8785 (JCS) canonicalised receipt
bodies.

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
| §3.7 | **Pacts** (NEW) | Pre-hoc behavioural commitments + slash mechanics |
| §3.4 | **Runtime fields** | Model / tokens / duration / cost (zero-content) |

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
