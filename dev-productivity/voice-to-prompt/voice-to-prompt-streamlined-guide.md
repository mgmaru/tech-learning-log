# 音声入力 → プロンプト整形 簡略化ガイド

## Fabric不要！音声からターミナルへ直接入力する方法

---

## このドキュメントについて

[voice-to-prompt-setup-guide.md](./voice-to-prompt-setup-guide.md) で構築した Amical → Fabric → Ollama のパイプラインでは、Fabricコマンドの手打ちがボトルネックになっています。

本ドキュメントでは、**Fabricを経由せずに**音声入力からターミナルへ直接プロンプトを流し込む方法を解説します。

### 従来のフロー（問題点あり）

```
Amical(音声入力) → クリップボード → Fabricコマンド手打ち → Ollama → CLI出力 → 手動コピペ
                                     ↑ ここがボトルネック              ↑ これも手間
```

### 簡略化後のフロー（本ドキュメントの目標）

```
Amical(音声入力) → 自動整形 → アクティブウィンドウ（Claude Code等）に直接ペースト
                   ↑ 手動操作なし！
```

---

## 前提条件

以下がセットアップ済みであることを前提とします。

- Windows 10/11 + WSL2（Ubuntu）
- Ollamaが **Windows側** にインストール・起動済み（`http://localhost:11434` でアクセス可能）
- Amicalが **Windows側** にインストール・設定済み（音声認識が動作する状態）
- Ollamaにモデルがダウンロード済み（例: `qwen3:14b`）

上記の環境構築がまだの方は、先に [voice-to-prompt-setup-guide.md](./voice-to-prompt-setup-guide.md) の Step 1〜4 を完了してください。

---

## Ollamaカスタムモデルによるプロンプト整形

### 概要

Ollamaの「Modelfile」機能を使い、開発プロンプト整形専用のカスタムモデルを作成します。

ポイントは **TEMPLATE** の書き換えです。Amicalは内部でOllamaに固定のシステムプロンプト（一般的なテキスト整形用）を送りますが、TEMPLATEから `{{ .System }}` を削除することで**Amicalのシステムプロンプトを無視**し、TEMPLATE内に埋め込んだ独自の指示を使わせます。

```
通常の流れ:
  Amical → [system: テキスト整形して] → Ollama → 整形されたテキスト

カスタムモデルの流れ:
  Amical → [system: テキスト整形して] → Ollama(TEMPLATE内の独自指示を使用) → 開発プロンプト
                                         ↑ Amicalの指示は無視される
```

### Step 1: Modelfileの作成

Windows側で **PowerShell** を開き、以下のコマンドを実行します。

> **重要**: プロンプトの原本は [prompt-templates/system-prompt.md](./prompt-templates/system-prompt.md) にあります。
> 内容を変更したい場合はそちらも併せて更新してください。

````powershell
# Modelfileの内容を作成
@"
FROM qwen3:14b

