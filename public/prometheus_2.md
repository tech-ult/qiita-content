---
title: Prometheus + Grafana で Webサービス監視環境を構築する - スロークエリ・エラーレート監視編 -
tags:
  - grafana
  - prometheus
  - スロークエリ
  - エラーレート
private: false
updated_at: '2025-05-25T10:21:11+09:00'
id: 5ef2200c414ce2b37396
organization_url_name: tech-ult
slide: false
ignorePublish: false
---
# はじめに

Webサービスのレスポンス速度が遅くて操作性が悪かったり、バグなどによる予期せぬエラーが発生していると、ユーザーに解約されるリスクが高まります。

- サービスが成長していくに従い、取り扱うデータ量も増大していき、データベースの検索スピードも遅くなり、ユーザー体験が悪化した
- サービスの新機能を他社よりも速くリリースして競合優位性を強化するため、クオリティーよりもデリバリーを優先してしまった結果、エラーが多発してユーザーからの問合せに対応するコストが増大した

上記のような事態に陥る前に、スロークエリやエラーをPrometheusで監視して、いち早く異常状態に気づけるようにしましょう！

# 前提条件

本記事は、以下について概要レベルの知識を有している読者を想定しています。

- Webアプリケーション開発
- [Docker環境構築方法](https://docs.docker.jp/index.html)
- [Slack APIの発行方法](https://api.slack.com/messaging/webhooks)
- [Gitコマンドの使い方](https://docs.github.com/ja/repositories/creating-and-managing-repositories/cloning-a-repository)

本記事で紹介する環境構築は、以下の環境で行いました。

- Windows11 WSL2(Ubuntu 22.04.5 LTS)
- Docker version 27.5.1
- Docker Compose version v2.34.0

本記事では、Prometheusによるモニタリングの中でも、**スロークエリやエラーレート等のモニタリング**に焦点を当てて紹介します。
PrometheusやGrafana自体の説明や、外形監視や短命ジョブの監視、および、検証のための環境構築方法については以下を参照してください。

[Prometheus + Grafana で Webサービス監視環境を構築する - 外形監視編 -](https://qiita.com/rharuki-tech-ult/items/b2e4cdf81e2bc97c6007)

[Prometheus + Grafana で Webサービス監視環境を構築する - 短命ジョブ監視編 -](https://qiita.com/rharuki-tech-ult/items/23381a9a555a44e9aeb6)

# 解説
## スロークエリについて

スロークエリとは、文字通り実行に時間がかかるクエリ（SQL）です。
ユーザーが画面上のボタンを押したけど、なかなか処理が進まないで待たされて、イライラがつのっていく。その原因の一つがスロークエリです。

主な発生原因は以下の通りです。

- インデックス設計漏れでテーブルをフルスキャンしている
- JOIN句で多数のテーブルを結合して検索している
- N + 1で大量のクエリを発行している
- トランザクションの粒度が粗く、ロック解除待ちに時間がかかっている
- そもそもデータベースの性能が低い

今回は監視がテーマなので、それぞれの詳細については、また別の機会に説明いたします。

## エラーレートについて

エラーレートとは、すべての通信の中でエラーが発生している割合のことです。
HTTPレスポンスのステータスコードを集計することで、エラーの割合を把握できます。

主なステータスコードは以下の通りです。

|ステータスコード|概要|
|--|--|
|200|OK: リクエスト成功|
|304|Not Modified: キャッシュ済みのレスポンスを利用する|
|403|Forbidden: リクエストされたリソースへのアクセス権がない|
|404|Not Found: リクエストされたリソースが存在しない|
|500|Internal Server Error: サーバー内部でエラーが発生|

エラーレートは、上記400系・500系のステータスコードの発生率を計算したものです。

なお、400系のエラーは、クライアント側の誤操作等で発生するエラーで、500系のエラーは、サーバー側の問題で発生するエラーです。
400系のエラーが短時間で多発していた場合、DDoS攻撃や脆弱性スキャンをされている可能性もあるため、時間当たりの発生率を監視する必要があります。
500系のエラーは、アプリケーション内部で何らかのバグが発生している可能性もあるため、発生したら即時にアラートを通知して調査・改修対応する必要があります。


## 監視環境の構成図（再掲）

今回構築する監視環境の構成図は以下の通りです。

![overview.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/de90ecb5-52d7-4a92-90ca-3e212060839d.png)

## 環境構築手順

### Grafanaのダッシュボード作成

今回は、Grafanaの公式ダッシュボードをインポートするのではなく、自分でパネルを作っていきます。
[Home > Dashboards > New dashboard] の画面で、[Add Visualization]ボタンを押下します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2c73e50d-d4a6-4f0e-9e9d-1155193bc665.png)
data sourceにはprometheusを選んでください。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/8292de36-8e05-41f5-805f-594b01dcb7de.png)
パネルの編集ページが表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/72d2dc0b-cd8d-4dc2-804d-225046b84dc6.png)
画面下部の[Enter a PromQL query...]の入力欄に、HTTPレスポンスのステータスコードごとの集計をするための以下のPromQLを入力して、[Run queries]ボタンを押下してください。
```SQL
sum by (status) (rate(nginx_http_response_count_total[1m]))
```
画面右上のVisualizationのドロップダウンで[Time series]がデフォルト選択されているため、時系列の折れ線グラフが表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/9e43d30e-803b-4622-b6e6-e03049551cb8.png)

ダッシュボードを忘れずに保存しておきましょう。[Save dashboard]ボタンから保存できます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/0535b419-a40f-4401-a82f-051ac0af4521.png)

