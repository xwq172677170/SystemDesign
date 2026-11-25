# Claude Certified Architect – Domain 3 Summary

**Configuration & Workflows** (20% of exam, version 1.0)

Summary of the course transcript in `claude certified architect.md`.

## Exam basics

- **5 domains:** agentic architecture & orchestration, tool design & MCP integration, **configuration & workflows**, prompt engineering & structured output, context management & reliability.
- Each exam draws **4 of 6 scenarios**. Domain 3 is the main domain in 3 of them, so at least one always shows up:
  - **#2 Code generation with Claude Code:** slash commands, CLAUDE.md, plan mode vs. direct execution
  - **#4 Developer productivity (Agent SDK):** exploring codebases, legacy systems, boilerplate, MCP
  - **#5 Claude Code for CI:** automated reviews, test generation, PR feedback with few false positives
- The material changes quickly, so check the exam version before you study and again before the exam.

---

## 1. CLAUDE.md: hierarchy, scoping, modularity

### What goes in it
CLAUDE.md is loaded when the session starts and **stays in context for every prompt**, so every line costs tokens. It gives **context, not enforcement**. For rules that must always hold, use permissions, hooks, or managed settings.

| Include | Leave out |
|---|---|
| Build / test / run commands | This week's task, current branch |
| Architecture overview (services, how they talk, where a change belongs) | Anything that goes stale fast (stale instructions get followed with full confidence) |
| Team conventions + preferred Claude behavior ("ask when unsure") | Things Claude can work out from the code |
| Decisions you can't see in the code (what was *agreed*, not just what was done) | Details you only need occasionally |

- Keep each file **under ~200 lines**. Bigger files leave less room for the actual work.

### Three levels
| Level | Location | Version-controlled? | Loads |
|---|---|---|---|
| **User** | `~/.claude/CLAUDE.md` | No, never reaches teammates | Every project, at start |
| **Project** | repo root `CLAUDE.md` or `.claude/CLAUDE.md` | Yes, changes go through PRs | At start |
| **Directory** | `some/subdir/CLAUDE.md` | Yes | **Only when Claude reads files in that directory** |

- Files **don't override each other. They're concatenated.** The file closest to the work comes last.

### Keeping it modular
1. **Directory-level CLAUDE.md:** the only option here that **actually saves context**.
2. **`@path/to/file.md` imports:** paths are relative to the importing file. **Not lazy**: imported files load at launch. You get better organization, not fewer tokens.
3. **Rules in `.claude/rules/*.md`:** one topic per file. Without front matter a rule is **always loaded**. With a `paths:` front matter it's path-scoped (see section 3).

**"Memory files"** is the exam term for everything auto-loaded: all CLAUDE.md files, imports, and rules.

### Diagnosing
- **`/memory`** lists every memory-file *location* (even ones that don't exist yet). Use it for "why does Claude behave differently for me vs. a teammate?"
- **`/context`** shows what is *actually loaded* and how many tokens each part uses.

### Exam tips
- **New teammate isn't getting conventions** → those instructions live at the **user** level on someone else's machine. Move them to project or directory level and commit.
- **Legacy folder needs a different style** → `legacy/CLAUDE.md`.
- **One rule file is ignored while others load** → it was saved **outside `.claude/rules/`**.

---

## 2. Custom slash commands & skills

### Commands
- A Markdown file whose body is a prompt: `.claude/commands/<name>.md`. The file name becomes the command name. Invoke with `/name args`. Arguments go into `$ARGUMENTS`.
- **Project:** `.claude/commands/`, committed and shared. **Personal:** `~/.claude/commands/`, only you.
- Optional YAML front matter:
  - `description`: shown in autocomplete
  - `argument-hint`: label for the expected input (a hint, not validation)
  - `allowed-tools`: **pre-approves** these tools. It **does not restrict** to them. To block a tool, use `disallowed-tools`.

### Skills
- A **directory** with `SKILL.md`: `.claude/skills/<name>/SKILL.md`. The **folder name** is the skill name.
- Can include supporting files (scripts, templates, reference docs), e.g. a deterministic Python parser plus an output template.
- **Two ways to invoke:** a human types `/name`, or **the model matches your request against the `description`**.
  - `disable-model-invocation: true` → only a human can trigger it (use for side effects like deploys)
  - `user-invocable: false` → only Claude can load it
- **The description is the matching surface.** Say *what* the skill does *and when to use it*.
- Only the description sits in context. The full body and supporting files load when the skill runs.
- Skills are now the preferred way to extend Claude Code. The commands folder still works.

| Use a command when… | Use a skill when… |
|---|---|
| It's a shortcut for one prompt you use often | It's a workflow with scripts or templates |
| You always trigger it explicitly | Claude should apply it automatically |
| One Markdown file is enough | You need more than one file |

**CLAUDE.md vs. skills:** CLAUDE.md holds project-wide *conventions* and loads every session. Skills hold specific *tasks* and load on demand.

### `context: fork`
- Runs the skill in a **subagent with its own context window**. Only the result comes back to the main conversation.
- Fixes **context pollution** (dozens of file reads crowding the window) and **lost focus**. In the demo, main-session usage dropped from ~72k to ~39k tokens.
- The subagent **starts clean, with no chat history**, so the skill prompt has to be self-contained.
- Use it for self-contained research, verbose operations (logs, commands), and anything that would bloat the main thread.

### Name precedence
**Enterprise (managed) > Personal > Project.** A personal skill with the *same name* silently overrides the team's version. To customize a team skill: **copy it, rename the folder, then edit it** so both versions can coexist.

### Exam tips
- Ask *who needs it*. If teammates should get it on clone → project folder, committed (not git-ignored). If it's only you → personal folder.
- A skill whose many file reads blow up the context → **`context: fork`** (not `allowed-tools`, not moving it into CLAUDE.md).

---

## 3. Path-specific rules

- A rule in `.claude/rules/` with a `paths:` glob list in its front matter. **It loads only when Claude reads, writes, or edits a matching file.** The front matter is stripped, so the scoping costs nothing in context.

```yaml
---
paths:
  - "src/api/**/*.ts"
  - "tests/**/*.test.ts"
---
```
- A compact comma-separated string also works. Use the list form once you have more than 2–3 patterns.
- **Malformed YAML → the paths are ignored → the rule becomes always-on.** Check the front matter first if a scoped rule loads everywhere.
- Glob notes:
  - `*` matches within **one** folder. `**` matches **any depth, including none**.
  - Patterns are **relative to the project root**: `terraform/**` matches only the top-level folder. Use `**/` to match at any depth (e.g. `**/*.sql`).
  - Brace expansion: `src/components/**/*.{ts,tsx}`
  - Matching **any one** pattern loads the rule.
- Benefits: a smaller context window, fewer tokens across the team, and sharper model focus (fewer conflicting rules).
- Subfolders like `rules/frontend/` work. Personal rules go in `~/.claude/rules/`.

### Rule vs. directory CLAUDE.md vs. skill
- **Directory CLAUDE.md** follows the **folder** (every file type inside it).
- **Path rule** follows the **file pattern**, even when matching files are scattered across the repo.
- **If you're about to copy CLAUDE.md content into a 2nd or 3rd directory, write a rule instead.** Duplicated files drift apart.
- **Skill** is a capability someone has to invoke, and nothing guarantees it gets invoked. A rule is a **standing convention** that applies automatically. Skills are the wrong tool for enforcing conventions.

### Exam tip
Many folders + one convention + applied automatically → **path-scoped rule with a glob** (e.g. 11 services' `migrations/` folders → one rule covering `**/migrations/**/*.py`).

---

## 4. Plan mode vs. direct execution

### Plan mode
- Claude explores and writes a plan. **Edits are blocked until you approve it.** A wrong approach costs a discarded plan instead of a half-finished rewrite.
- How to enter it: **Shift+Tab** (cycles permission modes), **`/plan [task]`**, or **`claude --permission-mode plan`**. Asking in plain language *may* work through EnterPlanMode/ExitPlanMode, but that's the model's call and not guaranteed.
- **Signals that call for a plan:** large-scale changes (library migrations), multiple valid approaches, architectural decisions, coupled or uncertain multi-file changes.
- **Anti-patterns:** editing as you go, writing the entire solution into the prompt, starting direct and switching to plan mode once it gets hard (the complexity was there all along).

### Explore subagent
- Built in and **read-only** (no Write/Edit). It scales its depth (quick / medium / very thorough) and returns a **summary**, so the raw file reads never enter the main context.
- Doesn't load your CLAUDE.md, so constraints go in the delegation prompt. `@agent-name` makes sure that specific agent runs.
- A general-purpose subagent has every tool and can modify code.

### Direct execution
- Not a separate mode. It just means skipping plan mode.
- Use it for a **known location, confirmed cause, one or a few files, no design choice left**: a stack-trace bug, adding validation, doc updates, a regression test, or replacing a hard-coded value with existing config.

### Hybrid workflow (the recommended one)
Spec → **Explore subagent** (discovery) → *(interview)* → **plan mode** → **human review/approval** (problems are cheapest to fix here) → save the plan to a file → **direct execution** of the approved plan.
"Think slow, act fast" (*How Big Things Get Done*, Flyvbjerg & Gardner).

### Exam tips
- 60 call sites in 12 services, undecided between an adapter and a rewrite → **plan mode first, compare approaches, review before any edit**.
- A trace through 30 files fills the context and Claude forgets constraints → **delegate the trace to the Explore subagent**.

---

## 5. Iterative refinement

### Few-shot examples
- Vague prose ("format nicely") gives output that is plausible but **inconsistent across runs**. Adding more rules just leaves new gaps.
- Add **2–3 input → exact output examples** (Anthropic docs suggest 3–5). Cover **different cases + a boundary case** and write the output *exactly*.
- Use them when the output has a shape (a table row, log line, or report entry), or when repeated runs come back formatted differently. Skip them when a schema or type already fixes the shape.

### Test-driven iteration
- Write the tests first. **Red → green → refactor.** Claude runs the tests and reads the failures itself, and the work is done when **all tests pass** (not when it "looks done").
- Cover expected behavior, **edge cases** (empty, null, boundaries, duplicates, malformed input), and **performance limits** (otherwise a slow but correct solution still passes).
- Tell Claude: "don't modify or weaken existing tests."
- Who writes the tests: you (a fixed target), Claude (if the spec is precise and you review the tests), or a split (one session writes tests, another writes code).

### Interview pattern
- Before building, Claude asks questions (with **AskUserQuestion**, **one at a time, with a recommended answer**) to surface edge cases, failure modes, and hidden assumptions.
- Use it when requirements are vague, the domain is unfamiliar, or a wrong guess is expensive to undo. Skip it when tests, examples, or a precise spec already settle things.
- In the workflow, it goes **after exploration and before planning**, so the questions are about real conflicts found in the code. Record the decisions in the plan.

### Batch vs. sequential fixes
- Ask: *would knowing the fix for one issue change how I fix another?*
  - **Yes (they interact)** → send them **together in one message** so the shared decision is made once.
  - **No (independent)** → fix them **one at a time** and verify each, so a failure points at one change.

### Exam tips
- Inconsistent format → **add examples**, not more prose, not CLAUDE.md, not plan mode.
- Unfamiliar domain (payment retries could double-charge) → **interview first**.

---

## 6. Claude Code in CI/CD

### Non-interactive mode
- **`claude -p "<prompt>"`** (`--print`) runs to completion, prints to stdout, and exits (exit code 0 = success).
- A CI job that hangs → **add `-p`**. Distractors that *don't* work: a made-up `CLAUDE_HEADLESS` env var, a fake `--batch` flag, `< /dev/null`, piping `yes`.
- A plain `-p` run **still loads CLAUDE.md** from the checked-out repo. Put CI context there: review criteria (what to flag, what to ignore, severity levels), testing standards, and what linters / static analysis already cover so Claude doesn't repeat them.
- **`--bare`** skips auto-discovery of CLAUDE.md, hooks, skills, plugins, MCP servers, and subagents. Runs are reproducible, but your review criteria are lost and you get a generic review.
- Pin the Claude Code version (or use Anthropic's GitHub Action) so runs stay reproducible. Authenticate with an API key, OAuth, or a cloud provider.

### Structured output
- Asking for JSON in the prompt is only a request.
- **`--output-format json`** → one JSON object with `result` (still free text), `is_error`, `num_turns`, `total_cost_usd`, `session_id`.
- **`--json-schema <schema>`** → validated payload in **`structured_output`** (e.g. `findings[]` with `file`, `line`, `severity` enum, `message`).
- **Check that `structured_output` exists** (and the exit status) before acting on it. Otherwise a failed run looks like a clean review with zero findings.
- **Permissions:** with nobody there to approve, any tool that isn't pre-approved is **denied silently and the run continues**. Use `--allowedTools` (e.g. `Read,Edit,Bash(npm test:*)`).

### Incremental reviews
CI runs have no memory, so findings get reposted on every commit and developers stop reading them. The fix:
1. Fetch prior findings (from PR comments, artifacts, or a cache). Use an empty list on the first run.
2. Inject them into the prompt (`<prior_findings>…</prior_findings>`) and **ask only for new issues**. A finding counts as the same if it's in the same file with the same problem, even if the line moved.
3. Review → schema-validated JSON.
4. Post only the new findings.
5. Save the full set for the next run.

### Test generation in CI
- Without context you get low-value tests (getters/setters, restating the implementation), convention drift, and duplicates.
- Fix: put **testing standards, test helpers, and "what's worth testing / what isn't" in CLAUDE.md**, and **inject the existing tests into the prompt**.
- Treat generated tests as suggestions for a person to pick up, not auto-pushed commits.

### Session isolation
- Self-review in the same session is biased because it reviews against its own reasoning. Adding "be highly critical" doesn't help.
- **Use a fresh session or separate subagent that sees only the diff.** Each CI run is already a fresh invocation. In the demo, a fresh session found 3 high-severity issues plus several mediums, vs. 1 high + 1 low from self-review.

### Exam tips
| Symptom | Fix |
|---|---|
| CI step hangs / times out | `-p` |
| The pipeline needs to parse output | `--output-format json` + `--json-schema`, then check `structured_output` exists |
| Reviews repeat old findings / tests duplicate existing ones | More context: prior findings, existing tests, CLAUDE.md criteria |
| Review isn't critical enough | A separate, isolated invocation (not a stricter prompt) |

---

## One-page recap
- **Where instructions live:** user / project / directory. Debug with `/memory` (locations) and `/context` (what's loaded).
- **What loads when:** CLAUDE.md + imports + unscoped rules every session. Directory CLAUDE.md and path rules when matching files are touched. Skills and commands on demand.
- **Scope a convention to file types** → a rule with a `paths:` glob.
- **Visible complexity** → plan mode. **Contained fix** → direct execution. **Heavy discovery** → Explore subagent.
- **Make vague requests checkable** → examples, tests first, interview.
- **CI** → `-p` to finish, a schema to parse, a fresh instance to review.
- **Next steps:** read the official exam guide (domain 3), practice each task statement on a real repo, work through the sample questions, then cover the other 4 domains.
