# Deep dive A: Entitlements → leak-free pack (working draft)

> Final English version lives in DESIGN.md (~900 words). Rows used: LP-0102, LP-0166, LP-0058, LP-0119 (+ LP-0031, LP-0007).

## Core idea: three independent barriers
1. **By construction (allow-list = what the LP CAN see):** each pack has a *plan* that lists what the LP **may** see. The assembler only receives that data: an excluded company's data **never enters** the assembly process, so there is nothing to "forget to delete".
2. **By inspection (deny-list = what the LP CANNOT see):** the final file is read (every XML, workbook, note, metadata, image file name) and searched for traces of anything forbidden. **The deny-list is not derived from the plan**: it comes straight from the side-letter data restrictions, through a separate code path. If the plan has a bug (e.g. an excuse was not applied), barrier 2 does not inherit it.

### Why two lists (and not one)
If barrier 2 searched for "whatever is not in the plan", it would be validating the plan against itself: an excuse missing from the plan would also be missing from the search list. With two lists from different sources:
- **allow-list** = full function (rules + holdings + overrides + defaults) → builds the pack.
- **deny-list** = minimal function, almost trivial to audit: it only reads the restrictions (excuses, unsigned joinder × restricted, FOI) → feeds the scanner.
- **Cross-check before assembly:**
  - **disjoint**: nothing can be in both (allow ∩ deny = ∅). If something is → bug in the plan → block.
  - **complete**: every holding × field in the fund is classified into exactly one of the two. Nothing is left "undecided" (which is exactly where omissions hide).
3. **By traceability (audit trail):** each delivered pack records the rules version, the plan, the scan result and who approved it. If the LP's lawyer calls, we can show what was sent and why.

