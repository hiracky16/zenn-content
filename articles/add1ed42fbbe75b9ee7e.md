---
title: "Google Cloud で作るサッカーのスターティングイレブン最適化パイプライン"
emoji: "⚽"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GoogleCloud", "BigQuery", "CloudRun", "CloudWorkflows"]
published: false
---

# 概要
2022 年はワールドカップが冬に開催ということでサッカーを見ながらアドベントカレンダーを執筆しております。
今回は FPL（後述）というサッカーに関連するゲームのデータを使ってベストなスターティングイレブンを数理最適化を用いて選んで行きたいと思います。
またゲームの都合上、週毎にデータを更新しなければならないので Google Cloud の技術を使いデータ更新からクレンジング、最適化までを一気通貫でできるようにしたのでそのアーキテクチャの紹介も兼ねます。
割とどの題材にも使えそうな構成だと思っております。

## Fantasy Premier League（FPL）

イングランドのプレミアリーグのリアルなデータを使ってやるゲームです。
実際の試合ごとに出た選手にはパフォーマンスによってポイントが加算され、ポイントの合計が一番多いプレイヤーが勝利となります。

# アーキテクチャ
ざっくりこんな感じです。

![](/images/optimize-archi.png)

以下これらを説明していきます。

# データ取得から作成まで
「ゴミデータを使ってなにかやってもゴミしか生まれない」とも言いますし良い最適化も良いデータを作ることが大切です。
更にゲームの要件的にそれを定期的に作成して最新の状態にしておく必要があります。

最適化のためのデータ作成は 3 フェーズに分かれて、それぞれを Cloud Run jobs にデプロイしています。
Cloud Run jobs は Docker コンテナの Entrypoint を実行してくれるバッチサービスでどんな言語でも Dockerfile にさえ記述できれば実行できます。
例えば `fetch` のタスクは API のレスポンスを GCS バケットに保存するだけなのでシェルスクリプトで記述しています。

```shell:entrypoint.sh
curl http://example.com > result.json # url は適当
gsutil mv result.json gs://fpl-rawfile-bucket/dt=2020-01-01/result.json
```

```docker:Dockerfile
FROM gcr.io/google.com/cloudsdktool/google-cloud-cli:alpine
RUN apk --update add curl
ENTRYPOINT ["sh", "entrypoint.sh"]
```

次に `parse` では `fetch` で保存した result.json を csv に変換する処理を行っているのですがデータがより扱いやすい python で記述しています。
`dbt` では dbt というデータモデリングのツールで SQL によりデータクレジングを行っています。
このように多様な処理を Cloud Run jobs で細切れでしかも違う言語で実現することができます。
解く最適化によってもデータは様々なので、今後その特性に対応するためにも自由度の高い Cloud Run を採用しています。
※ Cloud Run jobs は執筆当時は GA ではないので注意！

これらの順序関係は Cloud Workflows で管理しています。
Cloud Workflows は yaml を記述することにより Google Cloud サービスを簡単に順番に呼び出すことが可能です。
この処理は週次で呼び出す必要があるためこの Workflow を Cloud Scheduler にて呼び出して運用しています。（terraform でデプロイしているため若干記法が違う）

```yaml:workflow.yaml
cloudrun_jobs:
  params: [job]
  steps:
  - fetch:
      call: googleapis.run.v1.namespaces.jobs.run
      args:
        name: $${"namespaces/" + project_number + "/jobs/fetch"}
        location: asia-northeast1
      result: resp
  - parse:
      call: googleapis.run.v1.namespaces.jobs.run
      args:
        name: $${"namespaces/" + project_number + "/jobs/parse"}
        location: asia-northeast1
      result: resp
  ...
```

最終的に作られたデータは GCS に保存するのだが以下のようなフォーマットとなります。
最適化に使うのが以下のカラムです。

- name ... 選手名
- element_type_name ... ポジション名
- team_name ... チーム名
- now_cost ... 選手獲得にかかるコスト
- expected_points ... 今週獲得できそうなポイント

```
    id               name element_type_name      team_name  now_cost status  chance_of_playing  total_points  expected_points
0  501  NathanielPhillips               DEF      Liverpool        39      a                1.0             0              0.0
1  378         HarryArter               MID  Nott'm Forest        43      a                1.0             0              0.0
2  502        ScottCarson               GKP       Man City        38      a                1.0             0              0.0
3  393        LoïcMbe Soh               DEF  Nott'm Forest        42      a                1.0             0              0.0
4  328          PhilJones               DEF        Man Utd        39      a                1.0             0              0.0
```

