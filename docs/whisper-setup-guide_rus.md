# Установка OpenAI Whisper через UV

Инструкция по развёртыванию [openai/whisper](https://github.com/openai/whisper) как полноценного проекта UV с поддержкой GPU.

**Целевая система:** Windows · NVIDIA GeForce RTX 5070 Ti (16 ГБ VRAM) · архитектура Blackwell (compute capability **sm_120**).

> **Главное, что нужно понимать.** Whisper тянет за собой PyTorch. Обычная установка torch с PyPI даёт либо CPU-сборку, либо torch со старыми CUDA-ядрами, которые **не содержат kernel'ов для sm_120**. На GPU это проявляется ошибкой `CUDA error: no kernel image is available for execution on the device`. Поэтому torch ставится отдельно — из индекса с **CUDA 12.8 (cu128)**. Всё остальное берётся с обычного PyPI.

---

## 0. Предварительно: UV и драйвер

**UV.** Проверьте, что установлен:

```powershell
uv --version
```

Если нет — поставьте (PowerShell):

```powershell
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

После установки перезапустите терминал.

**Драйвер NVIDIA.** Для sm_120 нужен драйвер с поддержкой CUDA 12.8+. У вас драйвер уже свежий (ветка 32.0.15.x), отдельно ничего ставить не требуется — CUDA-runtime приедет вместе с колесом torch.

---

## 1. Версия Python

Whisper официально поддерживает Python **3.8–3.11**. Берём **3.11** как самый стабильный вариант. UV сам скачает нужный интерпретатор — вручную ставить Python не нужно (ваш системный 3.12 не трогаем).

---

## 2. Создание проекта

```powershell
uv init whisper-stt --python 3.11
cd whisper-stt
```

UV создаст папку проекта, файл `pyproject.toml` и зафиксирует версию Python в `.python-version`.

---

## 3. Настройка индекса PyTorch (cu128)

Откройте `pyproject.toml` и допишите в конец файла:

```toml
[[tool.uv.index]]
name = "pytorch-cu128"
url = "https://download.pytorch.org/whl/cu128"
explicit = true

[tool.uv.sources]
torch = { index = "pytorch-cu128" }
```

`explicit = true` означает, что этот индекс используется **только** для пакета torch. Все прочие зависимости (включая сам whisper) продолжают тянуться с обычного PyPI.

---

## 4. Установка зависимостей

Порядок важен: сначала torch (с cu128), затем whisper, который подхватит уже установленный torch.

```powershell
uv add torch
uv add openai-whisper
```

UV создаст виртуальное окружение `.venv`, поставит torch со сборкой `+cu128` и Whisper со всеми его зависимостями (tiktoken, numba, numpy и т. д.).

---

## 5. Установка ffmpeg

ffmpeg — это системная утилита, а не Python-пакет, поэтому через UV она не ставится. Нужен любой один из вариантов:

```powershell
scoop install ffmpeg      # без прав администратора (рекомендуется)
winget install ffmpeg     # встроенный менеджер Windows
choco install ffmpeg      # требует прав администратора
```

После установки **перезапустите терминал**, чтобы ffmpeg появился в `PATH`.

---

## 6. Проверка компонентов

Выполняйте из папки проекта (`whisper-stt`).

### 6.1. Окружение

```powershell
uv run python --version
```
Ожидается: `Python 3.11.x`

### 6.2. PyTorch + GPU

Это главная проверка. `torch.cuda.is_available()` показывает `True` даже при неправильных ядрах, поэтому обязательно выполняется **реальный расчёт на карте**:

```powershell
uv run python -c "import torch; print('torch:', torch.__version__); print('CUDA available:', torch.cuda.is_available()); print('Device:', torch.cuda.get_device_name(0)); print('Capability:', torch.cuda.get_device_capability()); print('GPU compute:', (torch.tensor([2.0]).cuda() * 2))"
```

Ожидаемый вывод:
```
torch: 2.9.1+cu128          (версия обязательно с +cu128)
CUDA available: True
Device: NVIDIA GeForce RTX 5070 Ti
Capability: (12, 0)
GPU compute: tensor([4.], device='cuda:0')
```

Если последняя строка падает с ошибкой `no kernel image` — значит установлен torch без поддержки sm_120 (см. раздел «Если что-то пошло не так»).

### 6.3. ffmpeg

```powershell
ffmpeg -version
```
Должна вывестись версия. Если «команда не распознана» — ffmpeg не в `PATH`, перезапустите терминал или переустановите.

### 6.4. Whisper CLI

```powershell
uv run whisper --help
```
Должна вывестись справка со списком опций.

### 6.5. Боевая проверка (реальная транскрибация)

Самый надёжный тест — прогнать настоящий аудиофайл:

```powershell
uv run whisper "C:\путь\к\audio.mp3" --model turbo --language Russian
```

При первом запуске модель (`turbo` ≈ 1.5 ГБ) скачается автоматически. Whisper создаст рядом файлы с расшифровкой (`.txt`, `.srt`, `.vtt` и др.). Если в логе видно использование CUDA и расшифровка идёт быстро — всё работает.

---

## 7. Как пользоваться

**Выбор модели** (у вас 16 ГБ VRAM — влезает любая):

| Модель | VRAM | Скорость | Когда брать |
|--------|------|----------|-------------|
| `turbo` | ~6 ГБ | ~8× | По умолчанию: быстро + высокая точность. **Не умеет перевод на английский.** |
| `large` | ~10 ГБ | 1× | Максимальная точность |
| `medium`| ~5 ГБ | ~2× | Если нужен именно перевод речи на английский |

**Транскрибация:**
```powershell
uv run whisper "audio.mp3" --model turbo --language Russian
```

**Перевод речи на английский** (только medium/large):
```powershell
uv run whisper "audio.mp3" --model medium --language Russian --task translate
```

**Из Python** (файл в проекте, запуск через `uv run python script.py`):
```python
import whisper

model = whisper.load_model("turbo")          # загрузится на GPU автоматически
result = model.transcribe("audio.mp3", language="ru")
print(result["text"])
```

---

## Если что-то пошло не так

**`CUDA error: no kernel image is available`** — стоит torch без ядер sm_120. Переустановите из cu128:
```powershell
uv remove torch
uv add torch        # при корректном pyproject.toml возьмётся из pytorch-cu128
```
Убедитесь, что блок `[[tool.uv.index]]` из шага 3 на месте.

**`torch.__version__` без `+cu128`** — torch приехал с PyPI. Причина та же: проверьте `[tool.uv.sources]` в `pyproject.toml`.

**`ffmpeg: команда не распознана`** — ffmpeg не в `PATH`. Перезапустите терминал; если не помогло — переустановите через scoop/winget.

**Ошибки при `uv add openai-whisper` про сборку tiktoken/rust** — обычно колёса есть готовые, но при необходимости установите Rust с https://www.rust-lang.org/learn/get-started и повторите.