画面右上の[Back to dashboard]ボタンを押して、一度ダッシュボードに戻り、画面上部の[Add]ドロップダウンから[Visualizagion]を選びましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/759021e0-c6e5-45b9-a4f4-dd86398f3782.png)
エラーレート表示用に、Visualizationから[Gauge]を選び、下記PromQLを入力してください。
```SQL
sum(rate(nginx_http_response_count_total{status=~"5.*|4.*"}[5m])) / sum(rate(nginx_http_response_count_total[5m]))
```
エラーが発生していないうちは、ゲージにはNo dataが表示されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2bef153c-6cd0-41d6-a678-b8bc50f263cc.png)
同様に、ステータスコードの総数を表示するため、Visualizationを追加しましょう。
[Bar chart]を選び、下記PromQLを入力してください。
```SQL
sum by (status) (nginx_http_response_count_total)
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/1b91ada5-fb9d-4df4-b157-5a846f3b7416.png)
同様に、ステータスコード分布を表示するため、Visualizationを追加しましょう。
[Pie chart]を選び、下記PromQLを入力してください。
```SQL
sum by (status) (nginx_http_response_count_total)
```
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/95204f64-ff9b-4b09-a8c0-8bc60d3b12f1.png)
同様に、スロークエリのTOP5を表示するため、Visualizationを追加しましょう。
[Table]を選び、下記PromQLを入力してください。
```SQL
avg_over_time(pg_stat_statements_mean_time[1m])
```
- `pg_stat_statements_mean_time` : クエリの平均実行時間
- `avg_over_time` : 指定した時間範囲（ここでは1分）にわたる平均値
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/058af9bc-aab2-40fe-a239-a445fc5b0f05.png)
TOP5だけを出したいので、画面下部の[Transformations]タブを選び、パネル表示内容を編集するため[Add transformation]ボタンを押下します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/33c867ce-3ece-42af-8fde-b936106c7307.png)
Reduceを選びます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/e5b00f70-c500-40f6-9cee-119c42dddcd2.png)
Caluculationsに[Max]を選択して、[Labels to fields]トグルをONにします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/4488cfa5-058e-4c82-978e-cedb8ee9220e.png)
表示するカラムを絞り込むため、Add another transformationからOrganize fieldsを選びます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/8e7f14a5-b828-487d-b696-9924d917b34c.png)
表示するカラムをqueryとMaxだけにして、その他のカラムを非表示に設定します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/e3aa8427-1504-4320-b2f8-0a641a4f5c7a.png)
クエリ実行時間の昇順に並び替えてTOP5を出すため、Add another transformationからSort byを選びます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/598f605b-20f0-433d-aa51-89f1a936192c.png)
FieldにMaxを選び、ReverseトグルをONにします。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/fc832dc8-8e47-467d-9772-7d159640e6e6.png)
5件のみ表示させるため、Add another transformationでLimitを追加します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/20ed0437-dd86-45d3-b599-79f878152b66.png)
Limitに5を設定します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/979feb53-8ef4-49c4-8310-3a483c3ce602.png)
作成したパネルは、ダッシュボード上で表示の大きさを替えたり、ドラッグアンドドロップで位置を替えたりできます。わかりやすいようにパネル名を編集するのも良いでしょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/ed632dbc-c971-43e6-afd9-73a8d43a23eb.png)


### アラート条件の設定

アラート条件の設定は`prometheus/alert_rules.yml`で定義しています。
今回は、以下のアラート条件を設定しました。

```YAML:prometheus/alert_rules.yml
  - name: nginx_alerts
    rules:
      - alert: AnyHttp5xxErrorDetected   # 5xxエラーが1回でも発生した場合のアラート
        expr: |
          sum(rate(nginx_http_response_count_total{status=~"5.."}[30s])) > 0
        for: 0s
        labels:
          severity: critical
        annotations:
          summary: "HTTP 5xx error detected"
          description: "At least one HTTP 5xx error occurred in the last 30 seconds."

      - alert: HighHttp4xxErrorRate    # 4xxエラー率が高い場合のアラート
        expr: |
          (sum(rate(nginx_http_response_count_total{status=~"4.."}[1m])))
          /
          (sum(rate(nginx_http_response_count_total[1m])))
          > 0.10
        for: 1m
        labels:
          severity: warning
        annotations:
          summary: "High HTTP 4xx error rate detected"
          description: "The HTTP 4xx error rate has exceeded 10% over the past 2 minutes."

  - name: postgres_alerts
    rules:
      # スロークエリを検知するアラート
      - alert: PostgresSlowQuery
        expr: max_over_time(pg_slow_queries_duration[1m]) > 5  # 過去1分間のうち、5秒以上実行されているクエリを検知
        for: 0s  # 検知したらすぐにアラート
        labels:
          severity: warning
        annotations:
          summary: "Slow PostgreSQL query detected"
          description: "Database: {{ $labels.database }}, User: {{ $labels.user }}, Query duration: {{ $value }}s, Query: {{ $labels.query }}"
