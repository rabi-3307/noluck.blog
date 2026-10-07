+++
title = "【第5回】環境づくり編 — SSH・ufw・Nginx・Hugoを準備する"
description = "VPSにSSHで入り、サーバーの更新、Nginxのインストール、ufwの設定、鍵認証への切り替えまでを行います。Nginxの設定ファイルを1行ずつ読み解き、実際にハマった落とし穴も解説します。"
date = 2026-09-25T09:05:00+09:00
lastmod = 2026-10-06T10:00:00+09:00
categories = ["beginners", "guide"]
aliases = ["/posts/nginx-config-yomitoku/", "/posts/ssh-key-auth/"]
+++

この記事を読み終えると、サーバーにSSHで入れて、ブラウザでNginxのページが表示され、手元のPCでHugoが使える状態になります。

コマンドの `xxx.xxx.xxx.xxx` はサーバーのIPアドレス、`example.com` は自分のドメインに置き換えてください。Hugo・SSH・Nginxがそれぞれ何をするものかは [第2回](/posts/series-02-kizai/) にまとめています。

## 操作手順

### 1. SSHでサーバーに入る

PowerShellで打ちます。

```
ssh root@xxx.xxx.xxx.xxx
```

初回は `Are you sure you want to continue connecting` と聞かれるので `yes`。続けてrootパスワードを入力します(入力中は何も表示されませんが、打てています)。

![初めてSSHで接続したときの画面](/images/posts/ssh-first-login.png "初めて接続したときの画面(IPアドレスなどは隠しています)。①で yes と打ち、②でrootパスワードを入力します。③の Welcome to Ubuntu が出れば接続できています")

`Connection timed out` で入れない場合は、ConoHaのセキュリティグループを確認してください。筆者はここで丸1日ハマりました。→ [ConoHa VPSでSSH接続できずハマった話](/posts/ssh-conoha-hamatta/)

### 2. サーバーを最新にする

ここからはサーバーの中で打ちます。

```
apt update && apt upgrade -y
```

### 3. Nginxを入れる

```
apt install nginx -y
systemctl status nginx
```

`active (running)` と出ればOK。ブラウザで `http://xxx.xxx.xxx.xxx` を開き、「Welcome to nginx!」が出れば成功です。

### 4. ファイアウォール(ufw)を設定する

```
ufw allow 22/tcp
ufw allow 80/tcp
ufw allow 443/tcp
ufw enable
ufw status
```

`22`(SSH)、`80`(HTTP)、`443`(HTTPS)が `ALLOW` になっていればOKです。

ConoHaの管理画面のセキュリティグループとは別の「サーバーの中の関所」なので、**両方で許可しないと通信は届きません。**

### 5. SSHを鍵認証にする

**手元のPC**のPowerShellで鍵を作ります。質問はすべてEnterでOKです。

```
ssh-keygen -t ed25519
```

公開鍵をサーバーに登録します。

```
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh root@xxx.xxx.xxx.xxx "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

もう一度 `ssh root@xxx.xxx.xxx.xxx` で入り、**パスワードを聞かれなければ成功**です。

最後に、サーバーでパスワードログインを止めます。

```
nano /etc/ssh/sshd_config
```

`PasswordAuthentication` の行を `PasswordAuthentication no` にして保存(Ctrl+O → Enter → Ctrl+X)し、再起動します。

```
systemctl restart ssh
```

### 6. サイトの置き場所とNginxの設定を作る

```
mkdir -p /var/www/yourlog/public
nano /etc/nginx/sites-available/yourlog
```

次の内容を貼って保存します。

```
server {
    listen 80;
    server_name example.com www.example.com;
    root /var/www/yourlog/public;
    index index.html;
}
```

設定を有効にして、確認してから反映します。

```
ln -s /etc/nginx/sites-available/yourlog /etc/nginx/sites-enabled/
rm /etc/nginx/sites-enabled/default
nginx -t
systemctl reload nginx
```

`nginx -t` で `syntax is ok` と `test is successful` が出てから `reload` してください。

### 7. 手元のPCでHugoのサイトを作る

第2回でHugoを入れていれば、PowerShellで作れます。

```
hugo new site mysite
cd mysite
```

見た目(テーマ)はHugo公式サイトから選ぶか自作します。このブログはHTML/CSSで自作しました。

## Nginxの設定ファイルを1行ずつ読む

手順6で貼った設定は、最初は筆者もコピペで使っていました。短いですが、1行ずつにはっきりした役割があります。

### `server { ... }`

「ここから1つのWebサイトの設定です」という枠です。1台のサーバーに `server { }` を複数並べれば、複数のサイトを動かすこともできます。

### `listen 80;`

**どのポート番号で通信を待つか**です。`80` はHTTP(暗号化なし)の決まった番号です。第6回でHTTPS化すると、Certbotが `listen 443 ssl;` という行を自動で足してくれます。`443` はHTTPS(暗号化あり)の番号です。

### `server_name example.com www.example.com;`

**どのドメイン名で来たときに、この設定を使うか**です。ここに書いていない名前で来たアクセスには、この設定は使われません。`www` ありとなしを両方書いているのは、どちらで来ても同じサイトを見せるためです(番外編1で、1つにまとめます)。

### `root /var/www/yourlog/public;`

**配るファイルが置いてある場所**です。誰かが `https://example.com/posts/abc/` を開くと、Nginxは `/var/www/yourlog/public/posts/abc/index.html` を探しに行きます。第7回で、Hugoが作った `public` フォルダの中身をここに送ります。

