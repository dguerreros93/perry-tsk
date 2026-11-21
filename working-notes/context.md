# What we know today: "what we have" (§4) and "what we were told" (§5)

Convention: ✅ = in the brief · 🔧 = technical fact (certain) · ⚠️ = technical fact that needs to be **verified** before it goes into the claims table · 💡 = design implication.

---

## 0. First: what a PPTX is on the inside (we need this for everything else)

🔧 A `.pptx` is a **ZIP** of folders of XML files. If you rename `deck.pptx` to `deck.zip` and open it, you see something like this:

```
ppt/
  presentation.xml        ← list and order of slides
  slides/slide1.xml ...   ← content of each slide (text boxes, images, tables)
  slides/_rels/           ← what each slide "points to" (images, charts, links to other slides)
  slideLayouts/, slideMasters/   ← the base template (green banner, side menu, footer)
  notesSlides/            ← the notes below the slide
  charts/chart1.xml       ← chart definition + a COPY of the numbers used to draw it
  embeddings/Microsoft_Excel_Worksheet1.xlsx  ← the Excel that opens with "Edit data"
  media/image1.jpg ...    ← photos (hence the 150 MB)
docProps/                 ← metadata: author, last editor, dates
```

💡 Generating a pack = copying that ZIP and modifying XMLs in a controlled way. You don't need PowerPoint to **create** the file. You do need something that **renders** it to produce the PDF or an image, and that's where the hard part begins.

---

## 1. What we have (§4)

### 1.1 Master template `HRP_Quarterly_Master_v7.pptx` (not shared; only the notes in Appendix A)

**What it is:** the 64-slide "master" deck the agency built. IR duplicates it and trims it down for each LP.

Every detail from Appendix A, translated into what it means technically:

