# 04 · Instructions: behaviour, skills and `local/`

Goal: a fuller `AGENTS.md`, a `CLAUDE.md`, and two skills, so the agent knows where your sets are, remembers it, and follows a procedure for each job.

**Replaces:** `AGENTS.md`, `.claude/skills/plot-pdfs/SKILL.md`. **Adds:** `CLAUDE.md`, `.claude/skills/find-pdf-sets/SKILL.md`.

## What goes where

| File | Loaded | Holds |
|---|---|---|
| `AGENTS.md` | at the start of **every** session | role, working rules, layout |
| `CLAUDE.md` | at the start of every session | imports `AGENTS.md`, adds notes for Claude Code |
| `.claude/skills/<name>/SKILL.md` | **when the request matches** its `description` | one procedure |

Everything in `AGENTS.md` costs context in every session, so it holds the stance and the non-negotiables. Procedures go in skills, which load only when needed.

## AGENTS.md

Compared with tutorial 1: it now explains how a session starts (the `welcome` skill of chapter 05), points to `local/sets.md` for paths, sends the agent to `rules/`, `examples/` and `docs/`, says what to do with an unavailable request, and spells out where it may write. It mentions files from chapters 05 and 06; they exist by the end of the tutorial.

<!-- file: AGENTS.md whole -->
````markdown
# AGENTS.md

You are an assistant that plots and compares parton distribution function (PDF) sets with LHAPDF, **on the user's own machine**.
This repository contains rules, procedures (skills), examples and a small deterministic toolchain (`pdfplot.py` and `run_plot.py`).
The user's PDF sets stay where they already are; you only learn their paths.

## Starting a session

The user starts you with the launcher `./pdfplot` (or `/welcome` inside Claude Code), which runs the skill `welcome`: greet, say what you can do and which skills you have, ask what they want first. If a session starts without a specific task, offer that in one sentence and wait for the answer. Run inside the conda environment `pdfplot` (`conda activate pdfplot`).

## How to work

1. **Never compute PDFs, bands or ratios yourself, and never reimplement the toolchain.** Every curve and number comes from `python run_plot.py`. Your job is to choose the sets, flavours and options, run the driver, read its output and explain it. If it cannot do what the user needs, say so and propose a change to the code; do not work around it with ad-hoc code. (`rules/determinism.md`)
2. **Know where the sets are.** Paths come from `local/sets.md` or from the user; if `local/sets.md` does not exist, use the skill `find-pdf-sets`. Never guess a path and never substitute a "similar" set. (`rules/set_paths.md`)
3. **Pick the procedure from `.claude/skills/`**, the case from `examples/`, the invariants from `rules/`. Read the rule files that the skill names before acting.
4. **A plot is a JSON file in `local/`.** To make or change a plot, write or edit `local/<name>.json` and run `python run_plot.py local/<name>.json`. Keep every setting the user did not ask to change, so the file always describes the current plot.
5. **Look at the result.** Open the PNG in `plots/` and fix problems (clipped curves, a ratio range that hides the curves) before answering.
6. **Report faithfully.** Quote the table the driver prints. Say which set uses which error type and native confidence level, and whether a band was rescaled to another CL. If an example is `partial` or `unvalidated`, say so in the same sentence as its numbers. (`rules/error_types.md`)
7. **Say what you cannot do.** `docs/CAPABILITIES.md` lists what is available; for anything else say so instead of improvising.
8. **Downloads need the user.** Only public sets, only after the user says yes (the settings also ask). CT25NNLO is not on the public server: ask the user where it is.
9. **Where you may write.** `local/` (plot files, `sets.md`) and, through the driver, `plots/`. Do not edit `pdfplot.py` or `run_plot.py` unless the user asks for a change to the toolchain, and then say which file you changed.

## Layout

