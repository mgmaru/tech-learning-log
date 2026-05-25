# Claude Skills と MCP — 概念整理と配信・管理方法の比較

## 概要

Claudeの能力拡張には大きく分けて **MCP (Model Context Protocol)** と **Agent Skills** の2系統がある。両者は混同されやすいが、目的も仕組みも独立しており、必要に応じて単独でも併用でも使える。

本ドキュメントは、

1. MCPとSkillsの本質的な違い
2. Skillsをどこに置き、どう配信するか(管理方法のバリエーション)
3. 各方式のメリット・デメリット

を備忘録として整理する。提案中の仕様や周辺ツールの参照リンクも末尾にまとめる。

---

## 1. MCP と Skills の違い

| 観点 | MCP | Skills |
|---|---|---|
| 役割 | 外部システムへの**接続層** | タスク遂行のための**指示書/ノウハウ** |
| 比喩 | 手足(外の世界に作用する) | マニュアル(やり方を教える) |
| 実体 | サーバーが提供する関数群 | `SKILL.md` を中心としたフォルダ |
| 実行時の挙動 | リクエスト時に外部APIを呼ぶ | 作業前に内容を読み込んでガイドにする |
| 通信 | あり(HTTP/SSE等) | 基本なし(ローカル読み込み) |
| 例 | Gmail / Asana / Google Drive 連携 | 「Wordドキュメント作成の流儀」 |

### 使い分けの判断軸

判断軸は独立した2つ:

- **外部システムと通信する必要があるか?** → MCPの要否
- **Claudeに毎回特定の作法を教え込みたいか?** → Skillsの要否

両方Yesなら併用。片方だけYesならその片方だけでよい。

### 併用が有効なケース

外部接続と独自ノウハウの両方が必要な場合。例えば社内Asana連携で「タイトルの接頭辞ルール」「優先度設定の社内基準」がある場合、Asana MCP + 運用ルールを書いたSkillの組み合わせが効く。

---

## 2. Skillsの管理方法 — 全体俯瞰

Skillsは「`SKILL.md`を中心としたフォルダ」という形式さえ守れば、どこにどう置くかは自由。コミュニティでは以下のような提案・実装が並行して動いている。

### 2.1 ローカルファイルシステム

最もシンプル。`~/.claude/skills/` (個人用) または `.claude/skills/` (プロジェクト単位) に配置する公式の標準的な方式。

**Pros**
- セットアップ不要
- ネットワーク不要、オフラインで動く
- バージョン管理はGitに任せられる

**Cons**
- マシン間で同期しない
- 他人と共有しにくい
- 動的更新・権限管理は不可

**向いている場面**: 個人の開発、単一マシン、プロジェクト固有のスキル。

---

### 2.2 Git URL / GitHub経由

`npx skills add <github-url>` 等で、Gitリポジトリから直接インストールする方式。Anthropic公式が現状推しているスタイル。

**Pros**
- 既存のGitワークフローに乗れる(レビュー、履歴、ブランチ)
- 公開・共有が容易
- 中央レジストリ不要(URL自体が識別子になる)

**Cons**
- インストール時のスナップショットなので、自動更新は別途仕組みが要る
- 依存解決やロックファイルは現状未整備(RFC段階)

**向いている場面**: チーム共有、OSSスキルの配布、再現性重視のセットアップ。

