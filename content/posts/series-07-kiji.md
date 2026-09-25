+++
title = "【第7回】記事の作成編 — Hugoで書いて公開するまで"
date = 2026-09-25T09:07:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、記事を書いて、自分のサイトに公開できるようになります。

## 操作手順

### 1. サイトのフォルダに移動する

PowerShellで、`hugo.toml` があるフォルダに移動します。

```
cd "C:\Users\ユーザー名\...\mysite"
```

エクスプローラーでそのフォルダを開き、アドレスバーに `powershell` と打ってEnterを押すと、最初からそのフォルダで開けます。

### 2. 記事ファイルを作る

```
hugo new posts/my-first-post.md
```

`content/posts/my-first-post.md` ができます。

### 3. VSCodeで書く

ファイルの先頭はこうなっています。

```
+++
title = "記事のタイトル"
date = 2026-09-25T10:00:00+09:00
draft = true
categories = ["beginners"]
+++
```

`+++` の下に本文を書きます。見出しは `##`、箇条書きは `-` です。

### 4. 手元で表示を確認する

```
hugo server -D
```

ブラウザで `http://localhost:1313` を開きます。保存するたびに自動で更新されます。終わるときは Ctrl+C。

### 5. 公開する準備をする

先頭の `draft = true` を `draft = false` に変えて保存します。

### 6. 公開用のファイルを作る

```
hugo
```

`public` フォルダに公開用のファイルができます。

### 7. サーバーに送る

```
scp -r public/* root@xxx.xxx.xxx.xxx:/var/www/yourlog/public/
```

### 8. 読み取り権限を直して反映する

```
ssh root@xxx.xxx.xxx.xxx "chmod -R 755 /var/www/yourlog && systemctl reload nginx"
```

ブラウザで自分のサイトを開き、記事が出ていれば公開完了です。

手順6〜8は、番外編(GitHub連携編)で `git push` 1回に自動化できます。

## つまずきやすい注意点

**`Unable to locate config file` と出る**

`hugo.toml` があるフォルダにいません。`cd` し直してください。一番よくあるミスです。

**記事が表示されない**

- `draft = true` のままになっていないか
- `date` が未来の日時になっていないか(未来の記事は公開されません)

**文字化けやエラーが出る**

VSCodeで、右下の表示が「UTF-8」になっているか確認してください。メモ帳で保存すると壊れることがあります。

**403 Forbidden になる**

手順8の `chmod` を忘れています。

操作だけまとめた版は [記事を公開する手順(操作だけ)](/posts/kiji-koukai-saitan/)、コマンド一覧は [Hugoコマンドリファレンス](/posts/hugo-command-reference/) にあります。

## 次の記事

[【第8回】Google Search Console編 — Googleにサイトを登録する](/posts/series-08-search-console/)
