## コードレビュー（日本語）

以下では、現在のサンプルコードをそのまま実行した場合に起こりうる問題点や、より堅牢・保守性の高い実装にするための改善提案をまとめます。  
改善点は「**何が問題か**」と「**どうすればいいか**」の２段階で説明します。

| # | 問題点 | 改善案 | コメント |
|---|--------|--------|----------|
| 1 | **例外処理がない** | `requests.exceptions.RequestException` でラップし、タイムアウトや接続失敗時に適切にハンドリング | ネットワークは不安定。`print()` でエラー表示すると追跡が難しい。 |
| 2 | **ステータスコードをチェックしていない** | `r.raise_for_status()` を呼び出す | 4xx・5xx のレスポンスで何も出力しないと分からない |
| 3 | **URL・モデル名をハードコーディング** | 定数化／環境変数読み込み | 設定の変更がコード変更に繋がる |
| 4 | **`json` モジュールを `import json` で重複** | `import json` は必要。`requests` も `json=` を使っているが、`json.loads` の方は `json` モジュールに依存 | 変更なし |
| 5 | **`line` が bytes である点を考慮していない** | `r.iter_lines(decode_unicode=True)` または `line.decode()` | 文字化けの原因になる |
| 6 | **`print()` をその場で使い、`flush=True` を毎回呼んでいる** | 1) `print(..., flush=True)` を 1 回だけ呼ぶか、 2) `sys.stdout.write` でバッファリング制御 | パフォーマンス面で軽減 |
| 7 | **`j.get("done")` が真になると break しているが、残りの行を無視している** | ループの最後で `break` するのは OK だが、`finally` でリソースを確実に解放 | `with` なら自動解放 |
| 8 | **関数化されていない** | `def generate(prompt: str) -> str` などにまとめ、テストしやすく | 再利用性向上 |
| 9 | **logging で情報を出力** | `logging` モジュールでデバッグ／情報ログを出力 | `print` で出力するとユーザーが見逃しやすい |
|10 | **環境変数や CLI 引数でパラメータを変更できない** | `argparse` で `--model`, `--url`, `--prompt` を受け取る | ユーザー体験向上 |
|11 | **`json.loads` 失敗時の例外が無視される** | 例外をキャッチしてリトライまたはエラーメッセージ | 部分的なデータが壊れている場合に有用 |
|12 | **長時間実行時に SIGINT を受けても途中で `print()` が残る** | `try/except KeyboardInterrupt` で graceful exit | ユーザーが中断しやすい |

---

## 改善後のサンプルコード

以下は上記改善点を取り入れた実装例です。  
Python 3.10+ の型ヒントや f‑string、`with` での `Session` を利用しています。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-

"""
AI 生成エンドポイントをストリーミングで呼び出すサンプル
"""

from __future__ import annotations

import argparse
import json
import logging
import os
import sys
from typing import Generator

import requests
from requests.exceptions import RequestException

# ---------- ① 設定 ----------
DEFAULT_URL = os.getenv("LLM_API_URL", "http://localhost:11434/api/generate")
DEFAULT_MODEL = os.getenv("LLM_MODEL", "gpt-oss:20b")

# ---------- ② ロギング ----------
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s | %(levelname)s | %(message)s",
    stream=sys.stderr,
)
logger = logging.getLogger(__name__)

# ---------- ③ ストリーム生成 ----------
def stream_generate(
    url: str, *, model: str, prompt: str, timeout: int = 30
) -> Generator[str, None, None]:
    """
    生成結果をストリーミングで取得するジェネレータ。

    Parameters
    ----------
    url : str
        エンドポイント URL
    model : str
        使用するモデル名
    prompt : str
        生成したいプロンプト
    timeout : int
        接続タイムアウト（秒）

    Yields
    ------
    str
        生成途中のテキスト
    """
    payload = {"model": model, "prompt": prompt}
    session = requests.Session()

    try:
        with session.post(url, json=payload, stream=True, timeout=timeout) as resp:
            resp.raise_for_status()
            logger.debug("Connection established, streaming response.")
            for raw_line in resp.iter_lines(decode_unicode=True):
                if not raw_line:
                    continue
                try:
                    data = json.loads(raw_line)
                except json.JSONDecodeError as exc:
                    logger.warning("JSON decode error: %s", exc)
                    continue

                # 「response」フィールドがあれば yield
                if "response" in data:
                    yield data["response"]

                # 「done」が true になったらループ終了
                if data.get("done"):
                    logger.debug("Server signaled completion.")
                    break
    except RequestException as exc:
        logger.error("Network error: %s", exc)
        raise
    finally:
        session.close()


