# Deep dive B: Template → deterministic assembly (working draft)

> The brief asks: "walk through exactly what your system does, step by step, using rows from Appendix B, and say how it fails". The final English version goes into DESIGN.md (~900 words). This is the long draft for thinking things through.

## Core idea
The template is converted **once per version** into a **manifest**: a "contract" that states which model slides exist and which slot in each one receives which piece of data. With that, assembling a pack is a **deterministic TypeScript program**: clone model slides, fill slots with values that are already computed and formatted, rewire links and clean up. Same input → same file, byte for byte. No LLM touches the XML or the numbers.

```
TEMPLATE v7 ──[0. onboarding]──► MANIFEST v7 (versioned)
                                     │
PACK PLAN (from A) ──[1. slide list]►│
SNAPSHOT ──[2. view models]─────────►│
                                     ▼
                        [3. TS assembler] ──► PPTX ──[4. structural validation]──► [data validation (A)] ──► IR
```

---

## Step 0: Template onboarding (once per version, ~2 per year)
1. **Inventory**: open the v7 ZIP (or, until we have it, a v7 pack from the archive) and list the slides, their shapes (name, type, placeholder, position, text), charts, images, links, animations, notes and whether they're hidden.
2. **Classify into archetypes**: cover, TOC, section divider, fund overview (KPIs + NAV bridge waterfall + 2 doughnuts + TVPI/DPI line), J-curve/benchmark (images), fund structure/timeline (SmartArt), capital account, schedule, company page, ESG annex, look-through annex, disclaimer. Hidden "old version" slides are marked `ignore`: **they are never cloned**.
3. **Map slots**: per archetype, each slot gets a `slotId`, a shape selector (name; if it has none, placeholder idx + type + approximate position), a type (`text | number | table | chart | image | kpiTiles | link`) and a formatting rule.
   - **Deterministic** automatic proposal: a script cross-references the shape text of an archive pack with that quarter's data (e.g. the shape that says "18.4" ↔ Vantrel's ARR). Whatever doesn't match is mapped **by a human** by hand. **No LLM**: a wrong mapping would put a number in the wrong slot across all 214 packs. And there are ~12 archetypes: doing it by hand is a day's work.
   - For Q4, the v7 manifest is a **hand-written JSON** reviewed in a PR. The onboarding UI is deferred to Q1.
4. **Validate the manifest**: every slot resolves to **exactly one** shape. Then an archive pack is re-assembled with its quarter's data and compared against the original (extracted text + render). If the diff is minimal, the manifest is approved.
5. **v8 arrives**: the inventory is repeated and diffed against the v7 manifest. Renamed or missing shapes = broken slots, and **generation with v8 is blocked** until they're re-mapped. We never generate with a partial manifest.

*(This is also the basis for Q1's "upload your own template": it's the same onboarding, assisted, not "just works".)*

## Step 1: Pack plan → slide list
From deep dive A we get, per LP×fund, the list of what's allowed. Example **LP-0102 Qamar, Fund III** (excused Pellory, PPTX, joinder ✓):
`cover → TOC → fund overview → J-curve → capital account → schedule (no Pellory row) → company pages ×30 (no Pellory) → disclaimers`. No annexes.

## Step 2: View models (data already computed and formatted)
The assembler **computes and decides nothing**: per slide it receives an object with final values (formatted string + raw value + `source_ref`). Examples with rows from B:

