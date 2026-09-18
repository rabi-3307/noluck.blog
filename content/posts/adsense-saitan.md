+++
title = "【最短手順】AdSenseコードの貼り方"
date = 2026-09-07
type = "posts"
categories = ["beginners"]
tags = ["AdSense", "最短手順"]
+++

## やること

1. `themes/(テーマ名)/layouts/_default/baseof.html` を開く
2. `</head>` の直前に、AdSenseが発行したコードを貼る
3. 保存する
4. 以下を実行する

```powershell
hugo
scp -r public\* root@(IPアドレス):/var/www/yourlog/public/
```

5. SSHで接続する

```powershell
ssh root@(IPアドレス)
```

6. 反映させる

```bash
sudo chmod -R 755 /var/www/yourlog && sudo systemctl restart nginx
```

7. ブラウザで自分のサイトを開き、正しく表示されるか確認する

## 詳しい説明が読みたい場合

→ [1つのファイルを直すだけで全ページに反映される仕組み(baseof.htmlとは)](/posts/baseof-shikumi/)
