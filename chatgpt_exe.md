## 1. ざっくりまとめると

| 目的 | 方法 | 主なポイント |
|------|------|--------------|
| **gpt‑oss‑20B をローカルで動かす** | 1. まず 20 B の重みを **4‑bit か 8‑bit で量子化** したファイルを入手。<br>2. `transformers` + `torch` (MPS) で推論。<br>3. それでもメモリ不足なら `llama.cpp` / `vllm` で **MPS／Metal** バックエンドを使う。 | 20 B は「純正」16‑bit だと **>20 GB** になるので、**量子化** と **Apple Silicon の GPU (MPS)** を活用することが必須。 |

> **注意**  
> - 20 B モデルは 32 GB RAM でも「メモリフットプリント + バッファ + アクティビティ」等で **25–30 GB** くらい必要になる。  
> - M4 Pro の GPU は 4 GB くらいしかないので、GPU で走らせても CPU のメモリを使う。  
> - 量子化 (4‑bit) すると **約 10 GB** 前後になるので、32 GB RAM なら OK。8‑bit だと 20 GB くらい。  

以下では「**4‑bit 量子化版**」を使った最もシンプルな手順と、必要に応じて `llama.cpp` で実行する方法を説明します。

---

## 2. 前提条件

| 項目 | 必要条件 | 備考 |
|------|----------|------|
| **OS** | macOS 14.0 以降 (Apple Silicon) | M4 Pro なら macOS 15 でも OK |
| **Python** | 3.10 以上 | `pyenv` で管理すると便利 |
| **必要なパッケージ** | `homebrew`（コマンドラインツール） | `brew install llvm cmake ninja` でビルド環境を整える |
| **メモリ** | 32 GB 以上 | 20 B モデルの 4‑bit 版で約 10 GB |
| **GPU** | MPS 対応 (Apple Silicon) | `torch.backends.mps.is_available()` で確認 |

---

## 3. 量子化済みモデルの入手

### 3‑1. gpt‑oss‑20B‑ggml‑q4‑fp16

