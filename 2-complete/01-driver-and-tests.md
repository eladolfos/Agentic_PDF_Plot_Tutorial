# 01 · A stronger driver and tests

Goal: make the driver check its input, record where each set came from, and keep every file in `plots/`; then add tests, so you can edit the code with confidence.

**Replaces:** `run_plot.py`. **Adds:** `tests/test_pdfplot.py`.

If the way the JSON plot file reaches `plot` is not clear, read section 4 of [tutorial 1](../1-basic.md) first: it explains the file, the dictionary and `plot(**config)` step by step.

## What changes in the driver

| Tutorial 1 | Now |
|---|---|
| Any key in the JSON goes to `plot` | Unknown keys are rejected with the list of valid ones |
| Errors are Python tracebacks | Expected errors become one line and exit code 1, which the agent can read |
| `output` can be any path | The file always goes to `plots/` |
| Nothing says which set was used | A **provenance** line per set: members, `ErrorType`, native CL, folder |
| Cannot look for sets | `--find <folder>` lists the sets under a folder, following symlinks |

The code is split into small functions so each can be tested. Replace `run_plot.py` with the following, in order.

<!-- file: run_plot.py head -->
```python
"""The driver: the only way plots are made. The agent (or you) calls it.

  python run_plot.py local/<name>.json     make the plot described by that file
  python run_plot.py --find <folder>       list the PDF sets found under a folder

A plot file is JSON whose keys are the arguments of pdfplot.plot, for example
{"sets": {"CT18NNLO": "pdfsets", "CT25NNLO": "pdfsets"}, "flavors": ["g", "u"], "Q": 2}
"""
import inspect
import json
import os
import sys

import numpy as np

from pdfplot import plot
```

**Finding sets and describing them.** A folder holding `<name>/<name>.info` is a set. `followlinks=True` matters: people often symlink sets into a project, and `os.walk` skips symlinked folders by default. `describe` is the provenance line.

<!-- file: run_plot.py -->
```python
AT = (0.001, 0.01, 0.1, 0.5)  # x values quoted in the summary table


def read_info(set_dir):
    """Parse <set_dir>/<name>.info into a dict of strings."""
    name = os.path.basename(set_dir)
    info = {}
    for line in open(f"{set_dir}/{name}.info"):
        if ":" in line:
            key, value = line.split(":", 1)
            info[key.strip()] = value.strip()
    return info


def find_sets(folder, max_depth=2):
    """Return [(name, path)] for every LHAPDF set under `folder`; path is the folder that contains the set."""
    folder = os.path.abspath(os.path.expanduser(folder))
    found = []
    for here, subfolders, files in os.walk(folder, followlinks=True):  # sets are often symlinked
        name = os.path.basename(here)
        if f"{name}.info" in files:  # a folder holding <name>.info is a PDF set
            found.append((name, os.path.dirname(here)))
            subfolders.clear()  # members are files; do not descend into a set
        elif here[len(folder):].count(os.sep) >= max_depth:
            subfolders.clear()
    return sorted(found)


def set_dir(name, path):
    """The folder of the set itself, whether `path` contains it or is it."""
    path = os.path.abspath(os.path.expanduser(path))
    return f"{path}/{name}" if os.path.isfile(f"{path}/{name}/{name}.info") else path


def describe(name, path):
    """One line about a set: size, kind of error set, native confidence level and where it is."""
    folder = set_dir(name, path)
    info = read_info(folder)
    return (f"{name}: {info.get('NumMembers')} members, ErrorType={info.get('ErrorType')}, "
            f"native CL={info.get('ErrorConfLevel')}%  path: {os.path.dirname(folder)}")
```

**The numeric table**, as in tutorial 1 but as functions, so it can be tested.

<!-- file: run_plot.py -->
```python
def interp(x, xi, curve):
    """Value of `curve` at xi, interpolated in log x."""
    return float(np.interp(np.log(xi), np.log(x), curve))


def summarize(sets, x, curves):
    """Text table: each set's ratio to the first one and its band half-width, at a few x values."""
    first = next(iter(sets))
    lines = [f"Ratios are to {first}."]
    for flavor in curves[first]:
        for name in sets:
            cells = []
            for xi in AT:
                if not x[0] <= xi <= x[-1]:
                    continue
                central, lower, upper = (interp(x, xi, c) for c in curves[name][flavor])
                ref = interp(x, xi, curves[first][flavor][0])
                ratio = central / ref if ref else float("nan")
                width = 50 * (upper - lower) / abs(central) if central else float("nan")
                cells.append(f"x={xi:g}: ratio {ratio:.3f}, band +-{width:.1f}%")
            lines.append(f"[{flavor}] {name}: " + " | ".join(cells))
    return lines
```