Principle: **fail closed**. When in doubt (a name that doesn't resolve, conflicting rules, a new company), the pack **is blocked** and escalated; it is never generated "just in case".

```
side-letter data ─┐
company registry ─┼─► [1] resolve references ─► [2] plan per LP×fund ─► [3] conflicts? ──yes──► BLOCK + counsel/IR
commitments ──────┘        (fail closed)            (pure function)            │no
                                                                                ▼
                                    [4] entitlements report (IR reviews ONLY the diff vs prior quarter)
                                                                                ▼
                                    [5] data filtered by plan ─► assembler (B)
                                                                                ▼
                                    [6] leak scanner + round-trip validator ─► [7] IR edits ─► [6] again ─► delivery + [8] audit
```

---

## Step 1: Resolve references (fail closed)
Side-letter rules name companies in free text ("Excused: Pellory Payments"). Before anything is assembled, **every reference must resolve to a `company_id`** in the registry:

| Reference | Sources where it appears | Resolution |
|---|---|---|
| "Pellory Payments" (LP-0102) | B1 "PELLORY PAYMENTS CO - CONV NOTE", B2 "Pellory" | → `cmp_pellory` (aliases confirmed by IR once, then reused) |
| "Vantrel Health" (LP-0166) | B1 "VANTREL HEALTH INC - SER B/C PFD" (III and Opps I), B2 "Vantrel Health" | → `cmp_vantrel` |

If a reference **does not resolve**, **all of that LP's packs** are blocked. This is the most dangerous failure mode in the system: an excuse that doesn't match gets silently ignored and the pack goes out with the company in it. That's why matching is deterministic (normalization + an approved alias table) and there is no "best guess".

## Step 2: Plan per LP×fund (pure function)
Input: the LP's rules (with their source clause) + the fund's holdings. Output: an explicit plan, **per company and per field**:

| Per-company field | Meaning |
|---|---|
| `page` | There is a company page |
| `identity` | Name, logo, description |
| `valuation` | Cost / fair value (schedule and page) |
| `financials` | KPIs, revenue chart, company-reported data |
| `commentary` | Deal team text |

Plus, at pack level: sections, annexes, currency, delivery format, cover note, disclaimer variants, footer legal name.

Combination rules: **default deny**. Every field starts as "no" and a rule enables it.

### Walkthrough with the B3 rows
**LP-0102 Qamar Crescent (Fund III)**: Excused Pellory · PPTX · joinder ✓
- Pellory: all fields `false` → no page, **no row in the schedule**, no commentary.
- The other 30 holdings: everything `true` (joinder signed → restricted ones too).
- Commentary for **other** companies: must not mention Pellory. Since commentary is generated per company × LP type (not per LP), the rule is simpler and stronger: **no commentary mentions other portfolio companies**. This is validated at generation time.
- PPTX → maximum exposure (see step 6).

**LP-0166 Hoshikawa Mutual (Opps I)**: JPY · look-through · Excused Vantrel · joinder ✓
- The excuse is **per LP × fund × company**. Vantrel is in III and in Opps I; Hoshikawa is only in Opps I, and there Vantrel is a large holding (cost 9.0m, FV 11.16m).
- Capital account: **taken as-is from the administrator** (the administrator already handles the economics of the excuse; we don't recalculate it).
- **Look-through annex**: sector and geography exposure **for this LP**. This calculation is ours: we have to exclude Vantrel and use Hoshikawa's share of each holding. Validator invariant: **look-through total = NAV share in the administrator's capital account**. If it doesn't reconcile, `needs_human`.
- JPY: a single closing exchange rate for the whole pack, recorded in the snapshot.

**LP-0058 Caldera State Teachers' (Fund III + Opps I)**: Reduced disclosure (FOI) · s.4.2 "company-level valuation detail each quarter" · joinder ✗
- FOI → fund-level aggregates only: `valuation=false`, `financials=false` and `commentary=false` for all companies (commentary is company-level information too); only identity (name, sector) remains. *(Updated after a second review.)*
- s.4.2 → `valuation=true` for all.
- **Conflict on `valuation`** → step 3.
- Joinder ✗ → `financials=false` for the 9 restricted companies (no conflict, both rules agree).

**LP-0119 Oskervale FoF IV (II, III, Opps I)**: PPTX · look-through · joinder ✗
- 3 packs. For the restricted companies present in each fund: `financials=false`, everything else `true`. Commentary in the "no financials" variant (already accounted for in cost-estimate).
- Look-through: aggregated by sector and geography, includes restricted companies (assumption: an aggregate is not "company financials").
- PPTX + a FoF that "lifts slides" → B's cleanup matters even more.

**LP-0031 Stichting (II + III)** EUR · ESG · joinder ✓ → no company restrictions; ESG annex; EUR for fund figures and capital account (assumption: the schedule too, since it's a fund figure; company KPIs stay in their own currency).
**LP-0007 Northmoor (III)** custom cover · PPTX · joinder ✓ → the cover note slot stays **blocked** until the partners' text comes in: the pack cannot move to `ready` without it.

## Step 3: Conflicts → block, never silent resolution
- Conflict = two rules for the same LP that give different values to the same field.
- **LP-0058**: both packs move to `blocked: entitlement_conflict` with the text of both clauses. A **preview using the most restrictive option** is generated so IR and counsel can see how it would look.
- The resolution is entered by an authorized person (counsel/IR): field, decision, who, when, why. It's stored as an **override with provenance**, valid in subsequent quarters until the side letter changes (at which point it is invalidated and the question is asked again).
- This is what my question 2 was about: the system flags the conflict, it doesn't pick.

> **Approved exceptions** *(added after a second review)*: overrides signed by counsel are a separate, audited input that **both** lists read. The deny-list = restrictions − approved exceptions. Otherwise a counsel-approved grant (e.g. Caldera s.4.2) would leave the scanner blocking the pack forever.

## Step 4: Entitlements report (what IR approves)
Per LP×fund, a readable table: what's in, what's out and why (with the clause). To avoid asking IR to review 214 plans every quarter, **only the diff against the prior quarter** is shown: new LPs, changed side letters, new companies, expired overrides.
- **New company in the fund**: is the LP excused from it? Excuses are per investment and the side-letter data may not cover a new investment → every LP with any excuse in that fund goes to `needs_review` for that company. Fail closed.
- This report is **the early deliverable** (plan-draft): it's useful even if assembly doesn't make it for Q4.

## Step 5: Data filtered by plan
The assembler requests data from a layer that **already applies the plan**: if `valuation=false`, the view model doesn't even carry the number. This also applies to **content baked into the template**: the master's sample company page probably contains real data from some company ("we copy last quarter's page and type over it"). Every slot is cleared before it's filled, and the scanner checks anyway.

## Step 6: Leak scanner (on the final file)
Each pack uses the **deny-list** (computed directly from the restrictions, not from the plan):
- Per excluded company: name, legal name, aliases, known product names, logo hash.
- Per forbidden field: the concrete **values** (e.g. Pellory's GMV 31.2, Vantrel's FV 11,160,000 / 11.16m / ¥ equivalent), normalized into several formats.

Where it looks: all slide, layout and master XML; tables; chart caches; **embedded workbooks**; notes; comments; alt text; `docProps`; **file names** in `media/` (e.g. `pellory_logo.png`); hyperlink targets; and the PDF text.

Results:
- Name or alias found → **block**, zero tolerance.
- Value found → `needs_human` with context, because it may be a coincidence (a "1.9%" can show up for other reasons).
- **Images**: text scanning cannot read them, so an **image allow-list by content hash** is used. It holds registry logos and headshots, the template's photos, and Finance's per-fund images, each registered and reviewed once on receipt. Any other image (e.g. a screenshot pasted from a board deck) → `needs_human`. *(Updated after a second review; replaces "Finance attests".)*

## Step 7: Re-scan after IR's edits
IR edits in PowerPoint. The incident two quarters ago probably came from **copying a page from another deck**. So on re-upload: scan + validator again. Without a `pass` scan after the last edit, the pack **cannot be delivered**.

## Step 8: Audit trail
Per delivered pack: file hash, rules version, plan, overrides applied, scan result, who approved and when, which portal and when. It's the answer to "if a number is wrong, I'm on the phone with the LP's lawyer".

---

## How it fails
| # | Failure | Example | Detection | Mitigation |
|---|---|---|---|---|
| 1 | **Excuse that doesn't resolve** to a company | "Pellory Payments" vs "PELLORY PAYMENTS CO" with no alias | Step 1 | Fail closed: blocks all of the LP's packs. |
| 2 | **Incorrect side-letter data** (counsel got it wrong) | Qamar's excuse is missing | Not detectable by the system | Diff report (step 4); outside our control: record it in the audit. |
| 3 | **New company** with no explicit rule | Opps I invests in a company in Q4 | Step 4 | `needs_review` for LPs with excuses in that fund. |
| 4 | **Inference from aggregates** | Qamar sees fund NAV; Pellory is the only 750k convertible note → it could be deduced | Not detectable | **Riskiest assumption**: fund aggregates are shown as the fund reports them. Needs validation with counsel. |
| 5 | **Cross-mention** in commentary | Another company's commentary: "partnership with Pellory" | Generation + scanner | Rule: commentary doesn't mention other portfolio companies. |
| 6 | **Baked-in content** in template or images | The master ships Vantrel's company page as a sample; Finance's J-curve has labels | Scanner (text); images not | Clear slots; Finance confirmation; OCR later. |
| 7 | **Different value format** escapes the scan | 11.16m vs 11,160,000 vs ¥1.7bn | Normalization | List of variants; the value isn't the main barrier (the allow-list is). |
| 8 | **Manual IR edit** reintroduces the company | IR pastes a slide from another pack | Re-scan (step 7) | Delivery blocked without a subsequent scan. |
| 9 | **Unmodeled conflict** | A new free-text rule the model can't type | Rules parser | Untyped rule = `needs_review`, not ignored. |
| 10 | **Ambiguous definition of "financials"** | Is a restricted company's valuation "financials"? | — | Explicit assumption (no) + question to counsel. |
| 11 | **Bug in the plan computation** (an excuse isn't applied) | Qamar's plan includes Pellory due to a code error | Cross-check allow ∩ deny ≠ ∅ before assembly + scanner with an independent deny-list | Two separate code paths; the deny-list is small and auditable. |

## Assumptions this adds (claims table)
| Assumption | Risk |
|---|---|
| Fund aggregates (NAV, TVPI, doughnuts, totals) are shown as the fund reports them, including to excused LPs | **High**: ask counsel |
| The administrator's capital account already reflects the excuses | Medium |
| "Financials" = company-reported data (KPIs, revenue); valuation is not | Medium |
| The LP's currency applies to fund figures, capital account and schedule; company KPIs stay in their own currency | Medium |
| Side-letter data comes with typed rules referencing the clause | Medium |
| Finance delivers images without company names | Medium |
| The 2 Swiss disclaimer rules are in the side-letter data | Low |
