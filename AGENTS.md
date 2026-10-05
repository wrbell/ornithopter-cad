# AGENTS.md

## Project

Parametric TPU wingtip for an 8-wing flapping-wing (ornithopter) drone. The
root section is the stock UIUC S1223 at 165 mm chord (it must match the
existing rib). Tip 80 mm, span 270 mm, 15° LE sweep, 8° washout, hollow shell
with three spanwise mounting holes. Mass target ≤ 110 g per tip. The whole
aircraft is about 4.2 kg with a thin hover margin, so every gram counts.

- `ornitho/` is the library (airfoil loader, section math, CadQuery geometry,
  hole validation, mass report, renders).
- `wingtip.py` is the human loop: edit `CONFIG`, run, look at `out/*.png`,
  read `out/report.txt`.
- `models/wingtip_agentcad.py` is the agent loop: the same geometry exposed to
  agentcad (versioned runs, previews, measure, diff, viewer) through its
  CadQuery runtime.
- [docs/AGENTCAD-MANUAL.md](docs/AGENTCAD-MANUAL.md) is the generated agentcad
  manual. Read it before CAD work with agentcad.
- [docs/agents/agentcad-usage.md](docs/agents/agentcad-usage.md) has the
  toolchain table, the agentcad rules and the gotchas for this repository.
  Read it before you run agentcad or edit `models/`.

The GitHub repository `wrbell/ornithopter-cad` is public.
Willem reviews every merge.

## Commands

Two venvs: `.venv` (pipeline, tests, renders) and `.venv-agentcad` (agentcad
CLI and MCP server). Never mix them. `make setup` builds both.

```sh
make setup                  # both venvs (Python 3.10-3.12)
make run                    # full build + STEP/STL + PNGs + report   (.venv)
make test                   # fast unit tests, no OCCT                (.venv)
make slow                   # OCCT geometry tests                     (.venv)
make lint                   # ruff check + ruff format --check        (.venv)
make fmt                    # ruff auto-fix + format                  (.venv)
make agentcad-dry LABEL=v1  # validity + metrics, no version consumed (.venv-agentcad)
make agentcad-run LABEL=thin ARGS="--params wall_thickness_mm=1.6"   # versioned run + renders
make agentcad-smoke         # end-to-end contract check of the integration
make agentcad-guide         # refresh the generated skill and docs/AGENTCAD-MANUAL.md
```

Standards checks: `pre-commit run --files <changed files>`,
`python3 tools/agents_md_lint.py .` and `python3 tools/check_docs.py .`.

## Code style

Follow the config files. Do not paste a style guide into this file.

- Python: `[tool.ruff]` in `pyproject.toml`. Run `make lint`; `make fmt`
  fixes. The collection Ruff config is not adopted here.
- Editor defaults: `.editorconfig`
- Markdown: `enforcement/markdownlint/.markdownlint-cli2.yaml`
- YAML: `enforcement/yamllint/.yamllint.yml`
- Spelling (information only): `enforcement/cspell/cspell.json`
- Prose for Willem: use the skill in
  `.agents/skills/simplified-technical-english/` (see Clarity).
- Generated files: `docs/AGENTCAD-MANUAL.md` (between its markers) and
  `.claude/skills/agentcad/SKILL.md`. Do not edit them by hand. Never run
  `agentcad instructions install` (details in
  [docs/agents/agentcad-usage.md](docs/agents/agentcad-usage.md)).
- Copied standards files (`tools/agents_md_lint.py`, `tools/check_docs.py`,
  `enforcement/`, `.github/workflows/standards.yml`, `.standards.json`): do
  not edit them; re-run the standards adopt tool.

Collection standards live in the private repository `wrbell/standards`.
When a standards file and this file disagree, this file wins.
Say that in the pull request body.

## Design rules

1. **Coordinate frame.** x chordwise, positive aft, root LE at 0. y spanwise,
   positive outboard, rib face at 0. z up. Sections stay in planes y = const.
   Twist is about the quarter chord, positive = nose down. If a view looks
   wrong, fix the viewer or the camera; never re-orient the geometry to match
   a picture.
2. **Hole validation must pass.** `ornitho.validate.check_holes` runs before
   any OCCT work. A `HoleValidationError` (agentcad reports `status: failed`
   with "mount hole validation failed") is a design problem: move or shrink
   the hole. Never lower `hole_min_wall_mm` below 1.0 or catch the exception
   to make it pass. When you change holes, report each hole's minimum
   clearance from the report or the warnings.
3. **Look at the renders before saying done.** After any geometry change, view
   `out/wingtip_iso.png`, `out/wingtip_root.png` and `out/section.png` (from
   `make run`), or `build/vN_label/preview.png` and
   `build/vN_label/renders/*.png` (from agentcad), and say what you saw: LE
   forward, span to +y, tip nose-down, cavity with solid trailing edge, three
   full-height boss webs with bores. Metrics are not a substitute for looking.
4. **Mass comes from `out/report.txt`.** It uses a fine triangulation of the
   solid plus the 2D wall table. agentcad's `metrics.volume` is OpenCascade's
   analytic integration, measured about 25 % low on this spline loft. Use it
   for validity, bounding box and diffs only; never quote it as mass.
