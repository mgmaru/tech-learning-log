# GitHub Actions 入門

## 1. GitHub Actionsとは

GitHub Actionsは、GitHub上でさまざまな処理を自動実行するための仕組みです。

代表的には、以下のような作業を自動化できます。

- テストの実行
- TypeScriptの型チェック
- Lintによるコードチェック
- アプリケーションのビルド
- デプロイ
- Dockerイメージの作成
- 定期実行バッチ
- Pull Requestへの自動チェック

一言でいうと、GitHub Actionsは、

> GitHubにコードをpushしたときやPull Requestを作成したときに、決められた処理を自動で実行してくれる仕組み

です。

---

## 2. CI/CDとの関係

GitHub Actionsは、CI/CDを実現するためによく使われます。

| 用語 | 意味 | 例 |
|---|---|---|
| CI | Continuous Integration / 継続的インテグレーション | pushやPull Request時に自動でテスト・ビルドする |
| CD | Continuous Delivery / Deployment / 継続的デリバリー・デプロイ | テスト成功後に自動でデプロイする |

たとえば、Reactアプリであれば以下のような流れを自動化できます。

```text
コードをpush
  ↓
GitHub Actionsが起動
  ↓
依存関係をインストール
  ↓
型チェック
  ↓
テスト
  ↓
ビルド
  ↓
必要に応じてデプロイ
```

---

## 3. GitHub Actionsでできること

## 3.1 テストの自動実行

開発者が用意したテストコードを、pushやPull Requestのタイミングで自動実行できます。

例：

```bash
npm run test
```

GitHub Actions自体がテスト内容を考えるわけではありません。
開発者が用意したテストを、GitHub Actionsが自動で実行します。

---

## 3.2 型チェック

TypeScriptプロジェクトでは、型エラーがないかを自動確認できます。

```bash
npm run typecheck
```

たとえば以下のようなコマンドが使われます。

```bash
tsc --noEmit
```

---

## 3.3 Lint

ESLintやBiomeなどを使って、コードの書き方や危険な書き方をチェックできます。

```bash
npm run lint
```

---

## 3.4 ビルド確認

アプリケーションが本番用にビルドできるか確認できます。

```bash
npm run build
```

個人開発では、まずこのビルド確認を自動化するだけでもかなり効果があります。

---

## 3.5 自動デプロイ

テストやビルドが成功したあと、Vercel、Firebase、AWS、GitHub Pagesなどへ自動デプロイできます。

ただし、初心者のうちはいきなり本番デプロイまで自動化せず、まずは以下の3つから始めるのがおすすめです。

```text
1. 型チェック
2. Lint
3. ビルド
```

---

## 4. GitHub Actionsの基本構成

GitHub Actionsでは、リポジトリ内にYAMLファイルを作成します。

基本的な配置場所は以下です。

```text
.github/workflows/ci.yml
```

構成は以下のようになります。

```text
Workflow
  └─ Job
      └─ Step
          └─ Action または コマンド
```

| 要素 | 説明 |
|---|---|
| Workflow | 自動化処理全体 |
| Event | Workflowを起動するきっかけ |
| Job | 実行単位 |
| Step | Jobの中の具体的な手順 |
| Action | 再利用できる処理部品 |
| Runner | 実際に処理を実行するマシン |

---

## 5. Event：いつ実行するか

GitHub Actionsは、特定のイベントをきっかけに実行されます。

| Event | 意味 |
|---|---|
| `push` | pushされたとき |
| `pull_request` | Pull Requestが作成・更新されたとき |
| `workflow_dispatch` | 手動実行 |
| `schedule` | 定期実行 |

例：pushとPull Requestで実行する場合

```yaml
on:
  push:
    branches: [main]
  pull_request:
```

---

## 6. Runner：どこで実行されるか

Runnerは、GitHub Actionsの処理を実際に実行するマシンです。

| Runner | 説明 |
|---|---|
| GitHub-hosted runner | GitHubが用意する実行環境 |
| self-hosted runner | 自分のPCやサーバーを実行環境として使う方式 |

通常の個人開発では、まずGitHub-hosted runnerを使えば十分です。

代表的には以下を使います。

```yaml
runs-on: ubuntu-latest
```

Webアプリ開発では、LinuxのRunnerで十分なケースが多いです。

---

## 7. 基本的なWorkflow例

React / TypeScriptプロジェクトの例です。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Test
        run: npm run test

      - name: Build
        run: npm run build
