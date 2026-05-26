# Tailscale併用のリモートデスクトップ比較

## 概要

MacBook から Windows PC をリモート操作する場合、Windows Home では標準の Windows リモートデスクトップ、いわゆる RDP の接続先には基本的になれません。

そのため、Windows Home 環境では以下のようなツールを検討することになります。

- RustDesk
- Sunshine + Moonlight
- NoMachine
- VNC系ツール
- Chrome Remote Desktop

この記事では、**Tailscaleと併用する前提**で、それぞれの特徴を比較します。

---

## 前提

想定する環境は以下です。

```text
MacBook
  ↓
Tailscale
  ↓
Windows PC
```

Windows PC は Windows Home を想定します。

用途は主に以下です。

```text
1. Windows PCの画面をMacから操作したい
2. 動画編集ソフトなどGUIアプリを操作したい
3. できるだけ軽く、低遅延で操作したい
```

なお、WSL上のソフトウェア開発については、リモートデスクトップではなく、VS Code Remote SSH を使う方が快適です。

---

# 結論

軽さ・低遅延を重視するなら、まず試す価値が高いのは **Sunshine + Moonlight** です。

一方で、汎用的なリモートデスクトップとしてバランスが良いのは **RustDesk** です。

| 用途 | おすすめ |
|---|---|
| 動画編集や画面描画の軽さ重視 | Sunshine + Moonlight |
| 普通のリモートデスクトップ用途 | RustDesk |
| RustDeskの代替として安定したGUI操作 | NoMachine |
| 軽い管理作業だけ | VNC系 |
| 非常用・簡単接続 | Chrome Remote Desktop |

---

# 比較表

| ツール | 軽さ・低遅延 | 設定の簡単さ | Tailscale併用 | Windows Home | 向いている用途 |
|---|---:|---:|---:|---:|---|
| RustDesk | 中〜高 | 高 | 良い | 可 | 一般的な遠隔操作 |
| Sunshine + Moonlight | 高い | 中 | 良い | 可 | 動画編集・低遅延操作 |
| NoMachine | 中〜高 | 中〜高 | 良い | 可 | 汎用リモートデスクトップ |
| VNC系 | 中 | 中 | 良い | 可 | 軽い管理作業 |
| Chrome Remote Desktop | 中 | 高 | 基本不要 | 可 | 簡単接続・非常用 |

---

# 1. RustDesk

## 概要

RustDesk は、TeamViewer や AnyDesk の代替として使えるリモートデスクトップソフトです。

Windows、macOS、Linux などに対応しており、Windows Home でも利用できます。

Tailscaleと併用する場合は、RustDeskの **Direct IP Access** を有効にし、Windows PC の Tailscale IP に直接接続します。

## 構成

```text
MacBook
  └ RustDesk
      ↓ Tailscale
Windows PC
  └ RustDesk
```

## RustDeskのメリット

```text
・設定が比較的簡単
・Windows Homeでも使える
・Tailscale IPで直接接続できる
・一般的なWindows操作に向いている
・ファイル転送などの機能もある
・リモートサポート用途にも使いやすい
```

## RustDeskのデメリット

```text
・動画編集のように画面変化が多い用途では重く感じる場合がある
・低遅延特化ではない
・画質や遅延はネットワーク環境に左右される
```

## 向いている用途

```text
・普通にWindowsを遠隔操作したい
・設定を簡単に済ませたい
・リモートサポート的に使いたい
・ファイル転送も使いたい
```

RustDeskは、最もバランスの良い選択肢です。

---

# 2. Sunshine + Moonlight

## 概要

Sunshine + Moonlight は、低遅延の画面ストリーミング構成です。

Windows側に **Sunshine** を入れ、Mac側に **Moonlight** を入れます。

もともとゲーム配信向けの技術に近いため、リモートデスクトップ専用ツールよりも、画面描画が軽く感じる可能性があります。

## 構成

```text
MacBook
  └ Moonlight
      ↓ Tailscale
Windows PC
  └ Sunshine
```

