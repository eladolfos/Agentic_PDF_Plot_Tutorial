# 05 · The welcome message, capabilities and skills

Goal: when the agent starts, it greets you, says what it can and cannot do, lists its skills and asks the first question. Nobody has to read the README to find out what to ask.

**Adds:** `docs/CAPABILITIES.md`, `.claude/skills/welcome/SKILL.md`, the launcher `pdfplot`.

## Three pieces

| Piece | Role |
|---|---|
| `docs/CAPABILITIES.md` | The single source of what the agent can and cannot do |
| `welcome` skill | The procedure that writes the message from that source |
| `pdfplot` launcher | Starts Claude Code with the welcome already requested |

Why three and not one prompt: the capabilities live in a **document** you update when you change the code; the welcome is a **skill**, so you can also summon it with `/welcome` at any time or when someone says "what can you do?"; and the launcher only saves typing.

## 1. Capabilities

One table, in plain words. It also lists what is **not** available, so nobody asks for it and the agent does not improvise. Keep it in step with `pdfplot.py`: when you add an option to `plot`, add it here and in the plot skill's options table.

<!-- file: docs/CAPABILITIES.md whole -->
````markdown
# Capabilities

What the agent can do. The `welcome` skill shows this table to the user, so keep it in step with `pdfplot.py`.

| Area | Available |
|---|---|
| Sets | Any LHAPDF set on disk: Monte Carlo (`replicas`), `hessian` and `symmhessian`; single-member sets without a band |
| Partons | `g u d s c b ubar dbar sbar cbar bbar`, and the valence combinations `uv dv` |
| Plots | Two kinds. The PDFs, x·f(x,Q) against x, with a ratio panel to the first set underneath; or **only the ratio** between the sets. Both with uncertainty bands at any confidence level (68 % by default); up to 6 sets, log or linear axes |
| Changes on request | Q, partons, confidence level, x range, axes, y limits, ratio range, legend text and title, a subset of replicas of a Monte Carlo set |
| Records | Which set, from which folder, with which error type and native CL; a table of ratios and band widths to quote |
| Not available | Plots against Q, luminosities, alpha_s, comparisons with data, dark theme. Say so; do not improvise |
| Needs from you | The folders that hold your sets. CT25NNLO is not on the public server |
````

## 2. The welcome skill

The skill builds the message from files instead of hard-coding it: the capabilities from `CAPABILITIES.md`, the skills from each `SKILL.md` header (so the list is always current, whatever skills you add later), the caveats from the docs. It ends with **one** question and runs nothing before the answer.

Create `.claude/skills/welcome/SKILL.md`:

<!-- file: .claude/skills/welcome/SKILL.md whole -->
````markdown
---
name: welcome
description: Greet the user when a session starts, say what this agent can do and which skills it has, and ask what they want first. Use at the start of a session started by the launcher, when the user types /welcome, says hello, or asks what you can do.
---

# welcome

Goal: in one short message the user knows what this agent does, what it cannot do, which skills it has, and what to ask first. Nothing is run before they answer.

Read first: `docs/CAPABILITIES.md`, `docs/LIMITATIONS.md`, and `local/sets.md` if it exists. Use the user's language.

## Steps

1. **Greet** in one or two sentences.
2. **Say in three lines what you will do:** find your PDF sets, plot or compare them, and change the plot when asked.
3. **Show what you can do:** the "Available" column of `docs/CAPABILITIES.md` as a short list, and the "Not available" row, so nobody asks for something that does not exist. Keep names in English.
4. **List your skills.** Read the `name` and `description` of every `.claude/skills/*/SKILL.md` and give one line each, so the list is always current.
5. **Mention the two caveats that matter to numbers:** CT25NNLO is not on the public server, and an example marked `partial` or `unvalidated` is not a result.
6. **Ask one question.** If `local/sets.md` does not exist: "Where are your PDF sets? Give me a folder and I will find them." If it exists: "Which sets do you want to plot?", with an example such as "CT18NNLO and CT25NNLO, gluon and up quark, Q = 2 GeV".

