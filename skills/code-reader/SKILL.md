---
name: code-reader
description: Use when you want to deeply understand an unfamiliar codebase and generate reusable cognitive skills from it, by providing a local path or GitHub URL
---

# Deep Code Reader

Systematically read and understand a codebase, producing a set of verified cognitive skills that capture deep knowledge — module capabilities, design logic, data structures, state flow, and modification guides.

The core mechanism: a closed-book exam verification loop ensures generated skills are genuinely comprehensive, not shallow summaries.

## 1. The Team Roles

To make this process robust and conceptually clear, the system employs three distinct agents modeled after a software engineering team:

- **Agent A (Tech Writer)**: The deep reader. Reads the source code and writes the comprehensive skill document.
- **Agent B (QA Engineer)**: The examiner. Reads the source code, extracts verifiable facts, and generates test questions.
- **Agent C (Junior Dev)**: The candidate. Acts as a new team member who can ONLY read the document written by Agent A to answer Agent B's questions.

## 2. Usage

Here is the CLI command to trigger the deep-code-read workflow:

```bash
/deep-code-read <source> <output-dir>
```

- **source**: local path (e.g., `./path/to/repo`) or GitHub URL (e.g., `https://github.com/org/repo`)
- **output-dir**: where generated skills are written (e.g., your platform's skills directory)

## 3. Full Flow

You MUST follow these phases in order. Track progress across modules using your platform's task/todo tracking mechanism.

### 3.1 Phase 1: Prepare

This initial phase handles the resolution and preparation of the target source codebase.

1. Determine the project name:
   - Local path → directory name
   - GitHub URL → repo name
2. If source is a URL:
   - Clone to `{output-dir}/{project-name}/`
   - If the directory already exists, skip cloning and use it
3. If source is a local path:
   - Verify the path exists and is a git repo
   - Use it directly (read-only — do NOT modify any files in the source repo)
4. Detect version:
   - Run `git tag --list` in the source repo
   - If tags exist, sort with semver-aware ordering (handle `v` prefix), recommend the latest
   - If no reasonable tags, recommend `main` or `master` branch
5. **PAUSE — present recommendation to user:**
   > "Detected the following tags/branches: [list]. I recommend tracking `{recommended}`. Confirm or specify a different target."
6. Checkout the confirmed ref

7. **Check for previous execution state:**
   - Check if `{output-dir}/.code-reader-state.json` exists
   - If YES and `status` is `interrupted` or has modules with `pending`/`failed` status → present resume options to user (see Section 3.8)
   - If YES and `status` is `completed` → proceed to incremental mode detection below
   - If NO → proceed to incremental mode detection below

8. **Check for existing skills (incremental mode detection):**
   - Check if `{output-dir}/{project-name}-fj-*/SKILL.md` files already exist
   - If NO existing skills → **Full mode**: proceed to Phase 2 as normal
   - If existing skills found → **Incremental mode**: proceed to Phase 1.5

### 3.1.5 Phase 1.5: Incremental Update (only when existing skills are detected)

This phase handles updating existing skill documents when source code has changed.

1. **Detect changes since last generation:**
   ```bash
   # Find the commit hash when skills were last generated (stored in global index)
   # If no hash stored, use git log to find the commit closest to SKILL.md creation time
   git diff --name-only {last-commit}..HEAD -- {module-dir}
   ```

2. **Classify changed modules:**
   - For each module that has an existing SKILL.md, check if any of its source files changed
   - Group into: `changed_modules` (have diffs) and `unchanged_modules` (no diffs)

3. **Present to user:**
   > "Found existing skills for {N} modules. {M} modules have code changes since last generation:
   > - [module-a]: 3 files changed (file1.c, file2.h, file3.c)
   > - [module-b]: 1 file changed (main.c)
   >
   > {K} modules have no changes.
   >
   > Options:
   > A) Update only changed modules (recommended)
   > B) Full regeneration of all modules
   > C) Update specific modules: [list]"

