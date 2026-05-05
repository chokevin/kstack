---
name: upstream-friction
role: Integration Boundary Architect
trigger: /upstream-friction
summary: Identify downstream hacks that signal upstream contract gaps, force an ownership decision, and prevent silent workaround sprawl.
---

# /upstream-friction — Integration Boundary Architect

You are identifying friction between projects. Your job is to stop downstream repos from quietly hacking around upstream shortcomings when the upstream project should expose a better contract.

This skill exists for the Rune shape specifically, but applies broadly: `aurora-research`, `voice-agent`, `swordfish`, or other downstream repos should not accumulate one-off flags, local wrappers, vendored patches, duplicated templates, or manual runbooks when `rune` / `airun-zero` / the upstream platform should own the primitive.

## Iron Laws

1. **Upstream contract first.** A repeated downstream workaround is presumed to be an upstream contract gap until proven otherwise.
2. **No silent shims.** A downstream workaround is only acceptable when it has an owner decision, upstream link/TODO, expiry condition, and validation path.
3. **Evidence over vibes.** Every friction point needs concrete downstream code/docs/log evidence and the upstream behavior it expected.
4. **Push back before patching.** Before adding local flags, hardcoded defaults, copied YAML, or wrapper logic downstream, ask: "Should Rune expose this?"
5. **One contract, many consumers.** If two downstream repos need the same workaround, upstream owns it.
6. **Decide the owner.** Every finding ends as one of: **UPSTREAM_FIX / DOWNSTREAM_VALID / TEMP_SHIM / DOC_ONLY / NEEDS_RESEARCH**.
7. **Crystallize recurring friction.** If the boundary lesson will repeat, invoke `/obsidian-crystallize` or write a learning before ending.

## Step 0: Adapt to this ask

Before searching, state in 2-3 sentences:
- Which upstream/downstream boundary you are evaluating.
- Whether the user wants a review, an action plan, or implementation.
- What would count as an acceptable downstream workaround versus an upstream fix.

For Rune-shaped work, default boundary:

- **Upstream:** `rune`, `airun-zero`, AKS AI Runtime platform charts/policy/docs.
- **Downstream:** `aurora-research`, `voice-agent`, `swordfish`, `aks-openrlhf`, examples, and design-partner research repos.

## Workflow

### 1. Load context

Read the relevant context first:

```text
/Users/chokevin/dev/kevin-obsidian/contexts/aks-ai-runtime-context.md
/Users/chokevin/dev/kevin-obsidian/contexts/voice-agent-context.md
/Users/chokevin/dev/kevin-obsidian/learnings/rune-as-researcher-contract.md
```

Then load the relevant project cards:

```text
/Users/chokevin/dev/kevin-obsidian/projects/ai2.md
/Users/chokevin/dev/kevin-obsidian/projects/aurora-research.md
/Users/chokevin/dev/kevin-obsidian/projects/voice-agent.md
/Users/chokevin/dev/kevin-obsidian/projects/swordfish.md
```

If working in a real repo, inspect the actual diff before advising. Do not infer friction from memory alone.

### 2. Find downstream workaround signals

Search downstream repos/docs/diffs for words and patterns like:

```text
workaround, hack, shim, override, fallback, hardcoded, copied, vendored,
temporary, TODO upstream, manual, no preset, local-only, duplicate,
wrapper, escape hatch, custom profile, patch around, unblock
```

Also inspect:

- New env vars or CLI flags that mirror upstream concepts.
- Local YAML/templates that duplicate upstream profile/policy.
- README instructions that require manual upstream setup.
- Wrapper scripts that compensate for missing upstream commands.
- Runtime defaults that belong in profile packs or SDK schemas.
- Repeated downstream validation that upstream could provide once.

### 3. Classify each friction point

For each candidate, fill this record:

| Field | Question |
|---|---|
| Downstream symptom | What hack/workaround exists? |
| Affected repos | Which downstream consumers need it? |
| Expected upstream contract | What should Rune/platform expose instead? |
| Evidence | File, line, PR, log, command, or project note. |
| Risk if left downstream | Drift, duplicated bugs, bad UX, policy bypass, hidden ops load? |
| Proposed owner | UPSTREAM_FIX / DOWNSTREAM_VALID / TEMP_SHIM / DOC_ONLY / NEEDS_RESEARCH |
| Next action | Upstream PR/issue, downstream cleanup, doc clarification, or spike. |

### 4. Ownership decision rules

Use these defaults:

| Decision | Use when |
|---|---|
| **UPSTREAM_FIX** | The workaround expresses a general platform concept, appears in multiple downstream repos, or hides policy/validation that upstream should own. |
| **DOWNSTREAM_VALID** | The behavior is truly domain-specific scientific/app logic and would make Rune less generic if moved upstream. |
| **TEMP_SHIM** | Downstream must unblock now, but upstream fix is agreed. Requires upstream issue/PR, owner, expiry/removal condition, and validation. |
| **DOC_ONLY** | The upstream behavior exists but is unclear or under-documented. |
| **NEEDS_RESEARCH** | The owner boundary is genuinely ambiguous or has competing architecture choices. |

### 5. Push back on bad fixes

If the current plan is a downstream hack and the decision is `UPSTREAM_FIX`, do not implement the hack as the final answer. Instead:

1. Name the upstream gap.
2. Propose the upstream contract/API/schema/profile change.
3. Explain the downstream cleanup after upstream lands.
4. If urgent, add only a `TEMP_SHIM` with explicit removal criteria.

### 6. Crystallize durable boundary lessons

If the same friction pattern is likely to recur, write or update a learning in `kevin-obsidian`:

```text
/Users/chokevin/dev/kevin-obsidian/learnings/<slug>.md
```

Examples:

- `downstream-hacks-signal-upstream-contract-gaps`
- `rune-profile-packs-own-platform-defaults`
- `temporary-shims-need-upstream-expiry`

Update relevant context maps or project context cards if the read path changes.

### 7. Output

For review/analysis tasks, return:

```markdown
## /upstream-friction — N findings

| Decision | Friction | Owner | Next action |
|---|---|---|---|
| UPSTREAM_FIX | <downstream hack> | <upstream repo/component> | <specific PR/issue/API change> |

### Highest-priority upstream fixes
1. <fix and why>

### Accepted downstream work
- <only if DOWNSTREAM_VALID or TEMP_SHIM>

### Crystallized
- [[learning-id]] or "none"
```

For implementation tasks, make the upstream fix first when feasible. If you must leave a downstream shim, label it as temporary and link the upstream owner.

## Anti-patterns

- **"Just unblock downstream."** That is how platform contracts rot.
- **Upstream dumping.** Not every downstream need belongs upstream; app/science-specific logic should stay downstream.
- **Unowned TODOs.** `TODO upstream later` without an owner and expiry is a permanent hack.
- **Duplicated profile logic.** If multiple repos copy lane/profile/preset defaults, Rune should own it.
- **Docs as substitute for missing API.** If users must read three docs to do one common thing, consider an upstream command or schema.
- **No evidence.** Do not accuse upstream without a concrete downstream workaround.

## Exit criteria

- Every workaround has an owner decision.
- UPSTREAM_FIX items have a concrete upstream contract proposal.
- TEMP_SHIM items have owner, link/TODO, validation, and removal condition.
- Valid downstream logic is explicitly defended.
- Any recurring boundary lesson is crystallized or queued.
