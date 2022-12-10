---
title: "Dataform cli を Cloud Run Jobs で実行！"
emoji: "🔥"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["gcp", "bigquery", "dataform"]
published: false
---

# 動機
- Dataform をイベント駆動で実行したい
- Dataform には REST API が用意されておりリクエストとすると実行が可能
- しかし REST API は Beta 版である(2022/07 時点)
- また REST API は不安定でレスポンスが返ってこないことが多々ある
- Dataform cli を実行するためのインフラとして Cloud Run (Jobs) を使ってみた

# Datafor cli

 - Dataform をイベント駆動で実行したい
 11 - Dataform には REST API が用意されておりリクエストとすると実行が可能
 12 - しかし REST API は Beta 版である(2022/07 時点)
 13 - また REST API は不安定でレスポンスが返ってこないことが多々ある
 14 - Dataform cli を実行するためのインフラとして Cloud Run (Jobs) を使ってみたhttps://docs.dataform.co/dataform-cli
