+++
title = "Hugoの環境構築でつまずいたこと(インストール〜初回起動まで)"
date = 2026-08-24
type = "posts"
categories = ["troubleshooting"]
tags = ["Hugo", "初心者"]
+++

ブログを一から作る過程で、最初の関門になったのが「Hugoを動かせる状態にする」ところでした。同じところでつまずく人のために、実際の手順とハマったポイントを残しておきます。

## 1. Hugo本体をインストールする

Windowsには`winget`という公式のインストールコマンドが標準で入っているので、これを使いました。

```powershell
winget install Hugo.Hugo.Extended
```

「拡張版(Extended)」を選ぶのがポイントです。無印版だと使えない機能があるため、迷ったら拡張版を選んでおくと安心です。

インストール後、PowerShellを開き直してから確認します。

```powershell
hugo version
```

バージョンが表示されればインストール成功です。

> ⚠️ 最初、`sudo apt install hugo`を試してエラーになりました。これはLinux用のコマンドで、普段使っているのがWindowsのPowerShellだったため動きませんでした。**自分が今、Windows・Mac・Linuxのどの環境で作業しているか**を意識するのが大事だと学びました。

## 2. サイトの雛形を用意する

Hugo本体をインストールしただけでは、まだ空っぽの状態です。サイトの骨組み(設定ファイルやデザインテーマ)一式をzipファイルとして用意し、それを解凍しました。

解凍すると、`yourlog-site`というフォルダが出てきて、中身はこんな構成になっていました。

```
yourlog-site/
├── hugo.toml       ← サイト全体の設定ファイル
├── content/        ← 記事を書く場所
└── themes/         ← デザインテンプレート
```

## 3. フォルダ移動でつまずいた話

解凍した`yourlog-site`をターミナルで開こうとしたとき、最初にやりがちなミスがありました。

**ミス:1つ上の階層で`hugo server`を実行してしまう**

```powershell
PS C:\Users\user\OneDrive\ドキュメント\job\ブログ> hugo server
...
ERROR command error: Unable to locate config file or config directory.
```

これは`yourlog-site`フォルダの**1つ手前**にいる状態で実行してしまったのが原因でした。`hugo.toml`はそのフォルダの中にあるので、Hugoから見つけられずエラーになります。

**解決方法**

```powershell
cd yourlog-site
```

さらに、PowerShellを再起動するたびに現在地は毎回リセットされる(`C:\Users\user`に戻る)ことも学びました。そのたびに`cd`し直すのが面倒だったので、今は**エクスプローラーでフォルダを開いてから、アドレスバーに`powershell`と入力して起動する**方法に落ち着いています。こうすると、そのフォルダを開いた状態でターミナルが立ち上がるので`cd`が不要になります。

## 4. ようやく起動

正しいフォルダに移動してから、改めて実行。

```powershell
cd yourlog-site
hugo server
```

```
Web Server is available at http://localhost:1313/
Press Ctrl+C to stop
```

この表示が出れば成功です。ブラウザで`http://localhost:1313`を開くと、自分のパソコンの中だけで動いているプレビュー画面が表示されました。

## つまずきポイントのまとめ

| つまずいたこと | 原因 | 学んだこと |
|---|---|---|
| `sudo apt install hugo`が失敗 | Linux用コマンドをWindowsで実行していた | 自分の作業環境(OS)を意識する |
| `hugo server`で config file エラー | 正しいフォルダに`cd`できていなかった | `hugo.toml`があるフォルダの中にいるかを確認する癖をつける |
| PowerShellを開き直すたびに迷子になる | ターミナルの現在地は毎回リセットされる | フォルダを開いてからターミナルを起動すると手間が減る |

環境構築は地味ですが、ここで「今どこのフォルダにいるか」を意識する習慣がついたのは、この後のVPS作業(SSHでの接続先を意識する場面)にもそのまま活きてくる気がしています。