| What the IR associate said | What it is technically | Why it hurts |
|---|---|---|
| "64 slides, 150 MB, it's the photos" | 🔧 Images in `ppt/media/`. | 214 packs × 150 MB ≈ **32 GB per cycle** if you copy everything. Slow to generate, upload and download. 💡 Images can be recompressed or reused. A pack is already smaller because slides get deleted. |
| "Hartwell Sans… on another machine it looks wrong" | 🔧 Fonts **don't travel** inside the PPTX unless they are embedded. If the machine doesn't have the font, it is substituted with another one with different metrics, and the text shifts, wraps or overflows. | Our Linux servers don't have it. Installing it there probably requires **another license** (the agency put it on 5 laptops, typical of a per-device license). ⚠️ Verify whether PowerPoint 365 Mac/Win allows embedding it and whether the license permits it. |
| "Green banner with background and quarter, menu on the left, clicking takes you to the section" | 🔧 The banner and menu probably live in the **layout/master**. Each menu item is an **internal hyperlink** pointing to a specific slide (via an internal relationship, not by number). | If I delete the "ESG annex" section for an LP that doesn't receive it, the menu link ends up **pointing at nothing** (broken link), or that item has to be removed. Same with "Back to portfolio". |
| "Index with page numbers, we fix them by hand" | Plain text, not an automatic field. | The generator has to **paginate first** (know how many slides come out) and **then** write the index. |
| Footer "Prepared for…", page number and date | Footer placeholders. | Names like "Stichting Bedrijfstakpensioenfonds voor de Noordelijke Scheepsbouw…" don't fit on one line. |
| "We copy the previous company's page and type over it" | There is no "clean" company page in the template. The de facto template is "the last one that was used". | We have to define **one** canonical company page (model slide) and clone it N times. v6→v7 rebuilt it, so the mapping changes between versions. |
| "4 KPI boxes that appear one by one" | 🔧 Animations are stored in a `<p:timing>` block on the slide that references the boxes **by their internal ID**. | If IDs change on regeneration or I delete a box (company with 3 KPIs), the animation **points to a box that doesn't exist**. ⚠️ PowerPoint's behavior in that case: verify (it may ignore it or ask to "repair file"). |
| "2 to 7 KPIs; if there are 3 we stretch, if there are 6 we pick" | Layout rules (resizing boxes) and an **editorial decision** (which 4 of 6). | Who picks? Fixed rule per company, LLM, or deal team. It's a product question, not a technical one. |
| "The revenue chart is a real chart, right-click edit numbers. Bars + line. We color the last bar green" | 🔧 A native chart stores the numbers **twice**: (a) a cache in `chart.xml` that PowerPoint uses to **draw**, and (b) the embedded Excel that opens when you edit. The color of a single bar is **per-point formatting** (`dPt`). | Both have to be updated and they have to match. If you only update the cache, the chart looks right, but the LP who opens "Edit data" sees **another company's or another LP's numbers** (those from the slide you copied) → **data leak**. The green formatting is tied to the point index: if the number of quarters changes, the green lands on the wrong bar. |
| "Long text shrinks on its own, almost always" | 🔧 "Shrink text on overflow": PowerPoint **measures** the text with the real font and stores a scale factor in the XML. | A generator **cannot measure** text without the font and a layout engine. If I write 120 words where there were 80, the XML keeps the old scale. Depending on who renders it, it overflows. ⚠️ Verify whether PowerPoint recalculates on open (I believe it does on edit; on open I'm not sure). |
| "NAV bridge is the waterfall chart" | 🔧 Waterfall is a **newer** chart type (Office 2016+), with a different XML format (`chartex`), unlike classic bars, lines and doughnuts. | Few libraries support it. ⚠️ LibreOffice probably **doesn't render it** (shows a placeholder or nothing). Verify: this is key for the PDF. |
| "Two doughnuts, a TVPI/DPI line chart" | Classic charts. | Same cache + embedded Excel pattern. Manageable. |
| "J-curve and benchmark are pictures, Finance sends them" | Images. | **A data source that isn't on the §4 list.** How do they arrive? Per fund? Per currency? Does an EUR LP get sent the J-curve in USD? |
| "Fund structure and timeline are SmartArt, we don't touch them" | 🔧 SmartArt = data model + a "cached drawing". | If we don't touch them, they are copied as is. ✅ Low risk. ⚠️ Rendering in LibreOffice may be imperfect. |
| "Schedule fits 14 rows per slide; Fund III is 3 slides; negatives in red in parentheses; totals row" | 🔧 PowerPoint tables **don't paginate on their own**. | The generator splits the table into blocks of 14, puts the totals row only on the last one and applies the negatives formatting (done by hand today). If I remove rows for an excused LP, the pagination changes **and so does the total**. |
| "Capital account: small table + bar chart" | Table + chart. | Data unique per LP. In the LP's currency. |
| "There are hidden slides at the end, old versions, we leave them" | 🔧 Slides with `show="0"`. **They're still in the file.** | The **31 LPs who receive PPTX** can see them. They may contain old data or data from excluded companies. 💡 Always remove them. |
| "Prep notes for partner calls below the slide" | 🔧 `notesSlides/` | Same risk, worse: internal notes for the partner ("this LP is upset about…"). 💡 Always delete notes from the output. |
| "Section pages with a full-bleed photo and 'slide across'" | Transition (Push or Morph). | Preserved in the PPTX. Irrelevant in the PDF. |
| "Round headshots" | Image cropped to an ellipse shape. | If we change the photo, the crop has to be preserved. |
| "Disclaimer the same for everyone, except 2 Swiss LPs with an extra paragraph" | Conditional variant. | **Not among the B3 flags.** Where does the rule come from? Probably a side letter clause that would need to be mapped. |
| "The agency revises it ~2 times a year; v6→v7 renamed things and redid the company page" | — | Any "by box name" mapping breaks with every version. 💡 We need an **onboarding step per template version** with validation ("are all the pieces I expect there?") before generating packs. |

### 1.2 Fund administrator export (quarterly Excel)

**What it contains:** capital accounts per LP, schedule of investments, cash flows, NAV. It is the **source of truth for the numbers**.

- ✅ It is signed off **~4 business days before** the contractual deadline. That is the entire window: generate, review 214 packs, have the partners rewrite ~20 cover notes, export and deliver.
- ✅ In **3 of 4 quarters** there was at least one valuation corrected **after** sign-off. Corrections have to be assumed as normal.
- Looking at B1:
  - Rows are per **instrument**, not per company ("VANTREL HEALTH INC - SER B PFD"). The company page aggregates several rows. We need to know that "VANTREL HEALTH INC" = "Vantrel Health" (B2) → **company identity table** (entity resolution).
  - Names in legal UPPERCASE with Unicode (ÅNGSTRÖM & SØN APS).
  - Free-text "Note" column ("Written off Q2", "Restated 14 Oct"): important information in an unstructured field.
  - Everything in USD. EUR/JPY LPs need an exchange rate. Where does it come from? It's not on the list. *(Assumption: the administrator provides it.)*
- 💡 It's an Excel from an external firm, so the format can change without notice. We need a **parser with schema validation** that fails clearly ("column X is missing") instead of misreading.

### 1.3 Portfolio company submissions (web form + attachments)

**What it contains:** KPIs reported by each startup, plus board decks and management accounts.

- ✅ **Different** KPIs per company, **different currencies**, **different fiscal calendars**. 10-15% don't submit on time.
- Looking at B2:
  - Vantrel: ARR in "USD m" (millions), NRR in %, runway in months → 3 KPIs.
  - Dunmarrow: Revenue in **absolute GBP** (2,140,000) → shown in GBP or converted? Period "Jul-Sep 2026" (another text format for Q3).
  - Pellory: "FY27 Q1 (Aug-Oct)": its quarter **doesn't match** calendar Q3. Submitted on **2 Nov**, probably after sign-off.
  - Ångström: didn't submit, and is also at 0 (written off).
- 💡 KPIs are shown "as the company reports them", with their unit and period, and a note when the period doesn't match. We don't normalize them (converting a KPI's currency or calendar would be inventing data).
- 💡 The revenue trend chart needs **history** (several quarters). Where does it come from? From previous submissions or from the archive.

### 1.4 Deal-team notes (bullets in Notion)

**What it contains:** informal notes per company. It is the raw material for the **80-120 words of commentary**.

- It's the only piece where an **LLM clearly makes sense**: turning bullets into prose in the house tone.
- Risks: the LLM **makes up** a fact, cites a number that doesn't match the administrator, or the bullets mention **another company** (e.g. "Pellory competitor"), which would leak to an LP excused from Pellory.
- 💡 Commentary is generated **per company (and per LP type)**, not per pack: 58 companies × a few variants. It's cheap and reviewable once.
- ⚠️ Notion access: API. Is there already an integration in Perry? Assumed not.

### 1.5 Perry's side-letter data (structured, verified by counsel)

**What it contains:** per LP, rights and restrictions already extracted. It is the **rules engine**.

- ✅ It already exists and fund counsel verifies it. We don't build it, we **consume** it.
- Looking at B3: the flags are human-readable ("Reduced disclosure… Side letter s.4.2: '…'"). Is there a structured form underneath (e.g. `excused_companies: [id]`) or is it text? *(Assumption: structured, with company IDs.)*
- 🚩 LP-0058: two conflicting rules, and the verified data contains it anyway. Verified ≠ consistent.
- Excuse rights name companies ("Pellory Payments"): again, entity resolution against B1/B2.

### 1.6 The archive (8 quarters of hand-made packs, PPTX + PDF)

**What it gives us:**
- 🔧 Every pack in the archive is a **derivative of the real template**. It carries the real masters, layouts, charts and SmartArt. Even if they don't give us the master, we have its structure.
- Real examples of IR decisions: which KPIs they picked out of 6, how they stretched 3, how they handled companies without a submission, what the cover notes ended up looking like.
- **Golden data** for tests: "LP-0102's Q3 pack exactly as it went out".
- House commentary style → examples for the prompt.
- ⚠️ It's also proof that there were errors (the excuse incident). Not all of the archive is "correct".
- ⚠️ It mixes v6 and v7: they need to be separated.

---

## 2. What we were told (§5): what they say and what's behind it

### Head of IR
> "Press a button… ready that afternoon. If it doesn't look like our agency made it, I can't send it. If a number is wrong, I'm on the phone with the LP's counsel."

These are **three requirements**, in order of severity:
1. **Correct** (non-negotiable): no wrong number, no prohibited data.
2. **Faithful** to the brand: it has to look like the agency's work, which implies font, real charts and layout.
3. **Fast**: generation in hours, not days.

Sizing numbers: 214 packs, 3 people, ~4 days → ~70 packs per person, ~18 per day, **~25 min per pack** to review everything. 💡 IR cannot review 214 packs thoroughly. Automated validations have to do the heavy lifting and the UI has to say **where to look**.

### IR associate
> "We'll still open decks in PowerPoint and tweak them…"

💡 The output **is not a final PDF**: it's a **working PPTX** that a human modifies. Consequences:
- The file has to open **perfectly** in PowerPoint (no "PowerPoint found a problem with content… Repair").
- After IR's edits, the system no longer knows what's in the file. If there's a numbers correction, how do we apply it without overwriting their changes? (Q5)
- If IR edits, they can also **introduce** an error (e.g. paste in an excused company). Do we re-validate after their edit? 💡 Yes: final scan before sending.

### Head of Product
> "From Q4 commentary tuned to LP type… two more customers for Q1 upload their own template and it just works."

- **Commentary per LP type in Q4**: technically cheap (prompt variants). The risk is about **quality and review**: more variants = more text someone has to review. And "pension funds want the DPI story": DPI is a fund-level number, so the text has to cite correct numbers.
- **"Upload your own template and it just works" in Q1**: that's **another product**. Our design for Halden Ridge depends on knowing their template. For an arbitrary template, the system would have to **understand** which box is which. 💡 Design template onboarding as an explicit step (a person or an LLM proposes the mapping, it's validated, it's approved) so it's reusable. "Just works" with no human doesn't fit in 15 eng-weeks.
- Note: neither of the two is a Halden Ridge requirement for Q4 itself. They're **commercial promises**.

### Head of Engineering
> "No Windows servers and no desktop Office running unattended… very good reason before a second language runtime."

What it rules out:
- ❌ Automating real PowerPoint (which would give perfect fidelity).
- ❌ python-pptx (Python = second runtime; the stack is TypeScript). Same for Java/.NET libraries (e.g. Aspose, which is commercial and used via Java or .NET).
- Gray area: **Microsoft 365 cloud APIs** (upload the PPTX to OneDrive/SharePoint and request PDF conversion via Microsoft Graph). It's the Office engine, but it runs on Microsoft's infra, not ours. ⚠️ Verify that the API supports PPTX→PDF, its size limits (150 MB?), whether it requires a customer tenant/license, and what happens with custom fonts (it probably doesn't have Hartwell Sans → same problem).
- ✅ Allowed: LibreOffice (already runs in their containers).

### Engineer, after a two-day spike
> "3 company slides… gave Claude the XML plus data, asked for edited XML back, stitched with python-pptx. 2 of 3 opened fine… previewed with LibreOffice. Most of the way there."

This one has to be taken apart carefully. It's the quote **designed to see whether you buy it**:
1. **2 of 3 = 67% per slide.** A pack has 30-75 slides. If each slide independently had a 67% chance of coming out right, the probability of a 40-slide pack coming out entirely right is ~0.67⁴⁰ ≈ **0.000001%**. Even at 99% per slide: 0.99⁴⁰ ≈ 67% of packs without a failure → ~70 broken packs per cycle.
2. **The LLM rewrites the numbers.** A probabilistic model copying figures = exactly the "phone with LP's counsel" scenario. And it's not reproducible: two runs can give different results.
3. **"Opened fine"** only proves the file isn't corrupt, not that the numbers or the "Edit data" data are right.
4. **He checked it with LibreOffice**, not PowerPoint. LibreOffice isn't what IR or the LPs use. Looking right in LibreOffice proves nothing about PowerPoint, and vice versa.
5. **python-pptx** violates the runtime constraint.
6. **He picked company slides**, which are the easy ones. He didn't test waterfall, hyperlinks, pagination or filtering.
7. **Cost** ⚠️ (estimate, verify): the XML of a slide with a chart can be tens of thousands of tokens. A round trip for ~10,000 slides per cycle is hundreds of millions of tokens, which could eat a significant share of the $1,500.

💡 What's salvageable: it confirms that **manipulating the XML directly** is viable and that LibreOffice is available. The right idea is **deterministic XML manipulation in code**, with the LLM only where there's natural language.

---

## 3. Numbers to keep at hand
| Item | Value | Implication |
|---|---|---|
| Packs per cycle | 214 (97 F-III, 62 F-II, 55 Opps) | Small volume: it's not a scale problem, it's a **correctness** problem. |
| Slides per pack | 30-75 | ~10,000 slides per cycle. |
| Holdings | F-III 31, F-II 20, Opps 14 | F-III schedule = 3 slides. |
| Window | ~4 business days from sign-off | Generation in hours; the rest for review. |
| Review | ~25 min available per pack | Strong automated validation needed. |
| PPTX delivery | 31 LPs | Full file cleanup. |
| Excuse | 11 LPs | Zero-tolerance filtering. |
| FOI | 17 LPs | "Aggregates only" variant. |
| Joinder | 9 restricted companies; 47 LPs not signed | Filters financials, not the whole company (assumption). |
| ESG / look-through | 26 / 12 | Conditional annexes. |
| FX | 14 LPs (EUR, JPY) | Closing exchange rate. |
| Personalized cover note | ~20 LPs | Workflow with partners. |
| Price | $1,500 per cycle ≈ $7 per pack | All variable cost has to fit in there. |
| Build | ~15 eng-weeks | Q4 in weeks 9-10. |

---

## 4. Gaps: things the brief doesn't say (candidates for assumptions)
- Where the exchange rates, Finance's images (J-curve) and the revenue history come from.
- Where the Swiss LPs rule comes from.
- Whether the side-letter data includes company IDs or only names.
- How delivery to the LP portal works today (API? manual upload?).
- Who picks the 4 KPIs out of 6.
- Whether restricted companies without a joinder lose the whole page or only the financials.

---

## 5. Clarifications (session 2026-11-20)

### 5.1 Python as a 2nd runtime? (pending, candidate for ALTERNATIVES.md)
- My position: it's justified by data-processing speed and the analysis ecosystem.
- Counterpoint: the volume is small (214 packs, Excels with hundreds of rows), so in Node speed isn't the bottleneck. The strong argument for Python would be the **maturity of python-pptx/openpyxl** for OOXML, not data analysis. But python-pptx doesn't support waterfall (chartex) either ⚠️ verify.
- Cost of adding it: another container image, another CI, another type system that has to be kept in sync with the TS models, and on-call has to know two stacks.
- Decision: deferred. It has to be presented as an explicit trade-off, not as a default.

### 5.2 Why the waterfall is special
- Technically it's **not a chart like the others**. Bars, lines and doughnuts use the classic format (`c:chart`, Office 2007). Waterfall, histogram, treemap, sunburst, box & whisker and funnel use **chartex** (`cx:chart`, Office 2016), a different XML schema with a different structure. The file stores it inside an `mc:AlternateContent` block with a "fallback" for older readers.
- Consequences: ⚠️ (all to be verified)
  - Generation libraries (python-pptx, PptxGenJS) don't create or edit it; the XML would have to be written by hand.
  - LibreOffice probably doesn't render it and shows the fallback or nothing, which hits the PDF directly.
- Nuance that lowers the risk: the NAV bridge is **fund-level**. For USD LPs it's identical within each fund. Real variants: 3 funds × (USD, EUR, JPY) ≈ up to 9 charts per cycle, not 214 (if the aggregates don't change due to excuse rights, see assumption in questions.md).
- Options: (a) edit the chartex by hand, with tests; (b) emulate it with a **classic stacked bar** with an invisible base series (standard trick): editable everywhere and easy to generate, but the look changes slightly and IR/the agency has to approve it; (c) generate the ≤9 variants once per cycle (or have IR update them by hand) and insert them.

### 5.3 "The font" = the Hartwell Sans font family
- Yes: the typeface. `.ttf`/`.otf` files installed in the operating system.
- The PPTX only stores the **name** ("Hartwell Sans"), not the font. Whatever renders the file (PowerPoint on a laptop, LibreOffice in a container) looks for that font on its machine. If it isn't there, it substitutes another one with **different metrics** (character width, line spacing): text wraps, overflows the boxes, and autofit is calculated differently.
- The PDF embeds the font used by whatever rendered it. If the container didn't have Hartwell Sans, the PDF comes out with the substitute: "doesn't look like the agency's".
- Three paths: (1) install Hartwell Sans in the container → requires a license that allows server use; (2) embed the font inside the PPTX (PowerPoint allows it if the font file permits it in its `fsType` field) ⚠️ verify support in Mac 365 and in LibreOffice; (3) have IR export the PDF from their laptops, which already have it.
- Hartwell Sans is fictional: for testing we use a proxy font with metrics different from the fallback's so that overflows are noticeable.

### 5.4 Firm constraint: no Windows or Office in OUR infra
Exact quote: *"No Windows servers and no desktop Office running unattended **in our infrastructure** this year."* What it allows and what it doesn't:

| Option for "rendering" the PPTX (PDF / preview) | Does it violate the constraint? | Note |
|---|---|---|
| PowerPoint on a Windows server of ours | ❌ Yes | Ruled out. |
| Headless LibreOffice in our Linux containers | ✅ No | Already in use. Fidelity to be verified (font, waterfall, autofit, SmartArt). |
| Microsoft 365 cloud conversion (Graph API) | 🟡 Gray area | Runs on Microsoft's infra, not ours. We need to ask the Head of Eng whether that counts as a "very good reason" or as a violation. ⚠️ Limits and fonts to be verified. |
| PowerPoint on IR's laptops, with a person in front of it | ✅ No | It's neither our infra nor unattended. It already happens today. |
| Our own rendering engine in TS | ✅ No | Not feasible in 15 eng-weeks. |

Conclusion: the constraint doesn't prohibit a high-fidelity PDF, it prohibits **us** producing it **with Office**. The real decision is: LibreOffice (ours, questionable fidelity), M365 cloud (high fidelity, gray area), or IR exports (perfect fidelity, manual step).