参考: [agentskills/agentskills Discussion #210 — Skill Package Manifest](https://github.com/agentskills/agentskills/discussions/210)

---

### 2.3 静的HTTPサーバー / CDN

`SKILL.md`を含むフォルダをそのままWebサーバーに置き、URL経由で動的にロードする方式。GitHub Pages、Netlify、S3 Static Hosting、nginxの素のディレクトリ公開などが該当。

**「静的」の意味**: サーバー側でプログラムが走らず、ファイルをそのまま返すだけのサーバー。動的サーバー(Next.js SSR、Rails APIなど)の対義語。

**Pros**
- セットアップが簡単(`git push` → GitHub Pagesで配信完了レベル)
- 安価・高速・CDNで世界中にキャッシュ可能
- 複数マシン間で常に最新を取得できる
- 運用負荷ほぼゼロ

**Cons**
- 動的処理は不可(権限の出し分け、利用ログ、検索など)
- スキル追加・削除のたびにデプロイが必要
- メタデータでのクエリが弱い(全件取得 → クライアント側フィルタになりがち)

**向いている場面**: 個人用Skillの複数マシン同期、社内/コミュニティへの単純な配布。

参考: [agentskills/agentskills Issue #42 — RFC: Remote Agent Skills (URL-based Skill Import)](https://github.com/agentskills/agentskills/issues/42)

---

### 2.4 OCIアーティファクト

コンテナイメージのようにOCIレジストリ(GHCR、Artifactoryなど)にスキルをプッシュ・プルする方式。`Project Cumasach` がリファレンス実装の一つ。

**Pros**
- バージョニング、署名、検証、ロールバックなど、コンテナエコシステムの成熟した仕組みを流用できる
- 既存のCI/CDパイプラインに組み込みやすい
- 大規模配布で実績ある仕組み

**Cons**
- 個人用途には過剰
- OCIツール(ORASなど)の学習コストがある

**向いている場面**: エンタープライズ、署名付き配布、強い再現性が必要な環境。

参考: [Project Cumasach](https://github.com/artur-ciocanu/project-cumasach)

---

### 2.5 データベース

Skillsの本文・メタデータをDBに格納し、カスタムプロバイダ経由でロードする方式。MicrosoftのAgent Skills SDKの`SkillProvider`抽象を使えば、数メソッド書くだけで実装できる。

**Pros**
- 動的編集が即反映できる(UIから編集して保存するだけ)
- メタデータ検索・フィルタが強力(タグ、最終更新、所有者などで絞り込み)
- 行レベルの権限管理が可能
- 利用ログ・分析・A/Bテストが容易
- pgvectorなどと組み合わせると埋め込みベクトル検索による精度の高い動的選択が可能

**Cons**
- インフラ運用が必要
- バージョン管理・差分レビューはGitに劣る
- 配布・公開には不向き(透明性が低い)

**向いている場面**: 動的編集が頻繁な環境、権限管理・分析が要件、社内ツールとしてのSkills管理基盤。

参考: [Microsoft Agent Skills SDK 提案](https://github.com/agentskills/agentskills/issues/139)

---

### 2.6 中央レジストリ(MCPレジストリ統合)

MCPレジストリにSkillsも統合する案。npm registryやDocker Hubに近い世界観。

**Pros**
- 横断的な発見性(検索、評価、ダウンロード数などのソーシャル指標)
- 標準化された配信フォーマット
- MCPサーバーがSkillsをネイティブに公開できる可能性

**Cons**
- まだ仕様策定中(RFC段階)
- 集中型の運用主体が必要

**向いている場面**: 将来的なエコシステム発展。現時点では発展途上。

参考: [modelcontextprotocol/registry Discussion #895 — skills.json format proposal](https://github.com/modelcontextprotocol/registry/discussions/895)

---

## 3. プロバイダ抽象という考え方

MicrosoftのAgent Skills SDKが導入した`SkillProvider`抽象は重要な発想で、上記2.1〜2.5を**同じインターフェースの別実装**として扱える。

インターフェースは `get_metadata`, `get_body`, `get_reference`, `get_script`, `get_asset` 程度のシンプルさ。

これにより、

1. 開発時 → **ファイルシステムプロバイダ**
2. 共有・複数マシン同期 → **静的HTTPプロバイダ**
3. 規模が大きくなり権限管理・分析が必要に → **DBプロバイダ**

という段階的な発展が、エージェント側のコードを変えずにできる。最初から大規模な仕組みを組む必要はなく、必要になった時点で差し替えればよい。

---

## 4. 段階的な発展パスの例(個人プロジェクト想定)

1. **ローカルファイル**で書き始める(`~/.claude/skills/`)
2. Git管理してGitHubに上げる → 他マシンでも `git clone` で展開
3. **GitHub Pagesで静的HTTP配信**して、URLで参照できるようにする
4. 必要に応じて**OCI**で署名付き配布、もしくは**DB**で動的管理に移行

各段階は不可逆ではなく、用途や規模に応じて行き来できる。

---

## 5. 現時点(2026年5月)のステータス整理

| 方式 | ステータス |
|---|---|
| ローカルファイル | 公式標準、安定 |
| Git URL | 公式CLI(`npx skills add`)で対応済み |
| 静的HTTP | RFC #42として議論中、サードパーティ実装あり |
| OCI | コミュニティで複数実装、収束中 |
| データベース | Microsoft SDK経由で実装可能、SkillProvider抽象 |
| 中央レジストリ | MCPレジストリ側で議論中、未確定 |

公式Anthropic側はGit URL経由のインストールを推している印象。HTTP・OCI・DBはコミュニティ主導で並行進化中。

---

## 6. 関連リンク

- [Anthropic公式 Agent Skills ドキュメント](https://platform.claude.com/docs/en/agents-and-tools/agent-skills/overview)
- [anthropics/skills リポジトリ](https://github.com/anthropics/skills)
- [agentskills.io](https://agentskills.io) — Agent Skills 標準
- [RFC #42 — Remote Agent Skills (URL Import)](https://github.com/agentskills/agentskills/issues/42)
- [Discussion #210 — Skill Package Manifest (依存解決とOCI)](https://github.com/agentskills/agentskills/discussions/210)
- [Issue #139 — Agent Skills SDK (プロバイダ抽象)](https://github.com/agentskills/agentskills/issues/139)
- [MCP Registry Discussion #895 — skills.json](https://github.com/modelcontextprotocol/registry/discussions/895)
- [Microsoft Agent Skills SDK 解説記事](https://techcommunity.microsoft.com/blog/azuredevcommunityblog/giving-your-ai-agents-reliable-skills-with-the-agent-skills-sdk/4497074)
- [OASR — リモートSkillレジストリ実装](https://github.com/JordanGunn/oasr)

---

## メモ

- 「Skills自体をデータベースに格納する」発想は、MicrosoftのSDKでは想定済みのカスタムプロバイダパターンに相当する
- 個人用Skills DBは、過去に作っていた `tech-learning-log` + 再利用可能プロンプトテンプレートの発想と地続き
- SQLite + Tauriフロントエンドでプロトタイプ可能。FrameWeaverと同じスタックで遊べる
- 将来的に埋め込みベクトル検索とSkills選択を組み合わせる方向に進む可能性がある(私見)
