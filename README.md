# icm-architect

> **This is a fork** of [RinDig/icm-architect](https://github.com/RinDig/icm-architect). It adds guidance for workspaces that small or local models will run. See [What this fork adds](#what-this-fork-adds). Everything else follows upstream.

A Claude skill that designs any process, idea, or problem into an **ICM workspace** — folder structure as agent architecture — or restructures an existing folder, repo, or vault into one.

ICM (Interpretable Context Methodology) replaces orchestration code with structure: numbered folders carry sequencing, hierarchy carries context scoping, plain markdown files carry state. One agent, reading the right files at the right moment, does the work of a multi-agent framework — and a human can open any folder and see exactly what state the system is in.

The workspace is a library. The routing files are the catalog: small, stable, they point at everything and store almost nothing. One librarian — one model — walks the building, and the question decides which shelf gets walked to.

- Paper: [Interpretable Context Methodology: Folder Structure as Agent Architecture](https://arxiv.org/abs/2603.16021) (Van Clief & McDermott)
- Community: [Clief Notes](https://www.skool.com/cliefnotes)

## What it does

Two modes:

- **Build** — extracts the structure already present in how you describe your work (the stages, the human gates, what's stable vs. per-run), picks one of six proven forms, and scaffolds the smallest workspace that carries it.
- **Restructure** — audits an existing folder, classifies every file (catalog / contract / factory / product / dead), proposes a migration map for approval, then migrates and validates.

Six forms, one skeleton: **Pipeline** (production line), **Umbrella** (portfolio of pipelines), **Record library** (people/clients/sessions), **Knowledge bundle** (a navigable brain), **Context map** (an organization as a graph), **System map** (a folder later agents will edit — nouns, movements, change-impact). They compose and recurse.

Every result is validated with the **walk test**: an agent with no memory must orient, act, and report status from the files alone.

## What this fork adds

Upstream assumes an agent that follows prose contracts. Running an ICM pipeline with a ~9B local model showed that this assumption doesn't hold for small models. The folder structure held up, but the gates didn't. The model did the visible half of each step and dropped the checking half: it approved its own stages, wrote its own format instead of the template, marked unfinished work `ok`, rewrote whole files to "fix" one compile error, and ran a regeneration script that deleted part of the reference library.

This fork adds what fixed that, written as generic ICM guidance:

- **[references/hardening.md](references/hardening.md)**, a new reference covering:
  - **Failure modes:** what small models actually do wrong, and what each looked like.
  - **The enforcement layer:** a `status` script that prints the one next action, instantiation by script, a validator that prints `FIX:` lines, and approval through a command only a human can run (interactive terminal only), tied to a hash of the approved file.
  - **Contracts a small model can follow:** a start-here loop, a literal never-list, numbered steps with exact commands, entry gates, word-for-word STOP messages.
  - **Constrained fix loops:** a wrapper that shows the first error only, counts attempts per error and prints STOP after three. It also tells the agent to stop at environment blocks (such as antivirus) and never work around them.
  - **Tested snippet libraries:** "copy exactly, then rename", verified at runtime and not only at build time, plus an error→fix table with exact messages.
  - **The trouble log:** a per-run, append-only record, written automatically by the scripts where possible, with a report across runs.
  - **Protecting the factory:** atomic regeneration of reference material, and regeneration scripts for humans only.
  - **Session hygiene** and a **rollout ladder:** add each guardrail only when a real failure calls for it.
- **SKILL.md** changes:
  - a new Build step, **"Decide how hard the gates must be"**
  - a walk-test question: walk it as the *weakest agent that will run it*
  - a **"Prose gates leak"** guardrail

Everything else is unchanged from upstream, which is merged into this fork as it moves.

## Install

**Claude Code:** copy this folder to `~/.claude/skills/icm-architect/` (or `.claude/skills/icm-architect/` inside a project), then ask Claude to "ICM this" / "structure this for agents" / "build me a workspace for X".

**Claude apps:** upload `icm-architect.skill` (build it with the skill-creator packager, or zip the `icm-architect/` folder itself — the folder is the zip root, not its contents) via [Customize → Skills](https://claude.ai/customize/skills).

## Layout

```
icm-architect/
├─ SKILL.md              the method: invariants, build mode, restructure mode, walk test
├─ references/
│  ├─ core.md                 five principles, five-layer hierarchy, naming, token discipline
│  ├─ forms.md                the six forms in depth: skeletons, moves, failure modes
│  ├─ system-map.md           audit pipeline for the System map form
│  ├─ reference-integrity.md  restructure move-safety gate
│  └─ hardening.md            enforcement for small/local models: validators, approval, fix loops, trouble log
└─ assets/templates/     copyable starters: CLAUDE.md, CONTEXT.md, stage contract,
                         node card, object/process cards, schema, questionnaire
```

MIT licensed, like the protocol it serves.
