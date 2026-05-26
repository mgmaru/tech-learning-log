# MacからWindows / WSLへSSH接続する方法

## 概要

この記事では、MacBook から Windows PC、および Windows 上の WSL へ SSH 接続する方法を整理します。

最終的に安定した方法は、**WSL に Tailscale を直接インストールし、Mac から WSL の Tailscale IP へ SSH 接続する構成**でした。

一方で、Windows 側の `portproxy` を使って WSL に転送する方法も試しましたが、WSL2 の内部 IP が変わることや、Windows から WSL への到達性が不安定になることがありました。そのため、失敗・検証記録として残します。

---

## 前提

この記事では、以下の環境を想定します。

- MacBook から Windows PC へ接続したい
- Windows には WSL2 が入っている
- Windows 本体へは OpenSSH Server で SSH 接続できる
- Tailscale を使用する
- Windows は Home エディションでもよい
- 開発用途では、Mac 側の VS Code から WSL に接続したい

---

## SSH接続の基本

SSH の接続形式は以下です。

```bash
ssh ユーザー名@接続先
```

例：

```bash
ssh user@100.x.x.x
```

ここで重要なのは、`@` の前は **マシン名ではなくユーザー名** という点です。

```text
ssh user@host
    ↑    ↑
    │    └ 接続先のIPアドレスまたはホスト名
    └ 接続先OS上のユーザー名
```

Windows のマシン名とユーザー名は別物です。

Windows 側で確認するには、以下を実行します。

```powershell
whoami
```

例：

```text
DESKTOP-XXXX\user
```

この場合、

```text
マシン名: DESKTOP-XXXX
ユーザー名: user
```

です。

---

# 方法1: MacからWindows本体へSSH接続する

## 構成

```text
MacBook
  ↓ SSH
Tailscale
  ↓
Windows PC
  └ OpenSSH Server
```

## Windows側でOpenSSH Serverを有効化する

Windows の PowerShell を管理者権限で開き、以下を実行します。

```powershell
Get-WindowsCapability -Online | Where-Object Name -like 'OpenSSH.Server*'
```

未インストールの場合は、以下でインストールします。

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
```

SSH サーバーを起動します。

```powershell
Start-Service sshd
Set-Service -Name sshd -StartupType Automatic
```

状態確認：

```powershell
Get-Service sshd
```

`Running` ならOKです。

## Tailscale IPを確認する

Windows 側で以下を実行します。

```powershell
tailscale ip -4
```

例：

```text
100.80.124.76
```

## Macから接続する

Mac 側で以下を実行します。

```bash
ssh Windowsユーザー名@WindowsのTailscale_IP
```

例：

```bash
ssh user@100.80.124.76
```

---

# Mac側のSSH configを使う

毎回 IP アドレスやユーザー名を入力するのは面倒です。  
Mac 側の `~/.ssh/config` に登録しておくと便利です。

```bash
nano ~/.ssh/config
```

例：

```sshconfig
Host win
  HostName 100.80.124.76
  User user
  Port 22
```

以後は以下だけで接続できます。

```bash
ssh win
```

---

# 方法2: WindowsのportproxyでWSLへSSH転送する方法

この方法は検証しましたが、最終的には不安定でした。  
ただし、仕組みを理解するうえでは重要なので記録します。

## 構成

```text
MacBook
  ↓ ssh user@Windows_Tailscale_IP -p 2222
Tailscale
  ↓
Windows PC:2222
  ↓ portproxy
WSL:22
```

図にすると以下のイメージです。

```text
[MacBook]
    |
    | SSH 2222
    v
[Windows Tailscale IP:2222]
    |
    | portproxy
    v
[WSL IP:22]
```

## なぜ2222番を使うのか

SSH の標準ポートは `22` です。  
Windows 本体の OpenSSH Server がすでに `22` を使っている場合、WSL 用に同じポートは使えません。

そのため、Windows 側に別の入口として `2222` を用意します。

```text
Windows本体SSH: 22
WSL用SSH入口: 2222
```

## WSL側でSSHサーバーを起動する

WSL 内で以下を実行します。

```bash
sudo apt update
sudo apt install -y openssh-server
sudo service ssh start
```

待ち受け確認：

```bash
ss -tlnp | grep :22
```

以下のように表示されれば、WSL 側の SSH サーバーは起動しています。

```text
LISTEN 0 128 0.0.0.0:22
LISTEN 0 128 [::]:22
```

## WSLのIPを確認する

WSL 内で以下を実行します。

```bash
hostname -I
```

例：

```text
172.27.250.45 172.17.0.1 172.18.0.1 ...
```

この場合、通常は一番左の IP を使います。

```text
172.27.250.45
```

複数表示される場合、Docker などのブリッジ IP が混ざっていることがあります。

## Windows側でportproxyを設定する

Windows の管理者コマンドプロンプトで実行します。

```cmd
netsh interface portproxy add v4tov4 listenaddress=0.0.0.0 listenport=2222 connectaddress=172.27.250.45 connectport=22
```

確認：

```cmd
netsh interface portproxy show all
```

表示例：

```text
Listen on ipv4:             Connect to ipv4:

Address         Port        Address         Port
--------------- ----------  --------------- ----------
0.0.0.0         2222        172.27.250.45   22
```

## Windows Firewallで2222番を許可する

Tailscale 経由だけ許可する場合は、以下のようにします。

```cmd
netsh advfirewall firewall add rule name="WSL SSH 2222" dir=in action=allow protocol=TCP localport=2222 remoteip=100.64.0.0/10 profile=any
```

これは、Tailscale の IP 範囲から来た TCP 2222 番への通信だけを許可する設定です。

## Mac側のSSH config

```sshconfig
Host wsl
  HostName 100.80.124.76
  User wsluser
  Port 2222
