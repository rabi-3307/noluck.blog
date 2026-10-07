+++
title = "記事を消したのに、サーバーには古いページが残り続けていた(scpは上書きするだけ)"
description = "記事を整理して数を減らしたのに、サーバーには消したはずのページやファイルが残り続ける。原因はscpによる公開が「上書き」しかしないことでした。新しいフォルダに丸ごと送って入れ替える方法と、Hugoのaliasesで古いURLを転送する方法をまとめました。"
date = 2026-10-07T11:00:00+09:00
draft = true
type = "posts"
categories = ["troubleshooting"]
tags = ["GitHub Actions", "デプロイ", "Hugo", "トラブルシューティング"]
+++

<!-- 公開前に:新しいデプロイでpushしたあと、古いURL(例:/posts/vps-first-setup/)を開いて、転送されること・古いページが消えたことを確かめてから draft = false にする -->

AdSenseの審査に落ちたのをきっかけに、似た記事をまとめて36本を18本にしました(→ [AdSense審査に落ちた話](/posts/adsense-shinsa-ochita/))。手元のフォルダからは記事が消えたので、これでサイトもすっきりした…と思っていたのですが、そう単純ではありませんでした。

**このブログの公開のしかたでは、サーバーにあるファイルは「上書き」されるだけで、「削除」はされません。** 手元で記事を消しても、サーバーには消した記事のページが残り続けます。

先に結論です。

- `scp`(とGitHub Actionsの `scp-action`)は、送ったファイルで上書きするだけ。サーバーにだけあるファイルは消さない
- 新しいフォルダに丸ごと送ってから入れ替える形にして、古いファイルが残らないようにした
- 消した記事の古いURLは、Hugoの `aliases` でまとめた先の記事に転送した

## 何が問題だったのか

このブログは、`git push` するとGitHub Actionsがサイトを作り、`/var/www/yourlog/public` に送っています(→ [番外編2](/posts/series-ex2-github/))。送るときの動きはこうです。

| 手元(作ったサイト)にある | サーバーにある | 送ったあと |
|---|---|---|
| ある | ある | 新しい内容で上書き |
| ある | 無い | 新しく置かれる |
| **無い** | **ある** | **そのまま残る** |

3行目が問題です。記事を消したりURLを変えたりしても、サーバーには古いページが残り、URLを知っている人(や検索エンジン)はそのまま見られてしまいます。

このブログで言えば、ブログを始めたころに書いて、その後書き直したり名前を変えたりした記事のページや、一時的に置いたファイルが、手元からは消えていてもサーバーには残っているはずでした。

<!-- ここに、実際に残っていた古いURLの例と、開いたときの様子を書く -->

### 何が困るのか

- **古い情報が見られる**:書き直す前の、間違いや古い手順が残ったページが読まれてしまう
- **中身の重なったページが増える**:Googleから見ると、同じような内容のページがたくさんあるサイトになる。AdSenseで「有用性の低いコンテンツ」と言われた原因の1つになりうる
- **見せるつもりのないファイルが残る**:作業用に一時的に置いたファイルなども、消したつもりでそのまま公開され続ける

## 直し方1:新しいフォルダに送ってから入れ替える

一番かんたんなのは「送る前にサーバーのフォルダを空にする」ですが、それだと送っている途中や、送るのに失敗したときに、サイトが空っぽになってしまいます。

そこで、こういう流れにしました。

```
1. 新しいサイトを、別のフォルダ(public_new)に丸ごと送る
2. 送れたことを確かめてから、今の public を public_old に名前変更
3. public_new を public に名前変更(ここで一瞬で切り替わる)
4. 持ち主と権限をそろえて、Nginxに反映
```

GitHub Actionsの手順書(`.github/workflows/deploy.yml`)の、送る部分と最後の部分をこう変えました。

```
      - name: Deploy via SCP
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "public/*,public/.*"
          target: "/var/www/yourlog/public_new"
          rm: true
          overwrite: true
          strip_components: 1

      - name: Switch to new files and fix permissions
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            set -e
            cd /var/www/yourlog
            test -f public_new/index.html
            rm -rf public_old
            if [ -d public ]; then mv public public_old; fi
            mv public_new public
            chown -R root:root /var/www/yourlog
            find /var/www/yourlog -type d -exec chmod 755 {} +
            find /var/www/yourlog -type f -exec chmod 644 {} +
            nginx -t
            systemctl reload nginx
```

