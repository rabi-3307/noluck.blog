+++
title = "【番外編2】GitHub連携編 — git pushだけで公開まで自動化する"
date = 2026-09-25T09:12:00+09:00
categories = ["beginners"]
+++

この記事を読み終えると、記事を書いて `git push` するだけで、ビルドからサーバーへの公開まで自動で終わるようになります。

## 何が変わるか

| 作業 | これまで(手動) | これから(自動) |
|---|---|---|
| ビルド | `hugo` を打つ | 自動 |
| サーバーへ転送 | `scp` を打つ | 自動 |
| 権限を直して再起動 | `ssh` で打つ | 自動 |
| 自分がやること | 上の3つ | `git push` だけ |

この仕組みをCI/CDと呼びます。今回はGitHub Actionsを使います。

## 操作手順

`xxx.xxx.xxx.xxx` はサーバーのIPアドレス、`ユーザー名` `リポジトリ名` は自分のものに置き換えてください。

### 1. GitHubにリポジトリを作る

1. GitHubにログインし、右上の「+」→「New repository」
2. リポジトリ名を入れ、「Private」を選んで「Create repository」

### 2. 手元のサイトをGitHubに上げる

サイトのフォルダ(`hugo.toml` がある場所)で、まず `.gitignore` というファイルを作り、次の2行を書いて保存します。`public` はGitHub側で作るので上げません。

```
public/
resources/
```

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

秘密鍵の中身は次のコマンドで表示できます。`-----BEGIN` から `-----END ...-----` までの全部をコピーします。

```
Get-Content $env:USERPROFILE\.ssh\github_actions_key
```

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

      - name: Fix permissions and restart Nginx
        uses: appleboy/ssh-action@master
        with:
          host: ${{ secrets.SERVER_IP }}
          username: root
          key: ${{ secrets.SSH_PRIVATE_KEY }}
          script: |
            chmod -R 755 /var/www/yourlog
            systemctl restart nginx
```

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

これ以降は、記事を書いたら `git add .` → `git commit -m "メッセージ"` → `git push` の3つだけで公開されます。VSCodeの「ソース管理」ボタンからでも同じことができます。

## つまずきやすい注意点

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

**秘密鍵をリポジトリに入れない**

秘密鍵はGitHubのSecretsに登録するだけです。サイトのフォルダにコピーしたり、コミットしたりしないでください。

## シリーズはここまで

計画からサーバー構築、公開、収益化、自動化まで、このブログでやったことはすべてこのシリーズに入っています。最初から読み直す場合はこちらです。

[【第1回】計画編 — WordPressか自作VPSか、最初に決めること](/posts/series-01-keikaku/)