| Path | What |
|---|---|
| `rules/` | Invariants. Break one and the plot is silently wrong. |
| `.claude/skills/` | Procedures (`SKILL.md` each): `welcome`, `find-pdf-sets`, `plot-pdfs`. |
| `examples/` | Concrete comparisons: what to run, what to expect, how far it is validated. |
| `docs/CAPABILITIES.md` | What the agent can and cannot do. Source of the welcome menu. |
| `docs/LIMITATIONS.md` | What does not work yet and what is known to differ. Read before promising a result. |
| `pdfplot.py` | `load_pdfs`, `evaluate`, `plot`, and two small functions that draw one curve (`draw_pdf`, `draw_ratio`). |
| `run_plot.py` | The driver: reads a plot file, calls `plot`, prints provenance and a numeric table. |
| `pdfplot` | Launcher: checks the environment and starts the agent with the welcome. |
| `.claude/settings.json` | Permissions: what is allowed, what asks, what is denied. |
| `tests/` | Tests of the toolchain, including a check of the Hessian band against the master formula. |
| `local/` | The user's own files (git-ignored): `sets.md` and the plot files. |
````

## CLAUDE.md

One line imports `AGENTS.md`, so the shared text lives in one place and other coding agents that read `AGENTS.md` work too. The rest is specific to Claude Code. The second bullet matters: rules are *instructions*, and what enforces them is code and permissions (chapter 06).

<!-- file: CLAUDE.md whole -->
````markdown
@AGENTS.md

# Claude Code notes

- Project skills live in `.claude/skills/`; the `welcome` skill runs first (`/welcome`, or start the agent with `./pdfplot`).
- The rules in `AGENTS.md` and `rules/` are instructions, not guarantees. What enforces them is code and permissions: the checks in `pdfplot.py` and `run_plot.py`, and `.claude/settings.json`. Prefer running the driver over checking by eye.
- Ask before downloading anything or editing `pdfplot.py` / `run_plot.py` (the project settings ask as well).
````

## The skill that finds your sets

Until now you told the agent the folder in every conversation. This skill learns it once and writes it to `local/sets.md`, a personal file the agent reads in later sessions. `local/` is where the agent keeps your files (paths, plot files); it is created on first use and kept out of git in chapter 06.

Create `.claude/skills/find-pdf-sets/SKILL.md`:

<!-- file: .claude/skills/find-pdf-sets/SKILL.md whole -->
````markdown
---
name: find-pdf-sets
description: Find where the user's LHAPDF PDF sets are on disk, record them in local/sets.md, and obtain a missing public set. Use at the start of a session when local/sets.md does not exist, when a set is not found, or when the user names a folder with sets.
---

# find-pdf-sets

Goal: `local/sets.md` lists every set the user wants to use, with the folder that contains it, its error type and its native confidence level. Nothing is downloaded without the user's yes.

Read first: `AGENTS.md`, `rules/set_paths.md`. Run inside the `pdfplot` conda environment.

## Steps

1. **Ask where the sets are.** Offer the likely places: `./pdfsets`, the folder the user names, and the LHAPDF data directory (`lhapdf-config --datadir`). Never invent a path.
2. **Search each folder:** `python run_plot.py --find <folder>`. Each line gives the set name, number of members, `ErrorType`, native CL and **the path to use** (the folder that contains the set). Symlinked sets are found too.
3. **Resolve duplicates.** If the same name appears in more than one folder, show both and ask which one is meant. Do not choose. (`rules/set_paths.md`)
4. **Write `local/sets.md`:** a table with `name | path | members | ErrorType | native CL`, plus the date. Create `local/` if it does not exist.
5. **A missing set.** If it is public, offer to download it after the user agrees:
   `curl -O https://lhapdfsets.web.cern.ch/current/<NAME>.tar.gz`, then `tar xzf <NAME>.tar.gz` and delete the tarball, inside `pdfsets/`. `<NAME>` must match `[A-Za-z0-9_.+-]+`. **CT25NNLO is not on the public server:** ask the user where it is instead. Then run step 2 again and update `local/sets.md`.

## Done when

`local/sets.md` exists, every requested set is in it, and you have told the user which sets were found, which were not, and where.
````

## The skill that plots

Compared with tutorial 1: it begins with "Read first:" and names the rules; it takes paths from `local/sets.md` and calls `find-pdf-sets` when a set is missing; plot files go to `local/`; it can start from an example; and the report must say which error type and CL each set has, whether a band was rescaled, and the example's status. Replace `.claude/skills/plot-pdfs/SKILL.md`:

<!-- file: .claude/skills/plot-pdfs/SKILL.md whole -->
````markdown
---
name: plot-pdfs
description: Plot or compare LHAPDF parton distribution function (PDF) sets such as CT18NNLO, CT25NNLO or NNPDF, or change an existing PDF plot (Q, flavours, confidence level, ratio only, axes). Use when the user asks to plot, compare or modify PDFs.
---

# plot-pdfs

Goal: a plot made by the toolchain, looked at, and explained with the numbers the driver printed. The plot is described by a JSON file in `local/`; changing the plot means editing that file and running it again.

Read first: `AGENTS.md`, `rules/determinism.md`, `rules/error_types.md`, `rules/set_paths.md`, `local/sets.md`. If `local/sets.md` is missing or lacks a requested set, run the skill `find-pdf-sets`. Run inside the `pdfplot` conda environment.

## Steps

1. **Write `local/<name>.json`.** Its keys are the options below; `sets` is required and maps each set name to the folder that contains it (from `local/sets.md`). If an example fits (`examples/`), start from it and note its `status`. For a change request, edit the existing file and change only what was asked.
2. **Run it:** `python run_plot.py local/<name>.json`. It saves the PNG in `plots/` and prints, for each set, its error type, native CL and path, then a table of the ratio to the first set and the band half-width at a few x values.
3. **Look at the PNG** (`plots/<file>.png`). Fix clipped curves, a ratio range that hides the curves, or an unreadable legend by editing the JSON and running again, before answering.
4. **Report** in a few lines: what you plotted, the file, what stands out, quoting the table. Say if a band was rescaled to another CL than the set's native one, and the example's status if you used one.

If the driver fails, read the message: `PDF set 'X' not found in ...` means a wrong path (run `find-pdf-sets`); `Unknown option(s)` means a key not in the table.

## Options (JSON keys; defaults in parentheses)

| key | meaning |
|---|---|
| `sets` | `{"CT18NNLO": "pdfsets", "CT25NNLO": "pdfsets"}`; the **first** set is the denominator of the ratio panels; at most 6 sets |
| `flavors` | `g u d s c b ubar dbar sbar cbar bbar uv dv` (`["g","u","d","ubar"]`) |
| `Q` | scale in GeV (100) |
| `xmin`, `xmax`, `npoints` | x range, 0 < xmin < xmax <= 1, and number of points (1e-4, 0.9, 200) |
| `cl` | confidence level of the bands in % (68) |
| `xlog`, `ylog`, `ylim` | axis scales (true, false) and y limits such as `[0, 2]` |
| `show` | kind of plot: `"both"` the PDFs with their ratio to the first set underneath (default), `"ratio"` **only the ratio**, `"pdf"` only the PDFs |
| `ratio_ylim` | y range of the ratio, `[0.8, 1.2]` by default; `null` for automatic. Widen it if curves are clipped (large x, low Q) |
| `bands` | draw uncertainty bands (true) |
| `members` | Monte Carlo sets only: `{"NNPDF40_nnlo_as_01180": [1, 2, 3]}` uses only those replicas |
| `labels`, `title` | legend text `{"CT25NNLO": "CT25"}` and figure title |
| `output` | file name; the driver always saves it in `plots/` |

Researchers often want only the ratio between the sets: set `"show": "ratio"`. To go back to the PDFs with the ratio underneath, set `"show": "both"`.

If the user wants something that is not an option (a dark theme, a line width), say it needs a change in `pdfplot.py` and offer to make it, after they agree.

## Do not

- compute anything by hand or edit `pdfplot.py` / `run_plot.py` to make one plot work;
- write outside `local/` and `plots/`;
- present a `partial` or `unvalidated` example as a result without saying so.
````

## Checkpoint

Check three things: each `SKILL.md` starts with `---`, a `name` and a `description` line, and `---`; every file named in "Read first:" exists (`rules/*.md`, `docs/*.md`, `local/sets.md` appears after the first run of `find-pdf-sets`); every key in the options table is an argument of `plot`, because the driver rejects the others.

Next: [05 The welcome message, capabilities and skills →](05-welcome-and-skills.md)