| 書いたこと | 意味 |
|---|---|
| `target: ".../public_new"` | 今公開しているフォルダではなく、別のフォルダに送る |
| `rm: true` | 送る前に、`public_new` を空にする(前回の残りを持ち越さない) |
| `set -e` | どれか1つでも失敗したら、そこで止める |
| `test -f public_new/index.html` | トップページがちゃんと届いているかを確かめる。無ければここで止まり、今のサイトはそのまま |
| `mv public public_old` → `mv public_new public` | 名前を変えるだけなので、切り替えは一瞬 |
| `nginx -t` | Nginxの設定に問題がないか確かめてから反映する |

ひとつ前の状態は `public_old` として1つだけ残るので、新しいサイトに問題があったときは、サーバーで次のように打てば元に戻せます。

```
cd /var/www/yourlog && mv public public_bad && mv public_old public
```

[CSSが404になったとき](/posts/css-404-permission/)は、転送が失敗しても権限を直すステップは必ず実行するように `if: always()` を付けていました。今回の形では、転送に失敗したら入れ替え自体をしないので、公開中のサイトには手を触れません。そのため `if: always()` は外しています。

## 直し方2:消した記事のURLは、まとめた先に転送する

古いページを消すと、そのURLは「ページが見つかりません(404)」になります。検索結果やほかのサイトからリンクされていた場合にもったいないので、まとめた先の記事に自動で移動するようにしました。

Hugoでは、まとめた先の記事のフロントマターに `aliases` を書くだけです。

```
+++
title = "【第3回】ConoHa編 — VPSを契約してサーバーを起動する"
aliases = ["/posts/vps-keiyaku-hamatta/", "/posts/vps-first-setup/"]
+++
```

`hugo` を打つと、`aliases` に書いたURLの場所に「新しいURLへ移動するだけのページ」が自動で作られます。中身はこれだけです。

```
<link rel="canonical" href="https://example.com/posts/series-03-conoha/">
<meta http-equiv="refresh" content="0; url=https://example.com/posts/series-03-conoha/">
```

`canonical` は「正式なURLはこちら」という印、`refresh` は「すぐにこのURLへ移動して」という指示です。この移動用のページはサイトマップにも載りません。

このブログでは、まとめた記事の21本に加えて、ブログを始めたころに名前を変えた記事のURLも、同じように今の記事へ転送しています。

## 手元の public フォルダにも注意

同じことは、手元のPCでも起きます。`hugo` コマンドは `public` フォルダに上書きで書き出すだけなので、消した記事のページや、昔一時的に置いたファイルが `public` に残り続けます。

手動で `scp -r public/* ...` で公開している場合は、この古いファイルまでサーバーに送ってしまうので、作り直す前に `public` を消すか、次のように打ちます。

```
hugo --cleanDestinationDir
```

GitHub Actionsは毎回まっさらな環境でサイトを作るので、こちらの問題は起きません。

## 確かめ方

新しいデプロイでpushしたあと、こう確かめます。

1. Actionsが緑のチェックになっているか
2. 消した記事の古いURLを開いて、まとめた先の記事に移動するか
3. サーバーで `ls /var/www/yourlog/public/posts/` を打って、今ある記事のフォルダだけになっているか

<!-- ここに、実際に確かめた結果を書く -->

## 学んだこと

| 思っていたこと | 実際 | 今のやり方 |
|---|---|---|
| 手元で消せば、サイトからも消える | 送るのは上書きだけ。サーバーにだけあるファイルは残る | 別のフォルダに丸ごと送って入れ替える |
| 記事を消したら、URLもなくなるだけ | 外からのリンクや検索結果が404になってもったいない | `aliases` でまとめた先へ転送する |
| `public` フォルダは毎回作り直される | `hugo` は上書きするだけ | 手動で公開するなら `--cleanDestinationDir` |

記事を整理したことで初めて気づいた問題でした。記事を書き足していくだけのうちは困らないので、気づきにくいところだと思います。

## あわせて読みたい

- [【番外編2】GitHub連携編 — git pushだけで公開まで自動化する](/posts/series-ex2-github/) — 自動デプロイの基本の形
- [git pushしたらデザインが消えた。CSSが404になった原因はフォルダの権限だった](/posts/css-404-permission/) — 自動デプロイで起きた、権限のトラブル
- [AdSense審査に「有用性の低いコンテンツ」で落ちたので、36本の記事を18本にまとめ直した](/posts/adsense-shinsa-ochita/) — 記事を整理することになったきっかけ
