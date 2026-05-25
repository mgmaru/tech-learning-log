# CODEOWNERS の基本 — 学習ノート

## 1. CODEOWNERSとは何か

CODEOWNERSは、**GitHub(およびGitLab等)の標準機能**で、リポジトリ内の特定のファイルやディレクトリに対する「所有者(レビュー責任者)」を宣言する仕組み。

ひとことで言うと、**「Gitリポジトリにレビュー責任の地図を貼る」** ためのファイル。

### なぜ必要か

複数人で開発するリポジトリでは、こんな問題が起きる:

- 「このファイルのレビュー、誰に頼めばいい?」が毎回不明
- 重要なファイルが誰にも見られずにマージされる
- 「自分の領域じゃないのに勝手に変更された」というすれ違い
- 新メンバーが「誰に聞けばいいか分からない」状態になる

CODEOWNERSはこれらを **コードベース上で明示化** することで解決する。誰が何の責任を持つかが、ドキュメントではなくリポジトリのファイルとして残る点が重要。

### 単独では強制力がないことに注意

CODEOWNERSファイル単体では「レビュアーが自動アサインされる」だけで、**マージを止める強制力はない**。強制力を持たせるにはGitHubの「ブランチ保護ルール」と組み合わせる必要がある。これは後述する。

---

## 2. ファイルの置き場所

`CODEOWNERS` という名前(拡張子なし)のファイルを、リポジトリ内の以下のいずれかに配置する:

```
your-repo/
├── CODEOWNERS              ← ルート直下(よく使われる)
├── docs/CODEOWNERS         ← docs/ 直下
└── .github/CODEOWNERS      ← .github/ 直下(推奨)
```

複数の場所に置くこともできるが、混乱を避けるため**1箇所だけ**にするのが推奨。一般的には `.github/CODEOWNERS` が最も多い(GitHub関連の設定ファイルが集まる場所だから)。

---

## 3. 基本的な書き方

### 構文の基本

各行は **「パターン スペース 所有者」** の形式で書く:

```
<パターン>    <所有者1> <所有者2> ...
```

- パターンは `.gitignore` と似た形式
- 所有者は `@ユーザー名` または `@組織名/チーム名` の形式
- `#` で始まる行はコメント
- 空行は無視される

### 最小の例

```
# 全ファイルのデフォルト所有者
*    @hiroaki
```

これだけで、リポジトリ内のすべてのファイル変更時に `@hiroaki` がレビュアーに自動アサインされる。

### もう少し実用的な例

```
# デフォルト(他のルールにマッチしないファイル全部)
*                           @hiroaki

# Rustコードはrust-team
*.rs                        @your-org/rust-team

# ドキュメントはdocs-team
/docs/                      @your-org/docs-team

# CI設定はインフラ担当
/.github/workflows/         @your-org/devops-team
```

---

## 4. パターンの書き方詳細

`.gitignore` と似ているが、**完全に同じではない**点に注意。

### 4.1 マッチングの種類

| パターン | 意味 | 例 |
|---|---|---|
| `*` | すべてのファイル | `*    @hiroaki` |
| `*.js` | 拡張子jsのファイル全部 | `*.js    @frontend-team` |
| `/path/` | 末尾スラッシュ = ディレクトリ全体 | `/skills/    @skills-team` |
| `/path` | 末尾スラッシュなし = ファイル名 | `/README.md    @hiroaki` |
| `path/file.md` | パス指定 | `docs/api.md    @api-team` |
| `**/foo/` | 任意の階層下の `foo/` ディレクトリ | `**/tests/    @qa-team` |

### 4.2 先頭スラッシュの意味

これがハマりやすいポイント:

```
# リポジトリのルート直下の docs/ だけにマッチ
/docs/    @docs-team

# 任意の階層にある docs/ にマッチ(ルートでなくてもOK)
docs/     @docs-team
```

ルート直下のディレクトリだけを指したい場合は、必ず先頭にスラッシュを付ける。

### 4.3 マッチしないケースに注意

`.gitignore` と違って、CODEOWNERSは **否定パターン(`!`)が使えない**。「これ以外を除外」みたいな書き方はできない。

```
# これはCODEOWNERSでは無効
!*.test.js    @some-team    ← エラーになる
```

---

## 5. ルールの優先順位 — 「最後にマッチしたルールが勝つ」

CODEOWNERSの最重要ルール:

> **ファイル内で後に書かれたルールが、前のルールを上書きする**

これは `.gitignore` と同じ挙動。具体例で見てみる:

```
# ルール1: 全ファイルのデフォルト
*                       @hiroaki

# ルール2: Rustファイルはrust-team
*.rs                    @rust-team

# ルール3: src/main.rs だけは特別
/src/main.rs            @hiroaki @senior-engineer
```

各ファイルの所有者は:

| ファイル | 適用ルール | 所有者 |
|---|---|---|
| `README.md` | ルール1 | `@hiroaki` |
| `src/lib.rs` | ルール2(ルール1を上書き) | `@rust-team` |
| `src/main.rs` | ルール3(ルール1と2を上書き) | `@hiroaki @senior-engineer` |

