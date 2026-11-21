# LP Reporting Packs: Design

## 1. Summary

After the fund administrator signs off, Perry builds 214 packs (one per LP per fund) from the agency's master template. Each pack is an editable PowerPoint file. It contains only what that LP's side letter allows, and every number is traced to its source. IR reviews and edits the packs in PowerPoint as they do today, and exports the PDFs from PowerPoint. Perry re-checks every edited file before it can be delivered.

Four decisions shape the design:

1. **Numbers never pass through an LLM.** Deterministic TypeScript builds the decks. The LLM writes company commentary and cover-note drafts only.
2. **Entitlements fail closed, through two independent lists.** An allow-list builds the pack. A deny-list, computed by a separate path, checks the finished file.
3. **The agency's template stays the source of design.** A per-version manifest maps its slides to data, so the output looks like the agency made it.
4. **No Office in our infrastructure.** We generate the PPTX ourselves. The delivered PDF comes from PowerPoint on IR's machines, which already have Hartwell Sans. LibreOffice is used only for previews and regression tests.

## 2. Architecture

```
 side letters ─┐
 admin xlsx ───┼─► INGEST ─► QUARTER SNAPSHOT ─► ENTITLEMENTS ─► allow-list ─► ASSEMBLER ─► VALIDATOR ─► IR REVIEW ─► VALIDATOR ─► deliver
 submissions ──┤   schema     versioned,          (Deep dive 1)                (Deep dive 2)  │ round-trip   edit in       (again)      + audit
 Notion notes ─┘   checks     company registry                  deny-list ──────────────────►│ + leak scan  PowerPoint
                                                  COMMENTARY (LLM, per company × LP type)
 TEMPLATE v7 ─► MANIFEST (once per template version)
```

All stages run as queue workers in the existing stack, with snapshots, plans and audit records in MongoDB and files in object storage. Ingest validates each source against its schema and fails loudly; it never guesses. The company registry maps "VANTREL HEALTH INC - SER B PFD", "Vantrel Health" and the side letter's "Vantrel Health" to one `company_id`. It uses normalisation plus an alias table that IR confirms once; there is no fuzzy best guess. Commentary runs days before sign-off, since it depends on notes and submissions rather than final valuations, so it can use the batch API. After sign-off only deterministic stages run, and those take minutes.

## 3. Deep dive 1: Entitlements → a pack that cannot leak

The incident that started this project was an excused LP receiving information about the company. Every LP, fund and company check follows from that.

**Step 1: Resolve references, failing closed.** Side-letter rules name companies in text. Before anything is built, each reference must resolve to a `company_id`. For example, LP-0102's "Excused: Pellory Payments" resolves to `cmp_pellory`, which also covers "PELLORY PAYMENTS CO - CONV NOTE" in B1 and "Pellory" in B2. A reference that does not resolve blocks every pack for that LP. This is the system's most dangerous failure: an unmatched excuse silently ships the company.

**Step 2: Build two lists by two code paths.**
- The **allow-list** is the full plan for one LP and fund. It combines rules, holdings, overrides and a default of deny, and it is set per company and per field: `page`, `identity`, `valuation`, `financials` and `commentary`. At pack level it adds annexes, currency, delivery format, cover note, disclaimer variants and footer name.
- The **deny-list** comes from a deliberately small function that reads only the restrictions: excuses, joinder not signed × restricted companies, and FOI. It subtracts **approved exceptions**: overrides signed by counsel, stored as a separate audited input that both paths read. Otherwise a counsel-approved grant would block the pack forever.

Before assembly, two checks run. The lists must be **disjoint**; overlap means a bug in the plan and blocks the pack. They must also be **complete**: every holding × field falls in exactly one list. If the scanner searched for "whatever is not in the plan", a forgotten excuse would be missing from both lists, and the scanner would be checking the plan against itself.