# ---------- ④ CLI ----------
def main() -> None:
    parser = argparse.ArgumentParser(
        description="Stream AI model response from LLM API."
    )
    parser.add_argument(
        "--url",
        default=DEFAULT_URL,
        help="LLM API endpoint (default: env LLM_API_URL)",
    )
    parser.add_argument(
        "--model",
        default=DEFAULT_MODEL,
        help="Model name to use (default: env LLM_MODEL)",
    )
    parser.add_argument(
        "--prompt",
        required=True,
        help="Prompt text for the model",
    )
    parser.add_argument(
        "--timeout",
        type=int,
        default=30,
        help="Connection timeout in seconds",
    )
    args = parser.parse_args()

    logger.info("Requesting generation with model=%s, prompt=%s", args.model, args.prompt)

    try:
        # 1. ストリームから出力を受け取って stdout に書き出す
        for chunk in stream_generate(
            args.url, model=args.model, prompt=args.prompt, timeout=args.timeout
        ):
            sys.stdout.write(chunk)
            sys.stdout.flush()
    except KeyboardInterrupt:
        logger.warning("Interrupted by user.")
        sys.exit(130)
    except Exception as exc:
        logger.exception("Unhandled exception: %s", exc)
        sys.exit(1)
    finally:
        print()  # 改行で終わらせる


if __name__ == "__main__":
    main()
```

### 主な変更点

| 変更 | 内容 |
|------|------|
| **設定を環境変数化** | `LLM_API_URL`、`LLM_MODEL` を使い、デフォルトも設定。 |
| **例外処理追加** | `RequestException` をキャッチし、ログに出力。 |
| **ステータスコード確認** | `resp.raise_for_status()` で 4xx/5xx を即時抑制。 |
| **`decode_unicode=True`** | `iter_lines` が文字列を返すようにし、`json.loads` のエラーを減らす。 |
| **ログレベル** | `logging` で情報・デバッグ・エラーを分離。 |
| **CLI** | `argparse` で `--url`・`--model`・`--prompt`・`--timeout` を受け取れる。 |
| **関数化** | `stream_generate()` で再利用性・テスト容易性を向上。 |
| **KeyboardInterrupt ハンドリング** | ユーザーが `Ctrl+C` で中断しても整合性を保つ。 |
| **最終改行** | `finally` で `print()` を呼び出し、整形した出力を保証。 |
| **型ヒント** | 予測可能性と IDE の補完を向上。 |

---

## さらに踏み込むと

| 項目 | 具体策 |
|------|--------|
| **非同期化** | `httpx.AsyncClient` で非同期ストリーミングに。 |
| **テスト** | `requests_mock` でストリームレスポンスをモックし、`unittest` / `pytest` で検証。 |
| **リトライ** | `tenacity` を使って一時的なネットワーク障害を自動リトライ。 |
| **UI** | `rich` ライブラリで進捗表示や色付きログを追加。 |
| **API の認証** | `Authorization` ヘッダーを環境変数で設定し、トークンを安全に扱う。 |
| **多言語対応** | `prompt` を多言語で渡せるように、`--lang` オプションを追加。 |

---

## まとめ

- **例外処理とステータスコード確認** を行うことで、ネットワーク障害や API 停止時の不具合を早期に検知できます。  
- **環境変数・CLI オプション** で設定を外部化すると、デプロイ環境への差分を減らせます。  
- **関数化** と **ロギング** の導入は、テスト容易性と運用時のトラブルシュートに直結します。  
- ストリーム処理は **`decode_unicode=True`** と **`json.loads` の例外処理** で安全性を担保。  

上記のサンプルをベースに、ご自身のプロジェクト要件（認証、複数モデルの切替、UI 等）に合わせて拡張してみてください。もし追加で疑問や実装に関して相談があれば、お気軽にどうぞ！
