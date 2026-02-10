# 音声入力 → プロンプト整形パイプライン 構築ガイド

## Amical（音声認識）→ Fabric（プロンプト整形）→ Ollama（推論）

---

## 全体像

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Amical    │ ──→ │    Fabric    │ ──→ │   Ollama     │
│（Windows側）│     │ （WSL2側）   │     │（Windows側） │
│  音声→テキスト│     │ パターンで整形 │     │ ローカルLLM  │
└─────────────┘     └──────────────┘     └──────────────┘

① 音声をテキストに変換（Amical）
② テキストをFabricのパターンでプロンプトとして整形
③ Ollamaのローカルモデルで推論・出力
```

---

## 前提条件

- Windows 10/11 + WSL2（Ubuntu）がセットアップ済み
- Ollamaが **Windows側** にインストール済み
- マイクが使用可能

---

## Step 1: Ollamaにモデルをダウンロード

Windows側のターミナル（PowerShell or コマンドプロンプト）で以下を実行します。

```powershell
# Qwen3 14B（メイン用、高品質）
ollama pull qwen3:14b

# Qwen3 8B（軽量・高速）
ollama pull qwen3:8b

# Gemma3 12B（Google製、バランス型）
ollama pull gemma3:12b
```

ダウンロード完了後、モデル一覧を確認します。

```powershell
ollama list
```

`qwen3:14b`、`qwen3:8b`、`gemma3:12b` が表示されればOKです。

### Ollamaの起動確認

Ollamaは通常、Windowsにインストールすると自動的にバックグラウンドで起動します。タスクトレイにOllamaのアイコンがあるか確認してください。

動作確認として、PowerShellで以下を実行します。

```powershell
curl http://localhost:11434/api/tags
```

モデル一覧がJSON形式で返ってくれば、Ollamaは正常に動作しています。

---

## Step 2: WSL2からOllamaへの接続確認

WSL2のターミナルを開き、Windows側のOllamaに接続できるか確認します。

### 2-1. Windows側のIPアドレスを取得

```bash
# WSL2からWindows側のIPを取得
WIN_HOST=$(cat /etc/resolv.conf | grep nameserver | awk '{print $2}')
echo $WIN_HOST
```

表示されたIPアドレス（例: `172.xx.xx.1`）がWindows側のアドレスです。

### 2-2. 接続テスト

```bash
curl http://$WIN_HOST:11434/api/tags
```

もしここで接続できない場合、Windows側のOllamaが全インターフェースでリッスンするように設定します。

#### 接続できない場合の対処法

1. Windows側でシステム環境変数を追加します
   - 変数名: `OLLAMA_HOST`
   - 値: `0.0.0.0`
2. Ollamaを再起動します（タスクトレイのアイコンを右クリック → 終了 → 再度起動）
3. 再度WSL2から接続テストを実行します

### 2-3. 環境変数の永続化（WSL2側）

接続先を毎回入力しなくて済むように、`.bashrc` に追加します。

```bash
echo 'export OLLAMA_HOST=$(cat /etc/resolv.conf | grep nameserver | awk '"'"'{print $2}'"'"')' >> ~/.bashrc
echo 'export OLLAMA_URL="http://$OLLAMA_HOST:11434"' >> ~/.bashrc
source ~/.bashrc
```

---

## Step 3: Fabricのインストール（WSL2側）

FabricはGo言語で書かれており、`go install` でインストールします。

### 3-1. Goのインストール

```bash
# 最新のGoバージョンを確認: https://go.dev/dl/
# 2025年2月時点では 1.24.x が最新安定版

wget https://dl.google.com/go/go1.24.0.linux-amd64.tar.gz

# 既存のGoがあれば削除してからインストール
sudo rm -rf /usr/local/go
sudo tar -C /usr/local -xzf go1.24.0.linux-amd64.tar.gz
rm go1.24.0.linux-amd64.tar.gz

# パスを設定
echo 'export GOROOT=/usr/local/go' >> ~/.bashrc
echo 'export GOPATH=$HOME/go' >> ~/.bashrc
echo 'export PATH=$GOPATH/bin:$GOROOT/bin:$PATH' >> ~/.bashrc
source ~/.bashrc

