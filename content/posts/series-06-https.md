+++
title = "【第6回】HTTPS編 — 無料の証明書で鍵マークを付ける"
date = 2026-09-25T09:06:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、サイトが `https://` で表示され、ブラウザに鍵マークが付いた状態になります。

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
5. ブラウザで `https://example.com` を開き、鍵マークが付いているか確認する
6. 自動更新が動くかテストする

```
certbot renew --dry-run
```

`Congratulations, all simulated renewals succeeded` と出ればOKです。

7. 手元のPCで `hugo.toml` の `baseURL` を `https://` に書き換える

```
baseURL = "https://example.com/"
```

## つまずきやすい注意点

**DNSが反映される前に実行すると失敗する**

Certbotは「そのドメインが本当にこのサーバーを指しているか」を確認します。`nslookup` でIPアドレスが出るまで待ってください。

**https が表示されない**

443番が閉じている可能性があります。ufwとConoHaのセキュリティグループの両方を確認してください。

**baseURLを http のままにしない**

サイトマップが「無効」と言われるなど、あとで別のトラブルになります。→ [baseURLとHTTPSでハマった話](/posts/baseurl-https-hamatta/)

**Certbotが書いた行は消さない**

CertbotはNginxの設定ファイルを自動で書き換えます。`# managed by Certbot` と付いた行は消さないでください。

**証明書の期限は90日**

自動更新されますが、ときどき確認しておくと安心です。

```
systemctl list-timers | grep certbot
```

HTTPとHTTPSの違いは [HTTPとHTTPSの違い](/posts/http-vs-https/) で解説しています。

## 次の記事

[【第7回】記事の作成編 — Hugoで書いて公開するまで](/posts/series-07-kiji/)