TEMPLATE """<|im_start|>system
あなたは、ソフトウェア開発者の音声入力テキストを、LLM（Claude Code等）に渡すための構造化されたプロンプトに変換する専門家です。

## あなたの役割
- 入力: 音声認識で変換された口語的・非構造的なテキスト
- 出力: Markdown形式で構造化された開発プロンプト

## 変換ルール

### 基本ルール
1. 口語表現を簡潔で明確な指示文に変換する
2. フィラーワード（えーと、あのー、まあ、なんか等）を除去する
3. 技術用語の誤認識を文脈から修正する（例: リアクト→React、ネクスト→Next.js、タイプスクリプト→TypeScript）
4. 出力は日本語で行う（技術用語・コード部分は英語のまま）
5. 余計な前置き・後書きは一切不要。構造化されたプロンプトのみを出力する

### 構造化ルール
入力内容からタスク種別を自動判定し、適切なセクション構成で出力する。
セクション見出しには「#」を使用する。
コードやコマンドはバッククォート3つで囲む。
不要なセクションは省略する（無理にすべてのセクションを埋めない）。

## タスク種別ごとの出力形式

### 新機能の実装
# 指示
{何を実装するかを1〜3文で簡潔に記述}

# 要件
- {箇条書きで要件を列挙}

# 技術スタック
- {使用する技術・ライブラリを列挙（言及があれば）}

# 実装例
（口頭で言及されたコード構造があればバッククォート3つで囲んで記述）

# 注意点
- {制約条件、考慮すべき点を列挙}

### バグ修正
# 指示
{何を修正するかを簡潔に記述}

# 現象
{発生している問題を記述}

# 再現手順
1. {手順があれば記述}

# 期待する動作
{本来どう動くべきかを記述}

# 注意点
- {修正時に注意すべき点を列挙}

### リファクタリング
# 指示
{何を改善するかを簡潔に記述}

# 対象
- {対象のファイル、関数、クラス等を列挙}

# 改善方針
- {どのように改善するかを記述}

# 注意点
- {既存の動作を壊さないための注意点を列挙}

### 調査・質問
# 調査内容
{何を知りたいかを簡潔に記述}

# 背景
{なぜ知りたいのか、文脈を記述}

# 知りたいこと
- {具体的に知りたいポイントを箇条書き}

## 変換例

### 例1: 新機能の実装
入力: えーとリアクトでログインフォーム作りたいんだけど、メールアドレスとパスワードの入力欄があって、バリデーションもつけてほしい。あとパスワードは8文字以上で。

出力:
# 指示
Reactでログインフォームコンポーネントを作成してください。

# 要件
- メールアドレス入力フィールド
- パスワード入力フィールド
- 各フィールドにバリデーションを実装する
- パスワードは8文字以上

# 注意点
- バリデーションエラー時はユーザーにメッセージを表示すること

### 例2: バグ修正
入力: ユーザー一覧ページでなんかページネーションが動かないんだよね。2ページ目をクリックしても1ページ目のデータが表示されたままになる。

出力:
# 指示
ユーザー一覧ページのページネーション機能を修正してください。

# 現象
2ページ目をクリックしても1ページ目のデータが表示されたままになる。

# 期待する動作
ページ番号をクリックしたら、該当ページのデータに切り替わること。

# 注意点
- ページ遷移時のAPIリクエストのパラメータ（offset/limit）を確認すること
- 状態管理でページ番号が正しく更新されているか確認すること

### 例3: リファクタリング
入力: APIのエラーハンドリングがあちこちにコピペされてるから共通化したい。トライキャッチの部分をまとめて、エラーメッセージの表示も統一したい。

出力:
# 指示
API呼び出しのエラーハンドリングを共通化してください。

# 対象
- API呼び出しを行っている各ファイルのtry-catchブロック
- エラーメッセージの表示処理

# 改善方針
- 共通のエラーハンドリングユーティリティを作成する
- 各API呼び出し箇所で共通ユーティリティを使用するようリファクタリングする
- エラーメッセージの表示形式を統一する

# 注意点
- 既存のエラーハンドリングの挙動を変えないこと
- API固有のエラー処理が必要な箇所は個別対応を残すこと

### 例4: 調査・質問
入力: ネクストJSのアップストリームキャッシュってどういう仕組みなの？今使ってるバージョン14から15に上げたいんだけどキャッシュ周りが変わったって聞いて。

出力:
# 調査内容
Next.js 14から15へのアップグレードにおけるキャッシュ機構の変更点を調査してください。

# 背景
Next.js 14から15へのバージョンアップを検討しており、キャッシュの仕組みが変更されたと聞いている。

# 知りたいこと
- Next.js 15でのキャッシュ機構の変更点
- Next.js 14との具体的な差分
- アップグレード時に必要な対応
- 破壊的変更の有無
<|im_end|>
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
{{ .Response }}<|im_end|>
"""

PARAMETER temperature 0.1
PARAMETER num_predict 4000
"@ | Set-Content -Path "$env:TEMP\Modelfile" -Encoding UTF8
````

