## MacBook で ChatGPT を使うための全パターンガイド

| 使い方 | 何ができるか | 手順（ざっくり） | 備考 |
|--------|--------------|-----------------|------|
| **① Web 版 (chat.openai.com)** | すぐに使えるチャット、画像生成、コード補助、会話ログ管理など | 1. Safari/Chrome/Edge で https://chat.openai.com にアクセス <br> 2. OpenAI アカウントでサインイン <br> 3. 使い始める | **最も手軽** ですが、ブラウザが必要。 |
| **② 公式 macOS アプリ (OpenAI App)** | ネイティブ感覚、通知・ショートカット、離線モード（API キー設定） | 1. App Store で “OpenAI” を検索 <br> 2. 「入手」→インストール <br> 3. アプリを起動しサインイン | 2024 年 7 月以降、OpenAI が公式に macOS アプリを公開。 |
| **③ API で自分のアプリ・CLI** | もっと自由にカスタマイズしたい時、Python・JavaScript で自動化 | 1. openai API キーを取得 <br> 2. `pip install openai`（Python） <br> 3. `openai.ChatCompletion.create(...)` で呼び出し | コードスニペットや自動化スクリプトに最適。 |
| **④ Terminal から CLI で使う** | ショートカットで素早く質問したい時 | 1. `pip install openai` <br> 2. `openai api chat completions.create -m gpt-4 --prompt "..."` | シェルスクリプトに組み込むと便利。 |
| **⑤ VS Code 拡張機能** | コーディング中に AI アシスタントを呼び出す | 1. VS Code を開く <br> 2. 拡張機能「ChatGPT for VS Code」をインストール <br> 3. API キーを設定 | コードの説明やデバッグに役立つ。 |
| **⑥ Notion・Roam Research などの統合** | ドキュメント内で直接 AI を利用 | 1. Notion で「+」→「Add a block」→「ChatGPT」<br> 2. API キーを設定 | Markdown や表形式での生成が簡単。 |
| **⑦ ブラウザ拡張 (ChatGPT for Safari/Chrome)** | どのページでも AI に質問できる | 1. Safari/Chrome の拡張機能ストアで検索 <br> 2. インストールし、API キーを入力 | いつでも「Ctrl+Space」などのショートカットで呼び出せる。 |
| **⑧ サードパーティのデスクトップアプリ** | よくある機能をカスタマイズしたい時 | 例: “ChatGPT Desktop” (Electron)、“MacGPT” <br> これらは公式アプリではないので注意 | アプリの安全性は自分で確認すること。 |

---

## ① Web 版で始める

1. **ブラウザを開く**  
   Safari、Chrome、Edge いずれでも OK。  
2. **ログイン**  
   `https://chat.openai.com/` にアクセスし、メールアドレスとパスワードでサインイン。  
   - まだアカウントが無い場合は「Sign up」で登録。  
3. **初期設定**  
   - **モデル選択**: GPT‑3.5 Turbo が無料、GPT‑4 はサブスク（ChatGPT Plus）で利用。  
   - **ショートカット**: `Cmd + Shift + ;` でチャットを開く（ブラウザによって異なる場合があります）。  
4. **使い方**  
   - 質問を入力 → Enter で送信。  
   - 画像生成が必要なら「New Image」ボタンをクリック。  
   - コード補助が欲しいときは「Python」や「JavaScript」モードを選択。  

> **ポイント**  
> - **会話ログ**: 左側のメニューから過去の会話を参照。  
> - **ファイル添付**: PDF などをアップロードして内容を質問できる。  

---

## ② 公式 macOS アプリの利用

1. **App Store で検索**  
   「OpenAI」または「ChatGPT」と入力。  
2. **インストール**  
   「入手」→「インストール」  
3. **起動 & サインイン**  
   - アプリを開くとログイン画面が表示。  
   - Web 版と同じアカウントを使う。  
4. **設定**  
   - **通知**: 新しいメッセージがあるときに通知を受け取る。  
   - **ショートカット**: 「⌘+⇧+;」でクイックアクセス。  
   - **テーマ**: ダーク/ライトモード。  

> **メリット**  
> - ウィンドウを切り替えることなく ChatGPT が起動し、デスクトップ上で作業しながら利用できる。  
> - ウィジェット機能でメモやメッセージを確認。  

---

## ③ API で自作アプリ／自動化

### 1. API キー取得
1. `https://platform.openai.com/account/api-keys` でサインイン。  
2. 「Create new secret key」→ キーをコピーして安全に保管。  

