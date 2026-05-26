# Raspberry Pi + UpSnap で Wake-on-LAN 環境を構築する

## 概要

このドキュメントは、自宅のPCをスリープ状態からインターネット越しに起動するための環境構築手順をまとめたものです。  
Raspberry Pi を中継サーバーとして利用し、UpSnap（WoL Webアプリ）と VPN（Tailscale）を組み合わせて構成します。

---

## 全体構成

```
外出先のブラウザ（スマホ・PC）
  │
  │（Tailscale VPN 経由）
  │
自宅ルーター
  ├─ Raspberry Pi Zero 2 W（常時起動）
  │    └─ UpSnap（WoL Webアプリ）
  │    └─ Tailscale（VPN）
  │
  └─ メインPC（スリープ中）← WoLで起動
         └─ 有線LAN接続必須
```

---

## 1. 基礎知識

### 1-1. Wake-on-LAN（WoL）とは

スリープ中のPCに「マジックパケット」と呼ばれる特殊なデータを送ることで、PCを起動する仕組みです。  
PCの電源が切れていても、NIC（ネットワークカード）だけは待機電力で動き続けており、マジックパケットを受信するとPCを起動します。

### 1-2. マジックパケットとは

送信先PCのMACアドレスを特定のフォーマットで繰り返したUDPパケットです。  
受け取ったNICがこれを「起動の合図」として認識します。

### 1-3. WoLの制約

| 項目 | 内容 |
|------|------|
| 接続方式 | **有線LAN必須**（Wi-Fiは基本的に非対応） |
| ネットワーク範囲 | 同一LANセグメント内のみ（ルーターを超えられない） |
| インターネット越し | 中継サーバーを経由する必要がある |

> **重要：** 起こされる側のPCは**有線LAN接続が必須**です。Wi-Fi接続のPCはWoLで起動できません。

---

## 2. 必要な機器・ソフトウェア

### 2-1. 中継サーバー（Raspberry Pi）

起こされる側のPCと同じLAN内に置き、常時起動させておきます。

**推奨モデル：Raspberry Pi Zero 2 W**

| スペック | 内容 |
|----------|------|
| CPU | 1GHz クアッドコア（64-bit ARM Cortex-A53） |
| RAM | 512MB |
| 無線 | Wi-Fi 2.4GHz / Bluetooth 4.2 |
| サイズ | 65mm × 30mm（超小型） |
| 価格 | 約2,500円 |
| 消費電力 | 非常に小さい（常時起動に最適） |

UpSnapのメモリ使用量はアイドル時約50MBなので、512MBで余裕があります。

**購入先：** スイッチサイエンス、秋月電子、Amazon など

**別途必要なもの：**
- microSDカード（8GB以上）
- microUSB電源アダプター（5V / 2.5A推奨）

### 2-2. 起こされる側のPC

- 有線LAN接続（必須）
- WoL対応NIC（最近のPCはほぼ対応済み）
- BIOSでWoL有効化（後述）

### 2-3. ソフトウェア

| ソフトウェア | 役割 |
|-------------|------|
| **UpSnap** | WoL Webアプリ。ブラウザからワンクリックでPCを起動できる |
| **Tailscale** | VPN。インターネット越しに安全にアクセスするために使用 |
| **Docker**（任意） | UpSnapの推奨実行環境 |

---

## 3. 起こされる側のPC設定

### 3-1. BIOSでWoLを有効化

1. PC起動時にBIOSに入る（`Delete` / `F2` / `F12` など機種により異なる）
2. `Power Management` や `Advanced` などのメニューを探す
3. `Wake on LAN` または `Power On By PCI-E` を **Enabled** に設定
4. 設定を保存して再起動

### 3-2. OSでWoLを有効化（Windows）

1. デバイスマネージャーを開く
2. `ネットワークアダプター` → 有線LANアダプターを右クリック → プロパティ
3. `電源の管理` タブ → 「このデバイスで、コンピューターのスタンバイ状態を解除できるようにする」にチェック
4. `詳細設定` タブ → `Wake on Magic Packet` を **Enabled** に設定

### 3-3. MACアドレスの確認（後で使用）

コマンドプロンプトで以下を実行し、有線LANアダプターの物理アドレスを控えておきます。

```cmd
ipconfig /all
```

`物理アドレス` の値（例：`A1-B2-C3-D4-E5-F6`）をメモしておきます。

---

## 4. Raspberry Pi の初期設定

### 4-1. Raspberry Pi OS のインストール