> **解説**
> - `FROM qwen3:14b` : ベースとなるモデルを指定します。`qwen3:8b` や `gemma3:12b` に変更可能です
> - `TEMPLATE` : プロンプト全体のフォーマットを定義します。`{{ .System }}` を含めていないため、Amicalが送るシステムプロンプトは無視されます
> - `temperature 0.1` : 出力の一貫性を高めるため低い値に設定しています
> - `num_predict 4000` : 構造化された出力に十分なトークン数です
>
> **`<think>` タグの除去について**
>
> qwen3は思考モード（thinking mode）で応答の前に `<think>...</think>` ブロックを出力します。
> 思考モード自体は回答品質に寄与するため維持しつつ、タグを除去するには以下の方法があります:
>
> - **Ollama v0.9.0以上**: APIレスポンスで `thinking` と `content` が自動分離される。Amical側が `content` のみ使用していれば、タグは表示されません
> - **Ollama v0.9.0未満 / Amicalでタグが表示される場合**: Ollamaのバージョンをアップグレードしてください（`ollama update`）

### Step 2: カスタムモデルの作成

同じPowerShellで以下を実行します。

```powershell
ollama create dev-prompt-refiner -f "$env:TEMP\Modelfile"
```

以下のような出力が表示されれば成功です。

```
transferring model data
using existing layer sha256:xxxx
creating new layer sha256:xxxx
writing manifest
success
```

### Step 3: カスタムモデルの動作確認

作成したモデルが正しく動作するかテストします。

```powershell
ollama run dev-prompt-refiner "えーとリアクトでログインフォーム作りたいんだけどメールアドレスとパスワードのバリデーション付きでお願いします"
```

**期待される出力例**:

```markdown
# 指示
Reactでログインフォームコンポーネントを作成してください。

# 要件
- メールアドレス入力フィールド
- パスワード入力フィールド
- 各フィールドにバリデーションを実装する
- パスワードは8文字以上

# 注意点
- バリデーションエラー時はユーザーにメッセージを表示すること
```

`#` 見出しで構造化されたプロンプトに変換されていればOKです。

### Step 4: Amicalの設定

1. **Amicalを起動**します

2. **Settings > AI Models > Language** タブを開きます

3. **Ollamaプロバイダーを接続**します
   - Ollamaのアコーディオンを展開
   - URL欄に `http://localhost:11434` と入力
   - 「**Connect**」ボタンをクリック
   - ステータスが**緑色**になることを確認

4. **モデルを同期**します
   - 「**Sync Models**」ボタンをクリック
   - モデル一覧に `dev-prompt-refiner` が表示されることを確認

5. **デフォルトモデルを設定**します
   - デフォルトLanguage Modelのドロップダウンから `dev-prompt-refiner` を選択

6. **フォーマッティングを有効化**します
   - **Settings > Dictation > Formatting Settings** を開く
   - フォーマッティングのトグルを **ON** にする
   - 使用するモデルに `dev-prompt-refiner` を選択

### Step 5: End-to-Endテスト

1. **Claude Codeのターミナル**をアクティブウィンドウにします

2. Amicalの**ホットキー**を押して録音開始します

3. テスト用に以下のように話します:
   > 「Next.jsでユーザー認証機能を実装したい。JWTトークンを使って、ログインとサインアップの画面を作りたい」

4. ホットキーを再度押して**録音停止**します

5. 数秒待つと、整形されたプロンプトが**Claude Codeのターミナルに直接ペースト**されます

**成功の判定基準**:
- `#` 見出しでセクションが構造化されている（例: `# 指示`、`# 要件`、`# 注意点`）
- タスク種別に応じた適切なセクション構成になっている
- 口語的な表現が簡潔な指示文に変換されている
- フィラーワードが除去されている
- 技術用語が正しく表記されている（例: React、Next.js、TypeScript）
- Claude Codeのターミナルに直接入力されている

### カスタムモデルのプロンプトを修正したい場合

整形の品質を調整したい場合は、Modelfileを編集して再作成します。

```powershell
# 1. Modelfileを編集（メモ帳で開く）
notepad "$env:TEMP\Modelfile"

# 2. 編集後、モデルを再作成（同名で上書き）
ollama create dev-prompt-refiner -f "$env:TEMP\Modelfile"
```

### カスタムモデルを削除したい場合

```powershell
ollama rm dev-prompt-refiner
```

