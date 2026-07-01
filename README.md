# Shutterai — Multi-GPU AI Image-Generation Automation

A small Python automation harness that batch-generates images from a spreadsheet of prompts by driving three ComfyUI instances in parallel across three GPUs.

## What it does

Given a bank of prompts in an Excel file, Shutterai fans each prompt out to multiple GPUs at once and saves the rendered images — turning an interactive Stable Diffusion tool (ComfyUI) into an unattended, high-throughput batch pipeline.

- **`main-gen.py`** — the orchestrator. Reads `prompt-bank.xlsx` (columns `Positive Prompt` / `Negative Prompt`) with pandas, then for each row launches `gpu0.py`, `gpu1.py`, and `gpu2.py` as parallel subprocesses, waits for the batch to finish, and reports total runtime. Output images are written to `output/` as `output_<row>_gpu<n>.png`.
- **`gpu0.py` / `gpu1.py` / `gpu2.py`** — near-identical workers, one per GPU/ComfyUI instance. They differ only by the ComfyUI API port they connect to (`8188` / `8189` / `8190`). Each worker uses [`comfy_script`](https://github.com/Chaoses-Ib/ComfyScript) to build and queue a ComfyUI workflow: load an SD 1.5 checkpoint (`v1-5-pruned-emaonly.ckpt`), encode the positive/negative prompts, sample with `KSampler` (21 steps, CFG 9, `euler_ancestral`), VAE-decode, and save the image. A distributed workflow (`NetDistAdvancedV2.json`) plus `RemoteChainStart`/`RemoteQueueWorker` nodes coordinate the remote render.
- **`prompt-bank.xlsx`** — the prompt library the run iterates over.

## Tech stack

- **Python 3** — `subprocess` orchestration, `pandas` for the prompt sheet, `argparse`
- **ComfyUI** (three instances on ports 8188–8190) driven programmatically via **ComfyScript**
- **Stable Diffusion 1.5** (`v1-5-pruned-emaonly.ckpt`)
- Multi-GPU parallelism via one ComfyUI process per GPU

## Setup / run

Prerequisites: three running ComfyUI instances (one per GPU) reachable on ports 8188/8189/8190, the SD 1.5 checkpoint installed, and the `NetDistAdvancedV2.json` workflow present.

```bash
pip install pandas openpyxl comfy-script

# Edit prompt-bank.xlsx to add "Positive Prompt" / "Negative Prompt" rows,
# then run the orchestrator:
python main-gen.py
```

Images land in `output/`.

> **Heads-up — this is environment-specific.** The workers hardcode an absolute workflow path (`/home/arash/ComfyUI/user/default/workflows/NetDistAdvancedV2.json`) and `/bin/python3`, so paths must be updated for your machine. The "progress bar" wait loop in the workers is a placeholder that just counts up rather than reading real ComfyUI progress. The scripts also assume three GPUs/instances are already up.

## Context

Shutterai is an AI image-generation automation tool — a batch pipeline that scales ComfyUI/Stable Diffusion across multiple GPUs from a simple prompt spreadsheet. It's a compact, practical example of GenAI tooling and multi-GPU orchestration.