## Do not

- run any command before the user answers;
- promise anything that `docs/CAPABILITIES.md` does not list.
````

## 3. The launcher

A plain `claude` opens an empty prompt. The launcher is a short script that checks the agent and LHAPDF exist, then starts Claude Code with a first message that asks for the welcome. It resolves its own location even through a symlink, so you can put a link on your `PATH` and start the agent from anywhere. `PDFPLOT_AGENT` lets another agent that reads `AGENTS.md` replace `claude`.

Create `pdfplot` (no extension) and make it executable:

<!-- file: pdfplot whole -->
```bash
#!/bin/bash
# pdfplot -- start the PDF plotting agent with its welcome message.
#
# Run it from anywhere (put a link on your PATH once:  ln -s "$PWD/pdfplot" ~/.local/bin/pdfplot).
# It checks that the agent and LHAPDF are available, then starts the agent with the `welcome` skill.
#
# Environment: PDFPLOT_AGENT=<command>  start another agent than `claude` (it must read AGENTS.md).

SRC="${BASH_SOURCE[0]}"
while [ -L "$SRC" ]; do
  D="$(cd -P "$(dirname "$SRC")" && pwd)"; SRC="$(readlink "$SRC")"; [[ "$SRC" != /* ]] && SRC="$D/$SRC"
done
cd "$(cd -P "$(dirname "$SRC")" && pwd)" || exit 1

AGENT="${PDFPLOT_AGENT:-claude}"
command -v "$AGENT" >/dev/null 2>&1 || {
  echo "pdfplot: '$AGENT' not found. Install Claude Code (https://claude.com/claude-code) or set PDFPLOT_AGENT."; exit 1; }
python -c "import lhapdf" 2>/dev/null || {
  echo "pdfplot: cannot import lhapdf. Activate the environment first: conda activate pdfplot"; exit 1; }

exec "$AGENT" "Start the welcome: follow the skill 'welcome'."
```

```bash
chmod +x pdfplot
```

## Try it

Test the launcher without starting the agent: make the "agent" a harmless command that prints what it receives.

```bash
PDFPLOT_AGENT=echo ./pdfplot
```

```text
Start the welcome: follow the skill 'welcome'.
```

Then the real thing:

```bash
./pdfplot
```

The welcome message looks like this. The wording varies, because the model writes it from your files; this is illustrative, not a recording:

```text
Hello! I plot and compare PDF sets with LHAPDF.

What I will do: find your PDF sets, plot or compare them, and change the plot when you ask.

What I can do:
- compare any LHAPDF set on disk (Monte Carlo, Hessian, symmetric Hessian), up to 6 at once
- plot g, u, d, s, c, b, their antiquarks, and u/d valence
- two kinds of plot: the PDFs with a ratio panel underneath, or only the ratio between the sets
- bands at any confidence level, log or linear axes
- change Q, x range, partons, y limits, legend and title, or use a subset of replicas
What I cannot do: plots against Q, luminosities, alpha_s, comparisons with data, a dark theme.

My skills:
- welcome: this message
- find-pdf-sets: find where your PDF sets are and remember it
- plot-pdfs: make and change plots

Two caveats: CT25NNLO is not on the public LHAPDF server, and an example marked partial or
unvalidated is not a result.

Where are your PDF sets? Give me a folder and I will find them.
```

Each line comes from a file. To change what the agent says it can do, edit `docs/CAPABILITIES.md`. To add a skill to the list, add a skill folder.

## Adding a skill later

Every new capability follows the same recipe: write the code in `pdfplot.py` and test it; add the option to the plot skill's table (or write a new skill with a clear `description`); add a line to `docs/CAPABILITIES.md`. The welcome picks it up by itself.

Next: [06 Guard rails →](06-guard-rails.md)