```

このWorkflowでは、以下の処理を自動実行しています。

```text
1. リポジトリのコードを取得
2. Node.jsをセットアップ
3. npm ciで依存関係をインストール
4. TypeScriptの型チェック
5. テスト実行
6. ビルド確認
```

---

## 8. GitHub Actionsにおける「テスト」とは

GitHub Actionsでいう「テスト」は、開発者が用意したテストコードやチェックコマンドを自動実行することです。

GitHub Actionsが勝手にアプリの正しさを判断するわけではありません。

```text
開発者がテストを書く
  ↓
GitHubにpushする
  ↓
GitHub Actionsがテストを自動実行する
  ↓
成功・失敗をPull Request上に表示する
```

---

## 9. テストの種類

| テストの種類 | 確認すること | 例 |
|---|---|---|
| 単体テスト | 関数や小さなロジックが正しく動くか | バリデーション関数、計算関数 |
| コンポーネントテスト | UI部品が正しく表示・動作するか | Reactのボタン、フォーム |
| 結合テスト | 複数の処理を組み合わせて正しく動くか | API呼び出し、DB保存 |
| E2Eテスト | ユーザー操作として画面全体が動くか | ログイン、投稿、タスク作成 |
| 型チェック | TypeScriptの型エラーがないか | `tsc --noEmit` |
| Lint | コードの書き方に問題がないか | ESLint、Biome |
| ビルド確認 | 本番用にビルドできるか | `npm run build` |

厳密には、型チェック・Lint・ビルド確認は「テストコード」ではありません。
ただし、CI上では品質チェックとしてテストと一緒に実行されることが多いです。

---

## 10. テストシナリオは誰が作るのか

テストシナリオは基本的に開発者が作成します。

たとえば、以下の仕様があるとします。

> タスク名が空の場合、登録できない

この仕様に対して、開発者は次のようなテストを書きます。

```ts
test("タスク名が空の場合、エラーになる", () => {
  const result = validateTaskName("");

  expect(result).toBe("タスク名は必須です");
});
```

GitHub Actionsは、このテストコードを自動で実行する役割を持ちます。

| 項目 | 担当 |
|---|---|
| 何をテストするか決める | 開発者 |
| テストコードを書く | 開発者 |
| テストコマンドを用意する | 開発者 |
| push時に自動実行する | GitHub Actions |
| 成功・失敗を表示する | GitHub Actions |

---

## 11. npm run test の仕組み

`npm run test`を実行すると、`package.json`の`scripts.test`に書かれたコマンドが実行されます。

例：

```json
{
  "scripts": {
    "test": "vitest"
  }
}
```

この場合、以下のコマンドを実行するのと同じです。

```bash
vitest
```

つまり、`npm run test`自体がテストファイルを探しているわけではありません。
実際にテストファイルを探すのは、VitestやJestなどのテストツールです。

```text
npm run test
  ↓
package.json の scripts.test を実行
  ↓
Vitest / Jest / Playwright などが起動
  ↓
テストツールのルールに従ってテストファイルを探す
```

---

## 12. テストコードの配置場所

テストコードの配置は、使用するテストツールのルールや設定によって決まります。

React / TypeScriptでは、主に以下の2パターンがあります。

---

## 12.1 対象ファイルの近くに置く

```text
src/
  features/
    task/
      taskValidator.ts
      taskValidator.test.ts
      TaskForm.tsx
      TaskForm.test.tsx
```

この置き方は、実装ファイルとテストファイルの対応関係がわかりやすいです。

```text
taskValidator.ts       ← 実装
taskValidator.test.ts  ← そのテスト
```

個人開発ではこの方法がおすすめです。

---

## 12.2 `__tests__` ディレクトリに置く

```text
src/
  features/
    task/
      taskValidator.ts
      TaskForm.tsx
      __tests__/
        taskValidator.test.ts
        TaskForm.test.tsx
```

テストファイルをまとめて管理したい場合に使われます。

---

## 13. Vitest / Jestが認識するファイル名

一般的には、以下のようなファイル名がテストとして認識されます。

```text
*.test.ts
*.test.tsx
*.spec.ts
*.spec.tsx
```

例：

```text
taskValidator.test.ts
TaskForm.test.tsx
taskValidator.spec.ts
TaskForm.spec.tsx
```

初心者のうちは、以下の命名にしておくとわかりやすいです。

```text
xxx.test.ts
xxx.test.tsx
```

---

## 14. 設定ファイルでテスト対象を指定する例

Vitestでは、`vitest.config.ts`などでテスト対象を指定できます。

```ts
import { defineConfig } from "vitest/config";

export default defineConfig({
  test: {
    include: ["src/**/*.test.{ts,tsx}"],
  },
});
```

これは、

```text
src/配下にある .test.ts または .test.tsx をテスト対象にする
```

という意味です。

---

## 15. E2Eテストの配置例

E2Eテストは、単体テストやコンポーネントテストとは別に分けることが多いです。

```text
tests/
  e2e/
    task-create.spec.ts
    login.spec.ts
