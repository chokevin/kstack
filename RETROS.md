# kstack retros

Append-only log. Newest at top.

---

## 2026-04-19 — bootstrap-through-first-research-dogfood

**Shipped:**
- Repo bootstrapped, 8 skills shipped (`/plan`, `/review`, `/investigate`, `/ship`, `/retro`, `/reckon`, `/airun-triage`, `/research`)
- `docs/principles.md` (paper-grounded, sets ~14% bounded-gain expectation per arXiv 2604.04323)
- Step 0 "Adapt to this ask" retrofitted across all 7 SKILL.md files
- First reckon memo: `docs/reckons/kstack-itself-20260418.md` → HARDEN
- First research memo: vLLM SOTA → ADAPT (in session-state, also belongs in `docs/research/`)
- 6 commits to main: `ab874bc`, `36197d0`, `5bf2cb2`, `6a76a84`, `384f11e`, `adee483`
- `~/.copilot/copilot-instructions.md` updated with full kstack discoverability + domain-skill section

**Worked:**
- `/reckon` against kstack itself produced a real binding decision (HARDEN with named non-goals) instead of mush. The format made it impossible to pat-on-back.
- `proj note` / `proj log` after each commit kept the registry warm with zero ceremony — invisible infra.
- arXiv 2604.04323 grounded the design in evidence; principles.md gives every future skill a "why this Iron Law" anchor.
- `/research` first dogfood (vLLM memo) — Iron Laws (decision, counter-evidence non-empty, ≥3 source types, footnoted citations) all enforceable and useful. Memo earned its keep.

**Hurt:**
1. **HARDEN constraint was non-binding.** The kstack-itself reckon explicitly said "no new skills until 5 retros worth of evidence." We shipped two new skills (`/airun-triage`, `/research`) in the same session. Each had documented justification but the constraint never bit. A decision the system can't enforce is a wish.
2. **Two consecutive "named pain → new skill" events without dogfood evidence.** Both `/airun-triage` and `/research` were built from "I think I need" not from observed friction with existing skills. Classic shape-of-problem-fits-shape-of-tool bias.
3. **Wrapper save path != skill canonical path.** `/research` skill specifies `docs/research/<slug>-<yyyymmdd>.md`. The research_task wrapper hardcoded a session-state path. Had to write to one and mention the other in the response. Real bug — undermines "skill is the source of truth."
4. **Step 0 retrofit was reactive.** Built `/airun-triage` and `/research` initially without Step 0; had to backport across all 7 files. Convention-by-retrofit means the next skill author won't know it's required.
5. **No skill validation before ship.** Paper says evals are the forcing function for skill quality. We have zero. Every skill is "looks right to me at the desk."
6. **N=1 dogfood for `/research`.** One memo doesn't validate a skill. The HARDEN memo asked for 5 retro datapoints. We have 1.

**kstack changes (concrete follow-ups):**
1. **Issue [#1](https://github.com/chokevin/kstack/issues/1):** Enforce HARDEN moratorium with a tripwire, not a vibe (`dogfood-log.md` + PR template).
2. **Issue [#2](https://github.com/chokevin/kstack/issues/2):** Reconcile wrapper save paths with skill canonical paths.
3. **Amendment to `skills/research/SKILL.md`:** Iron Law #N — "If the wrapper specifies a different save path, write to BOTH and flag the inconsistency in the handoff." Defensive until #2 is resolved.
4. **Amendment to `CONTRIBUTING.md` (or new `skills/SKILL_TEMPLATE.md`):** Step 0 is mandatory in all new skills. Make it copy-pasteable so it's not retrofit again.
5. **Defer:** Eval harness (paper-supported but premature at N=8 skills, N=2 dogfoods). Revisit after 5 more dogfood retros.
6. **Defer:** `/airun-triage` first real dogfood — wait for next stuck airun job, don't fabricate one.