**Walking through B3:**
- **LP-0102 Qamar (Fund III, excused from Pellory, PPTX).** Every Pellory field is denied: no page, no schedule row and no commentary. Commentary is generated per company × LP type, not per LP, so we use a stronger rule than "don't mention Pellory": no commentary mentions another portfolio company. The other 30 holdings are fully allowed, because the joinder is signed.
- **LP-0166 Hoshikawa (Opps I, excused from Vantrel, JPY, look-through).** Excuses are scoped to LP × fund × company. Vantrel's Series C in Opps I (cost 9.0m, fair value 11.16m) is denied. The capital account is taken as reported by the administrator, which already accounts for the excuse economics. The look-through annex is our own calculation, and it must exclude Vantrel. The validator checks that the look-through total equals the NAV share in the administrator's capital account. All JPY figures use one recorded quarter-end rate.
- **LP-0058 Caldera (Fund III and Opps I).** FOI means fund-level aggregates only. For all 17 FOI LPs, every company is denied `valuation`, `financials` and `commentary`, because commentary is company-level information too. Only identity (name and sector) remains, and the allocation charts and look-through stay aggregate. Side letter s.4.2, however, grants Caldera "company-level valuation detail". That is a conflict, so both packs go to `blocked: entitlement_conflict`, showing the text of both clauses. A preview using the restrictive reading lets counsel see the effect. Counsel records the decision as a signed exception with who, when and why, which both lists then read. It persists until the side letter changes. The system never picks a winner.
- **LP-0119 Oskervale (three funds, no joinder, PPTX).** The financials of the nine restricted companies are denied, and those companies get the no-financials commentary variant. The look-through annex stays aggregate.
- **LP-0007 Northmoor.** The cover-note slot is locked until the partners' text is in. The pack cannot reach `ready` without it.

**Step 3: IR approves the diff, not 214 plans.** A human-readable entitlement report lists, per pack, what is in, what is out and which clause decided it. IR reviews only the changes from last quarter: new LPs, changed side letters, expired overrides and new holdings. A new investment puts every LP with an excuse in that fund into `needs_review` for that company, because the existing data cannot say whether they are excused from it.

**Step 4: Data access respects the plan.** The assembler receives only the allowed data, so if `valuation=false` the number is never loaded. Every template slot is cleared before it is filled, because the master's company page is likely a copied real company.

**Step 5: Scan the final file with the deny-list.** The scanner covers:
- slide, layout and master XML
- tables
- chart caches and embedded workbooks
- notes and comments
- alt text
- `docProps`
- media file names, such as `pellory_logo.png`
- hyperlink targets
- after IR's export, the PDF text

A denied name or alias blocks the pack. A denied value, normalised across formats such as 11,160,000, 11.16m and the JPY equivalent, raises `needs_human` with its context, because a match can be a coincidence.

Text scanning cannot read images, so images pass through an **image allow-list keyed by content hash**. The list holds registry logos and headshots, the template's own photos, and Finance's per-fund images, each registered and reviewed once on receipt. Any other image raises `needs_human`, for example a screenshot that IR pasted from a board deck. That covers the realistic leak path without OCR.

**Step 6: Re-scan after IR edits.** Today IR builds company pages by copying the previous company's page, which is a plausible cause of the incident. A pack can be delivered only with a passing scan newer than its last edit.

**Step 7: Audit.** Each delivered pack records its file hash, rules version, plan, overrides, scan result, approver and delivery time.

**How it fails:**

