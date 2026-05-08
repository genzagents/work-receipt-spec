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

---

Conformance reports welcomed at the [genzagents/work-receipt-spec issues](https://github.com/genzagents/work-receipt-spec/issues).
