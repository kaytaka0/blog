# Blog執筆用リポジトリ

Hugo+hugo-book(theme)+Github Pagesによるブログサイト構成

## 記事の投稿までの手順

1. `content/docs/posts/`以下に追加する記事のmarkdownファイルを追加
1. `hugo server`コマンドを利用してローカル環境で記事の確認を行う
1. git add + git commit + git pushでリモートに変更を反映
1. github actionsでmainブランチの内容が自動的にビルド+デプロイされる


## ディレクトリ構成

- content/: 記事ファイル
- docs/: ビルドファイルの出力ディレクトリ
- static/: 静的ファイル置き場，画像はstatic/img/以下に格納する

## TODO
- OGPの自動生成機能の実装
