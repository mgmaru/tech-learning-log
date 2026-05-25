# Claude Code のモデル自動選択 — 現状と実用的なアプローチ

## 概要

Claude Code は Haiku / Sonnet / Opus の3つのモデルを使い分けられる。タスクに応じてモデルを切り替えれば、**コストを3〜4倍削減できる**ことが知られている。しかし「タスク内容を見て自動的に最適なモデルを選ぶ」という機能は、**現時点で標準提供されていない**。

本ドキュメントは、現状の課題と、実用的な代替アプローチ、そして今後の見通しを整理する。

---

## 1. なぜモデル選択が重要か

3つのモデルには明確な性格の違いがある。

| モデル | コスト | 速度 | 向いているタスク |
|---|---|---|---|
| Haiku | 安い | 速い | ファイル読み取り、grep、定型処理、フォーマット修正 |
| Sonnet | 中 | 中 | 標準的な実装、バグ修正、PRレビュー |
| Opus | 高い | 遅い | アーキテクチャ判断、複雑なリファクタリング |

実際の使用データでは、**Opusを使うかSonnetを使うかで3〜4倍のコスト差**が出ることが報告されている。

軽いタスクにOpusを使うのは無駄、重いタスクにHaikuを使うと品質が落ちる。**タスクに応じた選択が、品質とコストの両立に直結する**。

---

## 2. 現状の課題 — 標準では自動選択できない

Claude Code 公式には、現時点で **自動モデル選択機能はない**。

```
現状:
  ユーザーが /model コマンドで手動切り替え
  または settings.json でデフォルトを固定
        ↓
  全てのタスクが同じモデルで処理される
        ↓
  軽いタスクにOpusを使うとトークン浪費
  重いタスクにHaikuを使うと品質低下
```

これはコミュニティで **最もリクエストの多い機能カテゴリ** で、関連Issueが30件以上立てられている。Anthropicも認識しており、改善が議論されている領域。

### 公式が試した一つの解決策 — opusplan

`opusplan` という機能があり、「計画段階はOpus、実装段階はSonnet」という分担をある程度自動化する仕組み。ただし:

- ルーティングの粒度が粗い(plan vs non-plan の2分類のみ)
- 現状は動作に問題があると報告されている

完全には頼れない状態。

---

## 3. 実用的な5つのアプローチ

公式機能を待たずに実現する方法。

```
┌──────────────────────────────────────────────┐
│      モデル選択を実現する5つの方法             │
├──────────────────────────────────────────────┤
│                                              │
│  ① ジョブ分割パターン(GitHub Action向け)    │
│  ② Skillフロントマターでの指定(将来対応)    │
│  ③ Hook による動的切り替え(ローカル向け)    │
│  ④ Plan/Execute パターン(opusplan)         │
│  ⑤ サブエージェント設計                      │
│                                              │
└──────────────────────────────────────────────┘
```

それぞれの特徴を簡単に。

### ① ジョブ分割パターン(現時点の本命)

GitHub Action で、**トリガー条件ごとに異なるジョブを定義**し、各ジョブで使うモデルを指定する方法。

```yaml
jobs:
  # 軽量レビュー: Haiku
  quick-review:
    if: contains(github.event.pull_request.labels.*.name, 'chore')
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: claude-haiku-4-5-20251001

  # 通常タスク: Sonnet
  standard-task:
    if: github.event_name == 'pull_request'
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: claude-sonnet-4-6

  # 設計判断: Opus
  architecture:
    if: contains(github.event.comment.body, '@claude architect')
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: claude-opus-4-7
```

**利点**:

| 利点 | 内容 |
|---|---|
| 明示的 | どのトリガーでどのモデルが動くか一目瞭然 |
| 安定 | 公式機能のみで実現、動作が確実 |
| 段階的 | 最初は1ジョブから始めて拡張可能 |
| 予測可能 | コスト感が事前に分かる |

**欠点**: 完全自動ではない(条件を人間が設計する必要がある)

### ② Skill フロントマターでのモデル指定(将来)

提案中の機能。Skillの定義時にモデルも指定できるようにする。

```markdown
---
name: complex-refactoring
description: Used when major refactoring spans multiple files
model: claude-opus-4-7    ← Skill単位で指定
---
```

これが実装されると、**Skill選択(タスクの種類の判定)とモデル選択が同時に行われる** ようになる。

期待度は高いが、**現時点では未実装**。anthropics/claude-code のIssue #23462で議論中。

### ③ Hook による動的切り替え(ローカル向け)

ローカルのClaude Code限定。コミュニティ製のHookシステムを使う方法。

例: `tzachbon/claude-model-router-hook`

- プロンプトを複雑度で自動分類
- アクティブモデルを自動切り替え
- キーワードとパターンマッチングで判定(API呼び出しゼロ)
- プロンプト先頭に `~` で分類をバイパス可能

**仕組み**:

```
ユーザー入力: "fix the typo in README"
   ↓
UserPromptSubmit hook が起動
   ↓
キーワード分類: 軽量タスク
   ↓
settings.json の model を haiku に切り替え
   ↓
Claude Code が Haiku で処理
```

**注意**: GitHub Actionでは使えない。ローカル実行のClaude Code限定。

### ④ Plan / Execute パターン

opusplan の発想を応用したパターン。「考える段階はOpus、実装はSonnet」という分担。

```
Step 1: Plan(Opus)
   → アーキテクチャ判断、タスク分解、設計

Step 2: Execute(Sonnet)
   → 計画に従って実装
```