---

## トラブルシューティング

### 共通

#### Ollamaに接続できない

```powershell
# Windows側: Ollamaが起動しているか確認
curl http://localhost:11434/api/tags
```

起動していない場合はタスクトレイからOllamaを起動してください。

#### Ollamaの応答が遅い

- より軽量なモデルに変更してください（`qwen3:8b` は `qwen3:14b` より高速）
- Modelfileの `FROM` を変更して `ollama create` を再実行

### カスタムモデル

#### 整形結果がおかしい（一般的なテキスト整形になってしまう）

AmicalのシステムプロンプトがTEMPLATEの書き換えを上書きしている可能性があります。Ollamaのバージョンを最新に更新し、Modelfileを再作成してください。

#### カスタムモデルが表示されない

```powershell
# モデル一覧を確認
ollama list

# dev-prompt-refiner が表示されなければ、作成をやり直す
ollama create dev-prompt-refiner -f "$env:TEMP\Modelfile"
```

Amical側で「Sync Models」を再実行してください。

---

## まとめ

| 項目 | 内容 |
|---|---|
| Fabricの手打ち | 不要 |
| 手動操作 | 音声入力のみ |
| 整形エンジン | Ollama（カスタムModelfile） |
| 追加プロセス | なし |
| プロンプト変更 | Modelfile再作成 |
| 動作環境 | Windows + Amical + Ollama |

---

## 補足: TEMPLATEの仕組み

### AmicalがOllamaに送るシステムプロンプト