| Failure | Detection | Response |
|---|---|---|
| Excuse name does not resolve | Step 1 | Block all of the LP's packs |
| Bug in plan logic drops an excuse | Disjointness check; deny-list scan | Block |
| Side-letter data itself is wrong | Not detectable | Diff review by IR; audit shows the source |
| Inference from aggregates (Pellory's 750k note may be inferable from Fund III totals) | Not detectable | **Highest-risk assumption**: confirm with counsel |
| Restricted data inside an image | Image hash not in the allow-list | `needs_human`; registered images are reviewed once on receipt |
| Counsel-approved exception blocked by the scanner | Signed exception missing from the deny-list path | Both lists read the same signed exceptions |
| IR pastes a slide from another pack | Step 6 | Delivery blocked |

## 4. Deep dive 2: Template → deterministic assembly and round-trip validation

A .pptx is a zip of XML parts. Slides inherit from layouts and masters. A native chart holds its numbers twice: a cache used for drawing and an embedded workbook used by "Edit data".

**Step 0: Manifest, once per template version.** Every slide of v7 is inventoried and classified into about twelve archetypes, such as cover, TOC, fund overview, capital account, schedule, company page, annexes and disclaimer. Legacy hidden slides are marked `ignore` and are never cloned. For each archetype, the manifest maps slots (shape selector, type and format rule) to data fields. Until we receive v7 itself, we derive it from the archive's v7 packs. A script proposes mappings by matching shape text in an archived pack against that quarter's data, and an engineer resolves the rest by hand. There is no LLM in this step: a wrong mapping is a systematic error across 214 packs. For Q4 the manifest is hand-written JSON, reviewed in a pull request. It is accepted only when re-assembling an archived pack from its own quarter's data reproduces it. When a new template version arrives, it is diffed against the manifest, and missing or renamed shapes block generation until they are remapped.

**Step 1: View models.** The assembler computes nothing. Each slot receives a formatted value, the raw value and a `source_ref`. Examples from B1 and B2:
- **Vantrel, Fund III company page.** Series B plus Series C give cost 8.50m and fair value 17.35m. The schedule keeps both administrator rows.
- **Vantrel, Opps I.** Cost 9.00m and fair value 11.16m, with the same KPIs and commentary as Fund III.
- **KPI tiles.** Vantrel has three KPIs (ARR 18.4 USD m, NRR 118%, runway 14 months), so it gets the stretched three-tile layout. Dunmarrow has two, revenue GBP 2.14m in its own currency and 412 units, and carries the flag "restated 14 Oct" for IR.
- **Pellory.** Its period stays "FY27 Q1 (Aug–Oct 2026)", with a calendar-mismatch flag and a late-submission flag (submitted 2 Nov).
- **Ångström.** Written off with no submission, so it gets a schedule row only (configurable).
- **Fund III schedule.** 31 rows paginate as 14/14/3 plus the totals row. Qamar's 30 rows paginate as 14/14/2, and the total is labelled as excluding positions the LP is excused from.
- **LP-0031's legal name.** At about 120 characters it exceeds the footer's measured budget, so it uses an IR-approved short form.

**Step 2: Assemble.**
- **Base.** Start from the template's masters, layouts and theme with zero slides. For each planned slide, clone its archetype (XML, relationships, charts and workbook) and issue fresh IDs from one allocator.
- **Text.** Replace text while keeping the first run's formatting. Commentary has a word budget measured from the archive; we do not rely on autofit.
- **KPI tiles.** For more than four KPIs, a per-company priority list that IR can edit picks four. For two or three, a precomputed geometry is used. Removed tiles take their `<p:timing>` animation entries with them.
- **Charts.** Rewrite the cache and regenerate the embedded workbook from scratch, so it never contains another company's figures. The green latest bar becomes one `dPt` at the last index.
- **Waterfall.** The NAV bridge uses chartex and is handled the same way. It has at most nine variants (fund × currency).
- **Fixed content.** Images keep their geometry. SmartArt is copied untouched.
- **Links and TOC.** Menu links are re-resolved by section, and items for absent sections are removed. The TOC is numbered last.
- **Strip.** Remove notes, comments, hidden slides, custom XML and personal `docProps`, and prune unreferenced media.

**Step 3: Structural checks.**
- Every relationship resolves and there are no orphan parts.
- The OOXML schema validates. PowerPoint is strict and LibreOffice is tolerant, so LibreOffice is never the test of "opens fine".
- Every animation target and hyperlink exists, and no template placeholder text survives.
- For each release, a sample of packs is opened in real PowerPoint and must not show a repair prompt.

**Step 4: Round-trip data validator.** This is a separate module that shares no code with the assembler. It reads the finished file the way an LP would: text, tables, chart caches, workbook cells, alt text and footer. It maps values back to slots through the manifest and compares them with the snapshot, not with the view model, so an error in the view model still shows up. It also recomputes invariants independently:
- schedule rows sum to the administrator's **total investments at fair value**, not to NAV, since NAV also includes cash, other assets and liabilities
- for an excused LP, the fund's total investments minus the excluded positions equals the subtotal shown
- the NAV bridge ends at the administrator's NAV
- Vantrel's Fund III page = Series B + Series C
- chart cache = workbook
- the capital account ties out
- one currency and rate throughout

Every number must have a source, and every planned item must be present. The result is either `pass` or `needs_human`, listing slide, slot, expected and found. The validator runs again when IR uploads an edited file, so a number IR mistyped is also caught. The leak scanner from Deep dive 1 uses the same reader.

**How it fails:**

| Failure | Example | Detection / response |
|---|---|---|
| Template drift | v8 renames "KPI_1" | Manifest diff blocks v8; re-assembling an archived pack catches a rename that matches the wrong shape |
| PowerPoint repair prompt | Duplicate IDs across 30 cloned company pages | Schema and ID checks, plus a sample opened in PowerPoint |
| Text overflow (autofit not recomputed) | LP-0031 footer; 120-word commentary | Budgets per slot; preview; IR review |
| Stale chart workbook | Dunmarrow's chart keeps a copied company's data | Workbook always regenerated; scanner |
| Green bar on the wrong point | Pellory has 5 quarters of history, the template 8 | Unit test: `dPt` index equals the last point |
| Orphan animation | Dunmarrow's two removed tiles | Animation-target check |
| Waterfall not editable | NAV bridge in EUR | Week-1 spike; fallback is a classic stacked bar approved by IR |
| Misleading preview | LibreOffice substitutes the font | Preview labelled approximate; final check in PowerPoint |

## 5. Corrections after sign-off

In three of the last four quarters at least one valuation was corrected after sign-off. A new administrator file creates a new snapshot. A diff against the previous snapshot lists the affected values, and through `source_ref` the packs and slots that contain them, including commentary that cites the number. For Q4 we regenerate only the affected packs and show IR which slides changed, so they can re-apply their edits. Patching values in place inside an edited deck depends on our slot tags surviving PowerPoint edits, which is unverified, and comes after Q4. A pack that has already been delivered is never changed silently. IR decides whether to reissue it, and the reissue goes in the audit record.

## 6. The ten weeks

There are two engineers. Weeks 9–10 are the live Q4 cycle, which leaves about 15 engineer-weeks of build. Work that does not depend on the template comes first. The template's technical risks are tested in week 1.

| # | Piece | EW | Weeks |
|---|---|---|---|
| 1 | Spike: cloning, classic chart and chartex, repair-free open in PowerPoint | 1 | 1 |
| 2 | Ingest, schema checks, versioned snapshot | 1.5 | 1–2 |
| 3 | Company registry and aliases | 0.5 | 2 |
| 4 | Entitlements: rules model, allow/deny lists, conflicts, entitlement report, audit | 2 | 2–4 |
| 5 | PPTX reader: leak scanner and round-trip validator | 1.5 | 4–5 |
| 6 | Manifest v7 (by hand) and view models | 1 | 2–3 |
| 7 | Assembler core: clone, text, IDs, links, TOC, footer, strip | 2 | 3–5 |
| 8 | Charts, workbooks, schedule pagination, KPI layouts | 1.5 | 5–6 |
| 9 | Commentary (LLM, batch) with number and name checks | 1 | 6 |
| 10 | Review flow: pack status, `needs_human`, download and re-upload, re-validation | 1.5 | 6–7 |
| 11 | Shadow run: regenerate Q3 and compare with IR's hand-built packs | 1 | 7–8 |
| 12 | Corrections: snapshot diff, affected packs, regenerate | 0.5 | 8 |
| | **Total** | **15** | |

The entitlement report (piece 4) is useful on its own. If assembly slips, IR still builds Q4 by hand, but with a checked list of what each pack must and must not contain, and the scanner (piece 5) can check their hand-built files.

**Not ready for Q4:**
- Commentary tuned per LP type: Q4 ships one variant.
- Customer-uploaded templates and an onboarding UI.
- Server-side, high-fidelity PDF.
- In-place correction patches that preserve IR's edits.
- A partner cover-note workflow inside Perry: IR pastes the notes in.
- Automated upload to the LP portal.

## 7. Claims

No experiments were run for this document. Everything technical below is assumed until the week-1 spike or the shadow run checks it.

| Claim the design depends on | Status | How verified / how we will verify |
|---|---|---|
| Slides, charts and workbooks can be cloned and edited in Node without Office | Assumed | Week-1 spike, opened in PowerPoint (Mac and Windows) |
| Waterfall (chartex) data can be rewritten like a classic chart | Assumed, low confidence | Week-1 spike; stacked-bar fallback |
| Schema validation catches most causes of PowerPoint's repair prompt | Assumed | Spike: corrupt files deliberately and compare |
| LibreOffice is not a reliable test of "opens fine in PowerPoint" | Assumed (the spike suggests it) | Spike: compare both on the same files |
| Archived v7 packs contain every archetype in the master | Assumed | Inventory of the archive in week 2 |
| Shape names and tags survive normal editing in PowerPoint | Assumed | Spike: edit, save, re-read |
| The administrator's capital accounts already reflect excuses | Assumed | Shadow run on Q3 |
| Fund-level aggregates may be shown unchanged to excused LPs | Assumed, high risk | Ask counsel |
| "Financials" means company-reported data, not the fund's valuation | Assumed | Ask counsel |
| Finance's images contain no company names | Assumed | Finance attests |
| LLM cost is about $25–100 per cycle (2–7% of price) | List prices verified against Anthropic's price list (Nov 2026); token sizes assumed | Pilot on 20 companies; measure thinking tokens |
| Assembly takes minutes per cycle; LibreOffice preview about 1–3 minutes per pack | Assumed | Load test in the shadow run |
| IR exporting the PDF from PowerPoint is acceptable | Assumed | Confirm with IR (pending answer to question 3) |

## 8. Open questions (sent; answers pending)

The five questions as sent, and the working assumption the design runs on for each (template convention, who owns rights and conflicts, what IR reviews, what we test against, corrections), are listed in `notes_for_ali.md`, item 8.