## Sunshine + Moonlightのメリット

```text
・低遅延
・高FPSに向いている
・画面描画が軽く感じやすい
・GPUエンコードを活用できる
・動画編集ソフトの画面操作と相性が良い可能性が高い
```

特に、Windows PCにNVIDIA GPUなどがある場合、ハードウェアエンコードを活用できるため、体感が良くなる可能性があります。

## Sunshine + Moonlightのデメリット

```text
・RustDeskより初期設定が少し面倒
・リモートサポート用途には向かない
・ファイル転送機能は主目的ではない
・UAC画面や管理者操作との相性には注意が必要
・通常のリモートデスクトップというより画面ストリーミングに近い
```

## 向いている用途

```text
・動画編集
・画面描画が多いアプリの操作
・低遅延でWindowsを操作したい
・GPUを活かして軽く使いたい
```

動画編集用途なら、RustDeskよりも先に試す価値があります。

---

# 3. NoMachine

## 概要

NoMachine は、汎用的なリモートデスクトップソフトです。

Windows、macOS、Linux に対応しており、個人利用で使いやすい選択肢です。

RustDeskよりも「リモートデスクトップ製品」寄りの印象です。

## 構成

```text
MacBook
  └ NoMachine Client
      ↓ Tailscale
Windows PC
  └ NoMachine Server
```

## NoMachineのメリット

```text
・汎用リモートデスクトップとしてまとまっている
・Windows Homeでも使える
・音声やファイル転送なども扱いやすい
・RustDeskの代替として試しやすい
・画質やレスポンスが良い場合がある
```

## NoMachineのデメリット

```text
・RustDeskより設定項目が多い
・Sunshine + Moonlightほど低遅延特化ではない
・軽量というより多機能寄り
```

## 向いている用途

```text
・RustDesk以外の汎用リモートデスクトップを試したい
・Windows操作全般を行いたい
・ファイル転送や音声なども使いたい
```

RustDeskが合わなかった場合の代替候補として良いです。

---

# 4. VNC系ツール

## 概要

VNCは古くからあるリモートデスクトップ方式です。

代表的なものには以下があります。

```text
・TigerVNC
・TightVNC
・UltraVNC
・RealVNC
```

Tailscaleと併用する場合は、VNCサーバーをインターネットに直接公開せず、Tailscale IP 経由で接続します。

## 構成

```text
MacBook
  └ VNC Viewer
      ↓ Tailscale
Windows PC
  └ VNC Server
```

通常、VNCは `5900` 番ポートを使います。

```text
Mac → WindowsのTailscale IP:5900
```

## VNCのメリット

```text
・仕組みがシンプル
・軽い管理作業には十分
・Tailscaleと組み合わせると閉じたネットワーク内で使いやすい
・多くのクライアント・サーバー実装がある
```

## VNCのデメリット

```text
・動画編集や高FPS用途には弱い
・音声やファイル転送は製品依存
・セキュリティ設定は自分で注意する必要がある
・画面描画が重い用途では厳しいことがある
```

## 向いている用途

```text
・軽い設定変更
・サーバー管理
・画面を少し触る程度の作業
```

動画編集にはあまり向いていません。

---

# 5. Chrome Remote Desktop

## 概要

Chrome Remote Desktop は、Googleアカウントを使って簡単にリモート接続できるツールです。

Tailscaleを使わなくても接続できます。

## 構成

```text
MacBook
  └ Chrome Remote Desktop
      ↓ Googleアカウント経由
Windows PC
  └ Chrome Remote Desktop
```

## Chrome Remote Desktopのメリット

```text
・設定が簡単
・Googleアカウントで使える
・Tailscaleなしでも使える
・非常用の接続手段として便利
```

## Chrome Remote Desktopのデメリット

```text
・Tailscaleのプライベートネットワークを活かす構成ではない
・低遅延や動画編集向けではない
・Googleアカウントに依存する
・細かい画質や接続制御は弱め
```

## 向いている用途

```text
・非常用のバックアップ経路
・簡単に接続したい場合
・細かい設定をしたくない場合
```

