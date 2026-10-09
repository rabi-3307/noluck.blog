+++
title = "【番外編1】www統一編 — wwwあり・なしのURLを1つにまとめる"
description = "wwwありとwwwなしのURLを1つにまとめる手順です。Nginxの設定で自動転送し、canonicalタグを入れて、Googleの評価が2つに分かれるのを防ぎます。"
date = 2026-09-25T09:11:00+09:00
lastmod = 2026-10-09T09:00:00+09:00
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

## 実際に起きたこと:Search Consoleで「重複」と言われた

この設定をする前の9月上旬、このブログは `www` ありでもなしでも、同じページがそのまま表示される状態でした。そのころにGoogleが見に来たページが、後になってSearch Consoleの「ページのインデックス登録」にこう出てきました。

```
重複しています。ユーザーにより、正規ページとして選択されていません(8ページ)
```

開いてみると、8ページとも頭に `www.` が付いたURLで、Googleが見に来た日付は9月5日〜8日でした。

```
https://www.noluckblog.com/posts/…/
https://www.noluckblog.com/categories/beginners/
http://www.noluckblog.com/posts/…/
```

「同じ中身が2つのURLにあるので、Googleが本物を `www` なしの方に決めた」という意味です。本物にしてほしい方が選ばれているので、結果としては正しい状態でした。

念のため、今は `www` ありで開くと転送されるかを確かめました。

```
curl.exe -I https://www.noluckblog.com/
HTTP/1.1 301 Moved Permanently
Location: https://noluckblog.com/

curl.exe -I http://www.noluckblog.com/
HTTP/1.1 301 Moved Permanently
Location: https://noluckblog.com/
```

`https://` でも `http://` でも、`301` で `www` なしに転送されていました。この状態なら、Googleがもう一度見に来たときに「リダイレクトがあるページ」の方に移り、「重複」の数は減っていきます。

一度「修正を検証」を押したときは「失敗しました」になりました。転送の設定をする前の状態で見に来られていたためだと考えています。設定が効いているのを確かめてから、1〜2週間おいて押し直すのがおすすめです。

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

## あわせて読みたい

- [【第5回】環境づくり編 — SSH・ufw・Nginx・Hugoを準備する](/posts/series-05-kankyou/) — Nginxの設定ファイルを1行ずつ読む解説つき

## 次の記事

[【番外編2】GitHub連携編 — git pushだけで公開まで自動化する](/posts/series-ex2-github/)
