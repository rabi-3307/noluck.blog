+++
title = "【第9回】Google Analytics編 — 何人来たかを測る"
description = "Google Analytics(GA4)で、サイトに何人来たか、どの記事が読まれているかを測れるようにする手順です。Hugoのbaseof.htmlに1回貼るだけで全ページに入る仕組みと、実際に入れてみて分かった注意点も解説します。"
date = 2026-09-25T09:09:00+09:00
lastmod = 2026-10-06T10:00:00+09:00
categories = ["beginners", "guide"]
aliases = ["/posts/analytics-donyu/", "/posts/baseof-shikumi/"]
+++

この記事を読み終えると、サイトに何人来たか、どの記事が読まれているかが分かるようになります。

## Search Consoleとの違い

|  | Search Console(第8回) | Analytics(今回) |
|---|---|---|
| 分かること | 検索で何回表示・クリックされたか | サイトに来た人が何をしたか |
| 例 | どんな検索ワードで見つかったか | 何人来たか、どの記事を読んだか、どこから来たか |
| タイミング | サイトに来る前 | サイトに来た後 |

両方入れておくのがおすすめです。

## 操作手順

### 1. アカウントを作る

1. Google Analyticsを開き、「測定を開始」を押す
2. アカウント名を入力(自分が分かればいいので、ブログ名など)
3. プロパティ名を入力し、タイムゾーンを「日本」、通貨を「日本円」にする
4. 業種や規模、目的を選んで「作成」(筆者は目的に「サイトやアプリのエンゲージメントを測定する」を選びました)

### 2. サイトを登録する

1. プラットフォームで「ウェブ」を選ぶ
2. URLに `https://example.com` を入れ、ストリーム名(ブログ名など)を入力して「作成」
3. 「タグの実装手順を表示」→「手動でインストールする」を選ぶ
4. 表示されたコードをコピーする

コードはこんな形です。`G-XXXXXXXXXX` の部分が、自分専用の測定IDです。

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

URLは `https://` で登録します。第6回のHTTPS化を先に済ませておいたので、ここは迷わずに済みました。

### 3. サイトに貼る

1. VSCodeで `themes/(テーマ名)/layouts/_default/baseof.html` を開く
2. `<head>` と `</head>` の間、既存の行の下に貼り付けて保存する

貼ったあとはこうなります。

```html
<head>
  ...
  <link rel="stylesheet" href="{{ "css/style.css" | relURL }}">

  <!-- Google tag (gtag.js) -->
  <script async src="https://www.googletagmanager.com/gtag/js?id=G-XXXXXXXXXX"></script>
  <script>
    ...
  </script>
</head>
```

### 4. 公開して確認する

1. 第7回の手順で公開する(`hugo` → サーバーへ転送)
2. 自分のサイトをブラウザで開く
3. Analyticsの「レポート」→「リアルタイムの概要」で、1人以上表示されれば成功

![Google Analyticsのリアルタイムの概要](/images/posts/ga4-realtime.png "①「リアルタイムの概要」を開き、②が1以上なら計測できています。地図には見た人のだいたいの場所が出るので、この画像では消しています")

筆者が初めて開いたときは「過去30分のアクティブユーザー数: 4」と出て、一瞬喜びました。実際は、自分がいくつも開いていたブラウザのタブが別々に数えられていただけでした。リアルタイムに反応があることさえ確認できればOKです。

## なぜ1つのファイルに貼るだけで全ページに入るのか

Googleの説明には「各ページの `<head>` に貼ってください」と書いてあります。なのに、実際に直すのは `baseof.html` の1ファイルだけ。最初はこのズレに不安になったので、仕組みを確かめました。

### 普通のHTMLサイトの場合

HTMLを手書きでサイトを作ると、ページの数だけファイルがあります。

```
index.html      ← トップページ
about.html      ← このサイトについて
posts/a.html    ← 記事A
posts/b.html    ← 記事B
```

この場合は、本当に1ファイルずつ貼っていく必要があります。Googleの「各ページに貼ってください」は、この作り方を前提にした説明です。

### Hugoの場合

Hugoは、ページを1枚ずつ手書きするのではなく、**全ページ共通の「型」に、記事の中身を流し込んで、ページの数だけ自動で作ります。** その「型」が `baseof.html` です。

```
themes/yourlog/layouts/_default/baseof.html   ← 全ページ共通の型
                ↓ hugo コマンド
public/index.html
public/posts/a/index.html
public/posts/b/index.html
public/about/index.html
…(全部、同じ型から作られる)
```

`baseof.html` の中には `{{ block "main" . }}{{ end }}` という部分があり、ここに記事ごとの本文が差し込まれます。逆に言うと、`<head>` のような全ページ共通の部分は、`baseof.html` に1回書けば、作られる全ページに入ります。

### 本当に入ったか確かめる

`hugo` を打ったあと、`public` の中の違うページを2つ開いて見比べました。

```
public/index.html
public/posts/series-01-keikaku/index.html
```

どちらの `<head>` にも、同じGoogleのタグが入っていました。公開後なら、サイトで右クリック →「ページのソースを表示」→ `Ctrl + F` で `googletagmanager` を探しても確かめられます。トップページと記事ページの両方で見つかれば大丈夫です。

この仕組みは、第10回でAdSenseのコードを貼るときにもそのまま使います。

## つまずきやすい注意点

**貼るときに既存の行を消さない**

コードを貼るときに `<link rel="stylesheet" ...>` の行を消してしまい、サイト全体のデザインが崩れました。貼り付けは「追加」で、置き換えではありません。

**同じコードを2か所に貼らない**

1ページに同じタグが2回入ると、二重に数えられたり、正しく動かなかったりします。上の確認で、1ページに1回だけ出てくるか見てください。

**リアルタイム以外は反映が遅い**

普通のレポートに数字が出るまで24〜48時間かかります。0のままでも焦らなくて大丈夫です。

**自分のアクセスも数えられる**

記事の確認で自分が開いた分も1人として数えられます。始めたばかりのころは、ほぼ自分の数字です(筆者もそうでした)。

**ログイン情報は誰にも教えない**

`G-XXXXXXXXXX`(測定ID)はページのソースに出るので見られても問題ありませんが、Googleアカウントのパスワードは別です。

## 次の記事

[【第10回】収益化準備編 — AdSense申請とA8.net登録](/posts/series-10-shuueki/)
