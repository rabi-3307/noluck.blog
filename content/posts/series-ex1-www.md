+++
title = "【番外編1】www統一編 — wwwあり・なしのURLを1つにまとめる"
date = 2026-09-25T09:11:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、`www.example.com` にアクセスしても `example.com` に自動で転送され、Googleの評価が1つのURLにまとまる状態になります。

## なぜ必要か

Googleは `www.example.com` と `example.com` を別のページとして扱います。両方で同じ内容が見えていると「内容が重複したページ」とみなされ、検索の評価が2つに分かれてしまいます。

## 操作手順

このブログは「wwwなし」に統一しました。`example.com` は自分のドメインに置き換えてください。

### 1. 証明書に www が入っているか確認する

サーバーで打ちます。

```
certbot certificates
```

`Domains:` に `example.com` と `www.example.com` の両方があればOKです(第6回で両方指定していれば入っています)。

### 2. 設定ファイルのバックアップを取る

```
cp /etc/nginx/sites-available/yourlog /root/yourlog.bak
```

### 3. Nginxの設定を編集する

```
nano /etc/nginx/sites-available/yourlog
```

`listen 443 ssl` がある server ブロックの `server_name` から `www.example.com` を消し、`example.com` だけにします。

そのうえで、ファイルの一番下に次のブロックを追加します。

```
server {
    listen 443 ssl;
    server_name www.example.com;
    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;
    return 301 https://example.com$request_uri;
}
```

### 4. 確認してから反映する

```
nginx -t
systemctl reload nginx
```

### 5. 転送されているか確認する

手元のPowerShellで打ちます。

```
curl.exe -I https://www.example.com
```

`301` と `Location: https://example.com/` が出れば成功です。

### 6. canonical タグを入れる

`baseof.html` の `<head>` の中に1行追加します。「このページの正式なURLはこれです」とGoogleに伝えるタグです。

```
<link rel="canonical" href="{{ .Permalink }}">
```

保存して、第7回の手順で公開します。

## つまずきやすい注意点

**`nginx -t` が OK になるまで reload しない**

書き間違いのまま反映すると、サイト全体が表示されなくなることがあります。失敗したらバックアップから戻せます。

```
cp /root/yourlog.bak /etc/nginx/sites-available/yourlog
```

**ブラウザでの確認はあてにならない**

ブラウザは転送の結果を覚えてしまうので、設定を変えても前の動きのままに見えることがあります。`curl` かシークレットウィンドウで確認してください。

**301 を使う**

301は「引っ越しました(恒久)」、302は「一時的に移動中」という意味です。評価をまとめたいなら301です。

**すぐには検索結果に反映されない**

Googleが評価を1つにまとめるまで、数週間かかることがあります。

## 次の記事

[【番外編2】GitHub連携編 — git pushだけで公開まで自動化する](/posts/series-ex2-github/)
