# webサイト

1. `content/docs/posts/`以下に追加する記事のmarkdownファイルを追加
1. `hugo server`コマンドを利用してローカル環境で記事の確認を行う
1. `hugo`コマンドでビルドファイルを生成
1. git add + git commit + git pushでリモートに変更を反映
1. github actionsでmainブランチの内容が自動的にデプロイされる


- content/: 記事ファイル
- docs/: ビルドファイルの出力ディレクトリ
- static/: 静的ファイル置き場，画像はstatic/img/以下に格納する