# Mike + ICME Preflight

> Cryptographically verified responses for the open-source legal AI assistant.

[Preflight](https://docs.icme.io) compiles plain-English safety policies into
formal logic (SMT-LIB) and returns a cryptographic proof that a proposed
agent action is `SAT` (allowed) or `UNSAT` (blocked). This integration wraps
Mike's chat endpoint so every assistant response carries a verifiable receipt.

## Why

Legal AI has uniquely high downside risk:

- Unauthorized practice of law claims
- Jurisdiction-specific advice without disclaimers
- Cross-project privilege leakage
- Fabricated citations
- PII / privileged content in outputs

LLM-judge guardrails are probabilistic and jailbreakable. Preflight turns
each of these into a formally verifiable rule whose enforcement produces a
proof that a regulator, opposing counsel, or auditor can independently
re-check months later.

## What changes in Mike

The diff is intentionally small. Four new files, three light edits.

### Added

| File | Purpose |
|---|---|
| `backend/src/lib/preflight.ts` | HTTP client for `POST /v1/verify` |
| `backend/src/middleware/preflight.ts` | Express middleware on `POST /chat` |
| `backend/migrations/2026_01_preflight.sql` | Adds proof columns to `chat_messages` |
| `frontend/src/app/components/assistant/VerifiedBadge.tsx` | "Verified" pill linking to the proof |

### Edited

- `backend/src/routes/chat.ts` — middleware mounted on `POST /chat`; assistant
  insert now persists `preflight_check_id`, verdict, and policy version.
- `backend/.env.example` — three new vars (`ICME_API_KEY`, `ICME_POLICY_ID`,
  `ICME_PREFLIGHT_ENFORCE`).

The middleware is **fail-open by default** (`ICME_PREFLIGHT_ENFORCE=shadow`):
it records verdicts and proofs but never blocks. Flip to `enforce` once you
trust the policy.

## Setup

1. Apply the new migration in the Supabase SQL editor:

   ```sql
   \i backend/migrations/2026_01_preflight.sql
   ```

2. Add to `backend/.env`:

   ```
   ICME_API_KEY=...
   ICME_POLICY_ID=<uuid from icme dashboard>
   ICME_PREFLIGHT_ENFORCE=shadow
   ```

3. Render the badge wherever assistant messages are shown — e.g. in
   `AssistantMessage.tsx`:

   ```tsx
   import { VerifiedBadge } from "./VerifiedBadge";

   <VerifiedBadge
     info={{
       check_id: message.preflight_check_id,
       verdict: message.preflight_verdict,
       policy_id: message.preflight_policy_id,
       policy_version: message.preflight_policy_version,
     }}
   />
   ```

## Suggested demo policies

Write these in plain English in the ICME dashboard. Preflight compiles each
to SMT-LIB on save.

1. **No unauthorized legal advice** — outputs that recommend specific legal
   action must include a disclaimer and cite an authority.
2. **Privilege boundary** — references to documents must resolve to the
   current `project_id` only.
3. **PII egress** — no SSNs, account numbers, or DOBs in output.
4. **Citation integrity** — every cited case or statute must appear in the
   project's uploaded corpus.
5. **Escalation scope** — securities, healthcare, and M&A questions must be
   flagged for human review.

## Re-verifying a proof

Any third party can confirm a stored `check_id`:

```bash
curl https://api.icme.io/v1/proofs/<check_id>
```

The response is a self-contained, signed object — no Mike access required.

## Roadmap

- Two-stage verification: pre-LLM (intent) and post-LLM (output content).
- Per-project policies (lookup by `project_id` instead of a single env var).
- Streaming verification on tool calls (`runLLMStream` integration).
- Surface proof URLs in chat export / discovery bundles.

---

Built on top of [@willchen96/mike](https://github.com/willchen96/mike).
Integration by the team at [docs.icme.io](https://docs.icme.io).
