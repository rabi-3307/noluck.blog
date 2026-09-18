+++
title = "Google Analytics(GA4)を導入した話(測定IDをHugoに組み込む)"
date = 2026-09-05
type = "posts"
categories = ["guide"]
tags = ["GA4", "Google Analytics", "初心者"]
+++

Search Consoleの次に、アクセス数や読者の動きを見るために Google Analytics(GA4)を導入しました。Search Consoleほどのトラブルはありませんでしたが、Hugoへの組み込み方が最初分からなかったので、その手順を残しておきます。

## Google Analyticsとは

**サイトに何人来たか、どのページが読まれているか、どこから来たか、といったアクセスデータを計測してくれる無料ツール**です。Search Consoleが「検索エンジンから見えるサイトの姿」を教えてくれるのに対し、Analyticsは「実際に来た読者の行動」を教えてくれる、という役割の違いがあります。

## ステップ1:アカウントとプロパティの作成

[Google Analytics](https://analytics.google.com/)にアクセスし、以下の順で設定しました。

1. アカウント名を入力(自分が管理画面で分かればよいので、サイト名と同じにした)
2. プロパティ名、タイムゾーン(日本)、通貨(円)を設定
3. 業種やビジネス規模を選択(個人ブログに近いものを選択)
4. ビジネス目標:「サイトやアプリのエンゲージメントを測定する」を選択
5. プラットフォームで「ウェブ」を選び、サイトのURL(`https://noluckblog.com`)とストリーム名を入力

ここで1点注意したのが、**ウェブサイトのURLを`https://`で登録したこと**です。これは前提として、サイトが先にHTTPS化されている必要がありました。もしHTTPS化前にこの作業をしていたら、また入力し直しになっていたと思います。

## ステップ2:測定IDの取得

設定が完了すると、`G-`から始まる**測定ID**と、サイトに埋め込むためのタグ(コード)が発行されました。

```html
<!-- Google tag (gtag.js) -->
<script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
<script>
  window.dataLayer = window.dataLayer || [];
  function gtag(){dataLayer.push(arguments);}
  gtag('js', new Date());

  gtag('config', 'G-XXXXXXXXXX');
</script>
```

## ステップ3:Hugoのテンプレートに組み込む

このコードをどこに貼ればいいか最初迷いましたが、**全ページ共通の`<head>`部分に1回だけ貼れば、サイト全体に反映される**という仕組みでした。Hugoでは、それが`baseof.html`というファイルに該当します。

```
themes/(テーマ名)/layouts/_default/baseof.html
```

このファイルの`<head>`の中、`</head>`の直前に、先ほどのコードをそのまま追加しました。

```html
<head>
  ...
  <link rel="stylesheet" href="{{ "css/style.css" | relURL }}">

  <!-- Google tag (gtag.js) -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
  <script>
    window.dataLayer = window.dataLayer || [];
    function gtag(){dataLayer.push(arguments);}
    gtag('js', new Date());
    gtag('config', 'G-XXXXXXXXXX');
  </script>
</head>
```

## ステップ4:ビルドして公開、確認

いつも通りの手順でビルド・転送しました。

```powershell
hugo
scp -r public\* root@(IPアドレス):/var/www/yourlog/public/
```

## ステップ5:リアルタイムレポートで確認

Analyticsの管理画面から「レポート」→「リアルタイム」を開いた状態で、実際に自分のサイトをブラウザで開いてみると、

```
過去30分のアクティブユーザー数: 4
```

のように、即座に反応が表示されました。地図上にもマーカーが出て、確かに計測できていることが視覚的に確認できました。

> 補足:表示された人数(4人)は、実際の読者の数ではなく、自分が開いていた複数のブラウザタブなどが別セッションとしてカウントされた結果でした。慌てず、リアルタイムに反応があること自体を確認できればOKです。

## つまずかなかったが、注意しておいてよかった点

| 注意点 | 内容 |
|---|---|
| URLはhttpsで登録する | HTTPS化が済んでいない状態で登録すると、後で修正が必要になる可能性がある |
| コードは`<head>`内、1ページに1つだけ | 複数箇所に貼ると計測が重複したり正しく動かなかったりする恐れがある |
| 全ページ共通のテンプレートに貼る | 記事ページごとに貼る必要はなく、`baseof.html`のような共通部分に1回で済む |

Search Consoleの一件で散々苦労した直後だったので身構えていましたが、Analytics自体の導入は今回は比較的スムーズに終わりました。HTTPS化を先に済ませておいたことが、地味に効いていたのだと思います。
