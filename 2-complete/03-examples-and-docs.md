# 03 · Examples and docs

Goal: give the agent a concrete case it can start from, say how far that case is validated, and write down what does not work yet.

**Adds:** `examples/ct18-vs-ct25.md`, `examples/ct18-vs-ct25-ratio.md`, `docs/LIMITATIONS.md`.

## Examples: cases with a status

Skills are generic procedures; an **example** is one concrete case: what to run, with which inputs, what to expect, and how far it has been validated. (Larger projects often call these *recipes*; `examples/` says the same thing more plainly.)

Every example starts with a `status:` line: `validated` (reproduces a stated reference within a stated tolerance), `partial` (validated somewhere else, or only some outputs) or `unvalidated` (no reference comparison yet). That line stops the agent from presenting an unchecked setup as a result: the rules and the skill tell it to repeat the status next to the numbers.

```bash
mkdir examples docs
```

This example is `partial`, and says why: the band computation is validated against LHAPDF, but the figure has not been compared to a published one. It contains a plot file the agent copies and adapts. It is set at Q = 2 GeV, where the gluon is large and its uncertainty easy to see.

<!-- file: examples/ct18-vs-ct25.md whole -->
````markdown
# CT18NNLO vs CT25NNLO

status: partial

## What
Compare the gluon, up, down and anti-up distributions of CT18NNLO and CT25NNLO at Q = 2 GeV, with 68 % CL bands and a ratio to CT18NNLO underneath. At this low scale the gluon is large and its uncertainty is easy to see.

## Inputs
- `CT18NNLO`: hessian, 59 members, native CL 90 % (public LHAPDF server).
- `CT25NNLO`: hessian, 69 members, native CL 68 %. Not on the public server at the time of writing: the user supplies it.

## Plot file
An example is a case ready to run. Copy this to `local/ct18_vs_ct25.json` and set each path to the folder that contains the set (from `local/sets.md`):

```json
{
  "sets": {"CT18NNLO": "pdfsets", "CT25NNLO": "pdfsets"},
  "flavors": ["g", "u", "d", "ubar"],
  "Q": 2,
  "output": "ct18_vs_ct25.png"
}
```

Run: `python run_plot.py local/ct18_vs_ct25.json`

## Expected
The gluon ratio of CT25NNLO to CT18NNLO is about 0.98 at x = 0.001, 0.95 at x = 0.1 and 0.74 at x = 0.5. The gluon bands are wide at small x (about ±12 % for CT18NNLO and ±14 % for CT25NNLO at x = 0.001) and CT25NNLO's is the narrower from x = 0.01 on (±4 % against ±5 % there, ±15 % against ±19 % at x = 0.5). The u-quark ratio stays within about 3 % of 1.

## Validation
- Validated: our Hessian band equals LHAPDF's `PDFSet.uncertainty()` at the native CL (`tests/test_pdfplot.py`, CT18NNLO, gluon).
- Not validated: no comparison against the published CT25 figures. Until one is made, the status stays `partial`.
````

The second example is the same comparison showing **only the ratio**, the plot researchers ask for most often when the differences are what matters. Having it as an example means the agent can start from it when someone says "just show me the ratio".

<!-- file: examples/ct18-vs-ct25-ratio.md whole -->
````markdown
# CT18NNLO vs CT25NNLO: only the ratio

status: partial

## What
The same comparison as `ct18-vs-ct25`, but showing **only the ratio** of CT25NNLO to CT18NNLO, one panel per parton. Use it when the differences between the sets matter and the curves themselves do not.

## Inputs
The same two sets as `ct18-vs-ct25`.

## Plot file
Copy to `local/ct18_vs_ct25_ratio.json` and set each path to the folder that contains the set (from `local/sets.md`):

```json
{
  "sets": {"CT18NNLO": "pdfsets", "CT25NNLO": "pdfsets"},
  "flavors": ["g", "u", "d", "ubar"],
  "Q": 2,
  "show": "ratio",
  "output": "ct18_vs_ct25_ratio.png"
}
```

Run: `python run_plot.py local/ct18_vs_ct25_ratio.json`

## Expected
The reference set, CT18NNLO, is the line at 1 with its own band around it; CT25NNLO is drawn relative to it. The numbers printed by the driver are the same as for `ct18-vs-ct25`. Widen the range with `"ratio_ylim": [0.6, 1.4]` if the curves leave the default `[0.8, 1.2]` at large x.

## Validation
Same as `ct18-vs-ct25`: the band computation is validated against LHAPDF's own routine; no comparison with the published CT25 figures yet, so the status stays `partial`.
````

Add examples as you validate comparisons: `examples/ct18-vs-nnpdf40.md`, a large-x zoom, a valence-quark comparison. Start each as `unvalidated` and promote it only when it reproduces something you can point to.

## Limitations: be honest about the edges

An agent that does not know its own limits overpromises. This file is what it reads before saying "it will work", and what the welcome message of chapter 05 draws its caveats from.

<!-- file: docs/LIMITATIONS.md whole -->
````markdown
# Limitations

What does not work yet, and what is known to differ. Read this before telling a user "it will work".

| Topic | State | What to do |
|---|---|---|
| CT25NNLO | Not on the public LHAPDF server at the time of writing (its `.info` lists `Reference: tbc`) | The user supplies it; never try to download it |
| Outside the grid | LHAPDF extrapolates below a set's `XMin` and `QMin`; the driver does not warn | Check `XMin`, `QMin`, `QMax` in the set's `.info` before trusting a curve at very small x or Q |
| "68 %" convention | "68 %" means one sigma, 68.268949 %, LHAPDF's default; a literal `cl=68` in LHAPDF is 0.994 sigma. CT25's own level is listed as 68.0, so its one-sigma band is 0.6 % wider than its native eigenvector band | Say so if someone compares with a number computed at a literal 68 % |
| Monte Carlo bands | Our own formula: mean and standard deviation of the replicas times n_sigma (Gaussian assumption), not percentiles; the same for a custom subset of replicas | Say so if the replica distribution is far from Gaussian |
| Ratio bands | Each set's band is divided by the central value of the reference set; the uncertainty of the reference is not propagated into the band of the other set | Do not read the overlap of two ratio bands as a significance test |
| Hessian bands | Computed by LHAPDF's `PDFSet.uncertainty()` (the asymmetric master formula for `hessian`, the symmetric one for `symmhessian`) | None |
| Single-member sets | Drawn with a zero-width band | Do not read it as "no uncertainty" |
| Number of sets | At most 6 per plot (one colour each) | Plot in two runs |
| Speed | Every member is evaluated at every x point in Python: a first plot takes several seconds | Lower `npoints` for a quick look |
| Not yet tried | No live agent session was recorded; the skills were followed by hand | Report anything the agent does that the skills do not describe |
````

## Checkpoint

```bash
ls examples docs
```

```text
docs:
LIMITATIONS.md

examples:
ct18-vs-ct25.md  ct18-vs-ct25-ratio.md
```

Next: [04 Instructions →](04-instructions.md)
