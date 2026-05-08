# Publishing the spec to the public repo

This file is a checklist. Delete it after the public repo is live.

## Step 1 — create the GitHub repo (5 minutes)

```bash
# From the genzagents monorepo root
gh repo create genzagents/work-receipt-spec --public \
  --description "Open data format and signing scheme for AI agent work receipts. Privacy-tiered, counterparty-anchored, payment-rail-agnostic." \
  --homepage "https://genzagents.com/spec"
```

## Step 2 — push the staged contents

```bash
cd work-receipt-spec
git init
git add .
git branch -m main
git remote add origin https://github.com/genzagents/work-receipt-spec.git

git commit -m "v0.1.0 — initial public release

Open Work Receipt Format spec, with reference implementation in
@genzagentsio/receipts (npm) and genzagents (PyPI).

Co-author credits to the Moltbook builders who articulated the
receipt-as-primitive thesis: u/Rahcd, u/Arha_AGIRAILS, u/theagora,
u/Orac_garg, u/relayzero (see SPEC.md §14).

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>"

git push -u origin main
git tag v0.1.0
git push origin v0.1.0
```

## Step 3 — link from the marketing site

Edit `apps/web/app/(public)/spec/page.tsx` (create if missing) and add a top-level link to `https://github.com/genzagents/work-receipt-spec`. Already linked in the `did:web:` document at `apps/web/app/.well-known/did.json/route.ts`.

## Step 4 — email the Linux Foundation Agentic AI Foundation

See [`LF-EMAIL-DRAFT.md`](./LF-EMAIL-DRAFT.md) for the message body. Send to `agenticai@linuxfoundation.org` (verify the address by searching "Linux Foundation Agentic AI Foundation working group submissions" — it's been moving). CC `manraj@genzagents.com` for follow-up.

## Step 5 — DM design partners with the link

After the repo lands, DM:
- u/Rahcd (Moltbook) — "Your spec is now publicly maintained at github.com/genzagents/work-receipt-spec. PRs welcome."
- u/Vektor (SIGIL Protocol)
- u/jarvis-pact (Armalo AI) — `dev@armalo.ai`
- u/clawproof
- u/openclaw-frankops
- u/Orac_garg

Suggested DM body:

> Hi [name] — we just published the Work Receipt Format spec publicly at github.com/genzagents/work-receipt-spec under Apache-2.0 (code) and CC BY-SA 4.0 (spec text).
>
> [Specific reason this person is in the loop — referenced their post / their format uses the same primitive].
>
> Want to be a named co-author on v0.2 in exchange for a conformance test against [their product]? We'll do the integration work; you get permanent credit on the spec.
>
> [Their direct contact / link to their landing page].

## Step 6 — drop tracking link in next investor update

After all the above is shipped, the next monthly investor update should include:

> "Public spec donation: pushed `genzagents/work-receipt-spec` to GitHub on [date]. LF Agentic AI Foundation submission filed [date]. Two design partners (SIGIL, Armalo) confirmed conformance interest. This is the `D&B donating their data model to a standards body` move from our v3 thesis — schema commoditised, hosted dashboard remains the moat."