### 書き順の定石

このルールから導かれる書き方のベストプラクティス:

```
1. 一番上: デフォルト(* で全体)
2. 中間: ディレクトリ別・拡張子別
3. 一番下: 個別の特殊ルール
```

「広い範囲から狭い範囲へ」という順序で書く。逆に書くと意図通りに動かない。

```
# 悪い例: 順序が逆
/src/main.rs    @senior     ← この行は意味を失う
*               @hiroaki    ← すべてのファイルが@hiroaki扱いに
```

---

## 6. 所有者の指定方法

### 個人 vs チーム

```
# 個人を指定
*    @hiroaki

# 組織のチームを指定(推奨)
*    @your-org/skills-maintainers
```

**実運用ではチーム指定が圧倒的に推奨される**。理由は次の項で説明する。

### なぜチーム指定が望ましいか

個人指定の問題点:

| 個人指定の問題 | 詳細 |
|---|---|
| 休暇・退職リスク | その人が不在だとPRが止まる |
| 負荷集中 | レビュー依頼がその個人に集中する |
| メンテナンスコスト | 人の入れ替わりごとにCODEOWNERSを更新必要 |
| 異動への追従漏れ | 「もうそのチームじゃないのに」が発生 |

チーム指定の利点:

| チーム指定の利点 | 詳細 |
|---|---|
| 誰かが対応できる | チーム内の誰かがレビューすればOK |
| 負荷分散 | チームメンバー間で割り振り可能 |
| メンバー変更に強い | チーム所属の変更だけで済む |

組織のチームを使うには、GitHub上で組織のチーム機能を使う必要がある(個人アカウントのリポジトリだとチーム指定はできない)。

### 複数所有者の指定

スペース区切りで複数指定できる:

```
/critical/    @senior-engineer @your-org/security-team
```

この場合、**いずれか一人**のレビューがあれば要件を満たす(全員のレビューは要求されない)。

---

## 7. 何が起きるか — 機能の挙動

CODEOWNERSを配置すると、以下が自動的に起きる:

### 7.1 PR作成時のレビュアー自動アサイン

誰かがPRを作ると、**変更されたファイルにマッチするCODEOWNERSが自動的にreviewerに追加される**。

```
PR内容: src/lib.rs と docs/api.md を変更
   ↓
CODEOWNERSを参照
   ↓
自動アサイン:
  - @rust-team (src/lib.rs にマッチ)
  - @docs-team (docs/api.md にマッチ)
```

「誰にレビュー頼めばいい?」を毎回考えなくて済む。

### 7.2 レビュー必須化(ブランチ保護と組み合わせて)

GitHubの「Branch protection rules」で **「Require review from Code Owners」** を有効にすると:

- 変更されたファイルのCODEOWNERS全員の承認が必要
- 承認がないとマージボタンがグレーアウト
- 強制的に品質ゲートが機能する

### 7.3 責任所在の明示

「このディレクトリは誰の管轄か」がコードベース上で明確になる。新メンバーが「これ誰に聞けばいい?」と疑問に思ったとき、CODEOWNERSを見れば答えが分かる。

---

## 8. ブランチ保護ルールとの組み合わせ

これがCODEOWNERSを **本当に機能させる** ための重要な設定。

### 設定手順

GitHubの **Settings → Branches → Add branch protection rule** で、以下を有効にする:

```
保護するブランチ: main

✓ Require a pull request before merging
  ✓ Require approvals (1以上)
  ✓ Require review from Code Owners        ← これが重要
  ✓ Dismiss stale pull request approvals when new commits are pushed
  ✓ Require approval of the most recent reviewable push

✓ Require status checks to pass before merging
  ✓ Require branches to be up to date before merging
```

### この組み合わせで実現されること

```
┌────────────────────────────────────────┐
│  PRを作る                                │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  CODEOWNERS が自動でレビュアーを割り当て   │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  CIによる自動チェック                     │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  CODEOWNERのapproveを取得               │
└────────────────────────────────────────┘
              ↓
┌────────────────────────────────────────┐
│  全条件を満たして初めてマージ可能          │
└────────────────────────────────────────┘
```

CODEOWNERSとブランチ保護を組み合わせることで、**「誰かのレビューなしには本流にコードが入らない」状態**が作れる。

---

## 9. 実践的な書き方のコツ

実運用してて気づくポイントを整理する。

### コツ1: デフォルトを必ず設定する

`*` で始まるルールは必ず最初に書く。これがないと、CODEOWNERSにマッチしないファイルが「無主」になる。

```
# 必ず最初に書く
*    @hiroaki @your-org/maintainers
```

### コツ2: CODEOWNERS自体の所有者を厳しく設定

CODEOWNERSファイル自体を誰でも書き換えられると、レビュー責任の仕組みが骨抜きになる。必ず複数人のレビューを必須にする。

```
# CODEOWNERSの変更には複数人のレビューを必須化
/.github/CODEOWNERS    @hiroaki @your-org/security-team
```

### コツ3: ディレクトリ構造を所有権で設計する

逆説的だが、**「CODEOWNERSが書きやすい」ディレクトリ構造は良いディレクトリ構造**であることが多い。

