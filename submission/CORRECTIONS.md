# Corrections

All four moments come from one AI coding-assistant session (20–21 Nov 2026). I worked in Spanish, so each quote is given in the original with an English translation.

Evidence in `evidence/`:
- `excerpt-1-deep-dive-B-review.md` and `excerpt-2-allow-deny-lists.md` are verbatim transcript excerpts: the AI's message, my reply, and the AI's response.
- `original-file-lines.md` holds the lines the AI first wrote into its working files, pulled from the transcript's tool calls before they were edited.
- The full unedited transcript is attached separately.

---

## 1. The "independent" leak scanner checked the plan against itself

**What it proposed.** The entitlements design had "three independent barriers". An allow-list plan built each pack, and a leak scanner then searched the finished file for forbidden content. The AI described the scanner as "independiente del generador" (independent of the generator). However, the scanner's forbidden list was built from that same plan: *"Por pack se arma una lista de cosas prohibidas a partir del plan"* ("for each pack, a list of forbidden items is built from the plan").

**What was wrong.** The two barriers were not independent. Suppose a bug in the plan logic failed to apply LP-0102's Pellory excuse. Pellory would then be missing from the allow-list and from the forbidden list alike, so the scanner would pass a pack that leaks exactly what the project exists to prevent.

**How I knew.** I asked what the initial query would actually return:
> *"para que las barreras tengan sentido lo unico que serviria seria que esa consulta traiga lo que PUEDE y lo que NO PUEDE! … de lo contrario estarian validando lo mismo"*
> "For the barriers to make sense, that query has to return what the LP CAN see and what it CANNOT. Otherwise they'd be validating the same thing."

**What changed.**
- There are now two lists, built by separate code paths. The allow-list builds the pack. The deny-list comes from a small function that reads only the restrictions (excuses, joinder × restricted companies, FOI) and feeds the scanner.
- Before assembly, a cross-check requires the lists to be **disjoint** (any overlap means a plan bug and blocks the pack) and **complete** (every holding × field is classified).
- "Bug in plan logic" became an explicit failure mode in Deep dive 1.

*Evidence: `excerpt-2-allow-deny-lists.md`; `original-file-lines.md` (deep-dive-A.md, L7 and L93).*

---

## 2. The LLM kept creeping back into deterministic work

**What it proposed.** I had framed the LLM as being for commentary only (*"la ai se usaria solo para generar el commentary"*). Even so, the AI wrote two more uses into the design:
- In the template-onboarding step: *"Para lo ambiguo, el LLM propone"* ("for ambiguous cases, the LLM proposes" the mapping from template shapes to data fields).
- In the cost estimate, a line item for "data assistance": matching uncertain company names and suggesting which 4 of 6 KPIs to show.

The architecture note had also widened the principle to *"y, como mucho, para asistir el onboarding de templates"* ("and at most to assist template onboarding").

**What was wrong.** A wrong slot mapping is not a one-off error. It puts a number in the wrong place in all 214 packs, every quarter, until someone notices. Company-name matching is the step on which excuse enforcement depends, so it has to be deterministic and fail closed. The LLM was also unnecessary: about twelve slide archetypes can be mapped by hand in a day.

**How I knew.**
> *"me parece que el llm en el paso 0 sobra. y romperia la regla de solo usarlo para commentary"*
> "The LLM in step 0 is unnecessary, and it would break the rule of using it only for commentary."

**What changed.**
- The LLM was removed from template mapping and from entity resolution.
- For Q4, the v7 manifest is hand-written JSON, reviewed in a pull request and accepted only if it reproduces an archived pack.
- Company aliases are deterministic, with IR confirming them once.
- Rows D and F were deleted from the cost estimate.
- The principle now reads "LLM writes prose only".

*Evidence: `excerpt-1-deep-dive-B-review.md`; `original-file-lines.md` (architecture-draft.md L3, cost-estimate.md rows D and F, deep-dive-B.md L23).*

---

## 3. The plan treated entitlements as "whatever is left"