### 2. Python でのサンプル
```bash
pip install openai
```

```python
import openai

openai.api_key = "YOUR_API_KEY"

response = openai.ChatCompletion.create(
    model="gpt-4o-mini",
    messages=[
        {"role": "user", "content": "Hello, how are you?"}
    ]
)

print(response.choices[0].message.content)
```

### 3. コマンドラインで呼び出す
```bash
export OPENAI_API_KEY=YOUR_API_KEY
openai api chat completions.create -m gpt-4o-mini --prompt "Explain recursion in simple terms."
```

> **用途**  
> - スクリプトで定期的にデータを集計してレポート作成。  
> - macOS Automator で「Run Shell Script」→ ChatGPT で結果を取得。  

---

## ④ Terminal から直接 ChatGPT を呼び出す

1. **CLI をインストール**  
   ```bash
   pip install openai
   ```
2. **簡易質問**  
   ```bash
   openai api chat.completions.create -m gpt-4o-mini --prompt "Translate 'おはようございます' to English."
   ```

> **便利ポイント**  
> - **自動化スクリプト**：`bash` で日次タスクに組み込む。  
> - **ショートカット**：`~/.zshrc` に `alias gpt='openai api chat.completions.create -m gpt-4o-mini'` などを追加。

---

## ⑤ VS Code 拡張機能で開発をサポート

1. VS Code を開く。  
2. 「Extensions」→検索「ChatGPT for VS Code」。  
3. インストール → **Command Palette** (`⌘+⇧+P`) → `ChatGPT: Set API Key` でキーを入力。  
4. コードブロックを選択 → `ChatGPT: Ask for help` で即座に説明や改善案を取得。  

> **メリット**  
> - コードレビューやリファクタリングの提案。  
> - タスク管理やドキュメント生成にも使える。  

---

## ⑥ Notion で AI を直接使う

1. Notion を開き、任意のページにカーソルを置く。  
2. `+` をクリック → 「Add a block」→「ChatGPT」。  
3. 初回は API キーを入力。  
4. 質問を入力 → AI が回答をそのままページに挿入。  

> **活用例**  
> - 企画書作成時にアウトラインを生成。  
> - 週次レポートの要点を自動で作成。  

---

## ⑦ ブラウザ拡張（Safari / Chrome）

| ブラウザ | 拡張名 | 使い方 |
|----------|--------|--------|
| Safari | “ChatGPT for Safari” | インストール → キー入力 → `Ctrl+Space` でチャット |
| Chrome | “ChatGPT for Chrome” | インストール → キー入力 → 右下のアイコンで呼び出し |

> **便利ポイント**  
> - **ウェブページの要約**：ページを開いた状態で拡張を呼び出すと要点がまとめられる。  

---

## ⑧ サードパーティのデスクトップアプリ（非公式）

- **ChatGPT Desktop (Electron)**  
  - 公式の「OpenAI App」が無い時代の代表。  
  - 使いやすい UI だが、セキュリティは自分で確認。  

- **MacGPT**  
  - macOS のネイティブ感覚。  
  - 公式アプリと同様にショートカット・通知をサポート。  

> **注意**  
> - 公式アプリ以外は API キーを自分で入力する必要がある。  
> - 仕様変更が早く、サポートが遅れる場合がある。  

---

## 使い方のコツと注意点

| コツ | 詳細 |
|------|------|
| **ショートカット** | `⌘+⇧+;`（Web 版・App）でチャットを即時開く。 |
| **セキュリティ** | 公式サイト以外のキーは使わない。 |
| **プライバシー** | 会話ログは自動でクラウドに保存される。重要な情報は注意。 |
| **料金** | GPT‑4 は Plus（$20/月）または Enterprise で利用。 |
| **プラグイン** | Web 版で「Plugins」からブラウザ、API、Zapier などを追加。 |
| **日本語対応** | GPT‑4 は日本語の自然な文章生成が得意。 |

---

## まとめ

- **最速**: Safari/Chrome で `chat.openai.com` を開くだけ。  
- **本格的**: 公式 macOS アプリをインストールして、デスクトップ上で作業しながら AI を活用。  
- **カスタマイズ**: API キーを使って自作スクリプト・CLI・VS Code などに組み込む。  
- **統合**: Notion・Roam・Google Docs など多くのツールに直接埋め込める拡張機能も充実。  

自分のワークフローに合った方法を組み合わせて、MacBook で ChatGPT をフル活用してみてください。もし具体的に作りたいスクリプトや設定があれば、遠慮なく質問してください！
