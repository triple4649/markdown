## Python で PDF ファイルを結合（マージ）する方法

PDF を「結合する」＝複数の PDF を一つにまとめる処理は、**PyPDF2 / pypdf**（推奨）や **PyMuPDF / fitz**、**pdfrw** などで簡単に実装できます。  
以下では、最も使いやすく、公式にサポートされている **pypdf** を使った例を中心に紹介します。  
（pypdf は PyPDF2 の後継ライブラリで、Python 3.8+ で動作します。)

---

## 1. 必要なパッケージをインストール

```bash
# pip で pypdf をインストール
pip install pypdf
```

> **備考**  
> - `pypdf` は `PyPDF2` のフォークであり、同じインターフェースをほぼそのまま使えます。  
> - `pip install pypdf` で最新版（2024 版）を入れられます。  
> - `pdfrw` などを使う場合は `pip install pdfrw`、`PyMuPDF` は `pip install pymupdf` で入れます。

---

## 2. 基本的な結合（マージ）スクリプト

### 2‑1. 2 ファイルだけを結合する簡単例

```python
# merge_two.py
from pypdf import PdfMerger

def merge_two(pdf1: str, pdf2: str, out_path: str):
    merger = PdfMerger()
    merger.append(pdf1)
    merger.append(pdf2)
    merger.write(out_path)
    merger.close()
    print(f"マージ完了: {out_path}")

if __name__ == "__main__":
    merge_two("input1.pdf", "input2.pdf", "merged.pdf")
```

### 2‑2. フォルダ内の PDF をすべて順番に結合

```python
# merge_folder.py
import os
from pathlib import Path
from pypdf import PdfMerger

def merge_folder(folder_path: str, out_path: str, sort=True):
    folder = Path(folder_path)
    # PDF ファイルだけ抽出
    pdf_files = sorted(folder.glob("*.pdf")) if sort else folder.glob("*.pdf")

    merger = PdfMerger()
    for pdf in pdf_files:
        try:
            merger.append(str(pdf))
            print(f"追加: {pdf.name}")
        except Exception as e:
            print(f"⚠️  {pdf.name} でエラー: {e}")

    merger.write(out_path)
    merger.close()
    print(f"マージ完了: {out_path}")

if __name__ == "__main__":
    merge_folder("input_folder", "all_merged.pdf")
```

> **ポイント**  
> - `sorted()` でファイル名順に結合したい場合は `sort=True`。  
> - ファイル名に数字が含まれている場合は、`natsort` などを使って自然順に並べ替えると便利です。

---

## 3. さらに柔軟な結合（ページ範囲・ページ指定）

```python
# merge_with_range.py
from pypdf import PdfMerger

def merge_with_pages(pdf_path: str, pages: tuple[int, int], out_path: str):
    """
    pages: (start_index, end_index)  0-indexed, end_indexは含まない
    例: (0, 2) は 1 ページ目と 2 ページ目のみを抽出
    """
    merger = PdfMerger()
    merger.append(pdf_path, pages=pages)
    merger.write(out_path)
    merger.close()

# 使い方
if __name__ == "__main__":
    # 1.pdf の 1〜3 ページ目だけを抽出
    merge_with_pages("1.pdf", (0, 3), "partial_1.pdf")
    # 2.pdf 全ページを追加
    merger = PdfMerger()
    merger.append("partial_1.pdf")
    merger.append("2.pdf")
    merger.write("final.pdf")
    merger.close()
```

> **注意点**  
> - `pages` はタプル（開始インデックス, 終了インデックス）で、終了インデックスは「含まない」ので `(0, 3)` は 1〜3 ページ目を取り出します。  
> - 逆順で追加したい場合は `append(pdf, pages=(None, None))` で逆順に取得できます。

---

## 4. PDF に暗号がかかっている場合の対処

暗号化された PDF をマージしようとすると `PdfReadError` が発生します。  
`PdfMerger.append()` の引数 `password` でパスワードを渡せます。