メイン用途というより、バックアップ用として用意しておくと安心です。

---

# RustDeskとの比較

## RustDesk vs Sunshine + Moonlight

| 観点 | RustDesk | Sunshine + Moonlight |
|---|---|---|
| 軽さ | 中〜高 | 高い |
| 低遅延 | 普通〜良い | 強い |
| 動画編集 | 可能 | より向いている |
| 設定の簡単さ | 簡単 | やや手間 |
| ファイル転送 | あり | 主目的ではない |
| 管理操作 | 向いている | やや弱い |

動画編集や低遅延操作なら、Sunshine + Moonlight が有利です。

---

## RustDesk vs NoMachine

| 観点 | RustDesk | NoMachine |
|---|---|---|
| 設定の簡単さ | 簡単 | 中程度 |
| 汎用性 | 高い | 高い |
| ファイル転送 | あり | あり |
| 軽さ | 中〜高 | 中〜高 |
| 代替候補として | 基準 | 良い候補 |

NoMachineは、RustDeskが合わなかった場合の有力な代替です。

---

## RustDesk vs VNC

| 観点 | RustDesk | VNC |
|---|---|---|
| 使いやすさ | 高い | 実装による |
| 軽い管理作業 | 良い | 良い |
| 動画編集 | 普通 | 弱い |
| セキュリティ | RustDesk側にまとまっている | 設定次第 |
| Tailscale併用 | 良い | 良い |

VNCはシンプルですが、動画編集のような用途には向きません。

---

## RustDesk vs Chrome Remote Desktop

| 観点 | RustDesk | Chrome Remote Desktop |
|---|---|---|
| Tailscale併用 | 良い | 基本不要 |
| 設定の簡単さ | 簡単 | とても簡単 |
| 低遅延 | 中〜高 | 中 |
| Googleアカウント依存 | なし | あり |
| 非常用 | 良い | 良い |

Chrome Remote Desktopは非常用に向いています。

---

# 用途別おすすめ

## 動画編集を軽く操作したい

```text
Sunshine + Moonlight
```

理由：

```text
・低遅延
・高FPSに強い
・GPUエンコードを活用できる
・画面変化の多い操作に向いている
```

---

## 一般的なWindows操作をしたい

```text
RustDesk
```

理由：

```text
・設定が簡単
・機能バランスが良い
・ファイル転送も使える
・Tailscale IPで直接接続しやすい
```

---

## RustDesk以外の汎用リモートデスクトップを試したい

```text
NoMachine
```

理由：

```text
・汎用リモートデスクトップとしてまとまっている
・Windows/Mac/Linuxに対応
・RustDeskの代替として使いやすい
```

---

## 軽い管理作業だけしたい

```text
VNC系
```

理由：

```text
・構成がシンプル
・Tailscaleと組み合わせやすい
・軽い作業なら十分
```

---

## 非常用の接続経路が欲しい

```text
Chrome Remote Desktop
```

理由：

```text
・設定が簡単
・Tailscaleなしでも接続できる
・バックアップ経路として便利
```

---

# 最終的なおすすめ構成

今回の用途では、以下のように分けるのがよいです。

```text
開発:
Mac VS Code
  ↓ Remote SSH
WSL + Tailscale

Windows GUI操作:
Mac
  ↓ RustDesk or NoMachine
Windows

動画編集:
Mac
  ↓ Sunshine + Moonlight
Windows

非常用:
Chrome Remote Desktop
```

---

# まとめ

RustDeskは、Windows Homeで使えるリモートデスクトップとしてバランスが良い選択肢です。

ただし、動画編集のように画面変化が多く、低遅延が重要な用途では、**Sunshine + Moonlight** の方が軽く感じる可能性があります。

最初に試す順番としては、以下がおすすめです。

```text
1. Sunshine + Moonlight
2. RustDesk
3. NoMachine
4. Chrome Remote Desktop
```

特に、動画編集をMacから操作する目的であれば、**Sunshine + Moonlight** を第一候補にする価値があります。
