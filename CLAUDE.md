# CLAUDE.md — A1111 on Colab, Python 3.10 build

Context for anyone (human or Claude) picking this up. Read before changing the notebook.

## What this is

A Colab notebook running AUTOMATIC1111 Stable Diffusion WebUI v1.10.1 inside a
private Python 3.10 venv at `/content/sd310`, built with `uv`. Derived from
TheLastBen/fast-stable-diffusion (MIT), which broke when Colab moved to Python 3.13.

Cells: Connect Drive → Install/Update repo → **Requirements** → Model → LoRA →
ControlNet → **Start**. Only the two bold cells are new; the rest are upstream's.

Published in this repo as `fast_stable_diffusion_AUTOMATIC1111_py310.ipynb`. The
upstream A1111, ComfyUI and DreamBooth notebooks were removed — this is the only
notebook here now. `LICENSE` is TheLastBen's unmodified MIT text; the notice
("Copyright © 2022 Ben") travels with the redistribution, as MIT requires. A1111
itself is AGPL-3.0 but is cloned at runtime, not redistributed.

## Verified facts (checked, not assumed)

- Colab's current runtime is **Python 3.13.15**; torch wheels are `cp313`. The most
  recent *pinnable* past runtime (2026.07) is 3.12.13, so the pinnable list is
  misleading — check `googlecolab/backend-info` `apt-list.txt` for the real version.
- A1111 pins `Pillow==9.5.0`, `numpy==1.26.2`, `scikit-image==0.21.0`,
  `tokenizers==0.13.3` (via `transformers==4.30.2`). **None has a cp313 wheel or a
  pure-Python fallback.** numpy gained 3.13 support in 2.1, Pillow in 11.x.
- `Stability-AI/stablediffusion` returns **404** — removed from GitHub ~Dec 2025.
  A1111's dev branch uses `w-e-w/stablediffusion`. Verified that fork contains the
  pinned commit `cf1d67a6fd5ea1aa600c4df58e5b47da45f6bdbf`.
- openai CLIP's `setup.py` line 3 is `import pkg_resources`, removed in setuptools
  81+ (current is 84.0.0). pip's build isolation always fetches the newest
  setuptools, so the build fails regardless of the venv's version.
- `launch_utils.py`: `git_clone()` runs `git rev-parse HEAD` in any directory that
  already exists — a symlink or non-git folder aborts startup.
  `run_extensions_installers()` is gated behind `if not args.skip_install`.
- `launch.py` has **no `--exit` flag**; `main()` always calls `start()`. To install
  without launching, call `launch_utils.prepare_environment()` directly.
- The repos A1111 expects at pinned hashes: generative-models `45c443b3`,
  k-diffusion `ab527a9a`, BLIP `48211a15`, assets `6f7db241`.

## Design decisions — do not undo these

**Python 3.10 in a venv, not a pinned Colab runtime.** Runtime pinning works but
expires (past versions last one year) and gets undone by Colab upgrades. The venv is
self-contained in `/content`.

**Do not try to port A1111 to 3.13.** It was attempted. Unpinning the four packages
above just relocates the failure: pydantic v1 vs wandb's v2, `pytorch_lightning`
eagerly importing `WandbLogger`, then `clip`, then every extension dependency.
A1111 upstream is dormant (v1.10.1, mid-2024); no port is coming.

**`launch.py`, never `webui.py`.** Only `launch.py` runs `prepare_environment()`,
which clones repositories and runs each extension's `install.py`. Starting
`webui.py` leaves ADetailer, Dynamic Prompts and ControlNet without dependencies.

**Never add `--skip-install`.** It disables the extension installers.

**No source patching.** No `sed` on `modules/`. Cell 2 runs `git reset --hard`, so
source edits are wiped every session anyway. Use environment variables
(`STABLE_DIFFUSION_REPO`) and pre-installed packages instead.

**No `capture.capture_output()` in new cells.** That is what hid the original
failures behind a green ✔ for hours. Use `subprocess.run(capture_output=True)` and
print stdout/stderr only on non-zero exit. Quiet on success, loud on failure.

**Installation belongs in Requirements, not Start.** Requirements calls
`prepare_environment()` directly; Start passes `--skip-prepare-environment`.

## Known trade-offs

- The venv is ephemeral: every Requirements run re-downloads torch (~2.5 GB, 10–15
  min). To restart the server in a live session, run only the Start cell.
- `--opt-sdp-attention` replaces `--xformers` (no matching xformers build).
- Dropped from upstream: the `model.half()` sed (fp16 on load) and the `shared.py`
  quicksettings sed. Set quicksettings via Settings → User Interface instead — it
  persists in `config.json` and survives `git reset --hard`.
- `w-e-w/stablediffusion` is one person's fork of a deleted repo. Fragile.

## Open items

- Verify the full startup path end to end: gradio 3.41.2 booting, and the first-run
  extension installs for ADetailer / Dynamic Prompts / ControlNet. Still unverified —
  it needs a Colab GPU session, so nothing here has been run against the real runtime.
- The `Dreambooth/`, `Dependencies/` and `AUTOMATIC1111_files/` directories are left
  over from the removed notebooks. Nothing in this notebook reads them — the
  ControlNet cell pulls `CN_models*.txt` from TheLastBen's repo over HTTP, not from
  this one — so they can be deleted whenever.

## Debugging approach that worked

Check the actual source before theorising. Fetching `launch_utils.py`, reading the
real `setup.py`, reproducing the CLIP build failure locally, and curling repo URLs
for HTTP status each turned a guess into a fact — and twice disproved a confident
wrong theory. Do that first.
