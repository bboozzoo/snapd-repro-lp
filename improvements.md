# Prompt Improvements: Findings from Model Comparison

## Source

Comparison of `deepseek-4-flash.log` and `glm-5.1.log` executing bug #1662786
(snap list/find output hard to read in 80-column terminal).

## Key Findings

### 1. Plan lacked a concrete verification step

The plan's `expected_result` was qualitative ("lines that wrap beyond 80 columns, making the tabular output hard to read"). Both models had to improvise their own verification logic (awk line-length checks, COLUMNS comparison) during execution, wasting steps. The plan should require a **final step that is an automatable pass/fail command**.

### 2. No reproducible-script quality requirements in prompts

The prompt says "Include a complete reproducer script" but doesn't define "complete."

- deepseek: installed `lxd` (heavy, unrelated), no pass/fail exit code
- glm: installed `firefox` (heavy, slow), used `sleep 2` for daemon readiness (fragile), swallowed all errors with `2>/dev/null || true`

### 3. No "stop when confirmed" directive in execution prompt

deepseek ran 26 steps, of which ~10 were post-confirmation exploratory variants: Python subprocess experiments, `cat -A`, column separator analysis, `fold` simulations, installing extra snaps. The prompt says "check outputs between steps" but doesn't say to **stop exploring once the bug is confirmed**.

### 4. `report_result` tool schema is too loose on `script` field

The `script` description is just "The shell script that reproduces the bug." It should constrain the script to be self-contained, idempotent, and exit with a meaningful code.

### 5. No guidance on lightweight snap choices

Both models installed unnecessarily heavy snaps (lxd, firefox) just to populate `snap list`. The snapd domain knowledge section should recommend lightweight alternatives.

---

## Proposed Changes (in `prompt.go` and `tools.go`)

### A. Planning prompt — require a verification step as the final step

Add to `BuildPlanningPrompt` instructions:

> "The last step in your plan must be a concrete verification command that exits 0 if the bug is reproduced and non-zero otherwise. Do NOT make the final step 'observe output' — it must be an assertion."

Also update `expected_result` guidance:

> "The expected_result field should describe the automatable check, not just the visual symptom."

### B. Planning prompt — add reproducer script requirements

Add a guideline:

> "The plan should produce a reproducer script that is: (a) runnable on a fresh Ubuntu cloud image, (b) idempotent, (c) exits 0 if the bug is reproduced or 1 if not, (d) avoids heavy packages unless the bug requires them."

### C. Execution prompt — add "stop when confirmed" directive

Add to the "Important Guidelines" section:

> "Once you have confirmed the buggy behavior, do not run additional exploratory or diagnostic variants of the same command. Proceed directly to report_result."

### D. Execution prompt — tighten reproducer script guidelines

Expand "Include a complete reproducer script" to:

> "Include a self-contained reproducer script that: installs prerequisites, triggers the bug, and exits 0 if reproduced or 1 if not. Do not silently swallow all errors (use `2>/dev/null || true` only for expected failures). Use a polling loop rather than `sleep` for daemon readiness. Prefer lightweight snaps (hello-world, core18/20/22/24) over heavy ones (firefox, lxd, chromium) unless the bug specifically requires them."

### E. `report_result` tool schema — tighten `script` field description

Change from:

> "The shell script that reproduces the bug (if reproduced), or the best attempt (if not)."

To:

> "A self-contained shell script that: installs prerequisites, triggers the bug, and exits 0 if reproduced or 1 if not. Must be idempotent and runnable on a fresh Ubuntu instance."

### F. Snapd domain knowledge — add lightweight snap guidance

Append to `writeSnapdKnowledge`:

> "- For populating `snap list` output, prefer lightweight snaps (hello-world, core18, core20, core22, core24). Avoid heavy snaps like firefox, lxd, or chromium unless the bug specifically requires them."
