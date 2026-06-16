# Continue VSCode 拡張機能 — 調査レポート

> 調査日: 2026-06-15  
> 調査方法: 公式ドキュメント・GitHub ソースコード・5角度並列検索 → 25クレームを敵対的検証  
> 信頼度: **高 (20/25クレーム確認済、5クレーム棄却)**  
> ⚠️ 注意: Continue は 2026年4月に Cursor に買収され、リポジトリは read-only (アーカイブ) 状態。ドキュメントは現時点では有効だが今後の更新は未定。

---

## 1. Continue とは何か

Continue はオープンソースの AI コーディングアシスタントで、以下の形式で提供される:

| 形式 | 説明 |
|---|---|
| VS Code 拡張機能 | メイン。サイドバーにチャット UI、エディタにインライン補完 |
| JetBrains プラグイン | VS Code と同等機能（一部制限あり） |
| CLI (`cn`) | ターミナルから Continue を操作 |

### 4つの動作モード

| モード | 概要 |
|---|---|
| **Chat** | サイドバーでコードについて質問・相談 |
| **Autocomplete** | エディタでのリアルタイムコード補完 |
| **Edit** | 選択範囲を指示で書き換え (`Cmd/Ctrl+I`) |
| **Agent** | ファイル操作・マルチステップタスクを自律実行 |

### アーキテクチャ（VS Code 内部）

```
VS Code 拡張ホスト
├─ core/          ← ビジネスロジック（プロバイダー非依存）
├─ extension/     ← VS Code API アダプター
└─ gui/           ← React + Redux Toolkit の Webview パネル
```

3コンポーネントがメッセージパッシングで通信する設計。

---

## 2. 設定ファイル: config.yaml と config.json

### 現在の推奨形式

| | config.yaml | config.json |
|---|---|---|
| 状態 | **現在の推奨** | **非推奨 (Deprecated)** |
| 優先度 | 両方ある場合 yaml が優先 | yaml がなければ使われる |

`config.yaml` が存在する場合、`config.json` は無視される。

### config.yaml の必須トップレベルプロパティ

```yaml
name: my-config
version: 1.0.0
schema: v1
```

それ以外のプロパティはすべてオプション。

### config.json → config.yaml 移行の主な変更点

| 概念 | config.json (旧) | config.yaml (新) |
|---|---|---|
| チャットモデル | `models[]` | `models[]` + `roles: [chat]` |
| 補完モデル | `tabAutocompleteModel` | `models[]` + `roles: [autocomplete]` |
| 埋め込みモデル | `embeddingsProvider` | `models[]` + `roles: [embed]` |
| リランクモデル | `reranker` | `models[]` + `roles: [rerank]` |
| チャンクサイズ | `embeddingsProvider.maxEmbeddingChunkSize` | `embedOptions.maxChunkSize` |
| バッチサイズ | `embeddingsProvider.maxEmbeddingBatchSize` | `embedOptions.maxBatchSize` |

### config.yaml のモデル定義例

```yaml
models:
  - name: claude-3-5-sonnet
    provider: anthropic
    model: claude-3-5-sonnet-20241022
    apiKey: ${ANTHROPIC_API_KEY}
    roles: [chat, edit]
    defaultCompletionOptions:
      contextLength: 200000
      maxTokens: 4096
      temperature: 0.2

  - name: nomic-embed-text
    provider: ollama
    model: nomic-embed-text
    roles: [embed]
    embedOptions:
      maxChunkSize: 256
      maxBatchSize: 64
```

---

## 3. モデルパラメータ一覧

### `defaultCompletionOptions` — 生成パラメータ

