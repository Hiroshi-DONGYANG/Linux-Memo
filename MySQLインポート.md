# さくらのVPS に MySQL をインストールする手順

さくらのVPS（Ubuntu 24.04 LTS）に **MySQL 8.4 LTS** をインストールし、安全に使える状態にするまでの手順です。
Rocky Linux / AlmaLinux を使う場合は [付録A](#付録a-rocky-linux--almalinux-9-の場合) を参照してください。

> [!NOTE]
> - 対象 OS：Ubuntu 24.04 LTS（さくらのVPS 標準OSインストール）
> - 対象 MySQL：MySQL 8.4 LTS（MySQL 公式 APT リポジトリ）
> - 最終確認日：2026-11-11
> - コマンド中の `<...>` は自分の環境に合わせて置き換えてください。

---

## 目次

1. [前提条件](#1-前提条件)
2. [サーバーの初期準備](#2-サーバーの初期準備)
3. [MySQL 公式 APT リポジトリの追加](#3-mysql-公式-apt-リポジトリの追加)
4. [MySQL のインストール](#4-mysql-のインストール)
5. [初期セキュリティ設定](#5-初期セキュリティ設定)
6. [データベースとユーザーの作成](#6-データベースとユーザーの作成)
7. [設定ファイルの調整（小さいプラン向け）](#7-設定ファイルの調整小さいプラン向け)
8. [外部から接続する場合](#8-外部から接続する場合)
9. [バックアップ](#9-バックアップ)
10. [動作確認・よく使うコマンド](#10-動作確認よく使うコマンド)
11. [トラブルシューティング](#11-トラブルシューティング)
- [付録A: Rocky Linux / AlmaLinux 9 の場合](#付録a-rocky-linux--almalinux-9-の場合)
- [付録B: Ubuntu 標準リポジトリで手軽に入れる場合](#付録b-ubuntu-標準リポジトリで手軽に入れる場合)
- [参考リンク](#参考リンク)

---

## 1. 前提条件

| 項目 | 内容 |
| --- | --- |
| サーバー | さくらのVPS（メモリ 1GB 以上推奨。512MB でも動作はするが[7章](#7-設定ファイルの調整小さいプラン向け)の調整が必須） |
| OS | Ubuntu 24.04 LTS |
| ログインユーザー | `ubuntu`（さくらのVPS 標準OSの初期ユーザー。root で直接ログインはできない） |
| 接続方法 | ローカル PC から SSH で接続できること |

```bash
# ローカル PC から接続
ssh ubuntu@<サーバーのIPアドレス>
```

---

## 2. サーバーの初期準備

### 2-1. パッケージを最新化

```bash
sudo apt update
sudo apt -y upgrade
```

カーネル更新があった場合は再起動します。

```bash
# 再起動が必要か確認（ファイルがあれば再起動が必要）
ls /var/run/reboot-required && sudo reboot
```

### 2-2. タイムゾーンを日本時間に設定

ログやバックアップの時刻を分かりやすくするため、JST にしておきます。

```bash
sudo timedatectl set-timezone Asia/Tokyo
timedatectl
```

### 2-3. 必要なツールをインストール

```bash
sudo apt install -y wget gnupg lsb-release
```

---

## 3. MySQL 公式 APT リポジトリの追加

Ubuntu 24.04 の標準リポジトリでは MySQL **8.0** が入ります。長期サポート版の **8.4 LTS** を使うため、MySQL 公式の APT リポジトリを追加します。

### 3-1. 最新の設定パッケージ名を確認

[MySQL APT Repository のダウンロードページ](https://dev.mysql.com/downloads/repo/apt/) を開き、`mysql-apt-config_X.X.XX-X_all.deb` のファイル名を確認します。

> [!IMPORTANT]
> 古い `mysql-apt-config` を使うと、署名鍵の期限切れで `apt update` 時に `NO_PUBKEY` エラーになることがあります。**必ずダウンロードページで最新版を確認**してください。

### 3-2. ダウンロードとインストール

```bash
# ダウンロードページで確認したバージョンに置き換える
MYSQL_APT_CONFIG=mysql-apt-config_0.8.36-1_all.deb

cd /tmp
wget https://dev.mysql.com/get/${MYSQL_APT_CONFIG}
sudo dpkg -i ${MYSQL_APT_CONFIG}
```

設定画面が表示されたら以下を確認して `Ok` を選択します。

```
MySQL Server & Cluster (Currently selected: mysql-8.4-lts)
MySQL Tools & Connectors (Currently selected: Enabled)
Ok
```

`mysql-8.4-lts` 以外になっている場合は、その行を選んで `mysql-8.4-lts` に変更してください。

<details>
<summary>あとから選択を変更したい場合</summary>

```bash
sudo dpkg-reconfigure mysql-apt-config
sudo apt update
```

</details>

### 3-3. パッケージ情報を更新

```bash
sudo apt update
```

`repo.mysql.com` からパッケージ情報が取得されていれば OK です。

```bash
# 8.4 系がインストール候補になっていることを確認
apt-cache policy mysql-server
```

---

## 4. MySQL のインストール

```bash
sudo apt install -y mysql-server
```

インストール中に以下を聞かれます。

| 画面 | 入力内容 |
| --- | --- |
| Enter root password | MySQL の root パスワード（強固なものを設定し、パスワードマネージャー等で管理） |
| Re-enter root password | 同じパスワードを再入力 |

### 4-1. 起動確認と自動起動の設定

```bash
sudo systemctl status mysql
sudo systemctl enable mysql
```

`Active: active (running)` と表示されれば起動しています（`q` で終了）。

### 4-2. バージョン確認

```bash
mysql --version
# 例: mysql  Ver 8.4.x for Linux on x86_64 (MySQL Community Server - GPL)
```

---

## 5. 初期セキュリティ設定

```bash
sudo mysql_secure_installation
```

推奨の回答は以下のとおりです。

| 質問 | 推奨 | 説明 |
| --- | --- | --- |
| Enter password for user root | （4章で設定したパスワード） | |
| VALIDATE PASSWORD component を使うか | `y` | パスワード強度チェックを有効化 |
| パスワードポリシーのレベル | `1`（MEDIUM）以上 | 8文字以上・大小英字・数字・記号 |
| root パスワードを変更するか | `n` | 設定済みのため |
| 匿名ユーザーを削除するか | `y` | |
| root のリモートログインを禁止するか | `y` | root はローカルからのみ使う |
| test データベースを削除するか | `y` | |
| 権限テーブルを再読み込みするか | `y` | |

ログインできることを確認します。

```bash
mysql -u root -p
```

```sql
SELECT VERSION();
EXIT;
```

---

## 6. データベースとユーザーの作成

アプリケーションからは root を使わず、専用ユーザーを作成します。

```bash
mysql -u root -p
```

```sql
-- データベース作成（絵文字も扱える utf8mb4）
CREATE DATABASE <アプリ用DB名>
  CHARACTER SET utf8mb4
  COLLATE utf8mb4_0900_ai_ci;

-- ローカル接続専用ユーザーを作成
CREATE USER '<ユーザー名>'@'localhost' IDENTIFIED BY '<強固なパスワード>';

-- 作成したDBへの権限のみ付与
GRANT ALL PRIVILEGES ON <アプリ用DB名>.* TO '<ユーザー名>'@'localhost';

-- 確認
SHOW GRANTS FOR '<ユーザー名>'@'localhost';
EXIT;
```

作成したユーザーでログインできるか確認します。

```bash
mysql -u <ユーザー名> -p <アプリ用DB名>
```

> [!NOTE]
> MySQL 8.4 では認証方式の既定が `caching_sha2_password` で、`mysql_native_password` は既定で無効です。古いクライアントライブラリで接続エラーになる場合は、ライブラリを新しくしてください。

---

## 7. 設定ファイルの調整（小さいプラン向け）

MySQL は初期状態でもある程度メモリを使うため、さくらのVPS の 512MB〜1GB プランではメモリ不足で停止することがあります。プランに合わせて調整します。

### 7-1. 独自設定ファイルを作成

既存ファイルを直接編集せず、別ファイルで上書きします（アップデート時に設定が消えないようにするため）。

```bash
sudo nano /etc/mysql/mysql.conf.d/zz-custom.cnf
```

```ini
[mysqld]
# --- 文字コード ---
character-set-server = utf8mb4
collation-server     = utf8mb4_0900_ai_ci

# --- 接続 ---
# ローカルからのみ接続を受け付ける（外部接続する場合は8章参照）
bind-address = 127.0.0.1
max_connections = 50

# --- メモリ（プランに合わせて調整） ---
# 目安: 512MB → 64M / 1GB → 128M〜256M / 2GB → 512M / 4GB → 1G〜2G
innodb_buffer_pool_size = 128M

# 小さいプランでは performance_schema を無効にするとメモリを節約できる
performance_schema = OFF

# --- ログ ---
slow_query_log      = ON
slow_query_log_file = /var/log/mysql/mysql-slow.log
long_query_time     = 1

[client]
default-character-set = utf8mb4
```

### 7-2. 反映と確認

```bash
sudo systemctl restart mysql
sudo systemctl status mysql
```

```bash
mysql -u root -p -e "SHOW VARIABLES LIKE 'innodb_buffer_pool_size'; SHOW VARIABLES LIKE 'character_set_server';"
```

### 7-3. （任意）スワップの追加

メモリ 1GB 以下のプランでは、スワップがあるとメモリ不足による強制終了（OOM）を防ぎやすくなります。

```bash
# 既存のスワップを確認
free -h

# スワップが無い・少ない場合のみ（例: 1GB）
sudo fallocate -l 1G /swapfile
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile
echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
```

---

## 8. 外部から接続する場合

> [!WARNING]
> MySQL のポート（3306）をインターネットに公開するのは、総当たり攻撃の対象になるため推奨しません。
> **まずは 8-1 の SSH トンネルを検討してください。** ポート開放はどうしても必要な場合のみにします。

### 8-1. 推奨：SSH トンネル経由で接続

サーバー側の設定変更は不要です。ローカル PC で以下を実行します。

```bash
# ローカルの 13306 番ポートを、サーバーの 127.0.0.1:3306 に転送
ssh -N -L 13306:127.0.0.1:3306 ubuntu@<サーバーのIPアドレス>
```

別のターミナルや GUI ツール（MySQL Workbench、TablePlus、DBeaver など）から以下で接続します。

| 項目 | 値 |
| --- | --- |
| Host | `127.0.0.1` |
| Port | `13306` |
| User | 6章で作成したユーザー |

多くの GUI ツールには「SSH 経由で接続」の設定があるので、それを使っても同じです。

### 8-2. ポートを開放して接続する場合

接続元 IP を限定することが前提です。以下の **3か所** すべてで許可が必要です。

```
[接続元PC] → ① さくらのVPS パケットフィルター → ② UFW（OSのファイアウォール） → ③ MySQL の bind-address / ユーザーのホスト
```

#### ① さくらのVPS パケットフィルター

コントロールパネル → 対象サーバー → 「パケットフィルター」で、TCP 3306 を追加します。送信元を指定できる場合は、接続元の IP アドレスに限定してください。

> [!NOTE]
> 初期状態では SSH（22番）以外は拒否されています。パケットフィルターで許可していないと、OS 側でいくら設定しても `Connection timed out` になります。

#### ② UFW（使用している場合）

```bash
sudo ufw status
# 接続元 IP からの 3306 のみ許可
sudo ufw allow from <接続元IPアドレス> to any port 3306 proto tcp
```

#### ③ MySQL の設定

`/etc/mysql/mysql.conf.d/zz-custom.cnf` の `bind-address` を変更します。

```ini
bind-address = 0.0.0.0
```

```bash
sudo systemctl restart mysql
```

接続元 IP に限定したユーザーを作成します（`%` で全ホストを許可するのは避けます）。

```sql
CREATE USER '<ユーザー名>'@'<接続元IPアドレス>' IDENTIFIED BY '<強固なパスワード>';
GRANT ALL PRIVILEGES ON <アプリ用DB名>.* TO '<ユーザー名>'@'<接続元IPアドレス>';
```

ローカル PC から接続確認します。

```bash
mysql -h <サーバーのIPアドレス> -u <ユーザー名> -p
```

---

## 9. バックアップ

`mysqldump` で毎日バックアップを取り、古いものを自動削除する例です。

### 9-1. バックアップ用の認証情報ファイル

パスワードをスクリプトに直接書かないよう、権限を絞ったファイルに置きます。

```bash
sudo mkdir -p /root/.mysql
sudo nano /root/.mysql/backup.cnf
```

```ini
[client]
user=root
password=<rootのパスワード>
```

```bash
sudo chmod 600 /root/.mysql/backup.cnf
```

### 9-2. バックアップスクリプト

```bash
sudo nano /usr/local/bin/mysql-backup.sh
```

```bash
#!/bin/bash
set -euo pipefail

BACKUP_DIR=/var/backups/mysql
KEEP_DAYS=7
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "${BACKUP_DIR}"

mysqldump --defaults-extra-file=/root/.mysql/backup.cnf \
  --all-databases --single-transaction --routines --triggers --events \
  | gzip > "${BACKUP_DIR}/all_${DATE}.sql.gz"

# 保存期間を過ぎたファイルを削除
find "${BACKUP_DIR}" -name 'all_*.sql.gz' -mtime +${KEEP_DAYS} -delete
```

```bash
sudo chmod 700 /usr/local/bin/mysql-backup.sh
sudo /usr/local/bin/mysql-backup.sh   # 手動で1回実行して確認
ls -lh /var/backups/mysql
```

### 9-3. 毎日自動実行（cron）

```bash
sudo crontab -e
```

```cron
# 毎日 3:30 にバックアップ
30 3 * * * /usr/local/bin/mysql-backup.sh
```

> [!TIP]
> 同じサーバー内のバックアップだけでは、サーバー障害時に失われます。定期的に別の場所（ローカル PC、さくらのオブジェクトストレージなど）へもコピーしてください。

### 9-4. リストア

```bash
gunzip < /var/backups/mysql/all_YYYYMMDD_HHMMSS.sql.gz | mysql -u root -p
```

---

## 10. 動作確認・よく使うコマンド

```bash
# サービス操作
sudo systemctl status mysql
sudo systemctl restart mysql
sudo systemctl stop mysql
sudo systemctl start mysql

# ログ
sudo tail -n 100 /var/log/mysql/error.log
sudo tail -n 100 /var/log/mysql/mysql-slow.log

# 待ち受けポートの確認（127.0.0.1:3306 なら外部非公開）
sudo ss -tlnp | grep 3306

# メモリ使用量
free -h
```

```sql
-- ユーザー一覧
SELECT user, host, plugin FROM mysql.user;

-- データベース一覧
SHOW DATABASES;

-- 接続中のセッション
SHOW PROCESSLIST;
```

---

## 11. トラブルシューティング

<details>
<summary><code>apt update</code> で <code>NO_PUBKEY</code> / <code>is not signed</code> エラーになる</summary>

`mysql-apt-config` の署名鍵が古い可能性があります。[ダウンロードページ](https://dev.mysql.com/downloads/repo/apt/) から最新版を取得して入れ直してください。

```bash
cd /tmp
wget https://dev.mysql.com/get/<最新のmysql-apt-config>.deb
sudo dpkg -i <最新のmysql-apt-config>.deb
sudo apt update
```

</details>

<details>
<summary>MySQL が勝手に停止する / 起動しない</summary>

メモリ不足（OOM Killer）の可能性があります。

```bash
sudo journalctl -u mysql --no-pager -n 50
sudo dmesg | grep -i -E 'killed process|out of memory'
```

`Killed process ... (mysqld)` が出ていれば、[7章](#7-設定ファイルの調整小さいプラン向け) の `innodb_buffer_pool_size` を小さくする、`performance_schema = OFF` にする、スワップを追加する、などで対応します。

</details>

<details>
<summary>外部から接続すると <code>Connection timed out</code> になる</summary>

通信がどこかで遮断されています。以下の順に確認します。

1. さくらのVPS コントロールパネルのパケットフィルターで 3306 を許可しているか
2. `sudo ufw status` で 3306 を許可しているか
3. `sudo ss -tlnp | grep 3306` が `0.0.0.0:3306` になっているか

</details>

<details>
<summary><code>Host 'xxx' is not allowed to connect</code> / <code>Access denied</code> になる</summary>

ユーザーのホスト部分が接続元と一致していません。

```sql
SELECT user, host FROM mysql.user;
```

接続元 IP に合ったユーザーを作成してください（[8-2](#8-2-ポートを開放して接続する場合)）。

</details>

<details>
<summary>root のパスワードを忘れた</summary>

```bash
sudo systemctl stop mysql
sudo mkdir -p /var/run/mysqld && sudo chown mysql:mysql /var/run/mysqld
sudo mysqld_safe --skip-grant-tables --skip-networking &
mysql -u root
```

```sql
FLUSH PRIVILEGES;
ALTER USER 'root'@'localhost' IDENTIFIED BY '<新しいパスワード>';
EXIT;
```

```bash
sudo pkill mysqld
sudo systemctl start mysql
```

</details>

---

## 付録A: Rocky Linux / AlmaLinux 9 の場合

さくらのVPS で Rocky Linux / AlmaLinux 9 を選んだ場合の差分です。初期ユーザーは `rocky`（AlmaLinux は `almalinux`）です。

### A-1. MySQL 公式 Yum リポジトリを追加

[MySQL Yum Repository のダウンロードページ](https://dev.mysql.com/downloads/repo/yum/) で、EL9 向け `mysql84-community-release-el9-X.noarch.rpm` の最新ファイル名を確認します。

```bash
sudo dnf -y update
sudo dnf install -y https://dev.mysql.com/get/mysql84-community-release-el9-<X>.noarch.rpm

# OS 標準の mysql モジュールを無効化（公式版と競合させないため）
sudo dnf -y module disable mysql
```

### A-2. インストールと起動

```bash
sudo dnf install -y mysql-community-server
sudo systemctl enable --now mysqld
sudo systemctl status mysqld
```

### A-3. 初期 root パスワードの確認

Rocky/AlmaLinux 版では、インストール時に一時パスワードが自動生成されます。

```bash
sudo grep 'temporary password' /var/log/mysqld.log
```

この一時パスワードを使って `sudo mysql_secure_installation` を実行し、新しいパスワードを設定します。

### A-4. Ubuntu との主な違い

| 項目 | Ubuntu | Rocky / AlmaLinux |
| --- | --- | --- |
| サービス名 | `mysql` | `mysqld` |
| 設定ファイル | `/etc/mysql/mysql.conf.d/*.cnf` | `/etc/my.cnf` または `/etc/my.cnf.d/*.cnf` |
| エラーログ | `/var/log/mysql/error.log` | `/var/log/mysqld.log` |
| ファイアウォール | UFW | firewalld |

firewalld でポートを開放する場合（[8-2](#8-2-ポートを開放して接続する場合) の②の代わり）:

```bash
sudo firewall-cmd --permanent --add-rich-rule='rule family="ipv4" source address="<接続元IPアドレス>" port protocol="tcp" port="3306" accept'
sudo firewall-cmd --reload
```

---

## 付録B: Ubuntu 標準リポジトリで手軽に入れる場合

バージョンにこだわらず、最短で使いたい場合は Ubuntu 標準のパッケージでも構いません（Ubuntu 24.04 では MySQL 8.0 系）。

```bash
sudo apt update
sudo apt install -y mysql-server
sudo systemctl enable --now mysql
```

この場合、root はパスワードではなく **OS の root 権限（auth_socket）** でログインする設定になっています。

```bash
sudo mysql
```

以降は [5章](#5-初期セキュリティ設定) 以降と同じ手順で進められます。

> [!NOTE]
> MySQL 8.0 は 2026年4月にサポート終了（EOL）を迎えています。本番運用では 8.4 LTS の利用を推奨します。

---

## 参考リンク

- [MySQL 8.4 Reference Manual – Installing MySQL on Linux Using the MySQL APT Repository](https://dev.mysql.com/doc/refman/8.4/en/linux-installation-apt-repo.html)
- [MySQL APT Repository ダウンロード](https://dev.mysql.com/downloads/repo/apt/)
- [MySQL Yum Repository ダウンロード](https://dev.mysql.com/downloads/repo/yum/)
- [さくらのVPS マニュアル](https://manual.sakura.ad.jp/vps/)
