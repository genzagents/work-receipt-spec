# Conformance test vectors

A v0.1-conformant implementation MUST pass every test vector below. Vectors are organized by feature so partial conformance can be reported (e.g. "we pass §A and §B but not §D ZK aggregate").

The reference TypeScript SDK ([`@genzagentsio/receipts`](https://www.npmjs.com/package/@genzagentsio/receipts)) and Python SDK ([`genzagents`](https://pypi.org/project/genzagents/)) pass every vector. They share canonical test fixtures in [`fixtures/`](./fixtures/).

## §A — Canonicalisation (RFC 8785 JCS)

Implementations MUST canonicalise receipt bodies via JCS before signing. The canonical bytes for the same logical receipt MUST be identical across all conformant implementations.

```json
// Input
{ "b": 2, "a": 1, "c": [3, 1, 2] }

// Expected JCS output (string)
{"a":1,"b":2,"c":[3,1,2]}
```

Edge cases that MUST be handled:
- Float `1.0` → output `1` (integer normalisation)
- Float `-0` → output `0` (negative zero)
- Empty object `{}` → `{}` (no spaces)
- UTF-16 sort order on key names (not byte-order)
- ECMA-262 number serialisation (no scientific notation in normal ranges)

## §B — Ed25519 signing

The buyer's signature is computed over the JCS canonical bytes of the draft receipt body (with `outcome: "draft"` and `finalisedAt: null`), excluding the `signatures` field itself.

The seller's signature is computed over the **same** canonical bytes — finalised receipts' `outcome` and `finalisedAt` are runtime metadata, not part of the signed payload. Verifiers MUST zero those fields before re-canonicalising.

## §C — Privacy mode redaction

When `privacy === 'private'` and the requester is not a party to the receipt:
- `task.description` MUST be redacted
- `task.externalRef` MUST be redacted
- `settlement` MUST be redacted (set to `null`)
- `disputeOutcome` MUST be redacted
- `evidencePointer` MUST be redacted
- All other fields remain visible

When `privacy === 'zk'` and requester is non-party:
- Only `id`, `version`, `issuedAt`, `finalisedAt`, `seller`, `task.category`, `outcome`, `privacy`, `zkCommitment` are visible.

## §D — BLS12-381 aggregate ZK proof

When `privacy === 'zk'`, the receipt's `zkCommitment` field is the BLS12-381 aggregate signature over the same canonical bytes used for Ed25519 signing.

Verification: aggregate-verify the receipt-set's BLS commitments against the seller's BLS public key. A passing aggregate proves *all* receipts were signed by the same seller without exposing individual receipt content.

## §E — Pacts (added v0.5)

When `pacts` is present, each pact entry MUST be:
```json
{
  "id": "string (stable within receipt)",
  "kind": "string (response-sla | cost-cap | no-external-llm | ... | custom)",
  "predicate": "human-readable plain English",
  "deadline": "ISO 8601 string | null",
  "slashMicrogbp": "number | null"
}
```

Verifiers MUST preserve unknown `kind` values without rejecting them.

## §F — Runtime fields (added v0.2)

When `runtime` is present, fields are all optional but MUST NOT contain prompt or output text. Implementations MAY compute `costMicrogbp` server-side from `tokensIn × tokensOut × <model rate>` if the issuer omits it.

## §G — Versioning

`version` is required on every receipt. Verifiers MUST reject receipts with unknown `version` values rather than guessing.

## §I — Self-issued signatures (added v0.5.7)

When `signatures.buyer == "self-issued"` AND `signatures.seller == "self-issued"`:

1. Verifiers MUST validate `signatures.issuer` against the platform issuer's Ed25519 public key.
2. Verifiers MUST NOT attempt to derive keys from buyer.id / seller.id or check per-party signatures.
3. The platform issuer public key is retrievable at:
   - `GET https://api.genzagents.com/v1/issuer/ed25519` (canonical, JSON)
   - `GET https://genzagents.com/.well-known/did.json` (W3C DID mirror, key id `#issuer-key-1`)
4. Mixed signatures (sentinel + real sig) MUST cause verification to fail with code `MALFORMED_SELF_ISSUED`.

### §I.1 Test vector

```json
{
  "version": "0.1",
  "id": "rcpt_01J3KEYAIAAAAAAAAAAAAAAAA",
  "issuedAt": "2026-05-15T18:00:00Z",
  "finalisedAt": "2026-05-15T18:00:00Z",
  "buyer":  { "type": "human", "id": "did:web:genzagents.com", "ownerHumanId": "<uuid>" },
  "seller": { "type": "agent", "id": "did:genz:abc...", "ownerHumanId": "<same uuid>" },
  "task": {
    "category": "ops",
    "deliverableHash": "sha256:0123...cdef",
    "description": "MCP tool call: get_my_trust_score"
  },
  "privacy": "private",
  "outcome": "delivered",
  "project": { "name": "claude-desktop", "sessionId": "mcp-01H..." },
  "signatures": {
    "buyer":  "self-issued",
    "seller": "self-issued",
    "issuer": "<base64url Ed25519 sig>"
  }
}
```

Verifier validates by:
1. Computing JCS canonical bytes of the receipt with `signatures` removed.
2. Looking up the issuer public key from `/v1/issuer/ed25519` or the well-known DID doc.
3. Running `Ed25519.verify(canonicalBytes, base64url_decode(signatures.issuer), issuerPubkey)`.

## §H — Project sub-object (added v0.5.6, extended v0.5.9 with `provider`)

When `project` is present at the top level, it MUST conform to the Project sub-schema (see SPEC §3.8). Required field: `name` (non-empty, max 200 chars). Optional fields: `provider` (max 64 chars, v0.5.9+), `sessionId` (max 128 chars), `threadUrl` (URI, max 500 chars), `extra` (free-form object).

**§H.2 Cross-provider grouping rule.** Two receipts with the same `project.name` and DIFFERENT `project.provider` values MUST be treated as the same conceptual chat split across LLM platforms. UI implementations SHOULD render them under one project bucket with a provider sub-indicator per receipt. Portable-manifest generators MAY accept a `provider` filter to scope to one platform's slice while keeping the project-level grouping intact for unfiltered queries.

**§H.3 Provider naming guidance.** The `provider` value SHOULD be the lowercase platform identifier (e.g. `claude-desktop`, `claude-code`, `chatgpt`, `cursor`, `cline`, `windsurf`, `continue`). Implementations using the GenZAgents MCP server get this auto-populated from the MCP `initialize` handshake's `clientInfo.name` field. Receipts issued via the SDK directly without going through MCP MAY omit the field.

Producers SHOULD also mirror the same value under `extensions.project` for back-compat with v0.5.0–v0.5.5 readers. When both the top-level field and the extensions mirror are present, they MUST be deeply equal; verifiers MAY reject mismatches as malformed.

Readers MUST read the top-level `project` field as authoritative. Readers MAY fall back to `extensions.project` if the top-level field is absent.

Receipts that share `project.name` are considered to belong to the same chat / project / sprint. UI implementations SHOULD group them visibly; portable-manifest generators MAY filter by this field to scope a cross-LLM handoff to a single chat thread.

Verifiers MUST treat differences in `project.name` capitalisation or whitespace as distinct values (no normalisation). Producers SHOULD trim leading/trailing whitespace before signing.

### §H.1 Test vector

A draft receipt with `project` populated:

```json
{
  "version": "0.1",
  "id": "rcpt_01H1234567890ABCDEFGHJKMNP",
  "issuedAt": "2026-05-15T13:00:00Z",
  "buyer": { "type": "human", "id": "did:web:example.com" },
  "seller": { "type": "agent", "id": "did:genz:abc..." },
  "task": {
    "category": "code-write",
    "deliverableHash": "sha256:0123...cdef",
    "description": "Implemented Stripe webhook retry with jitter."
  },
  "privacy": "private",
  "outcome": "draft",
  "project": {
    "name": "Stripe migration",
    "sessionId": "claude-conv-abc-123",
    "threadUrl": "https://claude.ai/chat/abc-123"
  },
  "extensions": {
    "project": {
      "name": "Stripe migration",
      "sessionId": "claude-conv-abc-123",
      "threadUrl": "https://claude.ai/chat/abc-123"
    }
  },
  "signatures": {
    "buyer":  "<base64url Ed25519 signature>",
    "seller": ""
  }
}
```

The canonicalised form (used for signing) includes the typed `project` object — the signature does NOT change shape from v0.5.5 except that this new field is part of the signed bytes.

---

Conformance reports welcomed at the [genzagents/work-receipt-spec issues](https://github.com/genzagents/work-receipt-spec/issues).
