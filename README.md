
# whisper-stt

A guide to running speech transcription and translation locally using OpenAI Whisper on **Windows** with GPU acceleration <br/>
(WINDOWS 10, NVIDIA RTX 5070 Ti, CUDA 12.8 via `torch+cu128`).


https://github.com/openai/whisper


## Project setup

- Using `uv` + Python 3.11 (version 3.11 is well-tested).
- `torch` is installed from the PyTorch `cu128` index (not from the regular PyPI).
- Install `openai-whisper` and run it via `uv run whisper`.
- Verify CUDA with a real computation on the GPU.

### Creating the directory and initializing the project.
```sh
uv init whisper-stt --python 3.11
cd whisper-stt
```

### Important `pyproject.toml` snippet — edit the file.

```toml
[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
```

`explicit = true` means that the `pytorch-cu128` index is used only for `torch`, while the other packages are installed from the regular PyPI.


### Installing packages and dependencies
```sh
# Install the base dependency.
uv add torch
    # downloads 2 gigabytes

# If not yet installed - install it
choco install ffmpeg
ffmpeg -version

# Check the python version
uv run python --version

# Add the main package.
uv add openai-whisper
uv run whisper --help
```


### Verifying the installation

```sh
uv run python -c "import torch; print('torch:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('Device:', torch.cuda.get_device_name(0)); print('Capability:', torch.cuda.get_device_capability()); print('GPU compute:', (torch.tensor([2.0]).cuda() * 2))"
```

Output:

```text
torch: 2.11.0+cu128
CUDA available: True
Device: NVIDIA GeForce RTX 5070 Ti
Capability: (12, 0)
GPU compute: tensor([4.], device='cuda:0')
```

This confirms that Whisper can run on the GPU in this environment.

<br/>

## Quick start

Speech transcription:

```sh
uv run whisper "C:\PATH_TO_RECORDS\record.wav" --model large --language Russian
```

Transcription + translation to English:

```sh
uv run whisper "C:\PATH_TO_RECORDS\record.wav" --model large --task translate --language Russian
```


## Detailed documentation

A full step-by-step guide to installation, checks, and troubleshooting:

- [docs/whisper-setup-guide.md](docs/whisper-setup-guide.md)