**Running a plot file.** It checks the keys against the signature of `plot`, forces the output into `plots/`, calls `plot`, and returns the text: provenance first, table after.

<!-- file: run_plot.py -->
```python
def run(config_file):
    """Make the plot described by `config_file` and return the text to show the user."""
    config = json.load(open(config_file))  # the JSON file becomes a dictionary
    valid = set(inspect.signature(plot).parameters)
    unknown = sorted(set(config) - valid)
    if unknown:
        raise ValueError(f"Unknown option(s) {unknown}. Valid options: {sorted(valid)}")
    if "sets" not in config:
        raise ValueError("`sets` is required: {set name: folder that contains it}")
    # every plot goes to plots/, whatever the file asks for
    config["output"] = os.path.join("plots", os.path.basename(config.get("output", "pdf_comparison.png")))

    x, curves = plot(**config)  # `**` hands every key of the dictionary to plot() as an argument

    lines = [f"Saved {config['output']}", f"Bands at {config.get('cl', 68.0):g}% CL.", ""]
    for name, path in config["sets"].items():  # provenance: which set, from where
        lines.append(describe(name, path))
    return "\n".join(lines + [""] + summarize(config["sets"], x, curves))
```

**The entry point.** Expected errors (wrong path, wrong option, bad JSON) come back as one line of text and exit code 1.

<!-- file: run_plot.py -->
```python
def main(argv):
    try:
        if len(argv) == 3 and argv[1] == "--find":
            print("\n".join(describe(name, path) for name, path in find_sets(argv[2])) or "No LHAPDF sets found.")
        elif len(argv) == 2:
            print(run(argv[1]))
        else:
            print(__doc__)
            return 2
    except (OSError, ValueError, TypeError, KeyError) as err:  # json errors are ValueErrors
        print(f"Error ({type(err).__name__}): {err}", file=sys.stderr)
        return 1
    return 0


if __name__ == "__main__":
    sys.exit(main(sys.argv))
```

Plot files now live in `local/`, which the agent will own (chapter 04):

```bash
mkdir -p local
echo '{"sets": {"CT18NNLO": "pdfsets", "CT25NNLO": "pdfsets"}, "flavors": ["g", "u"], "Q": 2}' > local/test.json
python run_plot.py --find pdfsets
python run_plot.py local/test.json
```

```text
CT18NNLO: 59 members, ErrorType=hessian, native CL=90%  path: /home/you/PDFPlot/pdfsets
CT25NNLO: 69 members, ErrorType=hessian, native CL=68%  path: /home/you/PDFPlot/pdfsets
```

```text
Saved plots/pdf_comparison.png
Bands at 68% CL.

CT18NNLO: 59 members, ErrorType=hessian, native CL=90%  path: /home/you/PDFPlot/pdfsets
CT25NNLO: 69 members, ErrorType=hessian, native CL=68%  path: /home/you/PDFPlot/pdfsets

Ratios are to CT18NNLO.
[g] CT18NNLO: x=0.001: ratio 1.000, band +-12.2% | x=0.01: ratio 1.000, band +-5.0% | x=0.1: ratio 1.000, band +-4.6% | x=0.5: ratio 1.000, band +-18.8%
[g] CT25NNLO: x=0.001: ratio 0.979, band +-14.0% | x=0.01: ratio 1.012, band +-4.2% | x=0.1: ratio 0.945, band +-2.9% | x=0.5: ratio 0.737, band +-15.3%
[u] CT18NNLO: ...
```

Now break it on purpose: add `"colour": 1` to `local/test.json` and run again. You get `Error (ValueError): Unknown option(s) ['colour']. Valid options: [...]` and exit code 1.

## Tests

