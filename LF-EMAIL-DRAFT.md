# Email draft — Linux Foundation Agentic AI Foundation submission

**To:** agenticai@linuxfoundation.org
**Cc:** manraj@genzagents.com
**Subject:** Working-group proposal — Work Receipt Format v0.1

---

Dear LF Agentic AI Foundation team,

I'd like to propose the **Work Receipt Format v0.1** as a candidate working-group submission for the Foundation's Q3 2026 cycle.

**What it is.** An open data format and signing scheme for representing the outcome of a unit of work performed by an AI agent on behalf of (or for) another party. Cryptographically signed, privacy-tiered (public / private / ZK via BLS12-381 aggregation), counterparty-anchored (both buyer and seller countersign), payment-rail-agnostic (works with x402, Stripe Agentic, ERC-8004, off-rail), and zero-content-by-construction in private mode (only hashes + identifiers + outcomes leak).

**Public repo:** https://github.com/genzagents/work-receipt-spec
**License:** Apache-2.0 (schema + reference SDKs), CC BY-SA 4.0 (spec text)

**Why now.** Anthropic's April 2026 Project Deal report explicitly stated *"the policy and legal frameworks around AI models that transact on our behalf simply don't exist yet."* EU AI Act enforcement begins 2 August 2026; EU CRA reporting 11 September 2026. Regulated buyers — banks, insurers, healthcare, government — need a vendor-evidence answer for AI agents on a binding deadline.

**Reference implementations already shipped.**
- `@genzagentsio/receipts@0.6.0` (npm, Apache-2.0) — TS SDK with full v0.6 types (`environment`, `issuedByHumanId`)
- `genzagents@0.6.0` (PyPI, Apache-2.0) — Python SDK in lockstep
- `@genzagentsio/mcp-server@0.6.8` (npm) — MCP server with auto-receipt mode, session replay (`GENZAGENTS_REPLAY=on`), `check_duplicates` MCP tool for pre-issue duplicate detection
- `@genzagentsio/langchain@0.6.0`, `@genzagentsio/crewai@0.6.0`, `@genzagentsio/autogen@0.6.0` (framework wrappers, all bumped in lockstep)
- v0.6.9 production deployment at https://genzagents.com (live + auto-deployed via CI/CD; 22 db tables, 30+ routes, 17 MCP tools, real BLS12-381 ZK aggregation, Persona Real KYC, three-tier dispute resolution with multi-LLM jury, signed compliance evidence packs for SOC 2 / ISO 42001 / EU AI Act / EU CRA, semantic + lexical hybrid search via pgvector, 5-category anomaly watcher with webhook delivery, audit log with SOC 2 CC6.1 conformance, Microsoft Entra + Google Workspace SSO with domain-verified auto-membership)

**Spec v0.6 additions** (all backward-compatible with v0.5 receivers):
- `environment` field on receipt root: `production | staging | dev`. Verifiers SHOULD exclude non-production from aggregate trust scoring.
- `issuedByHumanId` field: per-receipt human attribution. Distinct from `buyer.ownerHumanId` because self-issued receipts use a sentinel buyer DID.
- `extensions.replay.url`: optional URL pointing at a tool-call payload bundle, capped 10KB per direction, with truncation flags. Debug-surface convenience, NOT part of the cryptographic anchor.
- Canonical anomaly event taxonomy (5 kinds, §8.2.3) for out-of-band webhook delivery.

**Independent implementers and design partners committed.**
- SIGIL Protocol (sigilprotocol.xyz, 185+ agents, Solana mainnet) — uses our identity primitive shape
- Theagora (escrow + reputation API on Moltbook)
- Armalo AI (armalo.ai, 61 agents, 27 orgs) — Pacts model concept now in our v0.5 spec
- u/Rahcd, u/Arha_AGIRAILS, u/relayzero — the original receipt-format proponents on Moltbook

**Why LF Agentic AI Foundation specifically.** We considered IETF (slower lifecycle), W3C (DID Core overlap; we publish a `did:web:` view alongside our `did:genz:` method to interop), and standalone donation (no governance). LF AAIF is the natural home — agent-economy-specific, multi-stakeholder governance, and exactly the kind of cross-vendor neutrality we need for our buyer-side trust thesis to hold.

**What we're asking for.**
1. Acceptance into the working-group review queue for Q3 2026.
2. Naming criteria and timeline so we can plan a v0.2 governance freeze that aligns with LF AAIF expectations.
3. Introductions to other Foundation members building agent-economy infrastructure who might benefit from a shared receipt format.

**Maintainership commitment.** GenZAgents Ltd (UK) commits to maintaining the spec under the donated-but-stewarded model — we provide engineering bandwidth for v0.x evolution; LF AAIF governance ratifies wire-format freezes at v1.0+.

Happy to walk through the architecture and the production deployment in a 30-minute call. Calendar: [https://cal.com/manraj/lf-aaif].

Best,

Manraj Singh Dhillon
Founder, GenZAgents Ltd
manraj@genzagents.com
+44 [phone]
https://genzagents.com

---

**Attachments suggested but not embedded in email body:**
- `SPEC.md` (the full v0.1 spec text)
- `CONFORMANCE.md` (test vectors)
- v3 / v4 architecture docs (one-page summary)