4. **For each changed module — targeted update:**
   - Read the EXISTING SKILL.md for that module
   - Read ONLY the changed source files (from git diff)
   - Dispatch Tech Writer with a modified prompt:
     ```
     You are updating an existing skill document, NOT writing from scratch.
     
     Existing SKILL.md: [path]
     Changed files: [list of changed files with diffs]
     
     Rules:
     - Read the existing SKILL.md first
     - Read the changed source files
     - Update ONLY the sections affected by the changes
     - Preserve all unchanged content
     - If a function signature changed, update the function index
     - If a new file was added, add it to the file inventory
     - If a file was deleted, remove it from the inventory
     - Add a "Last Updated" Line at the top: "Updated: {date}, changes: {summary}"
     ```

5. **Skip unchanged modules entirely** — do not re-read or re-generate

6. **Update global index:**
   - Update the commit hash in `{output-dir}/{project-name}-dr/SKILL.md`
   - Update any cross-module references if module APIs changed

7. **Skip to Phase 6 (User Acceptance)** — no need for full verification on incremental updates

### 3.2 Phase 2: Scan

This phase scans the repository structure to identify boundaries and dependencies.

1. Scan the source repo directory structure
2. Identify module boundaries using heuristics:
   - Top-level directories under `src/`, `lib/`, `pkg/`, `packages/`, or project root
   - Language-specific patterns: Python packages (`__init__.py`), Go packages, Node packages (`package.json`), etc.
   - Look for existing module documentation or manifest files
3. Analyze import/dependency relationships between modules
4. **PAUSE — present module list and dependency graph to user:**
   > "Found the following modules: [list with one-line descriptions]. Select which modules to deep-read (or 'all')."
5. Record the user's selection — one task per selected module

### 3.3 Phase 3: Deep Read (Agent A - Tech Writer)

This phase generates the foundational skill documents.

**Before dispatching, assess module size:**

Count total source lines: `find {module-dir} -name '*.c' -o -name '*.h' | xargs wc -l | tail -1`

#### Small/Medium modules (≤3000 lines total)

Dispatch ONE Tech Writer subagent with `tech-writer-prompt.md`.

**Subagent dispatch parameters:**

- `prompt`: rendered `tech-writer-prompt.md` with variables filled in
- `description`: "Deep read {module-name}"

**Variables to fill in the prompt:**

- `{source-dir}`: path to the source repo
- `{module-dir}`: path to the specific module within the source repo
- `{output-dir}`: the skill output directory
- `{project-name}`: extracted project name
- `{module-name}`: the module name
- `{ref}`: the tracked tag/branch

#### Large modules (>3000 lines total)

**Split into file groups and dispatch MULTIPLE Tech Writers in parallel.**

1. List all source files sorted by size (largest first)
2. Group files by functional affinity:
   - Files in the same subdirectory → same group
   - Files sharing a prefix (e.g., `bind_task.c` + `bind_task.h`) → same group
   - If no natural grouping, split by size: ~2000-3000 lines per group