[gpt‑oss‑github](https://github.com/LLM-OS-OSS/gpt-oss) で公開されている `gpt-oss-20b-ggml` を 4‑bit で量子化したファイルです。  
GitHub 上では `model.bin`（約 10 GB）と `config.json` を入手できます。

```bash
# 例: Homebrew で wget を入手
brew install wget

# 例: 直接ダウンロード
wget https://huggingface.co/gpt-oss-20b-ggml/resolve/main/model.bin
wget https://huggingface.co/gpt-oss-20b-ggml/resolve/main/config.json
```

> **備考**  
> - `wget` が 4‑bit 版を直接提供していない場合は、  
>   `https://huggingface.co/gpt-oss-20b-ggml/resolve/main/gpt-oss-20b.ggmlv3.q4_0.bin` などのファイルを使います。  
> - ファイル名はバージョンに応じて変わるので、**Hugging Face Hub** で最新を確認してください。  

### 3‑2. 量子化済みモデルを使うためのディレクトリ構成

```
~/gpt-oss-20b/
├── model.bin           # 4‑bit quantized weights
└── config.json         # モデル設定
```

---

## 4. `transformers` で推論（最も手軽）

```bash
# 1. 仮想環境作成（推奨）
python3 -m venv ~/venv/gpt-oss
source ~/venv/gpt-oss/bin/activate

# 2. 必要パッケージインストール
pip install --upgrade pip
pip install torch==2.5.0 transformers==4.44.2 sentencepiece
# torch の MPS ビルドを使う
pip install torch==2.5.0+cpu -f https://download.pytorch.org/whl/cpu/torch_stable.html
# ただし、Apple Silicon では公式に MPS がサポートされているので
# pip install torch==2.5.0+cpu を使うと CPU 版、Apple Silicon 版は
# https://developer.apple.com/metal/pytorch/ でビルド済みの wheel を使う
```

> **ポイント**  
> - `torch==2.5.0` は MPS バックエンドをサポートしています。  
> - もし `pip install torch==2.5.0` が `pip` で入らない場合は、  
>   `https://developer.apple.com/metal/pytorch/` から `torch-2.5.0-macos-arm64.whl` をダウンロードして `pip install <path>` してください。

### 4‑1. Python スクリプト例

```python
import torch
from transformers import AutoModelForCausalLM, AutoTokenizer

# 量子化モデルのパス
MODEL_DIR = "/Users/yourname/gpt-oss-20b"

tokenizer = AutoTokenizer.from_pretrained(MODEL_DIR)
# モデルは 4‑bit なので torch_dtype を float16 にして高速化
model = AutoModelForCausalLM.from_pretrained(
    MODEL_DIR,
    device_map="auto",  # 自動で MPS を使う
    torch_dtype=torch.float16,   # 4‑bit でも float16 で安全
    low_cpu_mem_usage=True,
    trust_remote_code=True  # quantized モデルを読むため
)

# 推論関数
def ask(question: str, max_new_tokens=256):
    input_ids = tokenizer(question, return_tensors="pt").input_ids.to("mps")
    # 生成
    gen = model.generate(
        input_ids,
        max_new_tokens=max_new_tokens,
        temperature=0.7,
        top_p=0.9,
        do_sample=True,
        pad_token_id=tokenizer.eos_token_id,
    )
    return tokenizer.decode(gen[0], skip_special_tokens=True)

# テスト
print(ask("こんにちは、GPT-OSS 20B で何ができますか？"))
```

> **実行コマンド**  
> ```bash
> python run_gpt_oss.py
> ```

### 4‑2. メモリ・速度のポイント

| 項目 | 設定 | 効果 |
|------|------|------|
| `device_map="auto"` | MPS が使えるなら GPU、CPU でメモリを使い切る | 速くなる |
| `torch_dtype=torch.float16` | float16 で計算 | メモリ使用量をさらに減らす |
| `low_cpu_mem_usage=True` | 生成時にメモリを最小化 | 生成中の GC を減らす |
| `max_new_tokens` | 少なめに | 生成コストを抑える |

---

## 5. `llama.cpp` で推論（さらに軽量化・高速化）

`llama.cpp` は C/C++ で書かれた小さな推論エンジンで、Apple Silicon の MPS (Metal) を利用できます。  
`llama.cpp` で 20 B 量子化版を走らせるには、まずビルドしてから推論します。

### 5‑1. `llama.cpp` のビルド

```bash
# 1. ソース取得
git clone https://github.com/ggerganov/llama.cpp.git
cd llama.cpp

# 2. 依存パッケージ
brew install cmake ninja

# 3. ビルド
make -j$(sysctl -n hw.logicalcpu)  # CPU でビルド
```

> **備考**  
> - 16‑bit モデルを走らせると「`model.bin` が 20 GB 以上」になるので、ビルドは 64‑bit で行う。  
> - 4‑bit 量子化版なら `make -j$(sysctl -n hw.logicalcpu) LLAMA_BUILD_SHARED=1` でも OK。  
> - もし `make` が失敗したら、`LLAMA_METAL=1` を設定して Metal（MPS）を有効にしてください。

### 5‑2. 量子化済みモデルの変換（必要に応じて）

`llama.cpp` では **ggml** 形式を使います。すでに `gpt-oss-20b.ggmlv3.q4_0.bin` を入手している場合はそのまま使えます。  
もし変換が必要なら、`cpp/convert-*.cpp` を使って `bin` → `ggml` 変換を行います。

```bash
# 例: 16‑bit → 4‑bit 変換
./convert-ggml-quantize gpt-oss-20b.bin gpt-oss-20b.ggmlv3.q4_0.bin q4_0
```

> **注意**  
> - 変換時に `gpt-oss-20b.bin` が 20 GB 以上になるので、メモリが足りないとクラッシュします。  
> - 変換は一度だけで OK。

### 5‑3. 推論実行

```bash
./main -m ~/gpt-oss-20b/gpt-oss-20b.ggmlv3.q4_0.bin -p "こんにちは、GPT‑OSS 20B は何ができますか？" -ngl 128
```

| オプション | 意味 | 推奨値 |
|------------|------|--------|
| `-m` | モデルファイル | `ggmlv3.q4_0.bin` |
| `-p` | プロンプト | 任意 |
| `-ngl` | GPU を使う場合に `-1` で全て使う | 128 くらいで OK |
| `-c` | コンテキスト長 | 2048 〜 4096 |
| `-ngl` | なし | CPU で走る |
| `-t` | スレッド数 | `$(sysctl -n hw.logicalcpu)` |

> **ポイント**  
> - `-ngl 128` で **Metal**（MPS）を使っているので、M4 Pro の GPU が 4 GB の場合でも 4‑bit 版ならメモリが足りる。  
> - `-ngl 0` で CPU 版を走らせると、`-t` を高めにすると CPU コア数に合わせて高速化。

---

## 6. 実際の推論例（コマンドライン＋Python）

### 6‑1. Python で `llama.cpp` を呼び出す

```python
import subprocess
import json

def llama_infer(prompt, model_path, max_tokens=256):
    cmd = [
        "./main",
        "-m", model_path,
        "-p", prompt,
        "-n", str(max_tokens),
        "-ngl", "128",     # GPU を使う
        "-t", str(8),      # CPU コア数
        "-f", "stdout",    # stdout に出力
    ]
    result = subprocess.run(cmd, capture_output=True, text=True)
    return result.stdout.strip()

print(llama_infer("こんにちは！", "~/gpt-oss-20b/gpt-oss-20b.ggmlv3.q4_0.bin"))
```

> **備考**  
> - `-f stdout` を指定しないと `main` が対話型で待機するので、スクリプトから呼び出すときは必ず `-f stdout` を入れる。  
> - `-ngl 0` で CPU 版を走らせたい場合は、`-ngl` を 0 に設定。

### 6‑2. 速度とメモリのチェック

```bash
# 速度計測（例）
time ./main -m ~/gpt-oss-20b/gpt-oss-20b.ggmlv3.q4_0.bin -p "テスト" -n 64 -ngl 128

# メモリ使用量確認（macOS Activity Monitor で MPS GPU を確認）
```

---

## 7. よくあるトラブルと対処

| トラブル | 原因 | 対処 |
|----------|------|------|
| **`model.bin` が読み込めない** | 4‑bit 版と 16‑bit 版を混同している | `config.json` を確認し、`ggml` 形式の `*.bin` を使う |
| **`torch.backends.mps.is_available()` が False** | PyTorch が MPS をサポートしていないビルド | Apple Silicon 用の wheel をダウンロードしてインストール |
| **OOM（メモリ不足）** | 量子化前の 16‑bit 版を読み込んでいる | 4‑bit `q4_0` 版を使う |
| **推論が非常に遅い** | `device_map="auto"` が CPU を選択 | `torch.device("mps")` を明示的に指定 |
| **エラー: `Segmentation fault`** | 変換時に `ggml` ファイルが壊れている | 再変換（`convert-ggml-quantize`）し直す |

---

## 8. まとめ

1. **量子化済み 4‑bit モデル** を入手。  
2. **PyTorch（MPS）** か **llama.cpp（Metal）** のいずれかを使って推論。  
3. Python で簡単に質問できるスクリプトを書けば、対話型でも高速に走ります。  
4. `llama.cpp` ならさらに軽量化され、M4 Pro の GPU で高速に動かせます。  

このセットアップにより、**macOS 上で手軽に GPT‑OSS 20B をローカルで実行**できるようになります。  
ぜひ、自分のプロジェクトに合わせてパラメータを調整し、快適にご利用ください！
