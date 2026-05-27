# RustDesk + Tailscale 導入ガイド

MacBook（クライアント）から自宅のWindows PC（ホスト）へリモートデスクトップ接続するための構成手順です。

---

## 構成概要

| | MacBook（外出先） | Windows（自宅） |
|---|---|---|
| RustDeskの役割 | クライアント（操作する側） | ホスト（操作される側） |
| Tailscaleの役割 | Tailnetに参加 | Tailnetに参加 |
| 主な設定 | ほぼデフォルトでOK | しっかり設定が必要 |

### なぜ Tailscale と併用するのか

- 中継サーバー（hbbs/hbbr）が不要 → インフラ管理ゼロ
- ポートフォワーディング不要
- Tailscale（WireGuard）と RustDesk（E2EE）の**二重暗号化**
- Tailnetに参加していないデバイスはそもそも接続不可

---

## 事前準備：両デバイス共通

1. [Tailscale](https://tailscale.com/download) を両デバイスにインストール
2. 同じ Tailscale アカウントでサインイン
3. `tailscale status` で両デバイスが同じ Tailnet に参加していることを確認

---

## Windows 側（ホスト）の設定

### 1. RustDesk をインストール

[GitHub Releases](https://github.com/rustdesk/rustdesk/releases) から最新版をダウンロード。

```
rustdesk-x.x.x-x86_64.exe
```

インストール後、要求される権限（画面キャプチャ等）をすべて許可する。

### 2. ダイレクトIP接続を有効化

```
RustDeskウィンドウ → ID横の三点メニュー（⋯）→ Settings → Security
→ 「Enable direct IP access」にチェック
```

### 3. 固定パスワードを設定

同じ Security セクションの Password 欄に任意のパスワードを入力。

> ワンタイムパスワードは再起動のたびにリセットされるため、固定パスワードの設定を推奨。

### 4. スタートアップ登録

タスクマネージャー → スタートアップ タブで RustDesk が **有効** になっているか確認。

### 5. スリープを無効化

```
設定 → システム → 電源とスリープ → スリープ → 「なし」
```

外出中に Windows がスリープすると接続できなくなるため必須。

### 6. Tailscale IP を確認・メモ

```powershell
tailscale ip
# 例: 100.x.x.x
```

このIPアドレスが MacBook から接続する際に使用するアドレスになります。

---

## MacBook 側（クライアント）の設定

### 1. RustDesk をインストール

M5チップ（Apple Silicon）のため、aarch64版をダウンロード。

```
rustdesk-x.x.x-aarch64.dmg
```

ダウンロード後、dmg を開いて Applications フォルダへドラッグ。

### 2. macOS の権限を許可

RustDesk 起動時に以下の権限を要求される。すべて許可すること。

```
システム設定 → プライバシーとセキュリティ → 画面収録 → RustDesk を許可
```

### 3. ダイレクトIP接続を有効化

Windows 側と同様に設定。

```
RustDeskウィンドウ → ID横の三点メニュー（⋯）→ Settings → Security
→ 「Enable direct IP access」にチェック
```

---

## 接続手順

1. Windows 側が起動・RustDesk が起動していることを確認
2. MacBook の RustDesk を起動
3. 「リモートIDを入力」欄に Windows の Tailscale IP（`100.x.x.x`）を入力
4. 「接続」をクリック
5. Windows 側で設定した固定パスワードを入力

> **注意：** RustDesk は IP アドレスのみ対応。Tailscale のデバイス名（`devicename.tailnet.ts.net`）は使用不可。

---

## 設定チェックリスト

### Windows（ホスト）

- [ ] RustDesk インストール済み（x86_64版）
- [ ] Direct IP access 有効化
- [ ] 固定パスワード設定済み
- [ ] スタートアップ登録済み
- [ ] スリープ無効化済み
- [ ] Tailscale インストール・参加済み
- [ ] Tailscale IP メモ済み（`100.x.x.x`）

### MacBook（クライアント）

- [ ] RustDesk インストール済み（aarch64版）
- [ ] 画面収録権限 許可済み
- [ ] Direct IP access 有効化
- [ ] Tailscale インストール・参加済み（同じアカウント）

---

## リポジトリ・公式リンク

| ツール | GitHub | 公式サイト |
|---|---|---|
| RustDesk | https://github.com/rustdesk/rustdesk | https://rustdesk.com |
| Tailscale | https://github.com/tailscale/tailscale | https://tailscale.com |