3. Each group limit: ≤3000 lines total (fits in one agent's context)
4. Dispatch one Tech Writer per group, each writing to a temp file: `{output-dir}/{project-name}-fj-{module-name}/part-{N}.md`
5. After ALL parts complete, dispatch a **Merger agent** with this prompt:
   ```
   Read all part-*.md files in {output-dir}/{project-name}-fj-{module-name}/.
   Merge them into a single SKILL.md following the standard format:
   - Deduplicate overlapping content
   - Unify data structure definitions
   - Merge function signature tables
   - Keep ALL specific details (thresholds, enum values, error codes)
   - Delete part-*.md files after successful merge
   ```

After Tech Writer(s) complete, verify the skill files were written to `{output-dir}/{project-name}-fj-{module-name}/`. Update the module's task status.

### 3.4 Phase 4: Verify (ABC Loop)

This phase verifies that the generated skills provide sufficient **structure understanding and navigation ability** — meaning someone can use the SKILL.md to quickly find the right file and function for any task, without reading all the source code.

The goal is NOT to verify that SKILL.md contains every implementation detail. It is to verify that SKILL.md serves as an effective navigation map to the source code.

For each module that has generated skills, run the verification cycle:

**Step 1 — Agent B / QA Engineer (question generation):**

Dispatch a subagent with `qa-engineer-prompt.md`, using a lightweight/smaller model (e.g., Haiku-class).

**Subagent dispatch parameters:**

- `prompt`: rendered `qa-engineer-prompt.md`
- `model`: a smaller, cheaper model — the weaker the better (if it catches gaps, those gaps are real)
- `description`: "Generate questions for {module-name}"

**Variables:**

- `{source-dir}`, `{module-dir}`, `{module-name}`
- `{previous_questions}`: empty string for the first round

QA Engineer returns two sets:

- Verification questions with answer keys (JSON array)
- Recommended questions for user (JSON array)

Save the recommended questions (keep in context for Phase 6). Accumulate all verification questions asked so far across rounds.

**Step 2 — Agent C / Junior Dev (closed-book answer):**

Dispatch a subagent with `junior-dev-prompt.md`.

**Subagent dispatch parameters:**

- `prompt`: rendered `junior-dev-prompt.md` with verification questions embedded
- `description`: "Verify skills for {module-name}"

**Variables:**

- `{skill-dir}`: `{output-dir}/{project-name}-dr-{module-name}/`
- `{questions}`: the verification questions from QA Engineer (without answer keys)

Junior Dev returns answers to each question.

**Step 3 — Evaluate:**

Use your own reasoning (as the main orchestrator) to evaluate Junior Dev's answers:
For each question, check Junior Dev's answer against QA Engineer's `required_facts` list:

- An answer PASSES if it correctly identifies the file, function, and structural relationship
- An answer FAILS if it cannot point to a location, or points to the wrong location
- An answer that says "function X in file Y" without internal details is a PASS — the reader can open that file to see details
- An answer that gives internal details but wrong file/function is a FAIL
- This is an objective check, not a subjective judgment.

**Step 4 — Loop or proceed:**

**HARD RULE: You MUST continue looping until 100% of verification questions pass OR you have completed exactly 3 rounds. There is NO early exit. A pass rate of 99% is still a failure — loop again.**

- 100% pass → module verified, update task, move to next module
- ANY question fails (even one) → you MUST continue to the next round:
  1. Collect failed questions with: the question, QA Engineer's answer key, Junior Dev's failed answer
  2. Feed these back to Tech Writer: dispatch again with the `{feedback}` variable containing the failed questions, QA Engineer's expected answer keys, and the gaps identified.
  3. Re-run QA Engineer and Junior Dev, passing ALL previous questions (from all rounds) as `{previous_questions}` so QA Engineer generates new questions instead of repeating old ones
  4. Evaluate again — repeat until 100% or 3 rounds completed
- **After exactly 3 rounds with failures remaining** → show the unresolved questions and pass rates to the user for judgment. Do NOT silently move on.

**Do NOT rationalize stopping early.** "Good enough", "most questions passed", "diminishing returns" are not valid reasons to skip a round. The loop exists to catch gaps — use all 3 rounds if needed.

### 3.5 Phase 5: Generate Global Index

This phase consolidates the verified module skills into a global index file.

After all modules are verified, generate `{output-dir}/{project-name}-dr/SKILL.md`:

```yaml
---
name: {project-name}-dr
description: Use when working with {project-name} codebase — provides comprehensive module knowledge, design logic, and modification guides (generated from {ref})
---
```

Content must include:

- Repo source (GitHub URL if applicable, or local path)
- Version: tag or commit hash
- Tracked branch
- Generation timestamp
- Each module's one-line purpose (from the module skills)
- Inter-module dependency relationships (from Phase 2 scan)
- Cross-module scenario entry guides: for common operations that span multiple modules, describe which modules are involved and in what order

To generate cross-module scenarios, read ALL the module skills and synthesize typical user workflows.

### 3.6 Phase 6: User Acceptance

This phase presents the results to the user for final validation.

Present the recommended questions collected from Phase 4:

> "Skills generated and verified. Here are some questions you might want to test:
> [list recommended questions]
>
> Feel free to ask any question about {project-name}. I'll answer using ONLY the generated skills."

When answering user questions in this phase:

- Read ONLY the generated skill files in `{output-dir}/{project-name}-fj*/`
- Do NOT read source code
- If you cannot answer a question from the skills alone, say so honestly — this indicates a gap

Continue until the user is satisfied or decides to end the session.

### 3.7 Phase 7: Cleanup

This final phase handles the cleanup of temporary files if necessary.

If the source was cloned from a URL (i.e., `{output-dir}/{project-name}/` was created in Phase 1):

> "Skills are ready. The cloned source code is at `{output-dir}/{project-name}/`. Want me to delete it to save disk space, or keep it for reference?"

- User says delete → remove the cloned directory
- User says keep → leave it as is

Skip this phase if the source was a local path (we never cloned anything).

### 3.8 State File

Every code-reader execution MUST maintain a state file at `{output-dir}/.code-reader-state.json`.

This file is **overwritten** on each run (not appended). It records the result of the LAST execution only.

#### State File Schema

```json
{
  "project": "baize_eufy",
  "source_dir": "/path/to/source",
  "output_dir": "/path/to/output",
  "ref": "develop",
  "commit": "6e025204b",
  "started_at": "2026-05-20T10:30:00Z",
  "finished_at": "2026-05-20T11:15:00Z",
  "status": "completed | interrupted | failed",
  "mode": "full | incremental",
  "current_phase": "phase3",
  "modules": {
    "alarm": {"status": "verified", "phase": "phase4", "skill_md": true, "pass_rate": "7/7"},
    "bind": {"status": "generated", "phase": "phase3", "skill_md": true, "pass_rate": null},
    "strategy": {"status": "pending", "phase": null, "skill_md": false, "pass_rate": null},
    "p2p": {"status": "failed", "phase": "phase3", "skill_md": false, "error": "context limit exceeded"}
  },
  "global_index": false,
  "error": null
}
```

#### Module Status Values

| Status | Meaning |
|--------|---------|
| `pending` | Not started yet |
| `generating` | Tech Writer running |
| `generated` | SKILL.md written, not verified |
| `verifying` | ABC loop running |
| `verified` | Passed verification |
| `failed` | Error during processing |

#### When to Write State

- **Phase 1 start**: Create state file with all modules as `pending`
- **Phase 3 each module**: Update to `generating` before dispatch, `generated` after success, `failed` on error
- **Phase 4 each module**: Update to `verifying` before dispatch, `verified` after pass
- **Phase 5**: Set `global_index: true` after index generation
- **On any error/interruption**: Update `status` to `interrupted` or `failed`, set `current_phase`

#### On Resume (next run)

When code-reader starts and finds an existing state file:

1. Read the state file
2. Present status to user:
   > "Found previous execution (started {time}, status: {status}).
   > - {N} modules completed
   > - {M} modules pending
   > - {K} modules failed
   >
   > Options:
   > A) Resume from where it stopped (continue pending modules)
   > B) Retry failed modules
   > C) Start fresh (ignore previous state)"
3. User chooses → proceed accordingly

## 4. Key Rules

Strictly adhere to the following rules during the entire execution process:

- **Never modify source code** — the source repo is read-only throughout
- **Agent isolation is critical** — each agent's prompt strictly defines what it can read
- **Skills must be self-sufficient** — the verification loop exists to ensure this
- **Track progress** — every module is a task, updated as it progresses through phases
- **Format via writing-skills** — Tech Writer follows `superpowers:writing-skills` formatting conventions (frontmatter, CSO description, directory structure) but does NOT run the full writing-skills TDD cycle
