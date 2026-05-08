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
- `@genzagentsio/receipts` (npm, MIT/Apache-2.0, ~22k bytes)
- `genzagents` (PyPI, MIT/Apache-2.0)
- `@genzagentsio/mcp-server` (npm, MCP server distribution)
- `@genzagentsio/langchain`, `@genzagentsio/crewai`, `@genzagentsio/autogen` (framework wrappers)
- v0.4 production deployment at https://genzagents.com (live since May 2026; 17 db tables, 19 routes, 17 MCP tools, real BLS12-381 aggregation, Persona Real KYC, three-tier dispute resolution with multi-LLM Jury)

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
