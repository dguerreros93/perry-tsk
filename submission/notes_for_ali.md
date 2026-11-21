# Notes for Ali

A few notes on how this submission was put together, so nothing in it is a surprise.

1. **The transcript is sanitized.** `transcript/session-transcript.md` is the full AI-assistant session for this task, with these removals and nothing else:
   1. System context the tool injects automatically: system reminders, the tool's local memory and a loaded API-reference skill.
   2. Tool results, meaning command output and file reads.
   3. Tool calls that handled the tool's local memory and the extraction of the transcript itself. Each one is marked "omitted" where it occurred.
   4. Personal data: my name, my email address and local file paths.
   5. References to my unrelated projects, tools and agents.

   Everything else is unedited: every message from me and from the AI, and the full content of every file the AI wrote or edited for this task.
2. **I worked in Spanish.** The working session and drafts are in Spanish. The deliverables (`DESIGN.md`, `ALTERNATIVES.md`, `CORRECTIONS.md`) are in English, and every quote in `CORRECTIONS.md` gives the Spanish original with an English translation.
3. **No experiments were run, by choice.** I treated this as a research and design exercise and did not install tooling for throwaway code. Every technical claim in `DESIGN.md` §7 is therefore marked *assumed*, with how it would be verified. Most of them are covered by the week-1 spike in the plan.
4. **The questions were sent before the design was finished.** The design runs on explicit working assumptions for each of the five questions (`DESIGN.md` §8). If the answers change any of them, I expect to update mainly §5 (corrections) and §7–8.
5. **`CORRECTIONS.md` has four counted entries and one marked "Not counted".** The fifth is kept for completeness. It was a misreading of my intent rather than a technical error, so I don't count it toward the four.
6. **Evidence files are verbatim extracts** from the same session. `evidence/original-file-lines.md` shows what the AI first wrote into its working files, before each correction. All timestamps are UTC.
7. **Word counts** exclude diagrams, as the brief allows: `DESIGN.md` has about 2,980 of 3,000 words and `ALTERNATIVES.md` about 580 of 600. Tables are included in both counts.
8. **The five questions I sent, and what the design assumes if they go unanswered.** These are the questions exactly as sent. Each working assumption is the one the design currently runs on (`DESIGN.md` §8).
   1. *"Can we ask them to prepare it to a convention/format we define (named placeholders, fixed chart names, a known structure) or does the system have to take whatever the agency produces and figure it out?"*
      **Assumption:** we map v7 ourselves once, as a hand-written manifest validated against archived packs. Any new version is diffed and blocked until it is remapped. If the agency adopts named slots from v8, the manifest becomes a validation step instead of a mapping step.
   2. *"Who owns the final call on what each LP can and can't see — is that signed off by someone on their side, or is it on us? And if an LP rights ever contradict each other, should the system flag it or just skip?"*
      **Assumption:** fund counsel owns the rules. Perry applies them, and a conflict blocks the pack and escalates; it is never resolved automatically. An example is LP-0058, where FOI-reduced disclosure conflicts with side letter s.4.2. Resolutions are recorded as overrides with who, when and why.
   3. *"Before a pack goes out, IR reviews it — are they reviewing numbers or mostly the cover note and layout?"*
      **Assumption:** automated validation is the primary check on numbers. That means round-trip comparison against source data, invariants and a deny-list leak scan, with a `needs_human` flag that escalates anything that doesn't reconcile. IR's review is the second layer.
   4. *"Since the real template can't be shared, what will we have to build and test against at the start?"*
      **Assumption:** we test against the archive of hand-built packs as the real v7 structure and golden output, synthetic fixtures for the template's features, and the six Appendix B LPs as data fixtures. A shadow run regenerates Q3 and compares it with IR's packs.
   5. *"When a number is corrected after sign-off (which happens most quarters), should we regenerate and re-send the affected packs? or what is the process?"*
      **Assumption:** a new snapshot is diffed against the previous one to find the affected packs and slides. For Q4 those packs are regenerated with a change list, so IR can re-apply their edits. A delivered pack is never changed silently: IR decides whether to reissue it, and the reissue is recorded in the audit trail.
