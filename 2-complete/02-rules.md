# 02 · Rules

Goal: write down what the agent must never get wrong, so it is not just in your head.

**Adds:** `rules/determinism.md`, `rules/error_types.md`, `rules/set_paths.md`.

## What a rule is

A short file with four headings: **Rule**, **Why**, **Enforced by**, **If violated**. A rule captures something an expert knows and a model does not. The "Enforced by" line is the honest part: it says where code checks the rule and where the agent is the only check.

Write a rule for every failure that would be **silent**. A wrong band still looks like a band; a set from the wrong folder still plots. Here there are three.

```bash
mkdir rules
```

## determinism

The design principle, written down: numbers come from the toolchain, never from the model. It also says what the agent may do, and what it may not do without being asked.

<!-- file: rules/determinism.md whole -->
````markdown
# Determinism

## Rule
Anything that produces a curve, a band, a ratio, a number or a pass/fail verdict comes from `pdfplot.py` through `run_plot.py`. The assistant selects the sets, flavours and options, runs the driver, and explains the output. It does not calculate, rescale, smooth or "fix up" results itself.

## Why
The same plot file must give the same figure and the same numbers whoever runs it. A model that re-derives a band or an interpolation each time introduces small, silent differences that cannot be traced afterwards.

## Allowed
- Writing or editing a plot file in `local/`, and `local/sets.md`.
- Running `python run_plot.py ...` and reading its output.
- Reading the PNG and describing it.

## Not allowed without the user asking
- Editing `pdfplot.py` or `run_plot.py` to make one plot work. If the toolchain needs a change, say what and why, make it as a separate change, and tell the user it affects every plot.
- Post-processing results (dropping points, rescaling a band, "correcting" a ratio) to make a comparison look right. If a result looks wrong, report it and find the cause.

## Enforced by
Partly. `.claude/settings.json` asks before any edit of `pdfplot.py` or `run_plot.py`, and `run_plot.py` saves every figure under `plots/`. Nothing stops the assistant from doing arithmetic in its reply: that part is a behaviour rule.

## If violated
Discard the hand-made numbers, rerun through the driver, and tell the user what happened.
````

## error_types

The lesson of tutorial 1 (mean ± std is wrong for Hessian sets) plus the confidence-level convention. This one is enforced in code: `evaluate()` sends Hessian sets to LHAPDF's routine and uses our own formula only for Monte Carlo, and the tests check both against numbers computed independently. The formulas are written out in tutorial 1.

<!-- file: rules/error_types.md whole -->
````markdown
# Error types and confidence levels

## Rule
A band is computed with the method of the set's `ErrorType` and shown at one confidence level for all sets:

- `hessian` and `symmhessian` sets (CT18, CT25, MSHT20): LHAPDF's own routine, `PDFSet.uncertainty(values, cl)`, over all members.
- `replicas` (Monte Carlo) sets, and a custom subset of replicas: our own formula, the mean and standard deviation of replicas 1…N, times n_sigma(cl).

Never use mean and standard deviation for a Hessian set. "68 %" means one sigma, 68.268949 %, which is LHAPDF's default confidence level, so sets built for different levels (CT18 for 90 %, CT25 for 68 %) are compared at the same one.

## Why
Hessian members are not random samples. For CT18, gluon at x = 0.1 and Q = 100 GeV, mean and standard deviation give about ±0.7 %, while the 68 % Hessian band is about ±2.4 %. Comparing a 90 % band with a 68 % band without rescaling makes one set look 1.6 times less certain. LHAPDF reads a literal `cl=68` as 0.994 sigma, and CT25's own level is listed as 68.0, so at one sigma LHAPDF widens CT25's native band by 0.6 %. That is LHAPDF's convention and we follow it.

## Enforced by
`evaluate()` in `pdfplot.py` sends Hessian and symmetric Hessian sets to LHAPDF, uses our formula only for replicas, refuses `members` on any other kind of set, and raises on an unknown error type. `tests/test_pdfplot.py` checks that the right values and confidence level reach LHAPDF, that the CT18 band equals the master formula written independently in the test, and that the Monte Carlo branch gives the hand-computed numbers. `run_plot.py` prints each set's `ErrorType` and native CL so the report can quote them.

## If violated or unsupported
For an error type that `evaluate()` does not support, say so; add a branch (and a test) only if the user asks. When reporting, always name the CL of the bands and say if any was rescaled.
````

## set_paths

Protects against the same set name pointing at different grids in different folders, by never guessing a path.

<!-- file: rules/set_paths.md whole -->
````markdown
# Set paths

## Rule
A PDF set name means something only together with the folder it was found in. Take paths from `local/sets.md` or from the user. Never guess a path and never substitute a "similar" set when the requested one is missing: ask.

## Why
The same set name can exist in two places (a shared and a personal LHAPDF folder, an old and a new download) with different grids. The name alone then gives different numbers depending on which folder LHAPDF searches first.

## Enforced by
`load_pdfs()` raises `FileNotFoundError` if the set is not in the folder it was given, and puts that folder first in LHAPDF's search path. `run_plot.py` prints the folder every set was loaded from. Nothing prevents two folders holding the same name; `find-pdf-sets` asks the user which one is meant.

## If violated
If a plot used a set from an unexpected folder, say so, rerun with the right path, and update `local/sets.md`.
````

Notice that the three rules name their enforcement in different places: settings, driver, tests, `evaluate()`, or nothing. Chapter 06 comes back to why that matters.

## Checkpoint

```bash
ls rules
```

```text
determinism.md  error_types.md  set_paths.md
```

The skills of chapter 04 will start with "Read first:" and name these files, so the agent reads the right invariants before it acts.

Next: [03 Examples and docs →](03-examples-and-docs.md)
