# Original file contents written by the AI (extracted verbatim from the session transcript's tool calls)

Source: the working session transcript. Timestamps UTC. Files were later edited after the corrections; these are the first versions.

#### Write architecture-draft.md — 2026-11-21T01:51:44.320Z (first version)
L3: Principio rector: **los números nunca pasan por un LLM**. Código determinista para todo lo que es dato. LLM solo para prosa (commentary, borradores de cover note) y, como mucho, para asistir el onboarding de templates.

#### Write cost-estimate.md — 2026-11-21T02:51:31.613Z (first version)
L22: | D. Asistencia de datos (matching de nombres dudosos, sugerir 4 de 6 KPIs) | Solo casos ambiguos | ~100 | ~2k | ~500 |
L32: | D. Asistencia de datos | 100 × (2k×$4 + 0,5k×$20)/1M = **$1,8** | **$0,9** | |
L24: | F. Onboarding de template (1 vez por versión, no por ciclo) | ~64 slides | ~64 | ~30k | ~3k |
L36: | F. Onboarding de template (por versión, ~2 por año) | 64 × (30k×$4 + 3k×$20)/1M ≈ **$12** | ≈ $6 | Despreciable. |

#### Write deep-dive-B.md — 2026-11-21T02:56:57.092Z (first version)
L23:    - Propuesta automática: un script cruza el texto de las shapes de un pack del archive con los datos de ese trimestre (p.ej. la shape que dice "18.4" ↔ ARR de Vantrel). Para lo ambiguo, el LLM propone. **Un humano confirma** en una UI simple. El LLM solo sugiere un mapeo que se revisa una vez; no genera contenido.
L109: | **Total B** | **~9 de 15** |
L63: ## Paso 4: Validación estructural (antes de los checks de datos de A)

#### Write deep-dive-A.md — 2026-11-21T03:17:09.204Z (first version)
L7: 2. **Por inspección (leak scanner):** se lee el archivo final (cada XML, workbook, nota, metadata, nombre de archivo de imagen) y se buscan rastros de lo excluido. Es independiente del generador.
L93: Por pack se arma una **lista de cosas prohibidas** a partir del plan:
