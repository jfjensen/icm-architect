# Hardening an ICM for weaker agents

A strong model follows prose contracts. A small or local model (7–14B), or any model on a long, error-heavy task, follows them *partially*: it skips the self-check, writes its own format instead of the template, marks unfinished work `ok`, approves its own stage, and rewrites whole files when a fix fails. The folder structure still holds. What breaks is the **gates**, because prose gates are only as strong as the agent reading them.

The fix is not more prose. **Every rule an agent has broken once becomes a script it cannot talk its way around.** Read this when the workspace will be run by a local/small model, when a run went off-contract, or when a pipeline's stages involve a fail-fix loop (code, data validation, builds).

These lessons come from running a code-generation pipeline with a ~9B local model. The patterns are domain-agnostic.

Contents: Failure modes · Enforcement layer · Contracts a small model can follow · Constrained fix loops · Tested snippet libraries · The trouble log · Protect the factory · Session hygiene · Rollout order

## Failure modes seen in practice

Expect all of these. Design against them before the first run.

| Failure | What it looked like |
|---|---|
| Skips the gate | Moved to the next stage without approval; approved its own stage by editing `status: approved`; invented side files (`x.md.status`) to record approval |
| Ignores the template | Wrote the plan in its own YAML instead of filling in the template's sections |
| Invents parameters | Made-up CLI flags and API names, where the exact command was one file away |
| Accepts impossible inputs | A brief combining options that can't work together got written *and* approved |
| Fake success | Marked a step `ok` with "needs investigation" in the notes; a README describing features that don't exist |
| Rewrite-as-fix | Answered each compile error by redesigning the whole step, which produced new errors; "three attempts" became three rewrites |
| Freelancing on the factory | Ran a docs-regeneration script unasked; it was interrupted halfway and deleted a chunk of the reference library |
| Stale context | Copied acceptance criteria from a *previous* attempt's brief still in its conversation |
| Folder hygiene | Hand-named run folders that broke the naming scheme; app code outside the product folder |

The pattern: the model does the *visible* part of each step (write a file, report progress) and drops the *checking* part.

## The enforcement layer

Put four small scripts between the agent and the gates. They are the workspace's immune system; keep them in `scripts/`.

**1. `status` → exactly one next action.** Scans the product folders and prints one line per unit of work: `NEXT: open <contract> and follow it from Step N`, `WAITING: the user must approve <file>`, or `NEXT: <file> is not ready, fix the FIX lines`. The entry file's first instruction is "run status, do what it says." The agent never has to work out where it is.

**2. `new_<unit>` → instantiate by script.** Copies the template folder with a validated name (`YYYY-MM-DD_<slug>`). Invariant 10 ("instantiate by copying") done by hand still drifts. `status` flags any folder it didn't create as invalid.

**3. `check_<unit>` → a validator that prints `FIX:` lines.** One line per problem, each saying *what to do*, not only what's wrong ("`02_plan.md`: missing section `## Build order`. Fill in the template, don't write your own format"). What to check:
- required template sections present, no `{placeholders}` left
- frontmatter values from a closed set
- **cross-field contradictions** (option A can't be combined with option B): catch them at the first stage, not after the build
- values that must match earlier stages (the plan's command matches the brief's choices)
- named references exist (a snippet file, a recipe, a doc page)
- blocklists of known hallucinations (old or non-existent API names, forbidden dependencies)
- structural facts about the product (declared modules, one definition per type, no stray files in the run folder)

Every stage contract's self-check step becomes "run the validator, fix every FIX line, repeat until `RESULT: OK`". `status` runs it too, so a failing file can never reach `WAITING`.

**4. `approve` → human-only approval, bound to content.** The agent writes `status: draft`. Approval happens only through a command the *human* runs:
- it refuses unless stdin and stdout are an interactive terminal (`sys.stdin.isatty() and sys.stdout.isatty()`). Agent shell tools are never interactive, and piping `yes` in doesn't pass either.
- it refuses while the validator still reports problems for that stage
- it asks the person to type a word (`approve`), then records `sha256` of the approved file in a hidden `.approvals.json`
- `status` treats a stage as approved **only** if the recorded hash matches the file on disk. A hand-edited `status: approved`, a side file, or an edit after approval all read as *not approved*.

This is tamper-*evident*, not tamper-proof: an agent with a shell could forge the JSON. It stops the realistic failure, a model that flips a status line to keep moving. Add "never create or edit `.approvals.json`" to the entry file's never-list.

## Contracts a small model can follow

- **Entry file = a loop, not a tour.** First section: "1. run status 2. do what its line says 3. follow the contract's steps in order, stop at STOP." Then a short **Never** list naming each observed failure literally ("never approve anything", "never rewrite a whole file to fix an error", "never run the regeneration script"). Then the map.
- **Numbered steps, one action each,** with the exact command or file path in the step. "Run the check command for your platform" fails. "Run `python scripts/check.py <run>`" works.
- **Entry gate** at the top of every contract: "Before you start: status must show stage N−1 approved. If not, stop."
- **A STOP step with the verbatim message** to send the user, including the exact approval command. Small models improvise badly at hand-offs.
- **"Edit the file that is already there. Keep every `##` heading."** Say it explicitly. Otherwise the template is treated as a suggestion.
- **Ask for rules, not only features.** The brief stage should ask for exact behaviour of the tricky parts (how a count or streak is computed, what happens on empty input). Vague rules become invented logic later.
- **One-line commands.** Multi-line shell continuations (`\` vs `` ` ``) break across shells. Write commands that paste into any of them.
- **Separate the lookup tables from the contract.** An error→fix table or a platform matrix is L3 material with its own file, loaded only by the step that needs it.

## Constrained fix loops

Any stage with a fail-fix loop (compiling code, validating data, running tests) needs the loop itself scripted:

- **A wrapper replaces the raw tool** (`check.py` instead of `cargo check`, a `validate` wrapper instead of the raw validator). It picks the right flags from the brief, so the agent never assembles a command.
- **Show only the first error.** Print "N more after this one: ignore them." Fixing the first error often clears the rest, and a wall of errors makes a small model rewrite everything.
- **Count attempts per error, in the wrapper.** Key on code + message + file, not line number (lines move as you edit). Print "attempts left: N". After three failed attempts print **STOP** and exit with a distinct code. The limit then enforces itself.
- **Define an attempt narrowly:** one small edit aimed at that one error. Rewriting a file, restructuring, or merging files is *not* an attempt and isn't allowed as a fix. When the first error changes, the count resets.
- **Escalation ladder per attempt:** 1) the error table, 2) compare with the snippet it was copied from, 3) the reference docs. Then stop.
- **Recognise environment failures** (antivirus quarantining a fresh binary, permission errors, locked files) and route them to "stop and tell the human". They don't count as attempts, and the agent must *never try to get around them*. Say so explicitly, or an eager agent will rename binaries and fiddle with build settings.