**What it proposed.** The first estimate for the template and assembly work came to about 9 of the 15 build weeks (*"Total B ~9 de 15"*). Entitlements were never estimated; they sat inside "the ~6 that remain" alongside ingest, commentary, review and corrections. The AI did notice that 9 weeks was a lot. Its proposed fix, however, was to trim assembly features, and it still did not budget entitlements.

**What was wrong.** Entitlements are the legally consequential core: the incident that started the project was an excused LP receiving company information. A plan that funds them from leftovers would arrive at week 9 with the riskiest part under-built.

**How I knew.**
> *"aunque se que el template y el armado son mucho mas dificiles, hablando de planificacion por semanas, estamos subestimando el A que abarca permisos y reglas para mostrar/ocultar. siento que este es el core del producto y legalmente es lo que mas impactaria"*
> "I know the template and assembly are much harder, but in the week-by-week plan we're underestimating A, the permissions and show/hide rules. It's the core of the product and legally the most consequential."

In the same message I asked to build the data and filtering work before the template work.

**What changed.**
- Entitlements got their own 3.5 engineer-weeks: the rules model and lists (2) and the reader, scanner and validator (1.5). Assembly was cut to about 6, using a hand-written manifest and no onboarding UI.
- The build order now puts template-independent work first, with only a one-week template risk spike in parallel.
- The entitlement report became a standalone fallback for Q4.

*Evidence: `excerpt-1-deep-dive-B-review.md`; `original-file-lines.md` (deep-dive-B.md L109).*

---

## 4. "Opens without errors" was being treated as "correct"

**What it proposed.** The assembly deep dive ended with structural validation only: relationships resolve, the schema validates, there are no orphan animations, and a sample opens in PowerPoint (*"Paso 4: Validación estructural (antes de los checks de datos de A)"*). Correctness of the numbers was deferred to checks that lived in a different section and were never wired into the assembly flow.

**What was wrong.** A file can open perfectly and still show Vantrel's Fund III fair value as 14.25m instead of 17.35m (Series B only), or carry another company's figures in a chart workbook. Nothing in the flow read the finished deck back and compared it with the source data.

**How I knew.**
> *"falta agregar una herramienta de validacion para confirmar que la data generada sea la correcta!"*
> "We still need a validation tool to confirm that the generated data is correct!"

**What changed.** A round-trip validator was added as a separate module that shares no code with the assembler.
- It extracts every value from the finished PPTX: text, tables, chart caches, workbook cells and footer.
- It compares those values with the snapshot, not with the assembler's intermediate model.
- It recomputes invariants such as schedule totals against NAV, Series B plus Series C, and chart cache against workbook.
- It returns `needs_human` with slide, slot, expected and found.
- It runs again on every file IR re-uploads.

The leak scanner now reuses its reader.

*Evidence: `excerpt-1-deep-dive-B-review.md`; `original-file-lines.md` (deep-dive-B.md L63).*

---

## Not counted (not valid): misreading two of my questions

Kept for completeness and marked not valid. It was a misinterpretation of my intent, not a technical error, and the AI's critique was reasonable on a literal reading.

**What happened.** The AI reviewed my five draft questions. It recommended dropping question 3 ("are they reviewing numbers or mostly the cover note and layout?") because the stakeholder quotes already answer it. It recommended dropping question 4 ("what will we have to build and test against?") because the archive is listed as available.

**My clarification.**
> *"con la #3 me refiero a que necesito saber si ellos validan los resultados o solo como luce … necesito un flag que escale a un humano cuando algo no cuadre. y con la #4 … como validamos que nuestros resultados esten ok? data de prueba? mock?"*
> "With #3 I need to know whether they validate the results or only the look. I need a flag that escalates to a human when something doesn't add up. With #4: how do we validate our results are right? Test data? Mocks?"

**What it led to.** The five questions stayed as sent. My clarification produced the `needs_human` escalation design and the testing strategy: the archive as a golden set, synthetic fixtures and a shadow run.

*Evidence: `evidence/excerpt-0-questions-not-valid.md`.*
