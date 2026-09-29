+++
title = "【最短手順】AdSenseコードの貼り方"
description = "HugoのブログにGoogle AdSenseのコードを貼る手順を、操作だけに絞ってまとめました。どのファイルのどこに貼るか、貼れたかの確かめ方、広告が出ないときに見るポイントまで分かります。"
date = 2026-09-07
type = "posts"
categories = ["beginners"]
tags = ["AdSense", "最短手順"]
+++

この記事を読み終えると、Google AdSenseのコードがサイトの全ページに入り、審査を待てる状態になります。

> **この記事の前提**
>
> - Hugoでブログを作っている(→ [第7回 記事の作成編](/posts/series-07-kiji/))
> - AdSenseに申し込み済み(→ [第10回 収益化準備編](/posts/series-10-shuueki/))
>
> 文中の `xxx.xxx.xxx.xxx` は自分のサーバーのIPアドレスに、`ca-pub-xxxxxxxxxxxxxxxx` は自分のIDに置き換えてください。

## 貼るコードはどんなものか

AdSenseの管理画面でサイトを追加すると、次のような形のコードが表示されます。

```html
<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-xxxxxxxxxxxxxxxx"
     crossorigin="anonymous"></script>
```

`ca-pub-` のあとの数字が、自分専用のID(パブリッシャーID)です。コードは画面からそのままコピーして、1文字も変えずに使います。

## 操作手順

### 1. 貼るファイルを開く

VSCodeで次のファイルを開きます。

```
themes/(テーマ名)/layouts/_default/baseof.html
```

`content` フォルダの中ではありません。`content` は記事を入れる場所なので、ここにHTMLのファイルを置くとHugoがエラーを出して止まります(実際にやってしまいました)。

### 2. `</head>` のすぐ上に貼る

ファイルの中から `</head>` という行を探して、そのすぐ上に貼り付けて保存します。

```html
<head>
<meta charset="UTF-8">
<title>...</title>
<link rel="stylesheet" href="...">

<!-- Google AdSense -->
<script async src="https://pagead2.googlesyndication.com/pagead/js/adsbygoogle.js?client=ca-pub-xxxxxxxxxxxxxxxx"
     crossorigin="anonymous"></script>
</head>
```

`<!-- Google AdSense -->` の行は、あとで見たときに何のコードか分かるようにするためのメモです。書かなくても動きます。

`baseof.html` は全ページ共通の土台なので、ここに1回貼るだけで全部のページに入ります。→ [baseof.htmlの仕組み](/posts/baseof-shikumi/)

### 3. 手元で確認する

```
hugo server
```

ブラウザで `http://localhost:1313` を開き、ページの上で右クリック →「ページのソースを表示」を選びます。`Ctrl+F` で `adsbygoogle` と検索して、見つかれば貼れています。

### 4. 公開する

GitHub連携をしている場合(→ [番外編2 GitHub連携編](/posts/series-ex2-github/))は、これだけです。

```
git add .
git commit -m "add adsense code"
git push
```

手動で公開している場合は、次の3つを順番に打ちます。

```
hugo
scp -r public/* root@xxx.xxx.xxx.xxx:/var/www/yourlog/public/
ssh root@xxx.xxx.xxx.xxx "chmod -R 755 /var/www/yourlog && systemctl reload nginx"
```

### 5. 本番のサイトで確認する

自分のサイトを開いて、手順3と同じように「ページのソースを表示」→ `adsbygoogle` を検索します。トップページだけでなく、記事のページでも見つかれば完了です。

## 貼ったのに広告が出ないとき

**審査に通るまでは、広告は出ません**

コードを貼っても、すぐには広告は表示されません。AdSenseの「サイト」画面で状態が「準備完了」になるまでは、何も出ないのが正常です。「準備中」は審査中という意味なので、そのまま待ちます。

**ページが開けない、デザインが崩れている**

審査は、Googleのロボットがサイトを見に来て行います。そのときにページが開けないと、審査に通りません。このブログでは、サーバーに送ったフォルダの読み取り権限がおかしくなっていて、CSSや記事が読めない状態になっていたことがありました。何ページか開いて、デザインと記事が正しく出ているか確認してください。→ [トラブル対応の記事](/categories/troubleshooting/)

**コードを2か所に貼っている**

テーマの別のファイルにも貼るなどして、1ページの中に同じコードが2回入ると、うまく動かないことがあります。手順5で `adsbygoogle` を検索したとき、コードが1回だけ出てくるか確認してください。

**広告ブロックを使っている**

ブラウザに広告ブロックの拡張機能が入っていると、自分のサイトの広告も見えません。確認するときは、拡張機能をオフにするか、シークレットモードで開いてください。

**ads.txtは別の作業**

コードを貼るのとは別に、`static/ads.txt` というファイルを置く作業があります。こちらは [第10回 収益化準備編](/posts/series-10-shuueki/) の手順を見てください。

## 審査を待つあいだにやること

審査の結果が出るまでは、記事を書き足しながら待ちます。中身の薄いページが多いと審査に通りにくいので、短すぎる記事は書き足すか、まとめてしまうのがおすすめです。
