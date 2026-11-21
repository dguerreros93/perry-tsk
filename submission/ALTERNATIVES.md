# Alternatives

Three approaches I considered seriously, all under the same constraints: no Windows or desktop Office in our infrastructure, Linux containers, a TypeScript monorepo, and a deck that must look like the agency made it.

## 1. Let the LLM edit the slide XML (the spike's approach)

Give the model a slide's XML plus the data, take edited XML back, and stitch the deck.

**For:** it is fast to prototype and copes with an unfamiliar template without mapping work.

**Against:** a probabilistic model rewrites every number in every pack, which is the failure the Head of IR cannot accept. Nothing is reproducible, so we could not show an LP's counsel why a pack came out as it did. The spike got two of three slides right. Even at 99% per slide, about a third of 40-slide packs would contain a broken slide. "Opened fine" was also judged in LibreOffice, which neither IR nor the LPs use.

**Cost is not the argument.** At about 10,700 slides per cycle and roughly 25k tokens in and 25k out per slide, this costs about $6,400 on Opus 5.5 but only about $160 on Haiku 5.5, well inside the $1,500 price. It fails on correctness, not on price.

## 2. Rebuild the deck in code from scratch (e.g. PptxGenJS)

Recreate the house style programmatically instead of starting from the agency's file.

**For:** it is fully deterministic, and there is no inherited hidden content.

**Against:**
- It will not look like the agency made it. SmartArt, the waterfall, masters, transitions and the photo treatment must all be re-implemented and kept pixel-matched.
- Each of the twice-yearly template revisions becomes an engineering project.
- It rules out "customers upload their own template".

## 3. Chosen: agency template plus manifest, deterministic OOXML assembly, independent validator

The agency's .pptx stays the source of design. Once per template version, a manifest records the slide archetypes and the data slots in each. Assembly clones archetypes and fills them with values that are already computed and formatted. It regenerates chart caches and their embedded workbooks, paginates, rewires links and the table of contents, and strips notes, hidden slides and metadata. A separate reader parses the finished file and checks it against the source data and against a deny-list built independently of the plan. The LLM writes prose only, never numbers or XML.

**Why:** the pack keeps the agency's design, every number is traceable to its source, and a new template version becomes a review step rather than a rewrite. The Q1 "own template" customers start from that same onboarding step.

**Cost:** we own a sizeable OOXML layer, including the waterfall (chartex), the least certain part. A week-1 spike decides between editing chartex and a classic stacked-bar waterfall approved by IR.

### Language: TypeScript, with Python as a fallback

A Python document service has a real case: python-pptx and openpyxl are more mature than the Node equivalents. I rejected it for now:
- Data volume is not the bottleneck: 214 packs per cycle, from spreadsheets of a few hundred rows.
- Python does not solve the hard parts. Chartex, rendering fidelity and entitlements are just as hard there.
- A second runtime adds a second CI pipeline and container image, types duplicated across the boundary, and an on-call team that must know both stacks.

If the week-1 spike shows a TypeScript OOXML gap that Python closes, that is the "very good reason" the Head of Engineering asked for. The assembler would then move behind the queue as an isolated worker.