```python
merger.append("protected.pdf", password="mypassword")
```

> **実装上の注意**  
> - すべての PDF が暗号化されている場合は、まず読み込めるかどうか確認してから追加してください。  
> - もしパスワードがわからない場合は、`pypdf` ではスキップして処理を継続するように例外処理を組み込むと安全です。

---

## 5. コマンドラインユーティリティ（簡易スクリプト）

```python
# pdf_merge_cli.py
import argparse
from pathlib import Path
from pypdf import PdfMerger

def parse_args():
    parser = argparse.ArgumentParser(description="PDF を結合する簡易ツール")
    parser.add_argument("files", nargs="+", help="結合したい PDF ファイル（順序に従う）")
    parser.add_argument("-o", "--output", required=True, help="出力ファイル名")
    return parser.parse_args()

def main():
    args = parse_args()
    merger = PdfMerger()
    for fp in args.files:
        merger.append(str(Path(fp)))
        print(f"追加: {fp}")
    merger.write(args.output)
    merger.close()
    print(f"=== 完了: {args.output}")

if __name__ == "__main__":
    main()
```

**実行例**

```bash
python pdf_merge_cli.py part1.pdf part2.pdf part3.pdf -o full.pdf
```

> **便利機能**  
> - `argparse` の `nargs='*'` や `--sort` オプションを追加すれば、ファイル名順に自動結合やファイルを除外することもできます。  
> - `Path.glob("*.pdf")` でフォルダ内の PDF を一括取得して自動結合するように拡張できます。

---

## 6. 代替手段：PyMuPDF（fitz）での結合

PyMuPDF は PDF だけでなく画像や PDF 内のテキスト抽出も高速に行えるライブラリです。結合は次のように実装できます。

```python
import fitz  # PyMuPDF

def merge_with_fitz(pdf_paths, output):
    doc = fitz.open()  # 新規ドキュメント
    for p in pdf_paths:
        src = fitz.open(p)
        for page in src:
            doc.insert_pdf(src, from_page=page, to_page=page)
        src.close()
    doc.save(output)
    doc.close()

merge_with_fitz(["a.pdf", "b.pdf"], "merged_fitz.pdf")
```

> **メリット**  
> - 大容量 PDF でも高速。  
> - 画像の埋め込みやページのリサイズなども容易に行える。  
> - ただし、PyMuPDF はライセンス（GPL/AGPL）に注意が必要です。  

---

## 7. まとめ

| ライブラリ | 主な特徴 | 推奨用途 |
|------------|----------|----------|
| **pypdf** | 公式サポート、Python 3.8+、簡潔な API | 一般的なマージ・分割・ページ操作 |
| **PyMuPDF (fitz)** | 高速、画像・テキスト抽出も同時に行える | 大容量 PDF の処理、画像編集など |
| **pdfrw** | オープンソース、シンプル | 既存の Python スクリプトに組み込みやすい |
| **PyPDF2** | pypdf の前身 | 互換性重視の古いコードベース |

> **実務でのヒント**  
> 1. **ファイルサイズが大きい**場合は `pypdf` で結合する前に `merge_pages` のようにページ単位で処理し、メモリ使用量を抑える。  
> 2. **結合後にページ番号を付与したい**場合は `pypdf` の `merge` で `bookmark` を付けることができる。  
> 3. **スクリプトを CI/CD で使う**場合は `requirements.txt` か `pyproject.toml` に `pypdf==X.Y.Z` を記載しておくと再現性が保てます。  

---

### さらに学びたい方へ

- **公式ドキュメント**  
  - [pypdf Documentation](https://pypdf.readthedocs.io/)  
  - [PyMuPDF Docs](https://pymupdf.readthedocs.io/)  

- **サンプルプロジェクト**  
  - GitHub で検索: `python pdf merge script` で実際のコード例が多数公開されています。  

もし、**特定の要件（例：ページ番号のリセット、PDF の再圧縮、ページ範囲の指定）**があれば、遠慮なく教えてください。さらに細かい実装例を提供します！