| パラメータ | 型 | 説明 |
|---|---|---|
| `contextLength` | number | モデルのコンテキストウィンドウ上限（トークン数）。**モデルの仕様に合わせて手動設定が必要** |
| `maxTokens` | number | 1回の補完で生成するトークン数の上限 |
| `temperature` | number (0.0〜1.0) | 出力のランダム性。0=決定論的、1=ランダム |
| `topP` | number | 核サンプリングの累積確率 |
| `topK` | number | 各ステップで考慮するトークン数の上限 |
| `stop` | string[] | 生成を停止するトークン列の配列 |
| `reasoning` | boolean | Anthropic / Ollama の思考(thinking)機能を有効化 |
| `reasoningBudgetTokens` | number | Claude 3.7+ の thinking に使うトークン予算 |
| `keepAlive` | number | Ollama でモデルをメモリに保持する秒数（デフォルト: 1800秒） |

> **`contextLength` と `maxTokens` の違い:**  
> - `contextLength` = モデルが受け取れるプロンプト全体の上限（入力）  
> - `maxTokens` = モデルが返答で生成できるトークン数の上限（出力）  
> 両方を設定する場合、`contextLength > maxTokens` になるように設定する。

### `chatOptions` — チャット/エージェントモード用

| パラメータ | 説明 |
|---|---|
| `baseSystemMessage` | Chat モード用のシステムプロンプト |
| `baseAgentSystemMessage` | Agent モード用のシステムプロンプト |
| `basePlanSystemMessage` | Plan モード用のシステムプロンプト |

### `requestOptions` — HTTP通信設定

| パラメータ | 説明 |
|---|---|
| `timeout` | リクエストタイムアウト（秒） |
| `verifySsl` | SSL証明書の検証有無 |
| `caBundlePath` | カスタム CA 証明書のパス |
| `proxy` | プロキシ URL |
| `headers` | カスタム HTTP ヘッダー |
| `clientCertificate` | クライアント証明書設定 |

### `autocompleteOptions` — 補完モード用

| パラメータ | 説明 |
|---|---|
| `debounceDelay` | 補完リクエスト送信までの待機時間（ms） |
| `maxPromptTokens` | 補完プロンプトの最大トークン数 |
| `modelTimeout` | 補完リクエストのタイムアウト（秒） |

### モデルパラメータの調べ方

```
1. 公式リファレンス:
   https://docs.continue.dev/reference
   → "models" セクション → 各パラメータの型・説明・デフォルト値

2. プロバイダー別のモデル仕様:
   - Anthropic: https://docs.anthropic.com/ja/docs/about-claude/models
     → 各モデルの context window / max output tokens を確認
   - OpenAI: https://platform.openai.com/docs/models
   - Ollama: `ollama show <model>` コマンドで context_length を確認

3. config_schema.json (ソースコード):
   https://github.com/continuedev/continue/blob/main/extensions/vscode/config_schema.json
   → JSON Schema 形式で全パラメータの型・説明が記載

4. Continue の GUI から確認:
   サイドバー → モデル選択 → 設定アイコン → パラメータ確認・変更
```

---

## 4. コードベースインデックスの仕組み

### インデックスの2層構造

```
~/.continue/index/
├─ LanceDB           ← ベクトルDB（埋め込みベクトル + ファイルパス）
└─ SQLite            ← code_snippets テーブル（コードエンティティ）
```

### インデックス生成フロー

```
ソースコード
    ↓
Tree-sitter (AST解析)      ← .scm クエリファイルで言語ごとに解析
    ↓
チャンク分割               ← maxChunkSize トークン以下に分割
    ↓
埋め込みモデル             ← テキスト → ベクトル変換
    ↓
LanceDB に保存             ← ~/.continue/index/ に永続化
```

### SQLite `code_snippets` テーブルの構造

| フィールド | 内容 |
|---|---|
| title | エンティティ名（関数名・クラス名など） |
| signature | シグネチャ（引数・戻り値の型） |
| content | エンティティの本文 |
| path | ファイルパス |
| cacheKey | インクリメンタル更新用キャッシュキー |
| startLine | 開始行番号 |
| endLine | 終了行番号 |

### デフォルト埋め込みモデル

| 環境 | デフォルト埋め込みモデル | 備考 |
|---|---|---|
| VS Code | `transformers.js` (all-MiniLM-L6-v2) | 拡張ホスト内でローカル実行 |
| JetBrains | **なし（要設定）** | transformers.js は使用不可 |

