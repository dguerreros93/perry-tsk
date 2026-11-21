# Glossary

## Terms given in the brief (with expanded explanation)
| Acronym | What it is | Why it matters for the design |
|---|---|---|
| **LP** (Limited Partner) | Investor in the fund (pension fund, insurer, family office, fund-of-funds). | Unit of personalization. 187 LPs, 214 LP×fund combinations = 214 packs. |
| **GP** (General Partner) *(doesn't appear, implied)* | The manager: Halden Ridge. The "partners" who sign the cover note. | They are the ones who rewrite cover notes. |
| **Side letter** | Private fund↔LP agreement that adds rights/restrictions on top of the main contract (LPA). | Source of the rules: what each LP can/can't see. Perry already has it extracted and **verified by fund counsel**. |
| **Portfolio company** | Startup the fund invested in. 58 in total, 7 in two funds. | Each one generates 1-2 slides. |
| **NAV / fair value** | Estimated value of the holdings at quarter end. Signed off every quarter. | The "official" number; if it's wrong → the LP's lawyer on the phone. |
| **NAV bridge** | Explains how the starting NAV got to the ending NAV (contributions, distributions, gains/losses). Drawn as a **waterfall chart**. | Difficult chart (see chartex below). |
| **Capital account** | Position of ONE LP: committed, paid-in, returned (distributed), its share of the NAV. | Unique per LP. |
| **TVPI** | Total Value to Paid-In: (distributed + NAV) / paid-in. | Only displayed; no need to calculate it. |
| **DPI** | Distributions to Paid-In: how much cash has already been returned / paid-in. "Pensions want the DPI story". | |
| **IRR** | Internal rate of return. | |
| **Excuse right** | The LP is left out of a specific investment: puts in no money, has no exposure. | 11 LPs. Their pack **must not contain anything** about that company. This was the incident 2 quarters ago. |
| **IR team** | Investor Relations: 3 people who currently build the packs by hand (~3 weeks). | Primary user. They review and touch up in PowerPoint. |
| **Fund administrator** | External firm that keeps the books and produces the official numbers (Excel). | Numerical source of truth. Signs off ~4 business days before the deadline; post-sign-off corrections in 3 of 4 quarters. |

## Domain terms that appear without definition
| Term | What it is |
|---|---|
| **Fund-of-funds (FoF)** | Fund that invests in other funds. They "lift slides" for their own reports → that's why they want the editable PPTX (31 LPs). |
| **Reduced disclosure / FOI** | Freedom-of-information: public bodies can be required to publish what they receive. That's why they only receive fund aggregates, with no per-company valuations/financials. 17 LPs. |
| **Confidentiality joinder** | Signed accession to a confidentiality agreement. Without it, financials for the 9 "restricted companies" are not received. 140/187 have signed. |
| **ESG annex** | Environmental/social/governance annex. 26 LPs. |
| **Look-through annex** | "Looking through" the fund: the LP's exposure by sector and geography at the level of the underlying companies. 12 LPs (mostly insurers, due to regulatory requirements such as Solvency II). |
| **Restatement in EUR/JPY** | Converting fund and capital account figures into another currency at the quarter-end exchange rate. 14 LPs. |
| **J-curve** | Typical fund performance chart (negative at first, rising later). Here it's an **image** sent by Finance. |
| **Benchmark comparison** | Comparison against indices/peers (e.g. Cambridge Associates). Also an image. |
| **Share series (Ser B PFD, Ser C PFD, Seed, Conv Note)** | Rounds/instruments: Series B, C preferred; seed; convertible note. A company can have several lines in the schedule (Vantrel has B and C in Fund III). |
| **Written off** | Value taken to 0 (Ångström Q2). |
| **Restated** | Figure corrected/restated afterwards (Dunmarrow, 14 Oct). |
| **ARR / NRR / runway / GMV / take rate** | Startup KPIs: annual recurring revenue, net revenue retention, months of cash, gross merchandise value, % the platform keeps. |
| **Stichting / W.L.L. / ApS / LLC** | Legal forms (NL foundation, Bahrain/Kuwait, Denmark, US). Relevant for the "Prepared for <legal name>" footer (very long names, special characters). |

## Relevant technical terms (PowerPoint / OOXML)
| Term | What it is | Why it matters |
|---|---|---|
| **OOXML / PPTX** | A .pptx is a ZIP of XMLs (slides, layouts, masters, charts, media, notes). | It can be manipulated without Office. |
| **Slide master / layout** | Base templates that slides inherit from (green banner, side menu). | Ideally the design lives here. |
| **Placeholder / shape name / alt text** | Named elements on a slide. | A way to "anchor" data without depending on positions. v6→v7 **renamed things**. |
| **Native chart + embedded workbook** | "Real" charts carry an embedded Excel (hence "right-click → edit data"). | Both the chart XML **and** the embedded Excel have to be rewritten. ⚠️ Data leak if old data is left in the workbook. |
| **chartex (Office 2016 charts)** | Waterfall, sunburst, etc. use a different schema (`cx:chart`), distinct from classic charts. | The NAV bridge is a waterfall → harder; poor support in libraries and in LibreOffice *(to be verified)*. |
| **SmartArt** | Diagrams with their own data model + cached drawing. | "We never touch them" → copy as is. LibreOffice renders them imperfectly *(to be verified)*. |
| **Autofit ("shrink text on overflow")** | PowerPoint recalculates the font size on open/edit. | The file stores a `fontScale`; a generator doesn't recalculate it → overflowing text in the PDF. |
| **Internal hyperlinks / action buttons** | Side menu and "Back to portfolio" point to specific slides. | They break when slides are deleted/duplicated. |
| **Hidden slides / speaker notes** | Hidden slides ("old versions") and partner prep notes. | ⚠️ In the 31 delivered PPTX files = information leak. |
| **Animations / transitions** | KPI tiles with fade, sections with "slide across" (Morph/Push). | They have to be preserved in the PPTX; in the PDF they don't matter. |
| **Font embedding** | Hartwell Sans is only installed on 5 laptops. | Rendering on a Linux server without the font = "doesn't look like the agency's". Font license for server use/embedding = legal question. |
| **python-pptx** | Python library for PPTX. | Violates "no 2nd runtime". |
| **LibreOffice (soffice headless)** | Already runs in their containers for conversion. | Preview ≠ PowerPoint fidelity. The spike validated with LibreOffice, not with PowerPoint in every case. |
| **Microsoft Graph PDF conversion** | M365 cloud API that converts PPTX→PDF with the Office engine. | Possible way around "no Office in our infra" *(to be verified: support, size limits ~150MB, customer license)*. |
