+++
title = "【番外編2】GitHub連携編 — git pushだけで公開まで自動化する"
description = "GitHub Actionsを使って、git pushするだけでHugoのビルドからVPSへの公開まで自動で終わるようにする手順です。自動デプロイ専用の鍵の作り方、GitHubへの登録方法、実際に起きた権限トラブルの防ぎ方も解説します。"
date = 2026-09-25T09:12:00+09:00
lastmod = 2026-10-06T10:00:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、記事を書いて `git push` するだけで、ビルドからサーバーへの公開まで自動で終わるようになります。

## 何が変わるか

| 作業 | これまで(手動) | これから(自動) |
|---|---|---|
| ビルド | `hugo` を打つ | 自動 |
| サーバーへ転送 | `scp` を打つ | 自動 |
| 権限を直して反映 | `ssh` で打つ | 自動 |
| 自分がやること | 上の3つ | `git push` だけ |

この仕組みをCI/CDと呼びます。今回はGitHub Actionsを使います。

## 操作手順

`xxx.xxx.xxx.xxx` はサーバーのIPアドレス、`ユーザー名` `リポジトリ名` は自分のものに置き換えてください。

### 1. GitHubにリポジトリを作る

1. GitHubにログインし、右上の「+」→「New repository」
2. リポジトリ名を入れ、「Private」を選んで「Create repository」

### 2. 手元のサイトをGitHubに上げる

サイトのフォルダ(`hugo.toml` がある場所)で、まず `.gitignore` というファイルを作り、次の内容を書いて保存します。ここに書いたものはGitHubに上がりません。

```
public/
resources/
.hugo_build.lock
*.key
github_actions_key*
```

`public` と `resources` はGitHub側で作り直すので上げません。下の2行は、**鍵のファイルを間違って上げないため**の保険です。筆者は鍵をサイトのフォルダに作ってしまい、気づかずにGitHubに上げていました。→ [自動デプロイ用の秘密鍵をGitHubに上げてしまったので、鍵を作り直した話](/posts/himitsukagi-github/)

続けてPowerShellで打ちます。

```
git init
git add .
git commit -m "first commit"
git branch -M main
git remote add origin https://github.com/ユーザー名/リポジトリ名.git
git push -u origin main
```

初回はブラウザが開いてGitHubへのログインを求められます。

### 3. 自動デプロイ専用の鍵を作る

手元のPowerShellで打ちます。パスフレーズを聞かれたら**何も入れずにEnter**を2回押します。

```
ssh-keygen -t ed25519 -f $env:USERPROFILE\.ssh\github_actions_key
```

`-f` のあとは、必ず `$env:USERPROFILE\.ssh\` から始まる場所にしてください。`-f github_actions_key` のようにファイル名だけにすると、**今いるフォルダ(=サイトのフォルダ)に鍵ができて**、GitHubに上がってしまう原因になります。

公開鍵をサーバーに登録します。

```
type $env:USERPROFILE\.ssh\github_actions_key.pub | ssh root@xxx.xxx.xxx.xxx "cat >> ~/.ssh/authorized_keys"
```

### 4. GitHubに秘密の情報を登録する

1. リポジトリの「Settings」→「Secrets and variables」→「Actions」
2. 「New repository secret」で次の2つを登録する

| Name | Secret |
|---|---|
| `SSH_PRIVATE_KEY` | 秘密鍵の中身すべて |
| `SERVER_IP` | サーバーのIPアドレス |

秘密鍵の中身は次のコマンドで表示できます。`-----BEGIN` から `-----END ...-----` までの全部をコピーして、Secretの入力欄に貼ります。

```
Get-Content $env:USERPROFILE\.ssh\github_actions_key
```

Secretに入れた値は、登録したあとは自分でも見られません。表示した秘密鍵は、ほかのどこにも貼らないでください。

### 5. 自動化の手順書を作る

サイトのフォルダに `.github\workflows\deploy.yml` を作り、次の内容を貼ります。`hugo-version` は手元で `hugo version` を打って出た番号に合わせてください。

```
name: Deploy to VPS

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Hugo
        uses: peaceiris/actions-hugo@v3
        with:
          hugo-version: '0.164.0'
          extended: true

      - name: Build
        run: hugo

      - name: Deploy via SCP
        uses: appleboy/scp-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          source: "public/*,public/.*"
          target: "/var/www/yourlog/public"
          overwrite: true
          strip_components: 1

      - name: Fix permissions and reload Nginx
        if: always()
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            chown -R root:root /var/www/yourlog
            find /var/www/yourlog -type d -exec chmod 755 {} +
            find /var/www/yourlog -type f -exec chmod 644 {} +
            systemctl reload nginx
