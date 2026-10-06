+++
title = "【第6回】HTTPS編 — 無料の証明書で鍵マークを付ける"
description = "Let's EncryptとCertbotを使って、無料でサイトをHTTPS化する手順です。HTTPとHTTPSの違い、証明書の役割、「https://と書いただけ」で接続できなくなった体験もあわせて解説します。"
date = 2026-09-25T09:06:00+09:00
lastmod = 2026-10-06T10:00:00+09:00
categories = ["beginners"]
aliases = ["/posts/http-vs-https/"]
+++

この記事を読み終えると、サイトが `https://` で表示され、ブラウザで「この接続は保護されています」と確認できる状態になります。

## HTTPとHTTPSの違い

|  | HTTP | HTTPS |
|---|---|---|
| 通信の中身 | そのまま(平文)でやり取りされる | 暗号化される |
| ポート番号 | 80 | 443 |
| ブラウザの表示 | 「保護されていない通信」と警告が出る | 「この接続は保護されています」 |
| 必要なもの | 特になし | SSL証明書 |

HTTPのままだと、通信の中身を途中で誰かに読まれたり、書き換えられたりするおそれがあります。公共のWi-Fiでログイン情報を送るときの危なさ、という説明がよくされます。

それに加えて、ブログを運営する側にとっては次の理由でも必須です。

- ブラウザに「保護されていない通信」と出ると、読む人が不安になる
- このあと登録するSearch ConsoleやAnalyticsには、`https://` のURLで登録しておくと、あとで直す手間がない

## 「https:// と書いただけ」では使えなかった話

筆者は、HTTPSを甘く見ていたせいで一度つまずいています。

Search Consoleにサイトマップを登録しようとしたとき、Hugoの設定ファイル(`hugo.toml`)の `baseURL` は、もう `https://` にしていました。

```
baseURL = "https://noluckblog.com/"
```

ところが、**サーバーにはまだ証明書を入れておらず、HTTPでしか公開していませんでした。** ブラウザで `https://noluckblog.com` を開くと、

```
接続できません
```

とはっきりエラーになりました。サイトマップも当然「取得できませんでした」のままです。

ここでようやく、**HTTPSは `https://` と書けば使えるものではなく、サーバーに証明書を用意して初めて使える**ということが腑に落ちました。

### 証明書とは何か

SSL証明書は、ざっくり言うと「このドメインの持ち主が、このサーバーで暗号化通信をすることを認めた」という証明書です。これがないと、ブラウザは暗号化通信を始められません。

この記事では、**Let's Encrypt**という無料の発行元から証明書をもらいます。Certbotというツールを使うと、証明書の取得からNginxの設定の書き換えまで、コマンド1つで自動でやってくれます。

## 始める前に確認すること

- `nslookup example.com` でサーバーのIPアドレスが出る(第4回)
- ConoHaのセキュリティグループに `IPv4v6-Web` が付いている(第3回)
- ufwで `80` と `443` を許可している(第5回)

## 操作手順

サーバーにSSHで入ってから打ちます。`example.com` は自分のドメインに置き換えてください。

1. Certbot(証明書を取ってくるツール)を入れる

```
apt install certbot python3-certbot-nginx -y
```

2. 証明書を取得する

```
certbot --nginx -d example.com -d www.example.com
```

3. 質問に答える

| 質問 | 答え |
|---|---|
| メールアドレス | 自分のメールアドレス(期限切れの通知が届く) |
| 利用規約に同意するか | `Y` |
| お知らせメールを受け取るか | `N` でOK |

4. `Successfully deployed certificate` と出れば成功
5. ブラウザで `https://example.com` を開き、アドレスバーの左にあるアイコンを押して「この接続は保護されています」と出るか確認する

![Chromeで接続が保護されていることを確認する画面](/images/posts/chrome-https.png "今のChromeでは、アドレスバーに鍵マークは出ません。①のアイコンを押すと、②のように鍵マーク付きで「この接続は保護されています」と表示されます")

6. 自動更新が動くかテストする

```
certbot renew --dry-run
```

`Congratulations, all simulated renewals succeeded` と出ればOKです。

7. 手元のPCで `hugo.toml` の `baseURL` を `https://` に書き換える

```
baseURL = "https://example.com/"
```

書き換えたら、第7回の手順で公開し直します。**証明書を入れてから** `baseURL` を `https://` にする、という順番が大事です。

## 証明書を入れたあとに変わったこと

- `https://noluckblog.com` が、警告なしで開けるようになった
- `http://` で来た人も、自動で `https://` に転送されるようになった(Certbotが設定してくれた)
- Search Consoleのサイトマップが「成功しました」になった

Certbotが何を書き足したかは、Nginxの設定ファイルを開くと分かります。

```
cat /etc/nginx/sites-available/yourlog
```

`listen 443 ssl;` や証明書の場所(`ssl_certificate`)の行が増え、末尾に `# managed by Certbot` と付いています。

## つまずきやすい注意点

**DNSが反映される前に実行すると失敗する**

Certbotは「そのドメインが本当にこのサーバーを指しているか」を確認します。`nslookup` でIPアドレスが出るまで待ってください。

**https が表示されない**

443番が閉じている可能性があります。ufwとConoHaのセキュリティグループの**両方**を確認してください。→ [ConoHa VPSでSSH接続できずハマった話](/posts/ssh-conoha-hamatta/)

**baseURLを http のままにしない**

サイトマップが「無効」と言われるなど、あとで別のトラブルになります。逆に、証明書を入れる前に `https://` にすると、上に書いたように「接続できません」になります。

**Certbotが書いた行は消さない**

CertbotはNginxの設定ファイルを自動で書き換えます。`# managed by Certbot` と付いた行は消さないでください。

**証明書の期限は90日**

自動更新されますが、ときどき確認しておくと安心です。

```
systemctl list-timers | grep certbot
```

## 次の記事

[【第7回】記事の作成編 — Hugoで書いて公開するまで](/posts/series-07-kiji/)