```
# 良い例: 領域ごとに分離されていて、CODEOWNERSが綺麗に書ける
src/
├── frontend/    @frontend-team
├── backend/     @backend-team
└── shared/      @your-org/all-engineers
```

```
# 悪い例: 領域が混在していて、CODEOWNERSが書きにくい
src/
├── components/    # フロントもバックも混在
└── utils/         # 誰のものか不明
```

CODEOWNERSが書きにくいと感じたら、ディレクトリ構造の見直しのサインかもしれない。

### コツ4: コメントで意図を残す

CODEOWNERSは設定ファイルなので、なぜそのルールにしたかが後で分かりにくくなる。コメントを積極的に活用する。

```
# === デフォルト ===
# プロジェクトリードがすべてのファイルのデフォルトレビュアー
*    @hiroaki

# === セキュリティ重要領域 ===
# 認証・暗号化系は必ずセキュリティチームが確認
/src/auth/         @your-org/security-team
/src/crypto/       @your-org/security-team

# === 領域別 ===
/frontend/         @your-org/frontend-team
/backend/          @your-org/backend-team
```

### コツ5: 過剰に細かく書かない

最初から完璧を目指さなくていい。**「デフォルト + 数個の領域別ルール」** から始めて、必要に応じて追加していく。

```
# 最初はこれくらいで十分
*                   @hiroaki @maintainers
/.github/           @hiroaki @devops
/docs/              @docs-team
```

過剰に細かくすると、変更のたびに「あの人にもレビュー依頼されちゃう」となって運用が破綻する。

---

## 10. よくある落とし穴

### 落とし穴1: 個人アカウントのリポジトリでチーム指定ができない

`@your-name/team-name` 形式のチーム指定は、**GitHub Organizationのリポジトリでのみ有効**。個人アカウントのリポジトリでは個人指定しかできない。

### 落とし穴2: CODEOWNERSの構文エラー

書式が間違っていてもGitHubはエラーを大きく表示しない。気づかないうちに無効になっていることがある。

確認方法: GitHubの **Settings → Code owners** ページで、有効なルールが表示されているか確認する(エラーがあればここで分かる)。

### 落とし穴3: ブランチ保護を設定し忘れる

CODEOWNERSだけ作って満足してしまうケース。**ブランチ保護ルールとセットで初めて機能する**ことを忘れずに。

### 落とし穴4: 「Require review from Code Owners」を有効にし忘れる

ブランチ保護を設定しても、このチェックボックスを有効にしないとCODEOWNERSが強制されない。

### 落とし穴5: チームがリポジトリへのアクセス権を持っていない

CODEOWNERSに書いたチームが、そのリポジトリに対する**書き込み権限**を持っていないと、レビュアーとして指定されない。組織の権限設定とCODEOWNERSは整合させる必要がある。

---

## 11. Skillsリポジトリでの応用例

チーム共有のSkillsリポジトリで使う場合、こんな構成が現実的:

```
# === デフォルト ===
*                              @hiroaki

# === Skillリポジトリの根幹 ===
# 重要設定ファイルは複数人レビュー必須
/.github/                      @hiroaki @your-org/devops
/.github/CODEOWNERS            @hiroaki @your-org/devops
/CONTRIBUTING.md               @hiroaki @your-org/maintainers
/README.md                     @hiroaki @your-org/maintainers

# === 汎用Skill ===
# 汎用Skillは複数領域の専門家でレビュー
/skills/general/               @your-org/maintainers

# === 領域別Skill ===
/skills/frontend/              @your-org/frontend-team
/skills/backend/               @your-org/backend-team
/skills/animation/             @your-org/animation-team
/skills/devops/                @your-org/devops

# === セキュリティ関連Skill ===
# 認証や暗号化に関わるSkillは厳しめに
/skills/security/              @your-org/security-team @your-org/maintainers
```

この構成のポイント:

- 領域別Skillは領域専門家がレビュー → ドメイン知識が反映される
- 汎用Skillは複数領域でレビュー → 偏った内容にならない
- セキュリティ関連は二重チェック → リスクの高いSkillは慎重に
- リポジトリ運営の根幹(`.github/`, `README.md` 等)は複数人レビュー → 暴走防止

---

## 12. まとめ

CODEOWNERSは小さな仕組みだが、チーム開発の品質に大きく寄与する。

| 押さえるべきポイント | 内容 |
|---|---|
| 配置場所 | `.github/CODEOWNERS` が推奨 |
| 構文 | `<パターン> <所有者>` のシンプル形式 |
| 優先順位 | 最後にマッチしたルールが勝つ |
| 個人 vs チーム | チーム指定が圧倒的に望ましい |
| 強制力 | ブランチ保護ルールとの組み合わせが必須 |
| 書き順 | 広いルール → 狭いルール の順 |
| デフォルト | 必ず `*` で全体カバーを宣言 |
| CODEOWNERS自体 | このファイル自体も保護対象に含める |

最初から完璧に設計しようとせず、**「デフォルト + 主要な領域別ルール」から始めて、運用しながら育てていく**のが現実的なアプローチ。