1. [Raspberry Pi Imager](https://www.raspberrypi.com/software/) をダウンロード・インストール
2. microSDカードをPCに挿入
3. Imagerで以下を選択：
   - OS：`Raspberry Pi OS Lite（64-bit）`（デスクトップ不要）
   - ストレージ：挿入したmicroSDカード
4. 歯車アイコン（詳細設定）から以下を設定：
   - ホスト名（例：`upsnap-pi`）
   - SSHを有効化
   - Wi-Fi設定（SSID・パスワード）
   - ユーザー名・パスワード
5. 書き込み実行

### 4-2. Raspberry Pi に SSH 接続

```bash
ssh pi@upsnap-pi.local
# または
ssh pi@<RaspberryPiのIPアドレス>
```

### 4-3. 初期アップデート

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 5. UpSnap のインストール

### 方法A：Docker を使う（推奨）

#### Dockerのインストール

```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
# 一度ログアウトして再接続
```

#### docker-compose.yml の作成

```bash
mkdir ~/upsnap && cd ~/upsnap
nano docker-compose.yml
```

以下の内容を貼り付け：

```yaml
version: "3"
services:
  upsnap:
    container_name: upsnap
    image: ghcr.io/seriousm4x/upsnap:4
    network_mode: host
    restart: unless-stopped
    volumes:
      - ./data:/app/pb_data
    environment:
      - TZ=Asia/Tokyo
      - UPSNAP_INTERVAL=@every 10s
      - UPSNAP_SCAN_RANGE=192.168.1.0/24  # 自宅のネットワーク範囲に合わせて変更
```

#### 起動

```bash
docker compose up -d
```

### 方法B：バイナリを直接実行する

```bash
# バイナリをダウンロード（arm64版）
wget https://github.com/seriousm4x/UpSnap/releases/latest/download/UpSnap_linux_arm64.tar.gz
tar -xzf UpSnap_linux_arm64.tar.gz

# raw socket権限を付与（root不要で動かすため）
sudo setcap cap_net_raw=+ep ./upsnap

# 起動
./upsnap serve --http=0.0.0.0:8090
```

---

## 6. UpSnap の初期設定

1. Raspberry Pi と同じネットワークに繋がったブラウザで `http://<RaspberryPiのIP>:8090` にアクセス
2. 管理者アカウントを作成
3. デバイスを追加：
   - デバイス名（例：`メインPC`）
   - IPアドレス（PCのローカルIP）
   - MACアドレス（手順3-3で控えたもの）
4. 「Wake」ボタンを押してPCが起動するか確認

---

## 7. Tailscale のインストール（インターネット越しアクセス）

UpSnapをインターネットに直接公開するのはセキュリティ上危険です。  
Tailscale（VPN）を経由することで、安全にどこからでもアクセスできます。

### 7-1. Raspberry Pi に Tailscale をインストール

```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

表示されたURLにブラウザでアクセスしてログイン・認証します。

### 7-2. 外出先のデバイスに Tailscale をインストール

- [Tailscale公式サイト](https://tailscale.com/download) からスマホ・PCにインストール
- 同じアカウントでログイン

### 7-3. アクセス確認

外出先から Tailscale 経由でアクセス：

```
http://<TailscaleのRaspberry PiのIP>:8090
```

Tailscaleの管理画面でRaspberry PiのIPアドレスを確認できます。

---

## 8. 使い方（運用）

1. 外出先でスマホ・PCのTailscaleを起動
2. ブラウザで `http://<TailscaleのRaspberry PiのIP>:8090` にアクセス
3. UpSnapの画面でPCの「Wake」ボタンを押す
4. 30秒〜数分待つ（OS起動時間）
5. リモートデスクトップ（RDP）で接続

---

## 9. トラブルシューティング

| 症状 | 確認事項 |
|------|----------|
| WoLが効かない | BIOSのWoL設定、OSのデバイスマネージャー設定、有線LAN接続を確認 |
| UpSnapにアクセスできない | Raspberry PiのIPアドレス、Dockerコンテナが起動しているか確認 |
| 外出先からアクセスできない | Tailscaleが両方のデバイスで起動しているか確認 |
| 起動後すぐ繋がらない | PCのOS起動完了まで1〜2分待ってからRDP接続する |

---

## 参考

- [UpSnap GitHub](https://github.com/seriousm4x/UpSnap)
- [Tailscale 公式サイト](https://tailscale.com)
- [Raspberry Pi Imager](https://www.raspberrypi.com/software/)
- [Tailscale + UpSnap 構築ガイド（英語）](https://tailscale.com/blog/wake-on-lan-tailscale-upsnap)