```

接続：

```bash
ssh wsl
```

## portproxy方式で発生した問題

この方式では、以下の問題が発生しました。

```text
Windows → WSLのIP:22 に接続できたり、できなくなったりする
WSLのIPが変わる
portproxyのconnectaddressを更新する必要がある
Windows FirewallやIP Helperサービスにも依存する
```

特に、Windows から WSL の IP に対して以下のように接続したとき、

```cmd
ssh wsluser@172.27.250.45
```

`Connection timed out` になることがありました。

この場合、Mac や Tailscale の問題ではなく、Windows から WSL 内部 IP への到達性が不安定になっている状態です。

そのため、最終的にはこの方式は採用しませんでした。

---

# 方法3: WSLにTailscaleを直接入れる方法

最終的に安定したのは、この方法です。

## 構成

```text
MacBook
  ↓ SSH
Tailscale
  ↓
WSL Ubuntu
  └ OpenSSH Server
```

図にすると、portproxy方式よりシンプルです。

```text
[MacBook]
    |
    | SSH 22
    v
[WSLのTailscale IP:22]
```

Windows を中継せず、WSL 自体を Tailscale の1台の端末として扱います。

---

## WSLでsystemdを有効化する

WSL 内で以下を開きます。

```bash
sudo nano /etc/wsl.conf
```

以下を追加します。

```ini
[boot]
systemd=true
```

保存後、Windows 側で WSL を停止します。

```cmd
wsl --shutdown
```

その後、WSL を再起動します。

```cmd
wsl
```

systemd が有効になっているか確認します。

```bash
ps -p 1 -o comm=
```

以下のように表示されればOKです。

```text
systemd
```

---

## WSLにTailscaleをインストールする

WSL 内で実行します。

```bash
curl -fsSL https://tailscale.com/install.sh | sh
```

Tailscale を起動します。

```bash
sudo tailscale up
```

表示されたURLをブラウザで開いて認証します。

認証後、状態確認します。

```bash
tailscale status
```

WSL の Tailscale IP を確認します。

```bash
tailscale ip -4
```

例：

```text
100.xxx.xxx.xxx
```

この IP が、Mac から WSL に直接 SSH するための接続先です。

---

## WSL側でSSHサーバーを起動する

WSL 内で以下を実行します。

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

確認：

```bash
sudo systemctl status ssh
```

または：

```bash
ss -tlnp | grep :22
```

---

## MacからWSLへ直接SSHする

Mac 側で以下を実行します。

```bash
ssh WSLユーザー名@WSLのTailscale_IP
```

例：

```bash
ssh wsluser@100.xxx.xxx.xxx
```

接続後、以下で Linux に入っていることを確認できます。

```bash
uname -a
```

---

## Mac側のSSH config

Mac の `~/.ssh/config` に登録します。

```sshconfig
Host win
  HostName 100.80.124.76
  User windowsuser
  Port 22

Host wsl
  HostName 100.xxx.xxx.xxx
  User wsluser
  Port 22
```

これで、Windows 本体には：

```bash
ssh win
```

WSL には：

```bash
ssh wsl
```

で接続できます。

---

# VS CodeからWSLへ接続する

VS Code 本家に Microsoft 公式の **Remote - SSH** 拡張を入れます。

VS Code の `settings.json` に以下を設定します。

```json
{
  "remote.SSH.remotePlatform": {
    "win": "windows",
    "wsl": "linux"
  },
  "remote.SSH.showLoginTerminal": true
}
```

接続します。

```text
Cmd + Shift + P
→ Remote-SSH: Connect to Host...
→ wsl
```

接続できたら、WSL 内のプロジェクトフォルダを開きます。

例：

```text
/home/wsluser/projects
```

---

# Antigravityについて

Antigravity では、VS Code 本家の Remote - SSH と同じようには動作しませんでした。

特に、Windows ホストに対して Linux 用の `bash -s` を送ってしまう挙動がありました。  
そのため、Remote SSH で安定して開発する場合は、現時点では **VS Code 本家 + Microsoft公式 Remote - SSH** を使う方が安全です。

---

# 最終的なおすすめ構成

用途ごとに分けるのが安定します。

```text
開発:
Mac VS Code
  ↓ Remote SSH
WSL + Tailscale

Windows本体管理:
Mac Terminal
  ↓ SSH
Windows + Tailscale

Windows GUI操作・動画編集:
Mac
  ↓ RustDesk + Tailscale
Windows
```

## 開発用

```text
MacBook
  ↓ VS Code Remote SSH
WSLのTailscale IP
```

## Windows操作用

```text
MacBook
  ↓ RustDesk
Windows PC
```

---

# まとめ

Mac から Windows 上の WSL へ SSH 接続する方法として、以下を試しました。

| 方法 | 結果 |
|---|---|
| Windows本体へSSH | 成功 |
| Windows portproxyでWSLへ転送 | 不安定 |
| WSLにTailscaleを直接入れる | 成功・安定 |

最終的には、**WSLにTailscaleを直接入れる方法が最も安定**しました。

この構成では、WSL2 の内部 IP 変更や Windows の `portproxy` に依存しないため、Mac の VS Code から WSL へ直接接続できます。

```text
MacBook
  ↓ Tailscale
WSL Ubuntu
```

WSL 上で開発する場合は、この構成が最もシンプルで安定しています。