# インストール確認
go version
```

`go version go1.24.0 linux/amd64` のように表示されればOKです。

### 3-2. Fabricのインストール

```bash
go install github.com/danielmiessler/fabric/cmd/fabric@latest
```

インストール後、バージョンを確認します。

```bash
fabric --version
```

### 3-3. Fabricの初期設定

```bash
fabric --setup
```

対話式のセットアッププロセスが始まります。以下のように進めてください。

1. **APIキーの設定**: OpenAI等のAPIキーを持っていない場合は、スキップして構いません（EnterやスキップのNumberを選んで進む）
2. **Ollama の設定**: セットアップの中で「Ollama」に関する項目が出てきたら、設定します
   - Ollamaのサーバーアドレスは `http://<Windows側IP>:11434` を指定します
   - `<Windows側IP>` は Step 2-1 で確認した値です
3. **デフォルトモデルの設定**: `qwen3:14b` をデフォルトに設定するのがおすすめです
4. **パターンの更新**: セットアップ完了後、パターンをダウンロードします

```bash
fabric --updatepatterns
```

### 3-4. モデル一覧の確認

```bash
fabric --listmodels
```

Ollamaのモデル（`qwen3:14b`, `qwen3:8b`, `gemma3:12b`）が表示されていれば、FabricとOllamaの連携は成功です。

### 3-5. Fabricの動作テスト

```bash
echo "人工知能は今後どのように発展するか" | fabric --pattern improve_prompt --model qwen3:14b
```

テキストが整形されて出力されれば成功です。

---

## Step 4: Amicalのインストール（Windows側）

### 4-1. ダウンロードとインストール