### `index index.html;`

**フォルダだけを指定されたときに、代わりに返すファイル名**です。`https://example.com/` のようにファイル名がないアクセスには、`index.html` を探して返します。

### ufw・セキュリティグループとの関係

`listen 80;` で待っていても、そのポートがConoHaのセキュリティグループと、サーバーの中のufwの**両方**で許可されていないと、外からの通信はそもそもNginxまで届きません。Nginxの設定は「届いた通信をどう扱うか」のルールで、「届くかどうか」は別の層の話です。この切り分けができるようになってから、トラブルの原因を探すのがかなり速くなりました。

### 書き換えたら必ず `nginx -t`

設定を書き換えたら、毎回この順番で打つようにしています。

```
nginx -t
systemctl reload nginx
```

`nginx -t` は「この設定ファイル、文法は正しい?」を確認するコマンドです。確認せずに反映すると、書き間違い1つでサイト全体が表示されなくなることがあります。

## つまずきやすい注意点

**SSHが `Connection timed out` になる**

ConoHaのセキュリティグループに `IPv4v6-SSH` が付いているか確認してください。サーバーの中の設定ではなく、管理画面側の問題です。→ [ConoHa VPSでSSH接続できずハマった話](/posts/ssh-conoha-hamatta/)

**ufwを有効にする前に22番を許可する**

順番を逆にすると、SSHで入れなくなって締め出されます。締め出されたらConoHaの管理画面の「コンソール」から入って直せます。

**パスワードログインを止める前に、鍵で入れることを確認する**

今のSSH画面は閉じずに、別のPowerShellで鍵ログインを試してください。失敗したまま止めると入れなくなります。

また、Ubuntu 24.04では `/etc/ssh/sshd_config.d/` の中のファイルにも `PasswordAuthentication yes` が書かれていて、そちらが優先されることがあります。止めたはずなのにパスワードを聞かれる場合はそこを確認してください。

**設定ファイルにコピペすると、文字が勝手に変わることがある**

チャットやメモアプリからコピーすると、`www.example.com` が `[www.example.com](http://www.example.com)` のようなリンクの書き方に化けて貼られることがあります。筆者は実際にこれで `nginx -t` が通らなくなりました。貼ったあとに中身を目で確認してください。→ [CSSが404になった原因はフォルダの権限だった](/posts/css-404-permission/)

**nanoの `.save` ファイルを残さない**

nanoが途中で閉じると、`yourlog.save` のようなファイルが残ることがあります。`sites-enabled` の中に残っていると、それも設定として読まれてエラーになります。`ls /etc/nginx/sites-enabled/` で余計なファイルがないか確認してください。

**ページが 403 Forbidden になる**

ファイルの読み取り権限が足りていません。

```
chmod -R 755 /var/www/yourlog
```

権限の仕組みと、筆者が何度もハマった話は [CSSが404になった原因はフォルダの権限だった](/posts/css-404-permission/) にまとめています。

**秘密鍵は絶対に人に渡さない**

`id_ed25519`(`.pub` が付いていない方)が秘密鍵です。渡してよいのは `.pub` が付いた公開鍵だけです。

## 次の記事

[【第6回】HTTPS編 — 無料の証明書で鍵マークを付ける](/posts/series-06-https/)