## Tested snippet libraries

For generative stages, give the model something to **copy** rather than write:

- A small library of canonical snippets (8–10 patterns covering most units of work), each headed `RECIPE / USE WHEN / DOCS / NEEDS`. Keep them in a buildable project so they can be re-verified with one command. Index them in a README table ("I need… → copy from").
- **Verify at runtime, not only at build time.** Two of nine snippets compiled cleanly and were still wrong: an error state that never reset, and a browser-only API that crashed server-side rendering. A snippet library with a bug spreads that bug into every run.
- The contract rule is **"copy exactly, then rename"**: keep the structure, hooks and nesting; change only names, types, text and styling. Each plan step names the snippet it starts from, and the validator checks that the file exists.
- Pair it with an **error→fix table** holding the *exact* message text. Trigger each common mistake on purpose in a scratch project and record the real message, because models search by message. Include the "compiles but wrong" cases too.

## The trouble log

Make every run leave a record of where it struggled. That record is how the factory improves.

- One append-only `trouble-log.md` per run: `time | stage | kind | what happened | attempts | outcome`.
- **Log automatically wherever a script can see it.** The fix-loop wrapper logs each error when it first appears, when it's fixed and when it's STOPped. The validator logs FIX lines when they appear and when they're resolved. Agents forget to log. Scripts don't.
- For what scripts can't see, give the agent a one-line logger with a closed set of kinds: `stuck` (didn't know how, and what it read), `runtime-bug`, `user-feedback`, `docs-gap`, `blocked`, `other`. Put the exact call in the contract step where it happens.
- A report script sums up all runs: problems by stage and kind, the most common messages (normalised: strip locations and numbers, keep error codes), and everything that ended blocked.
- After a failed run, **seed the log with reconstructed incidents** ("attempt 1 (reconstructed): …") before resetting, so the history survives the reset.
- Close the loop: the most frequent entries become new error-table rows, new validator checks, new snippets or new never-list lines. A kind that keeps recurring at one stage means that stage's contract needs splitting or scripting.

## Protect the factory

- **Regenerating reference material must be atomic.** Build into `<dir>.tmp`, swap it in only when it's complete, and refuse to swap if too many items failed. A script that deletes the output first and rewrites it second will, sooner or later, be interrupted halfway.
- **Regeneration scripts are human-only** (on the never-list). The factory is stable by definition, and an agent "investigating" should log a `docs-gap`, not rebuild the library.
- **Keep the factory under version control,** so any damage is one `git checkout -- <dir>` away. Keep product folders out of the commit until they're worth keeping.
- **Validate the factory's own snippets on every change** (build + runtime pass) before a run can copy them.

## Session hygiene

- **A fresh agent session per stage** (at least per run). Every stage's inputs are files, so the conversation is not needed, and it actively harms: old briefs, abandoned designs and earlier errors leak into the new attempt.
- **Resetting a run means resetting files:** stage files back to the templates, product folder removed, stray files deleted, approval records cleared. Keep a copy of the failed attempt *outside* the workspace, where the agent can't find and reuse it.

## Rollout order

Don't build all of this up front. Climb the same ladder as the rest of ICM:

1. Prose contracts + templates (every workspace).
2. When the agent is small, or a run goes off-contract: `status` + `new_<unit>` + checklist-style contracts with STOP messages.
3. When a template is ignored or impossible inputs get through: the validator, called from the contracts and from `status`.
4. When the agent approves itself: `approve` with content fingerprints.
5. When a fix loop spirals: the wrapper with first-error-only and attempt counting, the error table and the snippet library.
6. From the first failed run on: the trouble log and report.

Each rung answers a failure that actually happened. That is the same "three occurrences before it's a pattern" discipline, applied to guardrails.
