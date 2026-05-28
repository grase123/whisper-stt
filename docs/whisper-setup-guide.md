# Installing OpenAI Whisper via UV

A guide to deploying [openai/whisper](https://github.com/openai/whisper) as a full-fledged UV project with GPU support.

**Target system:** Windows · NVIDIA GeForce RTX 5070 Ti (16 GB VRAM) · Blackwell architecture (compute capability **sm_120**).

> **The main thing to understand.** Whisper pulls in PyTorch. A regular installation of torch from PyPI gives you either a CPU build, or a torch with older CUDA kernels that **do not contain kernels for sm_120**. On the GPU this manifests as the error `CUDA error: no kernel image is available for execution on the device`. Therefore torch is installed separately — from an index with **CUDA 12.8 (cu128)**. Everything else comes from the regular PyPI.

---

## 0. Prerequisites: UV and the driver

**UV.** Check that it is installed:

```powershell
uv --version
```

If not — install it (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

After installing, restart the terminal.

**NVIDIA driver.** For sm_120 you need a driver with CUDA 12.8+ support. Your driver is already up to date (the 32.0.15.x branch), nothing else needs to be installed separately — the CUDA runtime will come along with the torch wheel.

---

## 1. Python version

Whisper officially supports Python **3.8–3.11**. We take **3.11** as the most stable option. UV will download the right interpreter itself — there is no need to install Python manually (we don't touch your system 3.12).

---

## 2. Creating the project

```powershell
uv init whisper-stt --python 3.11
cd whisper-stt
```

UV will create the project folder, the `pyproject.toml` file, and pin the Python version in `.python-version`.

---

## 3. Configuring the PyTorch index (cu128)

Open `pyproject.toml` and append to the end of the file:

```toml
[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
```

`explicit = true` means that this index is used **only** for the torch package. All other dependencies (including whisper itself) continue to be pulled from the regular PyPI.

---

## 4. Installing dependencies

The order matters: first torch (with cu128), then whisper, which will pick up the already installed torch.

```powershell
uv add torch
uv add openai-whisper
```

UV will create the `.venv` virtual environment, install torch with the `+cu128` build, and Whisper with all its dependencies (tiktoken, numba, numpy, etc.).

---

## 5. Installing ffmpeg

ffmpeg is a system utility, not a Python package, so it is not installed via UV. You need any one of these options:

```powershell
scoop install ffmpeg      # no administrator rights required (recommended)
winget install ffmpeg     # Windows' built-in package manager
choco install ffmpeg      # requires administrator rights
```

After installing, **restart the terminal** so that ffmpeg appears in `PATH`.

---

## 6. Verifying the components

Run from the project folder (`whisper-stt`).

### 6.1. Environment

```powershell
uv run python --version
```
Expected: `Python 3.11.x`

### 6.2. PyTorch + GPU

This is the main check. `torch.cuda.is_available()` returns `True` even with the wrong kernels, so a **real computation on the card** is mandatory:

```powershell
uv run python -c "import torch; print('torch:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('Device:', torch.cuda.get_device_name(0)); print('Capability:', torch.cuda.get_device_capability()); print('GPU compute:', (torch.tensor([2.0]).cuda() * 2))"
```

Expected output:
```
torch: 2.9.1+cu128          (version must contain +cu128)
CUDA available: True
Device: NVIDIA GeForce RTX 5070 Ti
Capability: (12, 0)
GPU compute: tensor([4.], device='cuda:0')
```

If the last line fails with a `no kernel image` error — that means torch is installed without sm_120 support (see the "If something went wrong" section).

### 6.3. ffmpeg

```powershell
ffmpeg -version
```
The version should be printed. If you see "command not recognized" — ffmpeg is not in `PATH`, restart the terminal or reinstall.

### 6.4. Whisper CLI

```powershell
uv run whisper --help
```
Help with a list of options should be printed.

### 6.5. End-to-end check (real transcription)

The most reliable test is to run a real audio file through it:

```powershell
uv run whisper "C:\path\to\audio.mp3" --model turbo --language Russian
```

On the first run, the model (`turbo` ≈ 1.5 GB) will be downloaded automatically. Whisper will create files with the transcription next to it (`.txt`, `.srt`, `.vtt`, etc.). If the log shows CUDA being used and the transcription runs fast — everything works.

---

## 7. How to use

**Choosing a model** (you have 16 GB VRAM — any one fits):

| Model | VRAM | Speed | When to choose |
|--------|------|----------|-------------|
| `turbo` | ~6 GB | ~8× | Default: fast + high accuracy. **Cannot translate to English.** |
| `large` | ~10 GB | 1× | Maximum accuracy |
| `medium`| ~5 GB | ~2× | If you need speech translation to English |

**Transcription:**
```powershell
uv run whisper "audio.mp3" --model turbo --language Russian
```

**Speech translation to English** (medium/large only):
```powershell
uv run whisper "audio.mp3" --model medium --language Russian --task translate
```

**From Python** (a file in the project, run via `uv run python script.py`):
```python
import whisper

model = whisper.load_model("turbo")          # will load onto the GPU automatically
result = model.transcribe("audio.mp3", language="ru")
print(result["text"])
```

---

## If something went wrong

**`CUDA error: no kernel image is available`** — you have torch without sm_120 kernels installed. Reinstall from cu128:
```powershell
uv remove torch
uv add torch        # with a correct pyproject.toml it will be taken from pytorch-cu128
```
Make sure the `[[tool.uv.index]]` block from step 3 is in place.

**`torch.__version__` without `+cu128`** — torch came from PyPI. Same cause: check `[tool.uv.sources]` in `pyproject.toml`.

**`ffmpeg: command not recognized`** — ffmpeg is not in `PATH`. Restart the terminal; if that doesn't help — reinstall via scoop/winget.

**Errors during `uv add openai-whisper` about building tiktoken/rust** — usually prebuilt wheels are available, but if needed install Rust from https://www.rust-lang.org/learn/get-started and retry.