**推奨埋め込みモデル:**

| 用途 | プロバイダー | モデル |
|---|---|---|
| 品質最優先（クラウド） | Voyage AI | voyage-code-3（公式推奨） |
| ローカル実行 | Ollama | nomic-embed-text |
| OpenAI 利用 | OpenAI | text-embedding-3-small |

---

## 5. インデックスの上限・制約

### 確認された制約（ソースコード由来）

| 制約 | 値 | 根拠 |
|---|---|---|
| **1ファイルの最大サイズ** | **5MB** | `core/indexing/refreshIndex.ts` の `MAX_FILE_SIZE_BYTES` 定数 |
| maxChunkSize 最小値 | 128トークン | 公式ドキュメントに記載（コードでの強制は未確認） |
| maxBatchSize 最小値 | 1チャンク/リクエスト | 公式ドキュメントに記載 |

### 公式ドキュメントに記載のない制約

| 項目 | 状況 |
|---|---|
| インデックス対象ファイルの総数上限 | **記載なし**（LanceDB のキャパシティに依存） |
| コードベース全体のトークン数上限 | **記載なし** |
| コードベース全体のサイズ上限 | **記載なし** |

> **設計方針:** Continue はインデックスの総量制限をアプリケーション層で定義せず、使用するベクトルDB（LanceDB）と埋め込みプロバイダーのインフラ制約に委ねる設計。

### 大規模リポジトリでの実用的な注意点

- `5MB` を超えるファイル（バイナリ、minify済みJS、大規模 JSON など）は**自動的にスキップ**される
- `.gitignore` / `.continueignore` でインデックス対象を絞ることでパフォーマンスを改善できる
- 初回インデックス生成は大規模リポジトリで数分〜数十分かかる場合がある
- LanceDB のインデックスファイルは `~/.continue/index/` に保存され、ディスク容量を消費する

---

## 6. 調査で棄却されたクレーム（要注意）

以下は調査中に検索で出てきたが、**検証の結果誤りと判定された情報**:

| 誤情報 | 実際 |
|---|---|
| voyage-code-3 の最大コンテキスト長は 16,000 トークン | 未確認（公式ドキュメントに記載なし） |
| コードベースは「切り捨て / 固定行数分割 / AST再帰分割」の3戦略を使う | ソースコードに該当実装なし |
| codebase context provider は デフォルトで nRetrieve=50 / nFinal=10 | 確認できず（デフォルト値は不明） |

---

## 7. 参考ソース

| ソース | 種類 | URL |
|---|---|---|
| 公式ドキュメント | Primary | https://docs.continue.dev/ |
| config.yaml リファレンス | Primary | https://docs.continue.dev/reference |
| config.json リファレンス（非推奨） | Primary | https://docs.continue.dev/reference/json-reference |
| config.yaml 移行ガイド | Primary | https://docs.continue.dev/reference/yaml-migration |
| 埋め込みモデル設定 | Primary | https://docs.continue.dev/customize/model-roles/embeddings |
| カスタム RAG ガイド | Primary | https://docs.continue.dev/guides/custom-code-rag |
| GitHub リポジトリ（read-only） | Primary | https://github.com/continuedev/continue |
| config_schema.json | Primary | https://github.com/continuedev/continue/blob/main/extensions/vscode/config_schema.json |
| インデックス内部構造 (DeepWiki) | Secondary | https://deepwiki.com/continuedev/continue/3.4-codebase-indexing |

---

## 8. 未解決の疑問点

- `nRetrieve` / `nFinal`（ベクトル検索結果数）のデフォルト値と設定方法
- 増分インデックス（変更ファイルのみ再インデックス）の実装有無とトリガー条件
- Cursor 買収後の VS Code Marketplace 拡張機能の継続更新可否
- 大規模モノレポでの LanceDB パフォーマンス劣化の実際の閾値