実例として、典型的なコーディングセッションで **3〜4倍のコスト削減** を達成したとの報告がある。

公式の opusplan は現状不安定だが、自分でジョブを分けて手動でこのパターンを実装することは可能。

### ⑤ サブエージェント設計

複雑なタスクを **メインエージェント + サブエージェント** に分割する設計。

```
メイン: Opus
   ├─ アーキテクチャ判断
   ├─ タスク分解
   │
   ├─ サブ1: Haiku → ファイル読み込み、grep
   ├─ サブ2: Sonnet → 実装
   └─ サブ3: Sonnet → テスト作成
```

「考える人(Opus)」と「動く人(Haiku/Sonnet)」を分けることで、全体コストを抑えながら品質を維持する。

---

## 4. Hiroakiさん向けの実践レシピ

個人開発のGitHub Action文脈で、最も実用的なのは **①のジョブ分割パターン**。段階的に育てる方法を示す。

### ステップ1: 最小構成から始める

```yaml
name: Claude Code

on:
  pull_request:
    types: [opened, synchronize]

jobs:
  auto-review:
    runs-on: ubuntu-latest
    timeout-minutes: 10
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          claude_model: claude-sonnet-4-6
```

まずSonnet一本で運用してコスト感を掴む。

### ステップ2: 軽量タスクをHaikuに振り分け

```yaml
  format-check:
    if: contains(github.event.pull_request.labels.*.name, 'chore')
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: claude-haiku-4-5-20251001
```

`chore` ラベルが付いたPRはHaikuで安く処理。

### ステップ3: 重要タスクをOpusに

```yaml
  architecture:
    if: |
      contains(github.event.pull_request.labels.*.name, 'architecture') ||
      contains(github.event.comment.body, '@claude architect')
    timeout-minutes: 20
    steps:
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: claude-opus-4-7
```

設計判断が必要なケースだけOpusを起動。

### ラベル戦略の例

FrameWeaver のような複雑なプロジェクトでは、ラベルベースの制御が有効。

| ラベル | 起動ジョブ | モデル |
|---|---|---|
| `chore` | format-check | Haiku |
| `bug` | bug-fix | Sonnet |
| `feature` | feature-impl | Sonnet |
| `architecture` | architecture | Opus |
| `error-handling` | error-design | Opus |

エラーハンドリング設計みたいな重要領域に自動的にOpusが当たる仕組みを最初から作っておくと、品質とコストのバランスが取れる。

---

## 5. キーワードベースの簡易ルーター(おまけ)

「ラベルを付けるのも面倒」という場合、PRタイトルやコメント内容から自動判定する簡易ルーターも作れる。

```yaml
jobs:
  route-and-execute:
    runs-on: ubuntu-latest
    steps:
      - name: Determine model
        id: model
        run: |
          CONTENT="${{ github.event.comment.body }}${{ github.event.pull_request.title }}"
          if echo "$CONTENT" | grep -iE "refactor|architect|design"; then
            echo "model=claude-opus-4-7" >> $GITHUB_OUTPUT
          elif echo "$CONTENT" | grep -iE "typo|format|rename"; then
            echo "model=claude-haiku-4-5-20251001" >> $GITHUB_OUTPUT
          else
            echo "model=claude-sonnet-4-6" >> $GITHUB_OUTPUT
          fi
      
      - uses: anthropics/claude-code-action@v1
        with:
          claude_model: ${{ steps.model.outputs.model }}
```

完璧ではないが、明らかなパターンには十分対応できる。

---

## 6. 今後の見通し

この領域は急速に動いている。期待される進展:

| 機能 | 状態 | 期待度 |
|---|---|---|
| Skill frontmatter での model 指定 | 提案中(#23462) | 高 |
| 自動モデルルーティング | 議論中(#27665等) | 中 |
| opusplan の改善 | 議論中 | 中 |
| サブエージェントのデフォルトSonnet化 | 提案中(#26179) | 高(小さな変更) |

特に **Skill frontmatter でのモデル指定** は実装の動きがあり、近いうちに実現される可能性が高い。これが入ると、**「タスクの種類をSkillで判定 → そのSkillの推奨モデルが自動選択」** という流れになり、今のジョブ分割パターンよりも自然な自動化が可能になる。

進捗を追いたい場合は anthropics/claude-code のIssueトラッカーを時々確認するとよい。

---

## 7. まとめ

| 観点 | 答え |
|---|---|
| 公式の自動モデル選択は? | **現時点でなし**(最多リクエスト機能) |
| 現実的な解決策 | **GitHub Action のジョブ分割パターン** |
| ローカル向けの解決策 | **コミュニティ製の Hook システム** |
| 将来の本命 | **Skill frontmatter でのモデル指定** |
| コスト削減効果 | **3〜4倍**(典型的なケース) |

「完全自動」はまだ来ていないが、**「明示的な条件で半自動」は十分に実現可能**で、効果も大きい。

Hiroakiさんの場合、まずは **PRラベルベースの簡単な分岐** から始めるのが現実的。運用しながら「どのケースでOpusが必要か」「どのケースでHaikuで十分か」を学習し、ルールを育てていけば、無理なくコスト最適化された運用に近づける。

FrameWeaverのエラーハンドリング設計のような **重要領域には自動でOpusが当たる** 仕組みを最初に整備しておくと、品質を担保しつつ全体のコストを抑えられる。
