# debian-install-playbook

SystemRescue 起動環境から Debian sid をインストールする Ansible playbook。

## 対象環境

- ホスト: 192.168.77.92
- ブート: UEFI
- ディスク: /dev/sda (119.2GB SSD)
  - sda1: 512MB → /boot/efi (vfat)
  - sda2: 117.8GB → / (ext4)

## 使い方

```bash
ansible-playbook -i inventory.ini site.yml
```

インストール完了後、マシンを再起動して USB を抜く。
