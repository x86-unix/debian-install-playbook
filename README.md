# Debian sid Install Playbook

Ansible playbook で Debian sid をインストールし、Wayland デスクトップ環境をセットアップする。

## 前提条件

- ターゲットマシンが SystemRescue 等のライブ環境で起動済み
- ライブ環境で SSH が利用可能
- コントロールノードに `ansible` と `sshpass` がインストール済み

## 構成

```
site-base.yml              # ベースインストール（debootstrap）
site-extra.yml             # 拡張セットアップ（reboot 後に実行）

extra/
  extra-account.yml        # sudo 設定・ユーザー作成
  extra-services.yml       # システムサービス（cron・自動 apt upgrade）
  extra-cli-tools.yml      # CLI ツール（p7zip, gh, tree 等）
  extra-sound.yml          # サウンド（pipewire + wireplumber + pulseaudio-utils）
  extra-bluetooth.yml      # Bluetooth（bluez + blueman、ハード検出付き）
  extra-wayland.yml        # Wayland デスクトップ（labwc + waybar + foot + wofi + wlogout）
  extra-ime.yml            # IME（fcitx5 + SKK）
  extra-remote-desktop.yml # リモートデスクトップ（wayvnc + noVNC + Apache HTTPS + OTP）
  extra-gui-tools.yml      # GUI ツール（Google Chrome, VSCode, yazi 等）

files/
  frieren-blue-flower-anime-wallpaper-4k.png

group_vars/all.yml         # 共通設定値
inventory.ini              # ターゲットホスト定義
```

## パーティション構成

| # | 用途 | サイズ | FS |
|---|------|--------|----|
| 1 | /boot/efi | 512MB | vfat |
| 2 | / | 残り全部 | ext4 |

## 使い方

### 1. 設定を編集

```bash
vi inventory.ini
vi group_vars/all.yml
```

主な設定値（`group_vars/all.yml`）：

| 変数 | 説明 |
|------|------|
| `target_disk` | インストール先ディスク（例: `/dev/sda`） |
| `efi_partition` | EFI パーティション（例: `/dev/sda1`） |
| `root_partition` | root パーティション（例: `/dev/sda2`） |
| `hostname` | ホスト名 |
| `timezone` | タイムゾーン（例: `Asia/Tokyo`） |
| `locale` | ロケール（例: `ja_JP.UTF-8`） |
| `root_password` | root パスワード（平文、vault 推奨） |
| `default_user` | 作成するユーザー名 |
| `default_user_password` | ユーザーパスワード（平文、vault 推奨） |
| `sudo_nopasswd` | `true` で NOPASSWD sudo を有効化 |
| `wifi_ssid` | WiFi SSID（WiFi NIC 検出時のみ使用、vault 推奨） |
| `wifi_password` | WiFi パスワード（vault 推奨） |

### 2. ベースインストール

ライブ環境から実行する：

```bash
ansible-playbook -i inventory.ini site-base.yml
```

debootstrap で Debian sid を展開し、以下を設定する：

- fstab / hostname / hosts / sources.list
- タイムゾーン・ロケール
- カーネル・GRUB（EFI）
- NetworkManager（有線・WiFi）
- SSH（root ログイン許可）

完了後、マシンを再起動して USB を抜く。

### 3. 拡張セットアップ（reboot 後）

```bash
ansible-playbook -i inventory.ini site-extra.yml
```

実行順序：

1. ユーザーアカウント・sudo 設定
2. システムサービス（cron・自動 apt upgrade 03:00）
3. CLI ツール（p7zip, gh, tree）
4. サウンド（pipewire + wireplumber）
5. Bluetooth（bluez + blueman、ハード検出付き）
6. Wayland デスクトップ（labwc + waybar + foot + wofi + wlogout）
7. IME（fcitx5 + SKK）
8. リモートデスクトップ（wayvnc + noVNC + Apache HTTPS + OTP）
9. GUI ツール（Google Chrome, VSCode, yazi 等）

### 4. 個別実行

```bash
ansible-playbook -i inventory.ini extra/extra-wayland.yml
ansible-playbook -i inventory.ini extra/extra-remote-desktop.yml
```

## Bluetooth

- `extra-bluetooth.yml` 実行時にまず BT ハードウェアを検出（`lspci` / `lsusb` / `rfkill`）
- ハードが見つからない場合はメッセージを出して後続タスクをすべてスキップ（playbook は正常終了）
- ハードが検出された場合: `bluez` + `blueman` をインストールし `bluetooth.service` を有効化

## リモートデスクトップ

- `extra-remote-desktop.yml` で wayvnc + noVNC + Apache（HTTPS リバースプロキシ + OTP 認証）をインストール
- labwc autostart で `wayvnc` が自動起動し、`novnc.service` が `127.0.0.1:6080` でブリッジを提供
- Apache が HTTPS(443) で公開し、OTP 認証（TOTP）を前段で実施する
- ブラウザから `https://<host>/` でアクセス（自己署名証明書のため初回警告あり）
- noVNC は `127.0.0.1:6080` にバインドされ、外部から直接アクセス不可

### OTP ユーザー登録

playbook 実行後、ターゲットホストで以下を実行する：

```bash
novnc-otp-register <username>
```

出力された QR コード URL をブラウザで開き、Google Authenticator 等に登録する。

```bash
# 登録済みユーザー確認
cat /var/www/otp/users
```

登録後、ブラウザから `https://<host>/` にアクセスし、ユーザー名と 6 桁のワンタイムパスワードを入力する。

- OTP セッションは 8 時間有効（`OTPAuthMaxLinger 28800`）
- 証明書を Let's Encrypt 等に差し替える場合は `/etc/ssl/apache2/server.{crt,key}` を置き換えて `systemctl restart apache2`

### ポートバインド確認

```bash
ss -ltpn | grep -E ':(443|6080|5900)'
```

正常な状態：

```
0.0.0.0:443    ← Apache（外部公開、HTTPS + OTP）
127.0.0.1:6080 ← novnc（ローカルのみ）
127.0.0.1:5900 ← wayvnc（ローカルのみ）
```

### mod_authn_otp のビルド

`libapache2-mod-authn-otp` は Debian に存在しないため、ソースからビルドする：

```
/usr/local/src/mod-authn-otp  ← ソース
/usr/lib/apache2/modules/mod_authn_otp.so  ← インストール先
```

ビルド依存: `apache2-dev`, `liboath-dev`, `libssl-dev`, `pkg-config`, `autoconf`, `automake`, `libtool`

## WiFi

- WiFi NIC（`wlan*`, `wlp*`）が検出された場合のみ NetworkManager の接続設定を作成
- パスワードは `ansible-vault` で暗号化推奨：

```bash
ansible-vault encrypt_string 'your_password' --name 'wifi_password'
```

## 注意事項

- `group_vars/all.yml` に root パスワードとユーザーパスワードが平文で保存される。必要に応じて `ansible-vault` で暗号化すること
- `.vault_pass` ファイルに vault パスワードを記述しておくと `--vault-password-file .vault_pass` で復号できる
- `extra-gui-tools.yml` の yazi は `debian.griffo.io`（非公式リポジトリ）からインストールする
- `extra-wayland.yml` の `greetd` ユーザーは Debian パッケージが自動作成しないため playbook 内で作成している
