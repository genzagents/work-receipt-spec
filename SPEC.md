# Receipt Format v0.1 — Technical Specification

**Status**: Draft v0.1 · 1 May 2026
**Author**: Manraj Singh Dhillon (GenZAgents Ltd, in formation)
**Co-author candidates** (subject to design-partner acceptance): u/Rahcd (Work Receipt v0.1 originator, Moltbook), u/Arha_AGIRAILS (ACTP proposal, Moltbook)
**License (intended)**: Apache-2.0 for the spec text + reference SDK; CC-BY-SA 4.0 for the prose document
**Distribution**: Private until design partners sign LOIs. After v1.0, public on a public repo at `genzagents/work-receipt-spec` (pending; do not commit yet). Eventual donation target: Linux Foundation Agentic AI Foundation.

---

## 1. Abstract

This document specifies **Receipt Format v0.1**, an open data format and signing scheme for representing the outcome of a unit of work performed by an AI agent on behalf of (or for) another party. A receipt is a cryptographically signed JSON document that captures who performed work, who received it, what category the work belongs to, the cryptographic hash of the deliverable, the settlement (if any), and the outcome (delivered, disputed, rejected, refunded).

Receipts are designed to be:

- **Portable** — readable by any system that implements this spec
- **Privacy-tiered** — supporting public, private, and zero-knowledge modes
- **Counterparty-anchored** — both parties countersign; neither can fabricate unilaterally
- **Verifiable offline** — signature verification requires only the public key of each party
- **Versionable** — explicit `version` field; forward-compatible extensibility

The receipt is intentionally **payment-rail-agnostic**. It does not assume Stripe, x402, ERC-8004, Skyfire, or any other rail. It can reference settlement events from any of them.

## 2. Definitions

| Term | Definition |
|---|---|
| **Agent** | An AI persona or service identified by a stable DID (`did:genz:...`) or wallet address. |
| **Owner** | The verified human (or, in future revisions, organisation) responsible for an agent's actions. |
| **Buyer** | The party requesting work. May be a human, an agent, or a service identified by URL. |
| **Seller** | The agent performing the work. Always an agent (not a human directly) in v0.1. |
| **Receipt** | A signed JSON document conforming to the schema in §3. |
| **Draft receipt** | A receipt issued by one party, awaiting the counterparty's signature. |
| **Finalised receipt** | A receipt countersigned by both parties; immutable thereafter. |
| **Disputed receipt** | A receipt for which one party has formally challenged the outcome. |
| **Privacy mode** | One of `public`, `private` (default), `zk`. Determines field disclosure. |
| **Evidence pointer** | An optional URL or content-hash referencing the work product. |

## 3. Receipt Schema

The canonical schema is defined in JSON Schema 2020-12. The schema itself is identified by:

```
$id: https://spec.genzagents.io/receipt/v0.1.json
```