```

たとえばPlaywrightを使う場合、以下のようなコマンドになります。

```bash
npx playwright test
```

GitHub Actionsでは次のように実行できます。

```yaml
- name: Run E2E tests
  run: npx playwright test
```

---

## 16. GitHub Actionsとテストツールの役割分担

| 役割 | 担当 |
|---|---|
| テストファイルをどこに置くか | 開発者 |
| どの名前をテストとして認識するか | Vitest / Jest / Playwrightなど |
| `npm run test`で何を起動するか | `package.json` |
| CI上でコマンドを実行する | GitHub Actions |

GitHub Actionsは、テストファイルの場所を直接探すというより、`npm run test`などのコマンドを実行するだけです。

---

## 17. 個人開発でおすすめの導入順

最初から完璧な自動テスト環境を作る必要はありません。

おすすめの導入順は以下です。

```text
1. npm run build をGitHub Actionsで実行
2. npm run typecheck を追加
3. npm run lint を追加
4. 重要なロジックの単体テストを追加
5. 重要な画面操作のE2Eテストを追加
6. 必要に応じて自動デプロイを追加
```

まずは、mainブランチに壊れたコードを入れないことを目的にするとよいです。

---

## 18. GitHub Actionsの利用料金

GitHub Actionsは、パブリックリポジトリでは標準のGitHub-hosted runnerを無料で利用できます。

プライベートリポジトリでは、GitHubのプランごとに無料枠があり、それを超えると課金対象になります。

料金や無料枠は変更される可能性があるため、実際に使う前にはGitHub公式の料金ページを確認してください。

一般的には、個人開発で軽いCIを回す程度であれば、無料枠内に収まることが多いです。

特に注意が必要なのは以下です。

| 注意点 | 説明 |
|---|---|
| private repository | 無料枠を超えると課金対象になる可能性がある |
| macOS runner | Linux runnerより高額になりやすい |
| 長時間のE2Eテスト | 実行時間を消費しやすい |
| 頻繁なpush | CIの実行回数が増える |
| Artifact保存 | 保存容量を消費する |

Webアプリ開発では、基本的に以下を使うのがおすすめです。

```yaml
runs-on: ubuntu-latest
```

---

## 19. Secretsの利用

APIキーやトークンなどの秘密情報は、コードに直接書かず、GitHubのSecretsに保存します。

例：

```yaml
env:
  API_KEY: ${{ secrets.API_KEY }}
```

Secretsを使うことで、以下のような情報を安全に扱えます。

- APIキー
- デプロイトークン
- Firebaseの認証情報
- AWSの認証情報
- Slack通知用Webhook URL

---

## 20. 外部Actionを使うときの注意

GitHub Actionsでは、Marketplaceなどに公開されている外部Actionを使えます。

例：

```yaml
- uses: actions/checkout@v4
```

ただし、知らないActionを安易に使うのは注意が必要です。

理由は、ActionはWorkflow内でコードを実行できるためです。

なるべく以下を意識します。

- 公式Actionを優先する
- 利用者が多いActionを使う
- バージョンを固定する
- READMEや更新状況を確認する
- Secretsを扱うWorkflowでは特に慎重にする

---

## 21. まとめ

GitHub Actionsは、GitHub上でテスト・ビルド・デプロイなどを自動化する仕組みです。

重要なポイントは以下です。

| 観点 | 内容 |
|---|---|
| GitHub Actionsとは | GitHub上で処理を自動実行する仕組み |
| 主な用途 | テスト、型チェック、Lint、ビルド、デプロイ |
| 設定場所 | `.github/workflows/*.yml` |
| テストの内容 | 開発者が用意する |
| GitHub Actionsの役割 | 用意されたコマンドを自動実行する |
| テストファイルの探索 | Vitest、Jest、Playwrightなどが行う |
| 初心者の導入順 | build → typecheck → lint → test → deploy |

最初に導入するなら、以下のようなCIから始めるのがおすすめです。

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  ci:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout repository
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: 22

      - name: Install dependencies
        run: npm ci

      - name: Type check
        run: npm run typecheck

      - name: Lint
        run: npm run lint

      - name: Test
        run: npm run test

      - name: Build
        run: npm run build
```

このようにしておくと、Pull Requestやpushのたびに、アプリが壊れていないかを自動で確認できます。

GitHub Actionsは、テストを作ってくれるものではなく、

> 開発者が用意したチェックを、GitHub上で毎回自動実行してくれる仕組み

と理解するとわかりやすいです。
