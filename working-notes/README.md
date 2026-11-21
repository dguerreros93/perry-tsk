# Perry — LP Reporting Packs (take-home)

English translation of the working notes; the originals were written in Spanish.

A **design, not code** exercise. ~4h. Due: Saturday night. Followed by a 45-min conversation.

## Deliverables (section 8 of the brief)
| File | Limit | Must include |
|---|---|---|
| `DESIGN.md` | 3,000 words (tables count, diagrams don't) | 2 deep dives using rows from Appendix B + how it fails · 10-week plan with engineer-weeks per piece and what does NOT make Q4 · claims table (verified + how / assumed) |
| `ALTERNATIVES.md` | 600 words | 3 approaches seriously considered + why the chosen one |
| `CORRECTIONS.md` | — | ≥4 moments where the AI was wrong and YOU caught it (what it proposed, what was wrong, how you knew, what changed) + evidence with YOUR message. If the AI corrects itself, it doesn't count |
| Transcripts (optional, highly recommended) | — | Complete unedited sessions + experiments |

> ⚠️ Starting now: save the transcripts of these sessions. `CORRECTIONS.md` needs evidence, and the easiest way is to correct me explicitly when I get something wrong (and note it in `corrections-log.md`).

## Hard constraints
- 2 engineers × 10 weeks, but **~15 real engineer-weeks** (weeks 9-10 = live Q4 cycle + support rota).
- Stack: **TypeScript** monorepo (Next.js, Node API, queue workers, MongoDB, object storage), **Linux containers**, Claude + Gemini via API.
- **No Windows servers or unattended desktop Office** this year. **Avoid a 2nd runtime** (e.g. Python) without a very good reason → the spike's python-pptx clashes with this.
- Price: ~$1,500 per cycle (214 packs) → ~$7/pack of gross margin for everything (LLM, render, storage).
- 5 questions, a single email, answer within 1 business day. No need to wait for the answer to keep going.

## Language
- Working files (this folder): **Spanish**.
- Everything delivered (`DESIGN.md`, `ALTERNATIVES.md`, `CORRECTIONS.md`, evidence): **English**. Generated at the end, in a separate `submission/` folder (`../submission/`), from these drafts, respecting the word limits.

## Working files
- `source/` — original brief (PDF + extracted text) (not included in this repo; the brief is marked confidential)
- `glossary.md` — terms from the brief + technical terms that appear implicitly
- `findings.md` — traps / hidden details in the brief and Appendix B (not included in this repo)
- `questions.md` — evaluation of my 5 candidate questions + final proposal (not included in this repo)
- `corrections-log.md` — log of moments where the AI was wrong (raw material for CORRECTIONS.md)

## Status (end of 2026-11-20)
Done:
- Brief analyzed: `findings.md` (not included in this repo), `context.md`, `glossary.md`
- 5 questions sent: `questions.md` (not included in this repo) (answer expected ~2026-11-21)
- Architecture: `architecture-draft.md` (not included in this repo). Costs: `cost-estimate.md` (not included in this repo). Plan: `plan-draft.md` (not included in this repo)
- Deep dives: `deep-dive-B.md` (template/assembly + round-trip validator), `deep-dive-A.md` (allow/deny lists, fail closed)
- Corrections log: 5 entries (#2, #3, #5 strong; #4 medium; #1 weak)

Next steps:
1. Incorporate the answers to the 5 questions (adjust assumptions, plan and deep dives)
2. `ALTERNATIVES.md`: 3 approaches (LLM edits XML / deterministic OOXML in TS / Python or render service). Includes the Python trade-off
3. Consolidated claims table (verified vs assumed)
4. Short section on post sign-off corrections (C) within the DESIGN
5. Generate the English deliverables in `submission/` + transcript excerpts as evidence

## Status (2026-11-21)
`submission/` ready, in English: DESIGN.md (2,833 / 3,000 words, not counting the diagram), ALTERNATIVES.md (577 / 600), CORRECTIONS.md (4 corrections) + `evidence/` (verbatim transcript excerpts).
Pending: answers to the 5 questions → adjust §7-8 of DESIGN. The full transcript is attached in sanitized form (see `../submission/notes_for_ali.md`).

## Repo (2026-11-21)
- `submission/transcript/session-transcript.md`: **frozen**, not being regenerated for now.
- The original brief (`source/`) is not included because it is marked "Confidential".