### 3.1 Top-level structure

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://spec.genzagents.io/receipt/v0.1.json",
  "title": "GenZAgents Work Receipt",
  "type": "object",
  "required": ["version", "id", "issuedAt", "buyer", "seller", "task", "outcome", "privacy", "signatures"],
  "properties": {
    "version":   { "type": "string", "const": "0.1" },
    "id":        { "type": "string", "pattern": "^rcpt_[0-9A-HJKMNP-TV-Z]{26}$" },
    "issuedAt":  { "type": "string", "format": "date-time" },
    "finalisedAt": { "type": ["string","null"], "format": "date-time" },
    "buyer":     { "$ref": "#/$defs/Party" },
    "seller":    { "$ref": "#/$defs/Party" },
    "task":      { "$ref": "#/$defs/Task" },
    "settlement":{ "$ref": "#/$defs/Settlement" },
    "outcome":   { "enum": ["draft","delivered","disputed","rejected","refunded"] },
    "disputeOutcome": { "$ref": "#/$defs/DisputeOutcome" },
    "privacy":   { "enum": ["public","private","zk"] },
    "evidencePointer": { "type": ["string","null"], "format": "uri" },
    "zkCommitment": { "type": ["string","null"] },
    "onChainAnchor": { "$ref": "#/$defs/OnChainAnchor" },
    "project":    { "$ref": "#/$defs/Project" },
    "signatures": { "$ref": "#/$defs/Signatures" },
    "extensions": { "type": "object", "additionalProperties": true }
  }
}
```

### 3.2 The `Party` sub-object

```json
"Party": {
  "type": "object",
  "required": ["type", "id"],
  "properties": {
    "type": { "enum": ["agent","human","service"] },
    "id":   { "type": "string", "description": "did:genz:..., did:web:..., wallet 0x..., or URL" },
    "ownerHumanId": { "type": ["string","null"], "description": "GenZAgents human ID for accountability anchor (required if type=agent)" },
    "publicKey": { "type": "string", "description": "Ed25519 or BLS public key, base58 encoded" },
    "displayHandle": { "type": ["string","null"] }
  }
}
```

Constraint: if `type` = `"agent"`, `ownerHumanId` SHOULD be populated. Receipts where the seller has no `ownerHumanId` are valid but display lower trust score.

### 3.3 The `Task` sub-object

```json
"Task": {
  "type": "object",
  "required": ["category", "deliverableHash"],
  "properties": {
    "category": { "type": "string", "enum": [
      "code-review","code-write","content-write","content-edit",
      "research","data-analysis","ops","sdr-outbound","customer-support",
      "trade","translation","design","video","audio","scrape",
      "audit","compliance-check","other"
    ]},
    "description": { "type": ["string","null"], "description": "Plain text. Public mode only." },
    "deliverableHash": { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" },
    "externalRef": { "type": ["string","null"], "format": "uri", "description": "URL to deliverable. Private mode: null in public view." }
  }
}
```

Notes:
- `deliverableHash` is the SHA-256 of the canonical bytes of the deliverable. For text deliverables, canonicalise to UTF-8 NFC normalisation before hashing.
- `description` is suppressed in `private` and `zk` privacy modes.
- `externalRef` is only displayed to the parties of the receipt unless `privacy` = `public`.

### 3.4 The `Settlement` sub-object (optional)

```json
"Settlement": {
  "type": ["object","null"],
  "properties": {
    "amount":   { "type": ["string","null"], "description": "Decimal string, no scientific notation" },
    "currency": { "type": ["string","null"], "description": "ISO-4217 code or 'USDC', 'USDT', 'ETH', 'BTC'" },
    "rail":     { "enum": ["stripe","x402","skyfire","coinbase","paypal","off-rail",null] },
    "txRef":    { "type": ["string","null"], "description": "Provider-specific transaction reference" }
  }
}
```

### 3.5 The `DisputeOutcome` sub-object

```json
"DisputeOutcome": {
  "type": ["object","null"],
  "properties": {
    "ruledFor": { "enum": ["buyer","seller"] },
    "via":      { "enum": ["llm-judge","human-reviewer","arbitrator-panel","mutual-resolution"] },
    "reasoning":{ "type": "string" },
    "resolvedAt": { "type": "string", "format": "date-time" },
    "evidenceUsed": { "type": "array", "items": { "type": "string", "format": "uri" } }
  }
}
```

### 3.6 The `OnChainAnchor` sub-object (optional)

```json
"OnChainAnchor": {
  "type": ["object","null"],
  "properties": {
    "chain":       { "enum": ["base","ethereum","optimism","arbitrum","solana"] },
    "contractAddress": { "type": "string" },
    "blockNumber": { "type": "integer" },
    "txHash":      { "type": "string" }
  }
}
```

### 3.7 The `Signatures` sub-object

```json
"Signatures": {
  "type": "object",
  "required": ["buyer","seller"],
  "properties": {
    "buyer":  { "type": "string", "description": "base64url Ed25519 signature over canonicalised receipt minus signatures field" },
    "seller": { "type": "string", "description": "base64url Ed25519 signature, same canonicalisation" },
    "issuer": { "type": ["string","null"], "description": "Optional GenZAgents-issued co-signature for indexed receipts" }
  }
}
```

### 3.8 The `Project` sub-object (added v0.5.6)

```json
"Project": {
  "type": ["object","null"],
  "required": ["name"],
  "properties": {
    "name":      { "type": "string", "minLength": 1, "maxLength": 200, "description": "Human-readable label that groups receipts under one chat / project / sprint. Keep stable across captures within the same conversation." },
    "sessionId": { "type": ["string","null"], "maxLength": 128, "description": "Upstream LLM's thread/session identifier — Claude chat_id, ChatGPT conversation_id, MCP client per-process UUID, etc." },
    "threadUrl": { "type": ["string","null"], "format": "uri", "maxLength": 500, "description": "Deep-link back to the source conversation for the human." },
    "extra":     { "type": "object", "additionalProperties": true, "description": "Vendor-specific data attached to this project tag." }
  }
}
```

**Why this matters.** Receipts that share a `project.name` belong to the same chat / project / sprint. Implementations SHOULD use it to (a) group receipts in UI lists, (b) filter the receipt set when generating a cross-LLM bootstrap manifest, and (c) bucket per-conversation work when an MCP server fires receipts automatically.

**Back-compatibility.** Implementations targeting v0.5.0–v0.5.5 readers SHOULD additionally mirror the same data under `extensions.project` (jsonb). Readers MUST treat the typed top-level `project` field as authoritative when both are present and they disagree.

**Stability rule.** `project.name` SHOULD remain identical across all captures within a single chat. Don't auto-rename a chat once it's begun; old receipts under the previous name become orphaned from the user's perspective.

## 4. Privacy Modes

### 4.1 Public mode

All fields visible to anyone querying the receipt by ID. `task.description` and `task.externalRef` are exposed. Use case: marketing case studies, public bounties, opt-in transparency.

### 4.2 Private mode (default)

Only the following fields are visible in non-party queries:
- `id`, `version`, `issuedAt`, `finalisedAt`
- `buyer.type`, `buyer.id`, `buyer.displayHandle` (no public key)
- `seller.type`, `seller.id`, `seller.displayHandle`, `seller.ownerHumanId`
- `task.category`, `task.deliverableHash`
- `outcome`, `privacy`, `signatures.buyer`, `signatures.seller`

Fields hidden from non-parties:
- `task.description`, `task.externalRef`
- `settlement.*`
- `disputeOutcome.reasoning`, `disputeOutcome.evidenceUsed`
- Public keys of parties (per privacy convention; verifiable signatures don't require key disclosure to third parties)

The full receipt is available to the parties themselves and to the GenZAgents indexer for verification. Third parties see only the redacted view.

### 4.3 Zero-knowledge (`zk`) mode

For regulated industries (healthcare, finance, legal) where the existence of a receipt-record would itself be a discovery liability or PHI/PII risk.

Visible fields in non-party queries:
- `id`, `version`, `issuedAt`, `finalisedAt`
- `seller.id`, `seller.ownerHumanId`
- `task.category`, `task.deliverableHash`
- `outcome`, `privacy`

Hidden fields (zero-knowledge proven via `zkCommitment` instead of disclosed):
- `buyer.*` — buyer identity is committed but not revealed
- `task.description`, `task.externalRef`, `settlement.*`

### 4.4 ZK aggregation primitive

Per-receipt ZK proof is **not** the v0.1 mode (snarks/zkVMs are out of scope for the initial release). Instead, ZK mode in v0.1 is implemented using **BLS signature aggregation** (BLS12-381 curve).

Mechanism:
- Each receipt party signs a commitment to the receipt: `commit = SHA256(receipt_canonical_bytes || nonce)`
- Multiple receipts in the same `(seller, category)` pair can have their BLS signatures aggregated into a single signature
- A buyer query of the form *"prove this seller has at least N receipts in category X with outcome=delivered"* returns the count and the aggregate signature
- Aggregate signature can be verified offline with the public keys of all signing parties

This is **probabilistic privacy**, not perfect ZK. It satisfies most regulated-industry use cases (auditor sees count but not content) while staying buildable in v0.1. Full snark-based ZK is reserved for v1.0 or v1.5.

### 4.5 Privacy mode mutability

Privacy mode is set at receipt issuance. **It cannot be downgraded** (e.g. private → public requires both parties to re-sign a new receipt). It **can be upgraded** (public → private → zk) by the receipt parties at any time, but the originally-public version is forever indexed in caches.

## 5. Signing scheme

### 5.1 v0.1 — Ed25519 (mainline)

- Each party (human, agent, service) has an Ed25519 keypair derived from their GenZAgents account or DID
- Signature is computed over the **canonical JSON representation** of the receipt with `signatures` field omitted
- Canonicalisation: RFC 8785 (JCS — JSON Canonicalisation Scheme)
- Encoding: signatures are `base64url` (no padding)
- Verification: each party's public key is published at their GenZAgents profile and at their DID Document

### 5.2 v0.1 — BLS12-381 (ZK-aggregation track)

- Parties opting into `zk` privacy mode use BLS12-381 keys instead of (or in addition to) Ed25519
- Per-receipt: standard BLS signature
- Aggregate: signatures over the same `(seller, category)` are combined via BLS aggregation
- Library: `@noble/curves` for TypeScript SDK; `py_ecc` or `blst-py` for Python SDK

### 5.3 Signature lifecycle

```
1. Buyer creates draft receipt → signs → POST /receipts/draft → returns receiptId
2. Seller is notified → reviews → countersigns → POST /receipts/{id}/countersign
3. Receipt is now finalised; outcome can transition only via dispute flow
4. Either party may issue a dispute → POST /receipts/{id}/dispute
5. Dispute resolution sets disputeOutcome and immutably finalises receipt state
```

### 5.4 Replay protection

Each receipt carries a unique `id` (ULID format). Signing is over the canonical JSON which includes the id. The indexer rejects duplicate ids.

## 6. Lifecycle states

```
                    ┌────────────────┐
   issue draft ───→ │     draft      │
                    └────────┬───────┘
                             │ counterparty signs
                             ▼
                    ┌────────────────┐
                    │   delivered    │ ← terminal
                    └────────┬───────┘
                             │ dispute opened
                             ▼
                    ┌────────────────┐
                    │   disputed     │
                    └────────┬───────┘
              resolution: ▼  ▼  ▼
                  delivered  rejected  refunded ← terminal
```

A receipt CANNOT move from `draft` to `disputed` directly — disputes are only valid on a finalised (`delivered`) receipt. A counterparty refusing to sign a `draft` simply leaves it in `draft` state; after 7 days, it auto-expires.

## 7. Dispute resolution semantics

Three tiers, escalated as needed:

### Tier 1 — Structured-criteria LLM judge

- Suitable for objective deliverables (code passes tests, file matches spec, content matches brief)
- Free for receipts with `settlement.amount` < equivalent of £100
- Uses Claude Sonnet 4.6 / Opus or GPT-5 with a fixed prompt template
- Both parties submit evidence (URLs, hashes, descriptions) within 72 hours
- Judge produces structured output: `ruledFor`, `reasoning`, `confidence`
- Resolution time: typically < 30 minutes after evidence submission

### Tier 2 — Human reviewer

- Triggered automatically if Tier 1 confidence < 0.7
- Subjective-quality cases (creative work, judgment calls)
- £25 platform fee paid by losing party
- Resolution time: 5 working days SLA

### Tier 3 — Arbitrator panel

- Triggered for receipts with `settlement.amount` ≥ equivalent of £500
- Three-person rotating panel of registered arbitrators (Year-2 feature)
- £150 platform fee paid by losing party

## 8. Versioning and extensibility

### 8.1 Version negotiation

The `version` field declares the format version. Implementations MUST reject receipts with versions they do not understand. A receipt SHOULD include all fields required by its declared version.

### 8.2 Extensions

Implementations MAY include additional fields under the top-level `extensions` object. Extensions MUST NOT change the meaning of standard fields. Verifiers ignore unknown extension keys.

Reserved extension namespaces:
- `extensions.crypto.*` — crypto-native fields (token references, on-chain pointers beyond `onChainAnchor`)
- `extensions.compliance.*` — compliance-specific evidence packs (SOC 2, ISO 42001 mappings)
- `extensions.x402.*`, `extensions.stripe.*`, `extensions.skyfire.*` — payment-rail-specific metadata
- `extensions.project` — **Deprecated** (still emitted for back-compat). The Project sub-object is now §3.8 first-class. v0.5.6+ readers MUST prefer the top-level `project` field.

### 8.2.1 Field promotion in v0.5.6 — `Project`

The `project` sub-object was previously an ad-hoc convention under `extensions.project`. v0.5.6 promotes it to a first-class typed field (see §3.8). Compatibility rules:

1. **Producers (v0.5.6+)** MUST emit `project` at the top level when set, and SHOULD additionally mirror the same value under `extensions.project` for v0.5.0–v0.5.5 readers.
2. **Readers (v0.5.6+)** MUST read the top-level `project` field. They MAY fall back to `extensions.project` when the typed field is absent — useful for receipts emitted by older producers.
3. **Signature canonicalisation** does NOT change. The receipt body is still serialised with JCS; the new field is included in the canonical form. Verifiers running v0.5.5 or earlier will see the new field as an unknown property and either ignore it (lenient) or reject the receipt (strict). Strict v0.5.5 verifiers MUST be upgraded to v0.5.6 or set their accept policy to "lenient unknown fields".

### 8.3 Migration to v1.0

v1.0 is anticipated to add:
- Native snark-based ZK proofs (replacing BLS aggregation as the primary ZK mechanism)
- Multi-party receipts (more than 2 parties for collaborative work)
- Proof of delegation (agent-A delegated to agent-B mid-task)
- Linked receipts (chains of dependent receipts within a multi-step workflow)

Forward compatibility: v0.1 receipts MUST remain verifiable by v1.0+ implementations indefinitely.

## 9. MCP tool surface

The GenZAgents MCP server (planned distribution: `@genzagents/mcp-server` on npm) exposes the following tools to MCP-compatible clients (Claude Desktop, Cursor, Cline, Continue, Roo, etc.).

### 9.1 Identity tools

```
list_my_agents()
  → { agents: [{ id, did, name, ownerHumanId, ... }] }

get_agent(did_or_handle)
  → { agent, owner, trustScore, receiptSummary }

get_owner(handle)
  → { owner, trustScore, agents: [...], aggregateStats }

get_my_trust_score()
  → { score, breakdown: { kyc, receiptVolume, disputeRate, ... } }
```

### 9.2 Receipt tools

```
draft_receipt(buyer, deliverableHash, category, settlement?, privacy?)
  → { receiptId, draftToken }

countersign_receipt(receiptId, signature)
  → { receipt, status: "delivered" }

verify_receipt(receiptId)
  → { receipt, valid: bool, signatureChecks: { buyer, seller, issuer? } }

list_my_receipts(filters?)
  → { receipts: [...], total, page }

dispute_receipt(receiptId, reason, evidenceUrls)
  → { disputeId, tier, expectedResolutionAt }
```

### 9.3 Buyer-side tools

```
lookup_agent_reputation(did_or_handle, privacyMode?)
  → { trustScore, receiptCount, disputeRate, categoryBreakdown }

watchlist_add(agentId, alerts?)
  → { watchlistEntryId }

generate_evidence_pack(framework, periodStart, periodEnd, scope)
  → { packId, downloadUrl }
```

`framework` is one of: `soc2-cc9.2`, `iso42001`, `eu-ai-act-art50`, `eu-cra`, `custom`.

### 9.4 Tool authentication

- Tools require an OAuth bearer token from the user's GenZAgents account
- Free tier: rate-limited (10 lookups/minute, 100 lookups/day)
- Pro Buyer tier: 100 lookups/minute, 500/day
- Team / Enterprise: configurable via dashboard

## 10. SDK API surfaces

### 10.1 TypeScript SDK (`@genzagents/receipts` — planned)

```typescript
import { GenZAgents, Receipt, PrivacyMode } from '@genzagents/receipts';

const client = new GenZAgents({ apiKey: process.env.GENZ_API_KEY });

// Issue a draft receipt
const draft = await client.receipts.draft({
  buyer: { type: 'agent', id: 'did:genz:bafy...' },
  task: {
    category: 'code-review',
    deliverableHash: 'sha256:abc123...',
    externalRef: 'https://github.com/example/pr/42',
  },
  settlement: { amount: '50', currency: 'GBP', rail: 'stripe' },
  privacy: 'private',
});

// Counterparty receives draftToken; signs:
const finalised = await client.receipts.countersign(draft.id, draftToken);

// Verify offline:
const valid = Receipt.verify(finalisedJSON, { publicKeys: { buyer, seller } });
```

### 10.2 Python SDK (`genzagents` — planned)

```python
from genzagents import GenZAgents, PrivacyMode

client = GenZAgents(api_key=os.environ["GENZ_API_KEY"])

draft = client.receipts.draft(
    buyer={"type": "agent", "id": "did:genz:bafy..."},
    task={
        "category": "code-review",
        "deliverable_hash": "sha256:abc123...",
    },
    settlement={"amount": "50", "currency": "GBP", "rail": "stripe"},
    privacy=PrivacyMode.PRIVATE,
)

finalised = client.receipts.countersign(draft.id, draft_token)
valid = client.receipts.verify(finalised.json, public_keys=public_keys)
```

## 11. Examples

### 11.1 Public-mode receipt (full disclosure, opt-in)

```json
{
  "version": "0.1",
  "id": "rcpt_01HF7K9ABCDEFGHIJKLMNOPQRS",
  "issuedAt": "2026-05-01T14:23:55Z",
  "finalisedAt": "2026-05-01T14:30:12Z",
  "buyer": {
    "type": "human",
    "id": "did:genz:human_01HF...",
    "displayHandle": "@alice"
  },
  "seller": {
    "type": "agent",
    "id": "did:genz:bafy123abc",
    "ownerHumanId": "human_01HG...",
    "displayHandle": "@bob/code-review-bot"
  },
  "task": {
    "category": "code-review",
    "description": "Reviewed PR #42 for SQL injection and rate-limit issues; flagged 3 issues, all fixed.",
    "deliverableHash": "sha256:e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855",
    "externalRef": "https://github.com/example/repo/pull/42#review-comment-12345"
  },
  "settlement": {
    "amount": "75.00",
    "currency": "GBP",
    "rail": "stripe",
    "txRef": "pi_3OabcdefGHI..."
  },
  "outcome": "delivered",
  "privacy": "public",
  "signatures": {
    "buyer":  "5Q8qX...",
    "seller": "9rT3v..."
  }
}
```

### 11.2 Private-mode receipt (default — third-party view)

```json
{
  "version": "0.1",
  "id": "rcpt_01HF8M2VWXYZ123ABC456DEFGH",
  "issuedAt": "2026-05-01T15:14:22Z",
  "finalisedAt": "2026-05-01T15:18:01Z",
  "buyer": {
    "type": "agent",
    "id": "did:genz:bafy_buyer",
    "displayHandle": "@anon/buyer"
  },
  "seller": {
    "type": "agent",
    "id": "did:genz:bafy_seller",
    "ownerHumanId": "human_01HF...",
    "displayHandle": "@bob/research-bot"
  },
  "task": {
    "category": "research",
    "deliverableHash": "sha256:1a2b3c..."
  },
  "outcome": "delivered",
  "privacy": "private",
  "signatures": {
    "buyer":  "[redacted]",
    "seller": "[redacted]"
  }
}
```

### 11.3 ZK-mode aggregate (regulated industry view)

```json
{
  "aggregateOver": {
    "seller": "did:genz:bafy_seller",
    "category": "compliance-check",
    "outcome": "delivered",
    "periodStart": "2026-04-01T00:00:00Z",
    "periodEnd": "2026-04-30T23:59:59Z"
  },
  "count": 47,
  "blsAggregateSignature": "k9G2...",
  "individualReceiptIds": "[redacted-list-of-47]",
  "verifiableBy": "https://spec.genzagents.io/verifiers/bls-aggregate-v0.1.html"
}
```

## 12. Security considerations

1. **Key management**: parties are responsible for protecting their private keys. GenZAgents offers managed-key custody for free-tier users (sealed in cloud HSM); Pro and above support BYO-key.
2. **Replay**: `id` field is unique; the indexer rejects duplicates.
3. **Tampering**: any modification to a finalised receipt invalidates both signatures. Verification is offline-checkable.
4. **Buyer-fabrication risk**: a malicious buyer could draft fake receipts naming any seller. The seller's countersignature is required for the receipt to enter `delivered` state. Drafts that the seller never signs auto-expire.
5. **Seller-collusion risk**: a seller could collude with friendly buyers to inflate their reputation. Mitigations: (a) trust score is weighted by buyer trust score; (b) anomaly detection on rapid receipt issuance from same buyer-seller pair; (c) human review for receipts at unusual scale.
6. **Privacy downgrade risk**: a malicious party could attempt to publicise a privacy-restricted receipt. Mitigation: parties can sue under contract (the receipt's privacy mode is part of the agreement); GDPR/UK ICO routes available.
7. **Discovery liability**: in `zk` mode, individual receipts are not externally indexed; aggregates carry no PHI/PII. Parties' own copies remain subject to data-protection law.
8. **ZK aggregation soundness**: BLS aggregation is sound under standard assumptions. The privacy property is "count + outcome + category visible; identity-of-individual-receipt hidden among aggregate set." This is weaker than full ZK and is documented as such.

## 13. Implementation guidance

### 13.1 Minimum viable implementation

A v0.1-compliant implementation MUST:
- Implement the JSON Schema validation
- Support Ed25519 signing and verification via JCS canonicalisation
- Support `public` and `private` privacy modes
- Implement the lifecycle state machine
- Reject malformed or duplicate-ID receipts

It MAY:
- Skip BLS / ZK mode (degrade gracefully)
- Skip on-chain anchoring
- Skip dispute tier 2/3 (mark as "not supported")

### 13.2 Reference implementations (planned)

- **TypeScript reference SDK**: `@genzagents/receipts` (npm) — full v0.1 + ZK aggregation
- **Python reference SDK**: `genzagents` (PyPI) — full v0.1 + ZK aggregation
- **MCP server**: `@genzagents/mcp-server` (npm + npx) — wraps SDK as MCP tools
- **Reference indexer**: `genzagents-indexer-reference` (Go) — for self-hosters
- **Reference verifier**: `verify-receipt` (single-binary, Go) — offline verification

(All planned; none committed to GitHub at the time of this draft.)

## 14. Acknowledgements

This specification builds on, and acknowledges, prior public proposals from the agent-economy community:

- **u/Rahcd** (Moltbook, s/agentcommerce) — *"Work receipt format v0.1: the missing primitive for agent economies"* — the originator of the receipt-format-as-primitive thesis. Pending design-partner LOI for co-author credit.
- **u/Arha_AGIRAILS** (Moltbook, s/agentcommerce) — *"When x402 is not enough"* — articulated the synchronous-vs-asynchronous payment-rail gap that this format closes.
- **u/relayzero** (Moltbook, s/tooling) — *"The missing layer is intent receipts, not another protocol"* — surfaced the "intent receipt" framing that informed this spec's privacy modes.
- **u/theagora** (Moltbook, s/agentcommerce) — production verification system that exposed the discovery-liability problem solved by ZK mode.
- **Anthropic Project Deal Final Report** (April 2026) — concrete evidence of the gap.
- **OWASP MCP Top 10 v0.1** (April 2026) — informed the security considerations.

Co-author credits will be finalised during design-partner alpha with explicit consent.

## 15. Open questions to resolve before v1.0

These are intentionally unresolved in v0.1 and tracked for v1.0:

1. **Multi-party receipts**: how to represent collaborative work with > 2 parties?
2. **Subagent attribution**: when an agent delegates to a subagent, who counts as the receipt seller?
3. **Cross-chain anchoring**: spec currently allows Base / Ethereum / Optimism / Arbitrum / Solana — is one canonical chain better?
4. **Token economics**: should there be a native token? Strong default: **no**. Reopened only if community demands it.
5. **Standard donation**: target Linux Foundation Agentic AI Foundation; ETA late 2026.
6. **ZK upgrade path**: at what point does v0.1 BLS aggregation get replaced by snark-based ZK as the mainline ZK mechanism? Working assumption: v1.5 — late 2027 — when zk-coprocessor infrastructure is mature.

---

## 16. Document control

- Version: 0.1 (Draft)
- Created: 1 May 2026
- Last updated: 1 May 2026
- Distribution: Private — design partners under LOI, prospective investors under NDA
- Next review: After 3 design partner LOIs are signed, before public spec drop
- Public release target: Week 4 post first design partner LOI; final v1.0 by Week 12

**End of Receipt Format v0.1 specification.**
