# 06 · Guard rails

Goal: make the important rules hold even if the model ignores the text.

**Adds:** `.claude/settings.json`, `.gitignore`.

## Three levels of "must"

| Level | Mechanism | Strength | Here |
|---|---|---|---|
| 1 Text | `AGENTS.md`, skills, rules | the model usually obeys; no guarantee | "never compute by hand" |
| 2 Code | checks in the driver and in `evaluate()` | deterministic, but a person can bypass | output forced into `plots/`; unknown option rejected; unknown error type raises |
| 3 Permissions | `.claude/settings.json` | the tool call is blocked, or asks, before it runs | edits of the toolchain, downloads, `rm` |

Put each important rule at the highest level you can afford, and write the level in its "Enforced by" line (you did, in chapter 02).

## The permissions file

Claude Code evaluates `deny` first, then `ask`, then `allow`. `Edit` rules apply to every built-in tool that edits files, and a path starting with `/` is relative to the project folder.

Create `.claude/settings.json`:

<!-- file: .claude/settings.json whole -->
```json
{
  "permissions": {
    "allow": [
      "Bash(python run_plot.py *)",
      "Edit(/local/**)"
    ],
    "ask": [
      "Bash(curl *)",
      "Bash(wget *)",
      "Edit(/pdfplot.py)",
      "Edit(/run_plot.py)"
    ],
    "deny": [
      "Bash(rm *)"
    ]
  }
}
```

* **allow**: running the driver, and editing anything in `local/`, so the agent writes plot files without prompting you each time.
* **ask**: downloads (`curl`, `wget`), and any edit of `pdfplot.py` or `run_plot.py`. This is rule 9 of `AGENTS.md` made real: the agent may propose a change to the toolchain, but you confirm it.
* **deny**: `rm`. Your PDF sets are hundreds of megabytes and slow to fetch again; the agent has no reason to delete files.

These are guard rails against accidents and drift, not a sandbox: a pattern on a command cannot stop every way of doing the same thing.

## .gitignore

Your paths and plot files are personal, so `local/` stays out of git, together with the plots and the sets:

<!-- file: .gitignore whole -->
```text
# the PDF sets are large: download them as the tutorial explains
pdfsets
# your own files and generated plots
local/
plots/
__pycache__/
*.pyc
.pytest_cache/
.env
```

## Test the guard rails

Start the agent (`./pdfplot`) and try:

| You say | You should see |
|---|---|
| "Download CT25NNLO." | It says CT25NNLO is not on the public server and asks where yours is |
| "Change the line colours in `pdfplot.py`." | Claude Code asks you to confirm the edit |
| "Delete the plots folder." | The command is refused (`rm` is denied) |
| "Plot the gluon against Q." | It says that is not available (it is not in `docs/CAPABILITIES.md`) |
| "Show only the ratio." | It sets `"show": "ratio"` in the same JSON file and runs the driver again |
| "Compare CT18NNLO with CT25NNLO for the gluon." | It writes `local/<name>.json`, runs the driver, looks at the PNG and quotes the table, saying the example is `partial` if it used it |

The first row is level 1 (a rule in text), the second and third are level 3 (permissions), the fourth is level 1 backed by the capabilities document.

## What to add if the agent grows

This agent does nothing costly or irreversible, so it stops here. If yours could reach a cluster, spend an allocation or submit jobs, add:

* a **wrapper script** as the only channel to the remote machine, with an allow list of commands, a rate limit and a log;
* an **approval gate**: the driver only submits after the user has reviewed and authorized that exact configuration, recorded with a hash of the files, so changing them voids the approval;
* **deny** rules for `ssh`, `scp`, `sbatch` and similar commands, so everything goes through the wrapper.

## Where you ended up

* `pdfplot.py` is `load_pdfs`, `evaluate` and `plot` (with two small functions that draw one curve), and `run_plot.py` is the single, checked door to them.
* The instructions (`AGENTS.md`, three skills) tell the agent how to use that door and how to introduce itself; the rules say what must never go wrong, and how each is enforced.
* The model chooses and explains; the scripts compute and check; permissions decide what is allowed.

← [05 The welcome message](05-welcome-and-skills.md) · [Back to the start](../README.md)
