
# whisper-stt

Проект для локальной расшифровки речи через OpenAI Whisper на Windows с GPU-ускорением (NVIDIA RTX 5070 Ti, CUDA 12.8 через `torch+cu128`).

https://github.com/openai/whisper


## Создание проекта

- Используем `uv` + Python 3.11 (хорошо протестирована версия 3.11).
- `torch` устанавливается из индекса PyTorch `cu128` (а не из обычного PyPI).
- ставим `openai-whisper` и запускаем через `uv run whisper`.
- Проверка CUDA  реальным вычислением на GPU.

### Создание каталога и инициализация проекта. 
```sh
uv init whisper-stt --python 3.11 
cd whisper-stt
```

### Важный фрагмент `pyproject.toml`  - Редактируем файл. 

```toml
[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
```

`explicit = true` означает, что индекс `pytorch-cu128` используется только для `torch`, а остальные пакеты ставятся с обычного PyPI.


### Доставляем пакеты и зависимости 
```sh
# cтавим базовую зависимость. 
uv add torch
    # качает 2 гигабайта 

# если еще не установлена - ставим 
choco install ffmpeg
ffmpeg -version

# Проверяем версию python 
uv run python --version

# Добавляем основной пакет. 
uv add openai-whisper
uv run whisper --help
```


### Проверка установки

```sh
uv run python -c "import torch; print('torch:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('Device:', torch.cuda.get_device_name(0)); print('Capability:', torch.cuda.get_device_capability()); print('GPU compute:', (torch.tensor([2.0]).cuda() * 2))"
```

Вывод:

```text
torch: 2.11.0+cu128
CUDA available: True
Device: NVIDIA GeForce RTX 5070 Ti
Capability: (12, 0)
GPU compute: tensor([4.], device='cuda:0')
```

Это подтверждает, что Whisper может работать с GPU в этом окружении.

<br/>

## Быстрый запуск

Расшифровка речи:

```sh
uv run whisper "C:\PATH_TO_RECORDS\record.wav" --model large --language Russian
```

Расшифровка + перевод на английский:

```sh
uv run whisper "C:\PATH_TO_RECORDS\record.wav" --model large --task translate --language Russian
```


## Подробная документация

Полный пошаговый гайд по установке, проверкам и устранению проблем:

- [docs/whisper-setup-guide.md](docs/whisper-setup-guide.md)