Amicalのソースコード（[GitHub](https://github.com/amicalhq/amical) `apps/desktop/src/pipeline/providers/formatting/`）を調査した結果、AmicalはOllamaの `/api/chat` エンドポイントに以下の形式でリクエストを送信しています。

```typescript
// ollama-formatter.ts
const response = await fetch(`${this.ollamaUrl}/api/chat`, {
  method: "POST",
  body: JSON.stringify({
    model: this.model,
    messages: [
      { role: "system", content: systemPrompt },
      { role: "user", content: text },
    ],
    stream: false,
    options: { temperature: 0.1, num_predict: 2000 },
  }),
});
const data = await response.json();
const aiResponse = data.message?.content ?? "";
```

`systemPrompt` は `formatter-prompt.ts` の `constructFormatterPrompt()` 関数で動的に構築されます。デフォルト（一般的なアプリケーション使用時）のシステムプロンプト全文は以下の通りです。

```
You are a professional text formatter. Your task is to format transcribed text
to be clear, readable, and properly structured.

Instructions:
1. Fix any transcription errors based on context and custom vocabulary
2. Add proper punctuation and capitalization
3. Format paragraphs appropriately with sufficient line breaks
4. Maintain the original meaning and tone
5. Use the custom vocabulary to correct domain-specific terms
6. Remove unnecessary filler words (um, uh, etc.) but keep natural speech patterns
7. If the text is empty, return <formatted_text></formatted_text>
8. Return ONLY the formatted text enclosed in <formatted_text></formatted_text> tags
9. Do not include any commentary, explanations, or text outside the XML tags
10. Apply standard formatting for general text
11. Create logical paragraph breaks based on content flow
12. Maintain consistent formatting throughout
13. Preserve the original tone and style
```

上記はデフォルトのケースです。Amicalはフォアグラウンドのアプリケーション種別に応じて、コンテキスト固有のルールを動的に追加します。

| アプリ種別 | 追加されるルール |
|---|---|
| メールアプリ（Outlook, Gmail等） | メール構造、署名、メタデータ保持 |
| チャットアプリ（Slack, Discord等） | 会話的トーン、絵文字保持 |
| ノートアプリ（Notion, Evernote等） | 見出し、箇条書き、階層構造 |
| その他（デフォルト） | 標準的なテキスト整形 |

### {{ .System }} 削除によるシステムプロンプトの無視

OllamaはGoの `text/template` エンジンを使用してTEMPLATEを処理します。処理の流れは以下の通りです。

1. Amicalが `/api/chat` に送った `messages` 配列から `role: "system"` のメッセージが抽出され、`.System` 変数に格納される
2. TEMPLATEの実行時、`{{ .System }}` がTEMPLATEに存在しなければ、Goのテンプレートエンジンはその変数を**単純に無視**する（`Option("missingkey=zero")` 設定により、未参照変数はエラーにならず空文字列として扱われる）
3. 結果として、TEMPLATE内にハードコードした独自のシステムプロンプトのみがモデルに渡される

| TEMPLATEの種類 | {{ .System }} あり | {{ .System }} なし |
|---|---|---|
| レガシー形式（`.Prompt`/`.Response` 使用） | Amicalのプロンプトが挿入される | **Amicalのプロンプトは無視される** |
| モダン形式（`{{ range .Messages }}` 使用） | 挿入される | `.Messages` 配列経由で漏れる可能性あり |

本ドキュメントのModelfileはすべてレガシー形式を使用しているため、`{{ .System }}` を含めないことで確実にAmicalのシステムプロンプトは無視されます。

---

## モデル別のModelfileとパラメータ設定

Step 1 では `qwen3:14b` をベースモデルとして使用していますが、以下の4モデルも安定したプロンプト整形が可能です。各モデルのアーキテクチャに最適化したTEMPLATEとPARAMETERを記載します。

> **システムプロンプトについて**: 4つのModelfileすべてで、Step 1 と同じシステムプロンプト（原本: [prompt-templates/system-prompt.md](./prompt-templates/system-prompt.md)）を使用します。以下のModelfileでは構造を明確にするため、システムプロンプト本文を省略しています。`（※ システムプロンプト本文 — 全文は Step 1 または prompt-templates/system-prompt.md を参照）` の箇所に全文を挿入してください。

### 推奨パラメータ比較

| パラメータ | gemma-3-12b-it | gemma-3n-E4B-it | Ministral-3-8B | Qwen3-8B |
|---|---|---|---|---|
| `temperature` | 0.1 | 0.1 | 0.1 | 0.7 |
| `top_k` | 10 | 10 | 40 | 20 |
| `top_p` | 0.7 | 0.7 | 0.9 | 0.8 |
| `repeat_penalty` | 1.0 | 1.0 | 1.1 | 1.0 |
| `presence_penalty` | - | - | - | 1.5 |
| `num_ctx` | 8192 | 8192 | 8192 | 8192 |
| `num_predict` | 4000 | 4000 | 4000 | 4000 |
| `seed` | 42 | 42 | 42 | - |
| `stop` | `<end_of_turn>` | `<end_of_turn>` | `</s>` | `<\|im_end\|>` |

> **パラメータ解説**
>
> | パラメータ | 説明 |
> |---|---|
> | `temperature` | 低いほど出力が決定論的。Qwen3のみ公式推奨が0.7（低すぎると品質低下・ループ発生） |
> | `top_k` / `top_p` | サンプリング対象のトークン候補を制限。低temperatureとの組み合わせでは影響は小さい |
> | `repeat_penalty` | Gemma系はGoogle公式で1.0（無効）を推奨。Markdownの `#` `-` へのペナルティ防止のため |
> | `presence_penalty` | Qwen3のQ4_K_M版は繰り返しループが発生しやすく、Qwen公式が1.5を推奨 |
> | `seed` | 固定シードで出力の再現性を向上。Qwen3はtemperature 0.7で効果が薄く省略 |
> | `stop` | モデルが生成を停止するトークン。テンプレート形式に応じて異なる |

### モデル別の注意事項

| モデル | 注意点 |
|---|---|
| gemma-3-12b-it | KVキャッシュ量子化（`OLLAMA_KV_CACHE_TYPE` 環境変数）は設定しないこと（速度が激減する既知バグ） |
| gemma-3n-E4B-it | OllamaのデフォルトタグはQ4_K_M。Q8版は `gemma3n:e4b-it-q8_0` を明示的に指定すること |
| Ministral-3-8B | Ollama 0.13.1以上が必要 |
| Qwen3-8B | Thinkingモードの無効化を推奨（後述）。`presence_penalty` は1.5を超えると日本語に他言語が混ざる可能性あり |

### Modelfile: gemma-3-12b-it（Q4_K_M）

> **Gemma系モデルにはsystemロールが存在しません。** システムプロンプトはuserターン内に埋め込み、入力テキストとの間にセパレータを入れて区別します。

````
FROM gemma3:12b

TEMPLATE """<start_of_turn>user
あなたは、ソフトウェア開発者の音声入力テキストを、LLM（Claude Code等）に渡すための構造化されたプロンプトに変換する専門家です。

## あなたの役割
- 入力: 音声認識で変換された口語的・非構造的なテキスト
- 出力: Markdown形式で構造化された開発プロンプト

（※ システムプロンプト本文 — 全文は Step 1 または prompt-templates/system-prompt.md を参照）

---
以下の音声入力テキストを変換してください:
{{ .Prompt }}<end_of_turn>
<start_of_turn>model
{{ .Response }}<end_of_turn>
"""

PARAMETER temperature    0.1
PARAMETER top_k          10
PARAMETER top_p          0.7
PARAMETER repeat_penalty 1.0
PARAMETER num_ctx        8192
PARAMETER num_predict    4000
PARAMETER seed           42
PARAMETER stop           <end_of_turn>
````

### Modelfile: gemma-3n-E4B-it（Q8）

TEMPLATE構造はgemma-3-12b-itと同一です。`FROM` 行とstopトークンのみ異なります。

````
FROM gemma3n:e4b-it-q8_0

TEMPLATE """<start_of_turn>user
あなたは、ソフトウェア開発者の音声入力テキストを、LLM（Claude Code等）に渡すための構造化されたプロンプトに変換する専門家です。

## あなたの役割
- 入力: 音声認識で変換された口語的・非構造的なテキスト
- 出力: Markdown形式で構造化された開発プロンプト

（※ システムプロンプト本文 — 全文は Step 1 または prompt-templates/system-prompt.md を参照）

---
以下の音声入力テキストを変換してください:
{{ .Prompt }}<end_of_turn>
<start_of_turn>model
{{ .Response }}<end_of_turn>
"""

PARAMETER temperature    0.1
PARAMETER top_k          10
PARAMETER top_p          0.7
PARAMETER repeat_penalty 1.0
PARAMETER num_ctx        8192
PARAMETER num_predict    4000
PARAMETER seed           42
PARAMETER stop           <end_of_turn>
````

### Modelfile: Ministral-3-8B-Instruct-2512（Q4_K_M）

> Mistral系モデルは `[SYSTEM_PROMPT]`/`[/SYSTEM_PROMPT]` でシステムプロンプトを囲み、`[INST]`/`[/INST]` でユーザー入力を囲みます。

````
FROM ministral-3:8b

TEMPLATE """[SYSTEM_PROMPT]
あなたは、ソフトウェア開発者の音声入力テキストを、LLM（Claude Code等）に渡すための構造化されたプロンプトに変換する専門家です。

## あなたの役割
- 入力: 音声認識で変換された口語的・非構造的なテキスト
- 出力: Markdown形式で構造化された開発プロンプト

（※ システムプロンプト本文 — 全文は Step 1 または prompt-templates/system-prompt.md を参照）
[/SYSTEM_PROMPT][INST] {{ .Prompt }} [/INST]{{ .Response }}</s>
"""

PARAMETER temperature    0.1
PARAMETER top_k          40
PARAMETER top_p          0.9
PARAMETER repeat_penalty 1.1
PARAMETER num_ctx        8192
PARAMETER num_predict    4000
PARAMETER seed           42
PARAMETER stop           </s>
````

### Modelfile: Qwen3-8B（Q4_K_M）

> **Thinkingモードの無効化を推奨します。** プロンプト整形はテキスト変換タスクであり、複雑な推論は不要です。非Thinkingモードの方が低レイテンシでクリーンな出力が得られます。以下のModelfileではシステムプロンプトの先頭・末尾に `/no_think` を配置して無効化しています。
>
> **temperature が 0.7 と高い理由**: Qwen3は他モデルと設計思想が異なり、公式が非Thinkingモードでも0.7を推奨しています。これより低くすると出力品質が低下し、Thinkingモード有効時は0.6未満で無限ループに陥ります。

````
FROM qwen3:8b

TEMPLATE """<|im_start|>system
/no_think
あなたは、ソフトウェア開発者の音声入力テキストを、LLM（Claude Code等）に渡すための構造化されたプロンプトに変換する専門家です。

## あなたの役割
- 入力: 音声認識で変換された口語的・非構造的なテキスト
- 出力: Markdown形式で構造化された開発プロンプト

（※ システムプロンプト本文 — 全文は Step 1 または prompt-templates/system-prompt.md を参照）
/no_think
<|im_end|>
<|im_start|>user
{{ .Prompt }}<|im_end|>
<|im_start|>assistant
{{ .Response }}<|im_end|>
"""

PARAMETER temperature       0.7
PARAMETER top_k             20
PARAMETER top_p             0.8
PARAMETER repeat_penalty    1.0
PARAMETER presence_penalty  1.5
PARAMETER num_ctx           8192
PARAMETER num_predict       4000
PARAMETER stop              <|im_end|>
````

### カスタムモデルの作成手順

各Modelfileからカスタムモデルを作成する手順は共通です。PowerShellで以下を実行します。

```powershell
# 1. Modelfileを保存（例: gemma-3-12b-it の場合）
#    上記のModelfile内容をテキストエディタで作成し、任意のパスに保存
notepad "$env:TEMP\Modelfile-gemma12b"

# 2. カスタムモデルを作成
ollama create dev-prompt-refiner-gemma12b -f "$env:TEMP\Modelfile-gemma12b"
```

| ベースモデル | 推奨カスタムモデル名 | Modelfileパス例 |
|---|---|---|
| gemma3:12b | `dev-prompt-refiner-gemma12b` | `$env:TEMP\Modelfile-gemma12b` |
| gemma3n:e4b-it-q8_0 | `dev-prompt-refiner-gemma3n` | `$env:TEMP\Modelfile-gemma3n` |
| ministral-3:8b | `dev-prompt-refiner-ministral` | `$env:TEMP\Modelfile-ministral` |
| qwen3:8b | `dev-prompt-refiner-qwen3-8b` | `$env:TEMP\Modelfile-qwen3-8b` |

作成後、Amicalの Settings で「Sync Models」を実行し、作成したカスタムモデルを選択してください（Step 4 参照）。

### プロンプト整形タスクへの総合評価

| 観点 | gemma-3-12b-it | gemma-3n-E4B-it | Ministral-3-8B | Qwen3-8B |
|---|---|---|---|---|
| 出力の決定論性 | ◎（temp 0.1） | ◎（temp 0.1） | ◎（temp 0.1） | △（temp 0.7） |
| 量子化の安定性 | ○（Q4_K_M） | ◎（Q8） | ○（Q4_K_M） | △（ループ対策必要） |
| 設定のシンプルさ | ◎ | ◎ | ◎ | △（thinking制御が必要） |
| 日本語能力 | ◎ | ○ | ○ | ◎ |
| 推論速度 | ○ | ◎ | ◎ | ○ |

- **安定性重視**: gemma-3-12b-it または Ministral-3-8B（パラメータ調整が少なく扱いやすい）
- **速度重視**: gemma-3n-E4B-it（実効4Bで高速、Q8で精度維持）
- **日本語品質重視**: Qwen3-8B（日本語能力が高いが、追加の調整が必要）

---
## メモ
### モデル調査・検証
#### Llama系
- [x] Llama3.2-3b-Instruct：https://huggingface.co/bartowski/Llama-3.2-3B-Instruct-GGUF
→ 全然ダメ。プロンプトを作成するのではなくて、普通に回答してしまう。。
- [x] Llama3.1-8b-Instruct：https://huggingface.co/bartowski/Meta-Llama-3.1-8B-Instruct-GGUF
→ 全然ダメ。プロンプトを作成するのではなくて、普通に回答してしまう。
#### DeepSeek系
- [x] DeepSeek-R1-Distill-Llama-8B（Q4KM）：https://huggingface.co/bartowski/DeepSeek-R1-Distill-Llama-8B-GGUF
→ 全然ダメ。
- [x] DeepSeek-R1-Distill-Qwen-7B（Q4KM）：https://huggingface.co/deepseek-ai/DeepSeek-R1-Distill-Qwen-7B
→ 中国語で出力される
- [x] DeepSeek-R1-Distill-Qwen-14B-Japanese（Q4KM）：https://huggingface.co/cyberagent/DeepSeek-R1-Distill-Qwen-14B-Japanese
→ 惜しい（同内容が2回出力、生成が若干遅い、指示追従性はあまり高くない印象、出力安定しない）
#### Qwen系
- [x] Qwen3-8B（Q4KM）：https://huggingface.co/Qwen/Qwen3-8B
→ 精度はすごく良い。出力速度も速い。<think>タグさえ消せれば一番良さそう。
#### gemma系
- [x] gemma3-4b-it（Q4KM）：https://huggingface.co/unsloth/gemma-3-4b-it-GGUF
→ 全然ダメ。プロンプトを作成するのではなくて、普通に回答してしまう。
- [x] gemma3-4b-it（bf16）：https://huggingface.co/unsloth/gemma-3-4b-it-GGUF
→ ちょっと余計なタグが出現している（医療費控除に関する質問なのに、技術スタックのタグが存在する等）
- [x] gemma-3-12b-it（Q4KM）：https://huggingface.co/google/gemma-3-12b-it
→ 中々良い。実用レベル。指示追従性高い。冗長な文章もなく簡潔にプロンプトを作成してくれている。
- [x]x gemma-3n-E4B-it-GGUF（Q4KM）：https://huggingface.co/unsloth/gemma-3n-E4B-it-GGUF
→ あまり精度が良くない。日本語がところどころおかしい。gemma系は量子化がダメなのかも...
- [x] gemma-3n-E4B-it-GGUF（Q8）：https://huggingface.co/unsloth/gemma-3n-E4B-it-GGUF
→ よくわからないタグが出てきたが、内容は良い。システムプロンプトを修正すれば実用的になりそう。
- [x] gemma-2-9b-it（Q4KM）：https://huggingface.co/bartowski/gemma-2-9b-it-GGUF
→ 全然ダメ。プロンプトを作成するのではなくて、普通に回答してしまう。
- [x] gemma-2-2b-jpn-it（BF16）：https://huggingface.co/google/gemma-2-2b-jpn-it
→ ~~中々良い。実用レベル。生成速度も速い。~~ 全然ダメだった。
#### その他
- [x] granite-4.0-micro-GGUF（F16）：https://huggingface.co/ibm-granite/granite-4.0-micro-GGUF
→ 全然ダメ。プロンプトを作成するのではなくて、普通に回答してしまう。
- [x] LFM2.5-1.2B-JP（BF16）：https://huggingface.co/LiquidAI/LFM2.5-1.2B-JP-GGUF
→ Ollamaに入らなかった。
- [x] mistralai_Ministral-3-8B-Instruct-2512（Q4KM）：https://huggingface.co/bartowski/mistralai_Ministral-3-8B-Instruct-2512-GGUF
→　中々良い。実用的。指示追従性高い。若干冗長気味。

### 良さそうなモデル
- [] gemma-3-12b-it（Q4KM）
- [] gemma-3n-E4B-it-GGUF（Q8）
- [] mistralai_Ministral-3-8B-Instruct-2512（Q4KM）
- [] Qwen3-8B（Q4KM） 


### Amicalのエラーについて
- Yetiのマイクで音声入力したのにも関わらず、「configure mike」と出てきて、マイクが認識されない問題が発生。
→ 別のマイク（Webcam）で試したところ、問題なくできた。
→ もしかすると、セキュリティソフトにでブロックされている可能性も...？← なさそう。多分Yetiのマイクのせい。
→ ~~プッシュキーを変更すると治る？~~
→ どうやら、会話の終わりに、プッシュキーを最後まで押し続ける必要があるみたい（話し終わった後、1秒くらいは余分に押し続ける）
→ 1回、FormattingをOFFにして、音声入力してから、再度ＯＮにすると治る。