```

最後のステップは、送ったファイルの持ち主と読み取り権限を直して、Nginxに反映する部分です。

- `chown -R root:root` … 持ち主をサーバーの `root` にそろえる
- `find ... -type d ... chmod 755` … フォルダは「誰でも中に入れて読める」
- `find ... -type f ... chmod 644` … ファイルは「誰でも読める、書けるのは持ち主だけ」
- `if: always()` … 前のステップが失敗しても、このステップは必ず実行する

最初はここを `chmod -R 755` の1行にしていましたが、それだとサイトのデザインが丸ごと消えるトラブルが起きました。理由は下の注意点に書いています。

### 6. pushして動かす

```
git add .
git commit -m "add deploy workflow"
git push
```

### 7. 結果を確認する

1. GitHubのリポジトリで「Actions」タブを開く
2. 一番上の実行に緑のチェックが付けば成功
3. 自分のサイトを開き、変更が反映されているか確認する

![GitHub Actionsの画面](/images/posts/github-actions-success.png "①「Actions」タブを開き、②のように緑のチェックが付いていれば、公開まで自動で終わっています")

これ以降は、記事を書いたら `git add .` → `git commit -m "メッセージ"` → `git push` の3つだけで公開されます。VSCodeの「ソース管理」ボタンからでも同じことができます。

## つまずきやすい注意点

**pushしたのにサイトが変わらない、デザインが崩れる**

筆者が一番ハマったところです。Actionsで送ったフォルダが「持ち主しか読めない」状態になっていて、NginxがCSSを読めず、サイトが文字だけの画面になりました。しかも、転送のステップが失敗扱いになると、その後ろの「権限を直すステップ」は実行されません。上の `deploy.yml` の最後のステップは、この経験から直した形です。くわしくは [CSSが404になった原因はフォルダの権限だった](/posts/css-404-permission/) へ。

**YAMLはスペース1つのずれで動かない**

`Invalid workflow file` と出たら、インデント(行頭のスペース)がずれています。タブではなくスペースを使い、上のコードをそのまま貼り直すのが確実です。

**ファイルが二重のフォルダに入る・フォルダが空になる**

`target` と `strip_components` の組み合わせを間違えると、`public/public/` に入ったり、何も転送されなかったりします。上の組み合わせから変えないでください。

**`Repository not found` と出る**

リポジトリ名の打ち間違いです。GitHubの画面に出ている名前と完全に同じか確認してください(`.` や `-` の有無も)。

**`Permission denied to (別のアカウント名)` と出る**

学校用など、別のGitHubアカウントの認証情報がPCに残っています。PowerShellで次を打ってから、もう一度 `git push` すると、ログインし直せます。

```
cmdkey /delete:git:https://github.com
```

VSCodeを使う場合は、左下の人型アイコンからVSCode側のGitHubアカウントも別に切り替えてください。コマンドとVSCodeは別々にログイン情報を持っています。

**`no changes added to commit` と出る**

`git add .` を打たずに `git commit` しています。変更したファイルは、`add` してから `commit` します。

**秘密鍵をリポジトリに入れない**

秘密鍵はGitHubのSecretsに登録するだけです。サイトのフォルダにコピーしたり、コミットしたりしないでください。上げてしまったら、ファイルを消すだけでは足りません。鍵を作り直してください。

## あわせて読みたい

- [【第7回】記事の作成編 — Hugoで書いて公開するまで](/posts/series-07-kiji/) — 自動化する前の、手動での公開の流れ

## シリーズはここまで

計画からサーバー構築、公開、収益化、自動化まで、このブログでやったことはすべてこのシリーズに入っています。最初から読み直す場合はこちらです。

[【第1回】計画編 — WordPressか自作VPSか、費用と手間を比べて決める](/posts/series-01-keikaku/)