| Row | Resulting view model |
|---|---|
| **Vantrel B + C (HRP III)** | Fund III company page: cost 6.0 + 2.5 = **8.50m**, FV 14.25 + 3.10 = **17.35m**. The schedule keeps **2 rows** (as reported by the administrator). `source_ref` = both rows. |
| **Vantrel C (OPPS I)** | Opps I company page: cost **9.00m**, FV **11.16m**. Same KPIs and commentary as Fund III (it's the same company), different fund numbers. |
| **Vantrel KPIs** | 3 KPIs (ARR 18.4 USD m, NRR 118%, runway 14 months) → **3-tile** layout ("we stretch them"). |
| **Dunmarrow** | Revenue **GBP 2.14m** + Units 412 → **2-tile** layout. The KPI is shown in the company's currency, unconverted. The period "Jul-Sep 2026" is normalized to "Q3 2026". FV 9.80m flagged "restated 14 Oct" → alert to IR. |
| **Pellory** | GMV 31.2 USD m, take rate 1.9%; period **"FY27 Q1 (Aug–Oct 2026)"** shown as-is + warning "period doesn't match Q3". Submitted 2 Nov → late warning. |
| **Ångström** | FV **0** (written off Q2), **no submission** → "written off" company page variant with no tiles or chart, or just the schedule row (configurable rule; assumption: schedule only). |
| **Schedule HRP III** | 31 rows → pages of **14 / 14 / 3 + totals**. For Qamar (no Pellory): 30 rows → 14 / 14 / 2 + totals. Negatives in red and in parentheses, applied by code. |
| **LP-0031 footer** | "Prepared for Stichting Bedrijfstakpensioenfonds voor de Noordelijke Scheepsbouw en Maritieme Techniek · Confidential" ≈ 120 characters → if it exceeds the slot's budget (measured from the archive), an IR-approved `short_legal_name` is used, or it's escalated. |

## Step 3: Assembler (TypeScript, no Office)
1. **Base**: copy masters, layouts, theme and shared media from the template. **Start with no slides**.
2. **Clone** the archetype for each slide in the plan: slide XML + rels + charts + embedded workbook + notes. New part names and IDs (`sldId`, `r:id`, `cNvPr id`). Media deduplicated by hash.
3. **Text**: replace the content of the slot's runs while keeping the formatting (`rPr`) of the first one. Commentary has a slot **word budget**, learned from the archive (80-120). We don't rely on autofit.
4. **KPI tiles (2-7)**: if there are more than 4, the rule picks 4 (priority order per company, configurable by IR). With 2 or 3, a precomputed geometry is applied. Leftover tiles **are deleted together with their entries in `<p:timing>`** (fade animations), so no animations are left pointing at non-existent shapes.
5. **Classic charts** (revenue combo, doughnuts, TVPI/DPI, capital account):
   - rewrite the cache (`c:numCache`/`c:strCache`) **and regenerate the embedded Excel from scratch** (don't edit the copied one) → no data from another company remains;
   - "last bar green": delete the inherited `c:dPt` entries and add one at the index of the last point.
6. **NAV bridge (waterfall/chartex)**: same pattern on `cx:chart` (cache + workbook). Variants: fund × currency (≤9 per cycle). ⚠️ The most uncertain part of step 3: 2-3 day spike **in week 1**. Plan B: classic stacked bar approved by IR.
7. **Images**: logos, J-curve/benchmark (per fund, from Finance), headshots. The `blip` target is replaced and the geometry is kept (round crop).
8. **SmartArt**: copied intact.
9. **Links**: the manifest maps each menu item to a **section**. At the end, section → first slide → new `r:id` is resolved. If the section isn't in the pack (e.g. no ESG), **the item is removed**. "Back to portfolio" → portfolio index slide.
10. **TOC**: with the final order, write the page numbers by code.
11. **Footer**: legal name, page number and date.
12. **Cleanup** (critical for the 31 PPTX): no notes (prep notes are never copied), no comments, no hidden slides, `docProps` reset (author, last modified by, title), no custom XML, unreferenced media removed (which also reduces file size).

## Step 4: Structural validation (before A's data checks)
- Every `r:id` resolves, there are no orphan parts, `[Content_Types].xml` is consistent.
- Validation against the **OOXML XSD**. PowerPoint is strict; LibreOffice is lenient, so it's no use as a test.
- Every animation target and every hyperlink exists.
- Every manifest slot was filled (no template placeholders like "Lorem" or "[Company]").
- LibreOffice render → PNG for the preview and to catch gross overflows (text outside its box), bearing in mind the font is a substitute.
- Sample per release: open N packs **in real PowerPoint** (IR's Mac or ours) → no "repair" dialog.

## Step 5: Data validator (round-trip, independent of the generator)
A file that opens fine doesn't prove the numbers are correct. The validator is **a separate module**: it shares no code with the assembler and **reads the final PPTX** the way an LP would.
1. **Extract** every visible and hidden value: shape and table text, chart caches, embedded workbook cells, alt text, footer.
2. **Re-map** each value to its slot via the manifest and **compare against the snapshot** (the source: administrator, submissions), not against the view model. If the view model was computed wrong, the error still shows up.
3. **Recompute invariants** independently:
   - sum of schedule rows = the administrator's **total investments at fair value** (not NAV, which also includes cash, other assets and liabilities); for an excused LP, fund total investments − excluded positions = the subtotal shown; the NAV bridge ends at the administrator's NAV. *(Corrected after a second review.)*
   - Vantrel Fund III company page = Ser B + Ser C;
   - chart cache == embedded workbook;
   - capital account reconciles (committed / paid-in / distributed / NAV share);
   - currency and exchange rate consistent across the whole pack (LP-0031 EUR, LP-0166 JPY).
4. **Coverage**: every number in the pack has a `source_ref`, and every piece of data the plan requires is present.
5. **Result** per pack: `pass` / `needs_human` with the exact list (slide, slot, expected vs found). This is the escalation flag from question 3.
6. **It runs again** when IR re-uploads the edited PPTX: if IR mistyped a number by hand, that's caught too.

A's leak scan (excluded companies) uses the same extractor: a single "PPTX reader", two kinds of checks.

---

## How it fails (what the brief asks for)
| # | Failure | Example with rows from B | Detection | Mitigation |
|---|---|---|---|---|
| 1 | **Template drift** (v8 renames) | The company page's "KPI_1" shape is renamed "Tile 1" | Manifest diff during onboarding | Block the new version; assisted re-mapping. Residual risk: a rename that **does** match the wrong shape → hence re-assembling an archive pack as a test. |
| 2 | **PowerPoint asks to "Repair"** | Duplicate ID when cloning 30 company pages in Fund III | XSD + ID check; PowerPoint test per release | Centralized ID generator. LibreOffice is **not** usable as an oracle. |
| 3 | **Text overflows** (autofit not recalculated) | LP-0031 footer; 120-word commentary in a box designed for 80 | Per-slot character budget + render | Budget at commentary generation; `short_legal_name`. ⚠️ Verify whether PowerPoint recalculates `normAutofit` on open. |
| 4 | **Chart with stale or foreign data** | Dunmarrow's revenue chart keeps Vantrel's numbers in its workbook (copied slide) | Workbook scan (A) | Always regenerate the workbook, never edit it. |
| 5 | **Wrong green bar** | Pellory has 5 quarters of history and the template has 8 → `dPt` at index 7 | Test: dPt == last index | Delete and recreate the `dPt`. |
| 6 | **Orphan animation** | Dunmarrow has 2 KPIs; 2 tiles are deleted and their fade stays in `<p:timing>` | Animation target check | Delete timing entries together with the shapes. |
| 7 | **Non-editable waterfall** | HRP III NAV bridge in EUR for LP-0031 | Week 1 spike | Plan B stacked bar or variants prepared by IR. |
| 8 | **Company aggregated wrong** | Vantrel B+C summed in Fund III but not Opps I (or vice versa) | Reconciliation: sum of company pages = schedule total per fund | Registry `company_id` + explicit fund in the view model. |
| 9 | **Bad editorial decision** | Company with 6 KPIs: the system picks 4 that IR wouldn't have chosen | Not detected automatically → IR during review | Per-company KPI priority editable by IR; remembered quarter to quarter. |
| 10 | **Misleading preview** | IR approves a LibreOffice preview with a substitute font; in PowerPoint the text wraps differently | — | Label the preview "approximate"; final review happens in PowerPoint (which they do anyway). |
| 11 | **Memory/time** | 150 MB × 97 Fund III packs in parallel | Worker monitoring | ZIP streaming, media by reference/hash, bounded concurrency. |
| 12 | **IR edits break the anchors** for later corrections | IR rewrites Dunmarrow's company page and then the restatement arrives | → see C (corrections) | Slots carry their own tag (in the shape name or as an XML extension) that survives normal editing. ⚠️ Verify what PowerPoint preserves when editing. |

## Claims this deep dive adds to the table
| Claim | Status |
|---|---|
| Slides and charts can be cloned by manipulating OOXML in Node without Office | Assumed (high) → week 1 spike |
| chartex (waterfall) can be edited the same way as a classic chart | **Assumed (low)** → week 1 spike |
| PowerPoint recalculates `normAutofit` on open | Assumed → verify |
| Shape tags/names survive IR's editing in PowerPoint | Assumed → verify |
| LibreOffice is not usable as an oracle for "opens fine in PowerPoint" | Verified by the spike (2/3) + general documentation; reinforce |
| XSD validation catches most "Repair" cases | Assumed |
| A v7 archive pack contains all the master's archetypes | Assumed (hidden slides and some variants may be missing) |

## Estimate (revised; see plan-draft.md)
The first version gave ~9 EW for B: it underestimated A. Trimmed to **~6 EW**: hand-written v7 manifest, no onboarding UI, waterfall decided in the spike.
