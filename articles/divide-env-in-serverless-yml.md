---
title: "serverless.yml 内で環境ごとにリソースや環境変数を分ける"
emoji: "🌟"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

# 概要
毎回調べるので書いてみる。

AWS Lambda を使ったプロジェクトで重宝している serverless framework だが、環境ごとに使うリソースを分けたい場合のやり方を示していく。
例えば「 DynamonDB のテーブルを環境ごとに用意したい」や「本番と開発で環境変数を分けたい」などの要望がある場合にこの方法を使っている。

フォルダー構成は以下を想定

```
.
├── README.md
├── bin
│   └── main
├── env
│   ├── dev.yml
│   └── prod.yml
├── main.go
└── serverless.yml
```

# 環境変数を分ける
いつもやっているやり方は env フォルダを作ってその配下に環境ごとに yml ファイル（dev.yml など）を作ってそこに書いているやり方である。

```yml:dev.yml
BUCKET: bucket-dev
TABLE: table-dev
```