A wrong band still *looks* like a band, so mistakes in `evaluate` are silent. We test each branch (Monte Carlo, Hessian, symmetric Hessian, single member, valence quark), the two drawing functions and the three kinds of plot with a fake PDF object. For Monte Carlo the answers are numbers computed by hand; for Hessian sets the fake set records what we hand to LHAPDF (all members, member 0 first, the right confidence level). Then the driver, then the real sets: the most valuable test compares the CT18 band that LHAPDF gives with the master formula written independently in the test, so we do not just trust LHAPDF, and checks that one sigma is the native 90 % band divided by 1.645.

Create `tests/test_pdfplot.py`:

<!-- file: tests/test_pdfplot.py whole -->
```python
"""Tests of the toolchain. Run from the project folder:  python -m pytest -q

The real-set tests look for CT18NNLO and CT25NNLO in $PDFPLOT_SETS, then in ./pdfsets, then in ./
and are skipped if they are missing.
"""
import json
import os
import sys

import matplotlib.pyplot as plt
import numpy as np
import pytest

ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))
sys.path.insert(0, ROOT)

import run_plot  # noqa: E402
from pdfplot import draw_pdf, draw_ratio, evaluate, load_pdfs, n_sigma, plot  # noqa: E402


def _sets_dir():
    candidates = [os.environ.get("PDFPLOT_SETS"), f"{ROOT}/pdfsets", ROOT]
    for folder in filter(None, candidates):
        if all(os.path.isfile(f"{folder}/{n}/{n}.info") for n in ("CT18NNLO", "CT25NNLO")):
            return folder
    return None


SETS_DIR = _sets_dir()
need_sets = pytest.mark.skipif(SETS_DIR is None, reason="CT18NNLO and CT25NNLO not found (set PDFPLOT_SETS)")


# A stand-in for lhapdf.PDF whose x*f(x,Q) we control. Its set records every call to uncertainty(),
# so we can check what we hand to LHAPDF without testing LHAPDF itself.
class FakeSet:
    def __init__(self, error_type, cl):
        self.errorType, self.errorConfLevel, self.calls = error_type, cl, []

    def uncertainty(self, vals, cl):
        self.calls.append((list(vals), cl))
        return type("U", (), {"central": vals[0], "errplus": 3.0, "errminus": 1.0})()


class FakePDF:
    def __init__(self, value, error_type, cl=68.268949):
        self.value = value
        self._set = FakeSet(error_type, cl)

    def set(self):
        return self._set

    def xfxQ(self, pid, x, Q):
        return self.value * {2: 1.0, -2: 0.5}.get(pid, 1.0)  # u = value, ubar = value / 2


def pdfs(values, error_type, cl=68.268949):
    return [FakePDF(v, error_type, cl) for v in values]


X = np.array([0.1, 0.2])


# ---- evaluate: one test per branch -----------------------------------------------
def test_n_sigma():
    assert n_sigma(68) == pytest.approx(1.0, abs=1e-6)  # "68 %" means one sigma
    assert n_sigma(90) == pytest.approx(1.6449, abs=1e-3)
    assert n_sigma(95) == pytest.approx(1.9600, abs=1e-3)


@pytest.mark.parametrize("error_type", ["hessian", "symmhessian"])
def test_standard_sets_use_the_lhapdf_routine(error_type):
    members = pdfs([10, 13, 9, 11, 12], error_type)
    c, lo, up = evaluate(members, "g", X, 100)
    calls = members[0].set().calls
    assert len(calls) == len(X)  # one call per x point
    assert calls[0][0] == [10, 13, 9, 11, 12]  # all members, member 0 first
    assert calls[0][1] == pytest.approx(68.268949)  # "68 %" is handed over as one sigma
    assert (c, up, lo) == (pytest.approx([10, 10]), pytest.approx([13, 13]), pytest.approx([9, 9]))  # +3, -1
    evaluate(members, "g", X, 100, cl=90)
    assert members[0].set().calls[-1][1] == 90


def test_replicas_ignore_member_zero_and_use_sample_std():
    c, lo, up = evaluate(pdfs([999, 1, 2, 3, 4], "replicas"), "g", X, 100)  # member 0 must not enter
    assert c[0] == pytest.approx(2.5)
    assert up[0] - c[0] == pytest.approx(np.std([1, 2, 3, 4], ddof=1))


def test_replica_subset():
    c, _, _ = evaluate(pdfs([999, 1, 2, 3, 4], "replicas"), "g", X, 100, members=[3, 4])
    assert c[0] == pytest.approx(3.5)


def test_members_only_for_replica_sets():
    with pytest.raises(ValueError, match="only works for Monte Carlo"):
        evaluate(pdfs([10, 13, 9], "hessian"), "g", X, 100, members=[1])


def test_single_member_has_zero_width_band():
    c, lo, up = evaluate(pdfs([5.0], ""), "g", X, 100)
    assert (c == lo).all() and (c == up).all()


def test_valence_combination():
    c, _, _ = evaluate(pdfs([8.0], ""), "uv", X, 100)
    assert c[0] == pytest.approx(8.0 - 4.0)  # u - ubar


def test_unknown_flavor_and_error_type():
    with pytest.raises(ValueError, match="Unknown flavour"):
        evaluate(pdfs([1.0], ""), "top", X, 100)
    with pytest.raises(ValueError, match="Unsupported ErrorType"):
        evaluate(pdfs([1.0, 2.0], "banana"), "g", X, 100)


# ---- the driver, without any PDF set ---------------------------------------------
def make_set(folder, name, error_type="hessian"):
    set_folder = folder / name
    set_folder.mkdir(parents=True)
    (set_folder / f"{name}.info").write_text(f"NumMembers: 3\nErrorType: {error_type}\nErrorConfLevel: 90\n")
    return set_folder


def test_find_sets_depth_and_symlinks(tmp_path):
    make_set(tmp_path / "real" / "deep", "MySet")
    (tmp_path / "links").mkdir()
    (tmp_path / "links" / "MySet").symlink_to(tmp_path / "real" / "deep" / "MySet")
    assert run_plot.find_sets(tmp_path / "real") == [("MySet", str(tmp_path / "real" / "deep"))]
    assert run_plot.find_sets(tmp_path / "real", max_depth=1) == []  # too deep
    assert [name for name, _ in run_plot.find_sets(tmp_path / "links")] == ["MySet"]  # symlinked set


def test_describe_accepts_parent_or_set_folder(tmp_path):
    folder = make_set(tmp_path, "MySet")
    for path in (tmp_path, folder):
        assert run_plot.describe("MySet", str(path)) == (
            f"MySet: 3 members, ErrorType=hessian, native CL=90%  path: {tmp_path}")


def test_driver_reports_errors_instead_of_crashing(tmp_path, capsys):
    bad = tmp_path / "bad.json"
    bad.write_text(json.dumps({"sets": {"X": "."}, "colour": 1}))
    assert run_plot.main(["run_plot.py", str(bad)]) == 1
    assert "Unknown option(s) ['colour']" in capsys.readouterr().err
    bad.write_text(json.dumps({"sets": {"NOPE": str(tmp_path)}}))
    assert run_plot.main(["run_plot.py", str(bad)]) == 1
    assert "PDF set 'NOPE' not found" in capsys.readouterr().err
    bad.write_text(json.dumps({"flavors": ["g"]}))
    assert run_plot.main(["run_plot.py", str(bad)]) == 1
    assert "`sets` is required" in capsys.readouterr().err


def test_load_pdfs_missing_set(tmp_path):
    with pytest.raises(FileNotFoundError, match="NOPE"):
        load_pdfs("NOPE", tmp_path)


# ---- with the real sets ----------------------------------------------------------
@need_sets
def test_load_pdfs_accepts_parent_or_set_folder():
    assert len(load_pdfs("CT18NNLO", SETS_DIR)) == 59
    assert len(load_pdfs("CT18NNLO", f"{SETS_DIR}/CT18NNLO")) == 59


@need_sets
def test_hessian_band_matches_the_textbook_formula():
    """LHAPDF's band for CT18 (native CL 90 %) equals the master formula, written here independently."""
    members = load_pdfs("CT18NNLO", SETS_DIR)
    x = np.array([0.01, 0.1, 0.5])
    central, lower, upper = evaluate(members, "g", x, 10.0, cl=90)
    v = np.array([[m.xfxQ(21, float(xi), 10.0) for xi in x] for m in members])
    plus, minus = v[1::2], v[2::2]
    up = np.sqrt((np.maximum(np.maximum(plus - v[0], minus - v[0]), 0) ** 2).sum(axis=0))
    down = np.sqrt((np.maximum(np.maximum(v[0] - plus, v[0] - minus), 0) ** 2).sum(axis=0))
    assert central == pytest.approx(v[0])
    assert upper - central == pytest.approx(up, rel=1e-6)
    assert central - lower == pytest.approx(down, rel=1e-6)


@need_sets
def test_hessian_cl_rescaling():
    """One sigma (68 %) is the native 90 % band divided by n_sigma(90) = 1.645."""
    members = load_pdfs("CT18NNLO", SETS_DIR)
    x = np.array([0.1])
    central, _, up90 = evaluate(members, "g", x, 10.0, cl=90)
    _, _, up68 = evaluate(members, "g", x, 10.0, cl=68)
    assert (up90 - central) / (up68 - central) == pytest.approx(n_sigma(90), rel=1e-6)


@need_sets
def test_plot_writes_png_and_returns_curves(tmp_path):
    out = tmp_path / "a" / "b.png"
    x, curves = plot({"CT18NNLO": SETS_DIR, "CT25NNLO": SETS_DIR}, flavors=["g", "uv"],
                     npoints=20, output=str(out))
    assert out.stat().st_size > 10_000
    assert set(curves) == {"CT18NNLO", "CT25NNLO"} and set(curves["CT18NNLO"]) == {"g", "uv"}
    assert len(x) == 20


def test_draw_functions_draw_one_curve():
    x = np.array([0.1, 0.2, 0.3])
    curve = (np.array([2.0, 4.0, 6.0]), np.array([1.0, 3.0, 5.0]), np.array([3.0, 5.0, 7.0]))
    reference = (np.array([2.0, 2.0, 0.0]),) + curve[1:]  # zero at the last point
    _, ax = plt.subplots()
    draw_pdf(ax, x, curve, "red", "label")
    assert len(ax.lines) == 1 and len(ax.collections) == 1  # the curve and its band
    _, ax = plt.subplots()
    draw_ratio(ax, x, curve, reference, "red", bands=False)
    assert len(ax.collections) == 0  # no band
    y = ax.lines[0].get_ydata()
    assert y[:2] == pytest.approx([1.0, 2.0]) and np.isnan(y[2])  # central / reference; NaN where it is zero
    plt.close("all")


def test_show_must_be_known():
    with pytest.raises(ValueError, match="show must be"):
        plot({}, show="everything")


@need_sets
def test_the_three_kinds_of_plot(tmp_path):
    sizes = {}
    for show in ("both", "ratio", "pdf"):
        out = tmp_path / f"{show}.png"
        plot({"CT18NNLO": SETS_DIR, "CT25NNLO": SETS_DIR}, flavors=["g"], npoints=20, show=show, output=str(out))
        sizes[show] = out.stat().st_size
    assert min(sizes.values()) > 5_000
    assert len(set(sizes.values())) == 3  # three different figures


@need_sets
def test_driver_end_to_end_forces_output_into_plots(tmp_path, monkeypatch):
    monkeypatch.chdir(tmp_path)
    config = tmp_path / "p.json"
    config.write_text(json.dumps({"sets": {"CT18NNLO": SETS_DIR, "CT25NNLO": SETS_DIR}, "flavors": ["g"],
                                  "npoints": 30, "output": "../../escape.png"}))
    text = run_plot.run(str(config))
    assert (tmp_path / "plots" / "escape.png").exists()  # forced into plots/
    assert not (tmp_path.parent / "escape.png").exists()
    assert text.startswith("Saved plots/escape.png")
    assert "CT25NNLO: 69 members, ErrorType=hessian, native CL=68%" in text
    assert "[g] CT25NNLO: x=0.001: ratio" in text
```

```bash
pip install pytest        # or: conda install -c conda-forge pytest
python -m pytest -q
```

```text
.....................                                                    [100%]
21 passed
```

The tests that need the real sets look for them in `$PDFPLOT_SETS`, then `./pdfsets`, then `./`, and are skipped if they are not found.

Next: [02 Rules →](02-rules.md)
