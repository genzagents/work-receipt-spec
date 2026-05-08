# Contributing to the Work Receipt Format

Thank you for considering contributing to the spec.

## Quick links

- **Reference implementation (TypeScript):** [`@genzagentsio/receipts`](https://www.npmjs.com/package/@genzagentsio/receipts)
- **Reference implementation (Python):** [`genzagents`](https://pypi.org/project/genzagents/)
- **MCP server:** [`genzagentsio-mcp-server`](https://www.npmjs.com/package/genzagentsio-mcp-server)
- **Conformance test vectors:** [`./CONFORMANCE.md`](./CONFORMANCE.md)

## Spec change process

1. Open an issue describing the proposed change with a concrete use case.
2. Wait for at least one maintainer ack before drafting the PR.
3. PRs touching `SPEC.md` MUST include:
   - JSON schema changes in `receipt.schema.json`
   - At least one test vector in `CONFORMANCE.md` exercising the change
   - Updated reference implementation in the [`genzagents`](https://github.com/genzagentsio/genzagents) repo
4. Wire-format changes (anything that breaks existing receipt verifiers) require bumping `version` from `0.x` to `0.(x+1)`. Inside major version 0, breaking changes are still allowed but discouraged.

## Compatibility commitments

- v0.x — breaking changes permitted with version bump and migration notes
- v1.0+ — wire format frozen; new fields are additive only, must default to omitted, must not change verifier behaviour for existing receipts

## Privacy

- Spec changes that expose new content from `private` or `zk` modes are explicitly prohibited.
- Runtime fields (model, tokens, duration, cost) MUST NOT contain prompt or output text. Implementations that violate this lose conformance.

## Licensing

- Spec text (`SPEC.md`, `README.md`, `CONTRIBUTING.md`, `CONFORMANCE.md`) is **CC BY-SA 4.0**.
- JSON Schema (`receipt.schema.json`) and reference SDK code is **Apache-2.0**.

By submitting a PR you agree to release your contribution under these licenses.

## Maintainers

Initial maintainers (add yourself here in your PR):

- Manraj Singh Dhillon — `manraj@genzagents.com` — GenZAgents Ltd

## Co-author credits

The thesis behind this spec was articulated by these Moltbook authors before any of us started writing code. Their posts are cited in `SPEC.md §14`:

- u/Rahcd — "Work receipt format v0.1: the missing primitive for agent economies"
- u/Arha_AGIRAILS — "When x402 is not enough"
- u/theagora — "You hire an agent. You send payment. Did you get what you paid for?"
- u/Orac_garg — coined "outcome attestation" / "the 17% problem"
- u/relayzero — "intent receipts, not another protocol"

If you're one of these authors, please open a PR adding your real-world contact and we'll add proper co-author attribution.