# 最適化部分
今回の最適化では複数人の選手を選びポイントを最大化する必要があります。
ゲームの特性上毎週メンバーを選んでいるので「前週のメンバーから n 人変えてポイントを最大化する」という要件になります。
またルール上いくつか制約があるのでそれらをコードに落とし込んでいきます。
最適化に使うモジュールはご存知 `pulp` です。

## 制約一覧
- 選手獲得の予算
- 同じチームの選手は 3 人以下
- ポジションごとの制約
  - FW は 3 人
  - MF は 5 人
  - DF は 5 人
  - GK は 2 人
  - ※ 更にこの中から 11 人選ぶ

```python:optimizer.py
# lpVariable 生成
fun = lambda x: pulp.LpVariable(f'{x.id}_{x.element_type_name}_{x.team_name}', cat='Binary')
master['variables'] = list(master.apply(fun, axis=1))

# 最適化問題をオブジェクト化
prob = pulp.LpProblem('fpl_planner', sense = pulp.LpMaximize)

# 制約

# ポジション(element_type_name)ごとの人数の制約
PLAYER_LIMIT_BY_POSITIONS = {'GKP': 2, 'DEF': 5, 'MID': 5, 'FWD': 3}
for p in master.element_type_name.unique():
    dt = master[master.element_type_name == p]
    prob += pulp.lpSum(dt.variables) == PLAYER_LIMIT_BY_POSITIONS[p]

# 同じチーム 3 人の制約
SAME_TEAM_LIMIT = 3
for t in master.team_name.unique():
    dt = master[master.team_name == t]
    prob += pulp.lpSum(dt.variables) <= SAME_TEAM_LIMIT

# お気に入りチーム選手を絶対一人入れる
FAVORIT_TEAMS = ['Man Utd']
for t in FAVORIT_TEAMS:
    dt = master[master.team_name == t]
    prob += pulp.lpSum(dt.variables) >= 1

# 前週のスタメンとの交代選手数（例えば前週のメンバーと 3 人変える場合）
TOTAL_PLAYERS = 15
replacement = 3
dt = master[master.id.isin(current.element.values)]
prob += pulp.lpSum(dt.variables) == TOTAL_PLAYERS - replacemen

# コストの制限
COST_LIMIT = 1000
prob += pulp.lpDot(master.now_cost, master.variables) <= COST_LIMIT
# 全体の選手数の制限
TOTAL_PLAYERS = 15
prob += pulp.lpSum(master.variables) == TOTAL_PLAYERS
```

目的関数は以下。
`expected_points` はデータ作成時に計算した今週かくできそうなポイントです。
いずれはここを機械学習などを用いて算出できるようにしたいです。

```python:optimizer.py
# 目的関数
prob += pulp.lpDot(master.expected_points, master.variables)
```

これを解くと先週のメンバーからどの選手を変えたらよいか予算や人数の制約から選んでくれます。

```json:result
{
   "expected_points":38.599999999999994,
   "out_elements":[
      {
         "id":285,
         "element_type_name":"DEF",
         "name":"TrentAlexander-Arnold",
         "expected_points":0.2,
         "now_cost":72
      },
      {
         "id":340,
         "element_type_name":"MID",
         "name":"JadonSancho",
         "expected_points":0.6,
         "now_cost":72
      },
      {
         "id":80,
         "element_type_name":"FWD",
         "name":"IvanToney",
         "expected_points":0.0,
         "now_cost":74
      }
   ],
   "in_elements":[
      {
         "id":357,
         "element_type_name":"DEF",
         "name":"KieranTrippier",
         "expected_points":5.4,
         "now_cost":59
      },
      {
         "id":427,
         "element_type_name":"FWD",
         "name":"HarryKane",
         "expected_points":6.4,
         "now_cost":116
      },
      {
         "id":314,
         "element_type_name":"MID",
         "name":"PhilFoden",
         "expected_points":8.0,
         "now_cost":84
      }
   ],
}
```


# アプリケーション部分
最適化部分は Cloud Run のサービスとして Web API 化しました。
その API を Next.js ベースのアプリケーションから呼ぶことで最適化を呼び出すことができます。
画面はこんな感じ。

![](/images/optimize-screenshot.png)

# まとめ