5. **Keep the design anchors.** `root_chord_mm = 165` and the stock S1223 stay
   unless explicitly told otherwise. Keep CadQuery and the kernel pins; a
   build123d port or a pin bump needs an explicit request and a green
   `make slow`.
6. **Commit the evidence.** Commit geometry changes together with the
   regenerated `out/*.png`, `out/report.txt` and `out/wingtip_run.json`, so
   the git history stays visual. STEP/STL are gitignored.

## Tests

Definition of done: `make lint test` green; `make slow` if `ornitho/` changed;
`make run` and the renders viewed (design rule 3); `make agentcad-smoke` if the
model script, packaging or requirements changed; the mass versus the 110 g
target stated in the summary.

`make test` skips the `slow` marker (`pytest.ini`). CI
(`.github/workflows/ci.yml`) runs lint, both suites, a headless default build
and the agentcad contract check (`smoke_agentcad.py`).

Write or update the failing test first. Show that it fails, then make it pass.
Do not hard-code a value or a special case to pass a test.
If a test is wrong, say so.

## Pull requests and commits

Use a Conventional Commit subject. commitlint checks the message with
`enforcement/commitlint/commitlint.config.js`.
Open the pull request as a draft and fill in
`.github/PULL_REQUEST_TEMPLATE.md`. Wait for CI to pass.
List each assumption and each open tradeoff in the pull request body.
Do not edit files outside the task.
A new dependency needs a reason in the pull request.
Do not force-push, rewrite history, delete a branch, merge, or publish
unless Willem asks.

## Security

This repository is public. Do not commit a secret or an API key.
Do not put a secret in a prompt or a log.
gitleaks runs in pre-commit and in CI (`enforcement/gitleaks/.gitleaks.toml`).
Run unattended mode only in a sandbox.

## Clarity

Write explanations to Willem in Simplified Technical English. Use the same style
for a pull request description, a commit message body, and prose in a README or
another doc. Aim for about 80 percent of the rules. Full compliance with
ASD-STE100 is not the goal.

The rules are in the standards repository at `standards/writing-ste/ste.md`.
The upstream skill is
[simplified-technical-english](https://github.com/0xpili/simplified-technical-english/tree/1e148d670cba46685ad2b4c3f2354a637a7fdbbe)
(MIT, commit `1e148d670cba46685ad2b4c3f2354a637a7fdbbe`). Link to that skill.
Do not copy the skill into this repository again.

Code, identifiers, math, command-line output, and quoted error text are exempt.

When structure, flow, or architecture is the point, use a mermaid diagram.
For a complex result, offer a self-contained HTML page.
That page is a throwaway file.
Do not commit it unless Willem asks.

Make a video only when Willem asks for a video.
Do not add an API key or a secret.

`scripts/ste_check.py` in the upstream skill is an optional check on docs.
Do not use it as a CI gate.

<!-- standards:begin -->
## Collection standards

Every project under `/Users/willem/Code` follows the shared standards in
`/Users/willem/Code/standards/` (index: `standards/STANDARDS.md`; future
standards: `standards/ROADMAP.md`).

- **Presentations:** build every deck from
  `standards/powerpoint template/Willem-Default.potx` (theme "Helena": Neue Haas
  Grotesk Text Pro, 16:9, black on white with a gray ramp, template v2). Spec:
  `standards/powerpoint template/STANDARD.md`. Generate with
  `standards/powerpoint template/house_style.py` (open
  `Willem-Default-Base.pptx`, never the `.potx`) and gate with
  `standards/powerpoint template/deck_checks.py` before calling a deck done.
- **Deck rules:** no speaker notes in submitted decks; editable shapes, not
  chart images; numbered, linked superscript citations with a final References
  slide; no bottom rules, citation strips, or page counters; footer text only
  when a course or client requires it (for example `ME460 HWx`), which overrides
  the default of no footer; export the deliverable PDF with native PowerPoint
  and use LibreOffice renders only for QA.
- **Everything else:** do not invent facts, dates, or numbers; mark unknowns TBD
  and point at the source. Keep copyrighted course material out of git. This
  block is managed by `standards/tools/apply_standards.py`; edit
  `standards/ai-files/BLOCK-root.md`, not this copy.
- **AI use (school work):** no AI-generated or AI-modified images in any school
  deliverable; AI-written deliverable text only with written adviser
  pre-clearance (`docs/ai-clearances/`); never cite an AI tool as a source;
  never edit graded report text (the repo's `protected-paths.txt`; example:
  `standards/enforcement/senior-design-repo/sd-protected-paths.txt`). AI-use
  logging and attestation are opt-in per repo via `ai-attestation-roots.txt`;
  see `standards/standards/ai-use-disclosure/ai-use-disclosure.md`.
- **AI files:** one `AGENTS.md` (≤ 200 lines, Clarity verbatim); `CLAUDE.md` is
  `@AGENTS.md`. Gates: `standards/tools/agents_md_lint.py`, `ai_file_lint.py`.
<!-- standards:end -->
