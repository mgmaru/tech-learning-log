# Claude Code の設定ファイル：`settings.json` と `settings.local.json`

Claude Code は複数のスコープに分かれた設定ファイルを起動時に読み込み、それらを**マージ**して最終的な設定を決定する。本ドキュメントでは特に `settings.json`（プロジェクト）と `settings.local.json`（ローカル）の**使い分け**と**優先順位**を整理する。

---

## 1. 使い分け（最重要）

この2ファイルの役割を混同しないことが、チーム運用とセキュリティの両面で重要になる。

### `settings.json`（プロジェクト設定）

- **パス:** `<project-root>/.claude/settings.json`
- **目的:** チームで**共有**したい設定
- **Git:** コミットする（ソース管理にチェックイン）
- **置くもの:**
  - チーム共通の権限ルール（permissions）
  - 共有フック（hooks）
  - 必須の MCP サーバー設定
  - プロジェクト共通のモデル指定
- **⚠️ 注意:** リポジトリに入るファイルなので、**シークレット（API キー等）は絶対に置かない**

### `settings.local.json`（ローカル設定）

- **パス:** `<project-root>/.claude/settings.local.json`
- **目的:** そのリポジトリ内での**自分専用**の上書き
- **Git:** コミットしない（Claude Code が自動的に `.gitignore` 対象に設定する）
- **置くもの:**
  - API キーやシークレット
  - マシン固有のパス
  - 実験的な権限・フック・MCP 設定（共有前のステージング用途）
  - チームのモデル選択を個人的に上書きしたい場合

### まとめ表

| 観点 | `settings.json`（プロジェクト） | `settings.local.json`（ローカル） |
| --- | --- | --- |
| スコープ | チーム共有 | 個人・このリポジトリ限定 |
| Git | コミットする | コミットしない（自動 gitignore） |
| シークレット | **置かない** | 置いてよい |
| 主な用途 | 共通の権限・フック・MCP・モデル | 個人の上書き・実験・マシン固有設定 |

> **使い分けの原則:** 「チーム全員で共有すべきか」で判断する。共有するなら `settings.json`、自分だけ／このマシンだけなら `settings.local.json`。

---

## 2. 優先順位（precedence）

Claude Code は起動時に全ファイルを読み込んでマージする。同じキーが複数スコープに存在する場合、より優先度の高いスコープが勝つ。

完全な優先順位チェーン（高い順）：

```
managed settings  >  CLI フラグ  >  local settings  >  project settings  >  user settings
（組織強制）          （--model 等）   （settings.local） （settings.json）    （~/.claude/settings.json）
```

今回の2ファイルに絞ると：

```
settings.local.json  >  settings.json
（local が勝つ）          （project）
```

さらにグローバルなユーザー設定 `~/.claude/settings.json`（user スコープ）が最も弱い基盤として存在する。

### 各スコープの定義

| スコープ | パス | 適用範囲 |
| --- | --- | --- |
| user（最弱） | `~/.claude/settings.json` | 全プロジェクト共通のベースライン |
| project | `<project-root>/.claude/settings.json` | そのプロジェクト（チーム共有） |
| local（強い） | `<project-root>/.claude/settings.local.json` | そのプロジェクトの個人上書き |
| CLI フラグ | `claude --model opus` 等 | そのセッション限り |
| managed（最強） | 組織が Teams / Enterprise で配布 | 個別ファイルでは上書き不可 |

> Windows では `~/.claude` は `%USERPROFILE%\.claude` に解決される。

---

## 3. マージのされ方が項目で異なる（重要な落とし穴）

設定は単純に「上書き」されるとは限らない。キーの種類によって挙動が違う。

### スカラー値 → override（上書き）

モデル指定や各種フラグなど単一値のキーは、優先度の高いスコープが勝つ。

- 例: user で `spinnerTipsEnabled: true`、project で `false` → **project の `false` が適用**

### 権限ルール（permissions 配列）→ merge（マージ）

権限の配列は上書きではなく、スコープ間で**結合・重複排除**される。

- 例: user が `Bash(git status)` を allow、project が `Bash(npm run lint)` を allow → **両方が有効**

### セキュリティ上の注意

この merge 挙動が落とし穴になる。

- プロジェクトの `settings.json` に `deny` ルールを置いても、開発者は誰でも `settings.local.json` を作って上書きできる（local が project に勝つため）。
- **セキュリティ上重要な deny は `managed settings`（組織配布）に置く必要がある。** project レベルの deny は「ロックした」つもりでも、実際にはロックになっていない。

---

## 4. どの設定がどこから来たか確認する

迷ったら以下のコマンドで確認する。

- `/status` — 現在のセッションで読み込まれている**設定ソースの一覧**を表示する
  - ただし「どのソースが読まれたか」を示すだけで、**個々のキーをどのレイヤーが供給したか**までは表示しない
  - 設定ファイルに JSON エラーやバリデーションエラーがあれば、ここで報告される
- `/permissions` — 有効な全権限ルールを確認
- `/config` — 設定変更時に書き込み先（user / project / local）を選択できる

---

## 5. 実運用の指針（例: 個人プロジェクト）

公開・チーム共有を想定するリポジトリでの基本的な切り分け：

- **`settings.json`（コミットする）**
  - プロジェクト共通の権限ルール、共有フック、必須 MCP、モデル指定
- **`settings.local.json`（コミットしない）**
  - 自分のマシン固有のパス、API キー、試験中の権限・MCP 設定

---

## 参考リンク

- Claude Code 設定ドキュメント（公式）: https://code.claude.com/docs/en/settings
