+++
title = "【第5回】環境づくり編 — SSH・ufw・Nginx・Hugoを準備する"
date = 2026-09-25T09:05:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、サーバーにSSHで入れて、ブラウザでNginxのページが表示され、手元のPCでHugoが使える状態になります。

> **詳しい説明はこちら**
>
> この記事は操作だけに絞っています。仕組みを知りたい方はこちらへ。
>
> - [Hugo・VPS・SSH・Nginxの基礎](/posts/hugo-vps-ssh-nginx-kiso/)
> - [ufwとセキュリティグループの違い](/posts/ufw-vs-secgroup/)
> - [Nginxの設定ファイルを読み解く](/posts/nginx-config-yomitoku/)
> - [ConoHaでSSHがつながらなかった話](/posts/ssh-conoha-hamatta/)

コマンドの `xxx.xxx.xxx.xxx` はサーバーのIPアドレス、`example.com` は自分のドメインに置き換えてください。

## 操作手順

### 1. SSHでサーバーに入る

PowerShellで打ちます。

```
ssh root@xxx.xxx.xxx.xxx
```

初回は `Are you sure you want to continue connecting` と聞かれるので `yes`。続けてrootパスワードを入力します(入力中は何も表示されませんが、打てています)。

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

`22` `80` `443` が `ALLOW` になっていればOKです。

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

## つまずきやすい注意点

**SSHが `Connection timed out` になる**

ConoHaのセキュリティグループに `IPv4v6-SSH` が付いているか確認してください。サーバーの中の設定ではなく、管理画面側の問題です。→ [ConoHaでSSHがつながらなかった話](/posts/ssh-conoha-hamatta/)

**ufwを有効にする前に22番を許可する**

順番を逆にすると、SSHで入れなくなって締め出されます。締め出されたらConoHaの管理画面の「コンソール」から入って直せます。

**パスワードログインを止める前に、鍵で入れることを確認する**

今のSSH画面は閉じずに、別のPowerShellで鍵ログインを試してください。失敗したまま止めると入れなくなります。

また、Ubuntu 24.04では `/etc/ssh/sshd_config.d/` の中のファイルにも `PasswordAuthentication yes` が書かれていて、そちらが優先されることがあります。止めたはずなのにパスワードを聞かれる場合はそこを確認してください。

**ページが 403 Forbidden になる**

ファイルの読み取り権限が足りていません。

```
chmod -R 755 /var/www/yourlog
```

**秘密鍵は絶対に人に渡さない**

`id_ed25519`(`.pub` が付いていない方)が秘密鍵です。渡してよいのは `.pub` が付いた公開鍵だけです。

Nginxの設定の意味は [Nginxの設定ファイルを読み解く](/posts/nginx-config-yomitoku/) で解説しています。

## 次の記事

[【第6回】HTTPS編 — 無料の証明書で鍵マークを付ける](/posts/series-06-https/)
