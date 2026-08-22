# AUTOMATIC1111 on Google Colab — Python 3.10 build

A Colab notebook that runs [AUTOMATIC1111's Stable Diffusion WebUI](https://github.com/AUTOMATIC1111/stable-diffusion-webui) inside a private Python 3.10 virtual environment, instead of on whatever Python version Colab currently ships.

Derived from [TheLastBen/fast-stable-diffusion](https://github.com/TheLastBen/fast-stable-diffusion), which stopped working when Colab moved to Python 3.13.

## Why a separate Python

A1111 v1.10.1 (mid-2024) pins `Pillow==9.5.0`, `numpy==1.26.2`, `scikit-image==0.21.0` and `transformers==4.30.2`. None of those has a cp313 wheel or a pure-Python fallback, so on Colab's current runtime pip tries to build them from source against a C API that postdates them, and fails.

Unpinning them doesn't help: it just moves the incompatibility (pydantic v1 vs v2, `pytorch_lightning` importing `wandb`, then torch 2.11 against code written for 2.1). A1111 upstream is dormant, so no port is coming.

This notebook installs Python 3.10 with [uv](https://github.com/astral-sh/uv) into `/content/sd310` and runs the WebUI there. A1111's original pins install cleanly, and `launch.py` installs its own tested `torch==2.1.2+cu121`. Colab can move to 3.14 without affecting it.

## Usage

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/je4ngomes/fast-stable-diffusion/blob/main/fast_stable_diffusion_AUTOMATIC1111_py310.ipynb)

Select a GPU runtime, and run the cells top to bottom:

| Cell | Does |
|---|---|
| Connect Google Drive | Mounts Drive; the install persists there |
| Install/Update AUTOMATIC1111 repo | Clones/updates the WebUI |
| **Requirements** | Builds the 3.10 venv and runs A1111's installer |
| Model Download/Load | Optional — pick a model or point at your own folder |
| Download LoRA | Optional |
| ControlNet | Optional |
| **Start Stable-Diffusion** | Launches the server, prints the Gradio URL |

First run takes 10–15 minutes (torch is ~2.5 GB). The venv lives in `/content`, so it is rebuilt each session; the WebUI, models and extensions live on Drive and persist.

To restart the server within a session, run only the Start cell — re-running Requirements rebuilds the venv from scratch.

## Upstream breakages this works around

**Python version.** Colab ships 3.13; A1111 needs 3.10. Fixed with a uv-managed venv rather than pinning the Colab runtime, so it survives future Colab upgrades.

**`Stability-AI/stablediffusion` is gone.** The repository was removed from GitHub around December 2025, so the URL hardcoded in `launch_utils.py` 404s and git falls back to prompting for credentials. A1111's dev branch switched to [`w-e-w/stablediffusion`](https://github.com/w-e-w/stablediffusion); this notebook sets `STABLE_DIFFUSION_REPO` in the environment, which `launch_utils` already reads.

**openai CLIP won't build.** Its `setup.py` does `import pkg_resources`, removed in setuptools 81+, and pip's build isolation always fetches the newest setuptools. Installed with `setuptools==69.5.1` and `--no-build-isolation`. A1111 pins the same setuptools version, but only in `requirements_versions.txt`, which it installs *after* CLIP.

**Non-git entries in `repositories/`.** `git_clone()` runs `git rev-parse HEAD` in any directory that already exists, so a symlink or an extracted tarball aborts startup. The Requirements cell removes those so `launch.py` can manage the repositories itself.

**`webui.py` vs `launch.py`.** Extension installers run from `run_extensions_installers()` inside `prepare_environment()`, which only `launch.py` calls. Starting `webui.py` directly leaves ADetailer, Dynamic Prompts and ControlNet without their dependencies. Note also that `--skip-install` disables those installers.

## Notes

- Installation and startup are separate: Requirements calls `prepare_environment()` directly, Start passes `--skip-prepare-environment`. `launch.py` has no `--exit` flag.
- `--opt-sdp-attention` replaces `--xformers`, since no matching xformers build is installed.
- Output is quiet on success and verbose on failure — no `capture_output()` swallowing errors behind a green tick.
- Launched with `--api` enabled.

## Credits

- [AUTOMATIC1111/stable-diffusion-webui](https://github.com/AUTOMATIC1111/stable-diffusion-webui) — AGPL-3.0
- [TheLastBen/fast-stable-diffusion](https://github.com/TheLastBen/fast-stable-diffusion) — MIT; the Drive, repo, model, LoRA and ControlNet cells come from there
- [w-e-w/stablediffusion](https://github.com/w-e-w/stablediffusion) — the surviving fork of the removed upstream

## License

MIT — see [LICENSE](LICENSE), which carries TheLastBen's original notice (Copyright © 2022 Ben) as the MIT terms require.