1. [Amical Releases](https://github.com/amicalhq/amical/releases/latest) ページにアクセスします
2. Windows用のインストーラー（`.exe`）をダウンロードします
3. ダウンロードしたインストーラーを実行します
4. Windows SmartScreenの警告が出た場合は「詳細情報」→「実行」をクリックします

### 4-2. 初回起動と設定

1. Amicalを起動します
2. マイクのアクセス許可を求められた場合は「許可」をクリックします
3. Whisperモデルのダウンロード
   - Amicalはアプリ内からワンクリックでローカルのWhisperモデルをダウンロードできます
   - 設定画面で「Local Models」→ ダウンロードしたいモデルを選択します
   - 日本語の精度を重視する場合は、`large` または `large-v3-turbo` モデルを推奨します
4. 言語設定
   - 設定から言語を「Japanese / 日本語」に設定します

### 4-3. Amicalの基本的な使い方

- **ホットキー（デフォルト）** を押して録音開始
- 話し終わったら再度ホットキーを押して録音停止
- Amicalが音声をテキストに変換し、**アクティブなアプリケーションに直接入力**します
- テキストは **クリップボード** にもコピーされます

---

## Step 5: パイプラインの接続テスト

ここが最も重要なステップです。音声認識 → プロンプト整形 → LLM推論のパイプライン全体をテストします。

### 方式A: クリップボード経由（最も簡単）

Amicalはテキストをクリップボードにコピーする機能があるため、WSL2側で `powershell.exe` を使ってクリップボードを取得できます。

#### テスト手順

1. **Amicalで音声入力する**
   - Amicalのホットキーを押して、例えば以下のように話します:
   - 「Reactでログインフォームのコンポーネントを作成してください。メールアドレスとパスワードのバリデーション付きで」
   - ホットキーを再度押して録音停止

2. **WSL2のターミナルでクリップボードを取得してFabricに渡す**

```bash
# Windows側のクリップボードを取得してFabricに渡す
powershell.exe Get-Clipboard | fabric --pattern improve_prompt --model qwen3:14b
```

3. **結果を確認する**
   - Fabricの `improve_prompt` パターンによって整形されたプロンプトが出力されます

#### 別のパターンを試す

Fabricには多数のパターン（プロンプトテンプレート）が用意されています。

```bash
# 利用可能なパターン一覧を確認
fabric --listpatterns

# 代表的なパターンの例:

# プロンプトの改善
powershell.exe Get-Clipboard | fabric --pattern improve_prompt --model qwen3:14b

# テキストの要約
powershell.exe Get-Clipboard | fabric --pattern summarize --model qwen3:8b

# コードの生成
powershell.exe Get-Clipboard | fabric --pattern write_essay --model gemma3:12b

# アイデアの抽出
powershell.exe Get-Clipboard | fabric --pattern extract_wisdom --model qwen3:14b
```

### 方式B: テキストファイル経由

クリップボード方式がうまくいかない場合のバックアップ手順です。

1. **Amicalで音声入力する**（方式Aと同じ）

2. **テキストエディタにペーストして保存する**
   - Windows側のテキストエディタ（メモ帳等）にAmicalの出力をペースト
   - WSL2からアクセスできるパスに保存（例: `C:\Users\<ユーザー名>\voice_input.txt`）

3. **WSL2側でFabricに渡す**

```bash
cat /mnt/c/Users/<ユーザー名>/voice_input.txt | fabric --pattern improve_prompt --model qwen3:14b
```

### 方式C: ワンライナースクリプト化

毎回コマンドを打つのが面倒な場合は、シェルスクリプトにまとめます。

```bash
# ~/.local/bin/voice-prompt.sh として保存
cat << 'EOF' > ~/.local/bin/voice-prompt
#!/bin/bash
# 音声入力テキストをFabricで整形するスクリプト

PATTERN=${1:-improve_prompt}
MODEL=${2:-qwen3:14b}

echo "📋 クリップボードからテキストを取得中..."
INPUT=$(powershell.exe Get-Clipboard 2>/dev/null | tr -d '\r')

if [ -z "$INPUT" ]; then
    echo "❌ クリップボードが空です。Amicalで音声入力してください。"
    exit 1
fi

echo "🎤 入力テキスト:"
echo "---"
echo "$INPUT"
echo "---"
echo ""
echo "🔄 パターン '$PATTERN' で整形中（モデル: $MODEL）..."
echo ""

echo "$INPUT" | fabric --pattern "$PATTERN" --model "$MODEL"
EOF

chmod +x ~/.local/bin/voice-prompt
```

#### 使い方

```bash
# デフォルト（improve_prompt + qwen3:14b）
voice-prompt

# パターンを指定
voice-prompt summarize

# パターンとモデルを指定
voice-prompt improve_prompt gemma3:12b
```

---

## Step 6: End-to-End テスト（最終確認）

以下の手順で、パイプライン全体が動作することを確認します。

### テストシナリオ 1: プロンプト整形

1. **Amical** でホットキーを押し、以下のように話す:
   > 「Next.jsでユーザー認証機能を実装したい。JWTトークンを使って、ログインとサインアップの画面を作りたい。」

2. ホットキーを押して録音停止

3. **WSL2ターミナル** で実行:
   ```bash
   powershell.exe Get-Clipboard | fabric --pattern improve_prompt --model qwen3:14b
   ```

4. **期待される出力**: 元の口語的な指示が、構造化された明確なプロンプトに変換される

### テストシナリオ 2: 要約

1. **Amical** で何か長めの内容を話す

2. **WSL2ターミナル** で実行:
   ```bash
   powershell.exe Get-Clipboard | fabric --pattern summarize --model qwen3:8b
   ```

3. **期待される出力**: 話した内容が簡潔にまとめられる

### テストシナリオ 3: モデル比較

同じ入力に対して異なるモデルで結果を比較します。

```bash
# クリップボードの内容を一時ファイルに保存
powershell.exe Get-Clipboard | tr -d '\r' > /tmp/voice_input.txt

# Qwen3 14B
echo "=== Qwen3 14B ==="
cat /tmp/voice_input.txt | fabric --pattern improve_prompt --model qwen3:14b
echo ""

# Qwen3 8B
echo "=== Qwen3 8B ==="
cat /tmp/voice_input.txt | fabric --pattern improve_prompt --model qwen3:8b
echo ""

# Gemma3 12B
echo "=== Gemma3 12B ==="
cat /tmp/voice_input.txt | fabric --pattern improve_prompt --model gemma3:12b
```

---

## トラブルシューティング

### Fabricがモデルを認識しない

```bash
# Ollamaの接続を確認
curl http://$OLLAMA_HOST:11434/api/tags

# Fabricのセットアップをやり直す
fabric --setup
```

### WSL2からOllamaに接続できない

```bash
# Windows側のIPを確認
cat /etc/resolv.conf

# ファイアウォールの確認（Windows側のPowerShellで実行）
# ポート11434が許可されているか確認
netsh advfirewall firewall show rule name=all | findstr 11434
```

Windows Defenderファイアウォールでポート11434をブロックしている可能性があります。管理者権限のPowerShellで以下を実行してください。

```powershell
netsh advfirewall firewall add rule name="Ollama" dir=in action=allow protocol=TCP localport=11434
```

### Fabricのパターン一覧が空

```bash
fabric --updatepatterns
```

### 日本語の音声認識精度が低い

- Amicalの設定で、より大きなWhisperモデル（`large-v3` や `large-v3-turbo`）に変更してください
- カスタム辞書（Custom Vocabulary）にプログラミング用語を追加してください
  - 例: React, Next.js, TypeScript, コンポーネント, etc.

---

## 便利なFabricパターン（開発向け）

| パターン名 | 用途 |
|---|---|
| `improve_prompt` | 口語テキストを整形されたプロンプトに変換 |
| `summarize` | テキストの要約 |
| `extract_wisdom` | テキストからアイデアや知見を抽出 |
| `write_essay` | テーマに基づいてエッセイ・文章を生成 |
| `create_coding_project` | コーディングプロジェクトの骨格を生成 |
| `explain_code` | コードの説明を生成 |
| `improve_writing` | 文章の改善・推敲 |
| `create_summary` | 構造化された要約を生成 |

カスタムパターンを自分で作成することも可能です。

```bash
# パターンの保存場所を確認
ls ~/.config/fabric/patterns/

# カスタムパターンの作成例
mkdir -p ~/.config/fabric/patterns/voice_to_dev_prompt
cat << 'EOF' > ~/.config/fabric/patterns/voice_to_dev_prompt/system.md
# IDENTITY and PURPOSE

あなたは、ソフトウェア開発者の音声入力を受け取り、
Claude CodeやChatGPT等のLLMに渡すための高品質なプロンプトに変換する専門家です。

# STEPS

1. 入力された口語テキストを分析する
2. 開発タスクの要件を抽出する
3. 技術スタックや制約条件を特定する
4. 構造化されたプロンプトに整形する

# OUTPUT FORMAT

以下の形式で出力してください:

## タスク概要
[1-2文でタスクを簡潔に説明]

## 要件
- [箇条書きで要件を列挙]

## 技術スタック
- [使用する技術を列挙]

## 詳細な指示
[LLMに渡すための具体的な指示文]
EOF
```

---

## Step 7: 簡略化フロー（Fabric不要）

上記Step 1〜6のフロー（Amical → Fabric → Ollama）は動作しますが、Fabricコマンドの手打ちがボトルネックになります。
以下の2つの方式を組み合わせることで、**Fabricを経由せずに**音声入力から直接開発作業に入れます。

### 方式A: Amical + Ollama 直接連携

AmicalにはOllama連携機能が組み込まれており、音声認識結果を自動的にOllamaで整形してアクティブウィンドウにペーストできます。

#### 設定手順

1. **Ollamaが起動していることを確認**（Step 1参照）

2. **Amicalの設定画面を開く**
   - Settings > AI Models > **Language** タブ

3. **Ollamaプロバイダーを接続**
   - Ollamaのアコーディオンを展開
   - URL欄に `http://localhost:11434` を入力
   - 「Connect」ボタンをクリック → ステータスが緑になればOK
   - 「Sync Models」をクリック → Ollamaのモデル一覧が同期される

4. **デフォルトモデルを選択**
   - 同期されたモデル一覧から `qwen3:14b` 等を選択

5. **フォーマッティングを有効化**
   - Settings > Dictation > **Formatting Settings**
   - フォーマッティングのトグルをONにする
   - 使用するモデルを選択

#### 使い方

1. Claude Codeのターミナルをアクティブにする
2. Amicalのホットキーを押して音声入力
3. ホットキーを再度押して録音停止
4. Amicalが自動的に：音声認識 → Ollamaで整形 → **アクティブウィンドウ（Claude Code）に直接ペースト**

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Amical    │ ──→ │   Whisper    │ ──→ │   Ollama     │ ──→ アクティブウィンドウ
│  音声入力    │     │  音声認識     │     │  テキスト整形  │     に自動ペースト
└─────────────┘     └──────────────┘     └──────────────┘
         ※ Fabricコマンドの手打ち不要！
```

#### 注意点

- Amicalの整形プロンプトは「句読点追加・フィラーワード除去・読みやすい文章化」が目的です
- 開発プロンプトへの変換（Fabricの`improve_prompt`相当）は行いません
- ただしClaude Codeは口語的なテキストでも意図を正確に解釈できるため、問題ありません
- カスタムプロンプト機能は今後のバージョンで実装予定（[Issue #55](https://github.com/amicalhq/amical/issues/55)）

#### カスタム語彙の登録（推奨）

Settings > Vocabulary から開発用語を登録すると、音声認識と整形の精度が向上します。

登録推奨の用語例:
- フレームワーク: React, Next.js, TypeScript, Tailwind CSS
- ツール: Claude Code, Ollama, Docker, WSL2
- 概念: コンポーネント, API, エンドポイント, リファクタリング
- コマンド: git, npm, pip, cargo

---

### 方式C: Claude Code `/voice` カスタムコマンド

方式Aでアクティブウィンドウへの自動ペーストがうまくいかない場合や、
クリップボード経由で確実に入力したい場合に使います。

#### セットアップ

`.claude/commands/voice.md` を以下の内容で作成済みです:

```markdown
以下はクリップボードから取得した音声入力テキストです:

!`powershell.exe Get-Clipboard 2>/dev/null | tr -d '\r'`

上記は音声認識(Amical)で入力された口語的なテキストです。
以下のルールに従って処理してください:

1. 口語的な表現や曖昧な指示を正確に解釈する
2. 開発タスクとして意図を理解し、そのまま実行に移す
3. 不明点がある場合のみ確認し、それ以外は直接作業を開始する
```

#### 使い方

1. Amicalで音声入力する（テキストがクリップボードにコピーされる）
2. Claude Codeで `/voice` と入力してEnter

```
┌─────────────┐     ┌──────────────┐     ┌──────────────┐
│   Amical    │ ──→ │ クリップボード │ ──→ │ Claude Code  │
│  音声入力    │     │  自動コピー    │     │  /voice で   │
│             │     │              │     │  直接解釈・実行│
└─────────────┘     └──────────────┘     └──────────────┘
         ※ Fabric不要、Ollama整形も不要！
         ※ Claude Code自身が口語テキストを理解
```

#### メリット

- `/voice` の6文字を打つだけ
- FabricもOllamaによる中間整形も不要（Claude Code自身がLLM）
- クリップボード経由なので確実に動作する

---

### 方式の使い分け

| 状況 | 推奨方式 |
|---|---|
| Claude Codeのターミナルがアクティブ | 方式A（完全ハンズフリー） |
| 自動ペーストがうまくいかない場合 | 方式C（`/voice`で確実に入力） |
| 複雑な開発指示を出したい場合 | 方式C（Claude Codeが直接解釈） |
| Claude Code以外のツールに入力したい場合 | 方式A（任意のアクティブウィンドウに対応） |

---

## まとめ

### 現在のフロー（Step 1〜6: Fabric経由）

| ツール | 役割 | 動作環境 |
|---|---|---|
| Amical | 音声 → テキスト変換 | Windows（デスクトップアプリ） |
| Fabric | テキスト → プロンプト整形 | WSL2 |
| Ollama | プロンプト → LLM推論 | Windows（WSL2からアクセス） |

### 簡略化フロー（Step 7: Fabric不要）

| ツール | 役割 | 動作環境 |
|---|---|---|
| Amical | 音声 → テキスト → Ollama整形 → 自動ペースト | Windows |
| Ollama | Amicalからのテキスト整形 | Windows |
| Claude Code | `/voice` で口語テキストを直接解釈・実行 | WSL2 |

簡略化フローでは**Fabricコマンドの手打ちが完全に不要**になり、音声入力から開発作業開始までのステップが大幅に削減されます。