```

- AnyHttp5xxErrorDetected
  - 500系のエラーが発生したら即時にアラート通知
- HighHttp4xxErrorRate
  - 400系のエラー発生率が10%を超えた状態が1分間継続した場合にアラート通知
- PostgresSlowQuery
  - 処理時間が5秒を超えるクエリがあった場合にアラート通知

## エラーレート・スロークエリのアラート通知検証

### 自作したWebアプリから、エラーやスロークエリを発生させる

http://localhost/
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/983ed308-46b5-4515-8661-af69e0939f67.png)

[404エラー][403エラー]ボタンを押して、エラーを発生させてみましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/6e638086-0c25-4147-ac0c-4026769ff1d5.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/8a79a5cd-f29d-4b05-a1df-6a46d7c7cd57.png)

Grafanaのダッシュボード上で、エラーを確認できます。エラーレートが10%を超えていないので、アラート通知されません。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2552a4fa-473f-4339-8bd2-14a3ffd8dfee.png)

エラーレートが10%を超えた状態を1分間継続させるため、[404エラー][403エラー]ボタンを定期的に押し続けてみましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/bdcd6dd0-219f-487b-984b-4747ad18d773.png)

Prometheusの画面上では、PENDING状態になっています。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/4bd6ae7f-34b3-4ac2-9421-a2867637b3f8.png)

1分間継続すると、FIRING状態になり、アラートが通知されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/43b102ed-37b8-4cfe-863d-dc196d02c5b9.png)

Slackにもアラートが連携されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/0b201969-773f-4165-b910-7cda5cfea75f.png)

次に、500系のエラーのアラート通知も確認しましょう。
[500エラー]ボタンを押下してください。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/767194ea-d367-4bf5-962d-1293434dfa06.png)

エラー検知します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/b274fa97-9825-45d3-a2d4-36ed7df541c0.png)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/0f2e9f30-6917-48a5-a183-fde0daa46544.png)

Slackにも通知されます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/544c3cfc-3eca-4a2e-b328-09559384a93a.png)

スロークエリも試してみましょう。
自作アプリケーションの[ユーザー一覧（Slowクエリ）]ボタンの内部処理は下記のとおり、10秒スリープ後にPostgreSQLのユーザー一覧を返却しています。

```javascript:app/app.js
// /user → 10秒待ってからユーザー一覧
app.get('/user', async (req, res, next) => {
  try {
    await client.query('SELECT pg_sleep(10)'); // 10秒待つ(スロークエリ)
    const result = await client.query('SELECT usename FROM pg_user');
    res.json(result.rows);
  } catch (err) {
    next(new AppError('Failed to fetch users', 500));
  }
});
```

スロークエリのアラート設定は前述通り、実行時間が5秒を超えるクエリが1分間継続することを条件としているため、[ユーザー一覧（Slowクエリ）]ボタンを1分間の間（計6回）押し続けてみましょう。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/a5171ae0-333b-41b8-86c1-1550d689dfa1.png)
そのうち、エラー検知します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/5dfc0b6d-54c4-41ab-a399-81941fc2a148.png)
ダッシュボードのスロークエリTOP5では、1位がsleepのクエリ（10秒）になっていることが確認できます。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/1d16727e-a75b-4718-b732-8097cf6bd80e.png)
Slackにもアラート通知します。
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/7f0d4cf1-b2ab-42e0-96b8-8af0a7ee1c27.png)

# まとめ

本記事では、Webサービスの品質劣化を未然に防ぐため、PrometheusとGrafanaを活用してスロークエリやHTTPエラーを監視する方法を解説しました。

サービスの成長に伴い、処理の遅延やエラーは避けがたいものですが、適切な監視体制を整えることで、ユーザー体験の悪化を未然に防ぐことができます。とくにスロークエリやエラーレートといった指標は、サービス運用上のボトルネックや障害の兆候を早期にキャッチするための重要な観測点です。

実際のプロダクトで問題が顕在化してから対応するのではなく、本記事で紹介したようなモニタリング環境を構築して、異常を「先に知る」体制を整えておくことが、信頼性の高いサービス運用につながります。

ぜひ、自社の環境にあわせてカスタマイズしながら、監視の精度と迅速なアラート対応体制を高めていきましょう。

一緒にOSSについて学んでいきたいなど、ご興味を持たれた方は、[弊社ホームページ](https://www.tech-ult.com/contact-8)からお問い合わせいただければ幸いです。

# 参考

[HTTP レスポンスステータスコード](https://developer.mozilla.org/ja/docs/Web/HTTP/Reference/Status)
