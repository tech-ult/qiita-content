---
title: Prometheus + Grafana で Webサービス監視環境を構築する - クライアントライブラリ監視編 -
tags:
  - grafana
  - prometheus
  - prom-client
private: false
updated_at: '2025-06-01T10:28:35+09:00'
id: c13771f54a64d8889fee
organization_url_name: tech-ult
slide: false
ignorePublish: false
---
# はじめに

これまで紹介してきたPrometheusのExporterによるメトリクス収集により、提供しているサービスが安定的に稼働しているかを確認できます。

しかし、Exporterでは以下に挙げるビジネス的な観点のメトリクス収集はできません。

- ユーザーのプロファイル（年代、性別、地域といった属性）毎の機能利用率
- 検索 → 詳細閲覧 → カート追加 → 購入のファネル追跡
- ユーザーが使用するデバイスごとの商品購入完了率

上記のようなメトリクスは、必要な項目をプログラム内部でログに出力して、週次・月次など定期的にレポーティングすることは可能です。
プレミアム会員の退会なども、退会機能を利用したユーザーのプロファイルをログに出力して、常時ログ監視しておけばアラート通知することは可能でしょう。

ただし、重要なビジネスメトリクスを迅速に収集しアラート通知したい場合、わざわざログ出力するよりも、Prometheusに直接メトリクスを収集させた方がリアルタイム性が高まります。

サービスに特化したビジネスメトリクスをプログラム内部でPrometheusに収集させるためには、Prometheusの公式クライアントライブラリを利用します。

今回は、クライアントライブラリをプログラム内部で呼び出し、Prometheusでメトリクスを収集する方法を紹介します。

# 前提

本記事は、以下について概要レベルの知識を有している読者を想定しています。

- Webアプリケーション開発
- [Docker環境構築方法](https://docs.docker.jp/index.html)
- [Slack APIの発行方法](https://api.slack.com/messaging/webhooks)
- [Gitコマンドの使い方](https://docs.github.com/ja/repositories/creating-and-managing-repositories/cloning-a-repository)

本記事で紹介する環境構築は、以下の環境で行いました。

- Windows11 WSL2(Ubuntu 22.04.5 LTS)
- Docker version 27.5.1
- Docker Compose version v2.34.0

本記事では、Prometheusによるモニタリングの中でも、**クライアントライブラリによるビジネスメトリクスのモニタリング**に焦点を当てて紹介します。
PrometheusやGrafana自体の説明や、外形監視やスロークエリ、短命ジョブの監視、および、検証のための環境構築方法については以下を参照してください。

[Prometheus + Grafana で Webサービス監視環境を構築する - 外形監視編 -](https://qiita.com/rharuki-tech-ult/items/b2e4cdf81e2bc97c6007)

[Prometheus + Grafana で Webサービス監視環境を構築する - スロークエリ・エラーレート監視編 -](https://qiita.com/rharuki-tech-ult/items/5ef2200c414ce2b37396)

[Prometheus + Grafana で Webサービス監視環境を構築する - 短命ジョブ監視編 -](https://qiita.com/rharuki-tech-ult/items/23381a9a555a44e9aeb6)

# 解説
## Exporterについて

Exporterは、監視対象のサーバーやプロセスにアクセスできる環境にインストール（デプロイ）して利用します。
PrometheusからのHTTPリクエストを受け取ると、監視対象のメトリクスを収集し、TEXT形式に整形してレスポンスします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/827b3f10-107f-4bf7-b617-dc2d8ea15bb2.png)

監視対象の種類（DB、ハードウェアなど）により様々なExporterが[公開](https://prometheus.io/docs/instrumenting/exporters/)されており、用途により適切なものを選んで利用します。

公式のExporterが欲しいメトリクスを収集していない場合や、そもそも公式のExporter自体が存在していない場合は、Exporterを自作して利用することも可能です。

## クライアントライブラリについて

クライアントライブラリは、開発言語ごとにPrometheusが公式に用意しているものや、コミュニティが有志で用意しているものが存在するので、アプリケーションの環境に応じて選定します。

| 開発言語    | クライアントライブラリ                                                                       |
| ------- | --------------------------------------------------------------------------------- |
| Go      | [prometheus/client\_golang](https://github.com/prometheus/client_golang)          |
| Java    | [prometheus/client\_java](https://github.com/prometheus/client_java)              |
| Python  | [prometheus/client\_python](https://github.com/prometheus/client_python)          |
| Ruby    | [prometheus/client\_ruby](https://github.com/prometheus/client_ruby)              |
| .NET    | [prometheus-net/prometheus-net](https://github.com/prometheus-net/prometheus-net) |
| Node.js | [siimon/prom-client](https://github.com/siimon/prom-client)                       |

クライアントライブラリは、監視対象のアプリケーションの中に組み込みます。
アプリケーションは、ユーザーの操作など任意のタイミングでクライアントライブラリの機能を使いメトリクスを都度記録していきます。

アプリケーション側でPrometheus用のメトリクスエンドポイントを用意し、Prometheusからリクエストを受け付けた際に内部で記録しておいたメトリクスをレスポンスします。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/23763aab-3b01-4f3f-85c6-54bf22eca275.png)

## クライアントライブラリとExporterの使い分けについて

クライアントライブラリは、アプリケーションと同一プロセスに組み込まれるため、アプリケーション自体のリソース使用率が高まります。重要な機能に絞ってメトリクスの収集を行い、処理負荷が高まりすぎないように注意が必要です。

一方で、Exporterは、アプリケーションとは別プロセスで起動するため、クライアントライブラリに比べてアプリケーションのリソース使用率を高めません。

クライアントライブラリとExporterの比較は以下の通りです。

| 項目                 | クライアントライブラリ                | Exporter                    |
| ------------------ | -------------------------------------- | ------------------------------------ |
| **取得可能なメトリクス**     | アプリケーション内部の詳細メトリクス（ビジネスロジック、ユーザーアクション） | システムやミドルウェアの外部メトリクス（CPU使用率、DB統計など）   |
| **リアルタイム性・精度**     | アプリケーション処理と同期して計測でき、ミリ秒単位での精密な監視が可能    | ポーリングベースのため遅延が発生する可能性あり、精度はやや劣る      |
| **カスタマイズ性**        | 任意のKPIや複雑な条件付きメトリクスを自由に定義可能            | Exporterの仕様に依存、独自メトリクスの追加は困難または制限あり  |
| **アプリへの影響**        | メトリクス収集処理がアプリ本体に影響を与える可能性あり            | アプリとは分離されており、Exporterの障害がアプリに影響を与えない |
| **保守性・コード管理**      | ロジックと監視コードが混在し、保守性や可読性が下がる場合がある        | アプリと分離されているため、役割分担が明確でコードも分離可能       |
| **障害耐性**           | アプリケーションがダウンするとメトリクスの取得も停止する                 | Exporterが別で動作していればメトリクス収集は継続可能       |
| **スケーラビリティと運用コスト** | アプリ内で処理するため、高トラフィック時にリソース競合が起こり得る      | Exporter側でリソースを分離・調整可能だが、運用負荷も増加     |

重要なビジネスメトリクスはクライアントライブラリで収集し、それ以外の技術的なメトリクスはExporterで収集させるなど、適材適所で使い分けて利用するのが良いでしょう。

## 監視環境の構成図（再掲）

今回構築する監視環境の構成図は以下の通りです。

![overview.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/de90ecb5-52d7-4a92-90ca-3e212060839d.png)

## 環境構築手順
### Grafanaのダッシュボード作成

独自メトリクスを可視化するためには、ダッシュボードを自作する必要があります。

[Home > Dashboards > New dashboard > Edit panel] からメトリクスごとのパネルを作成しましょう。

http_requests_total, http_request_duration_seconds, database_connections_active, database_query_duration_seconds, database_queries_total, user_actions_total, health_check_status, application_errors_total, slow_queries_total

**平均応答速度の可視化**

VisualizationにStatを選択して、以下のPromQLを入力しましょう。

```SQL
rate(http_request_duration_seconds_sum[5m]) / rate(http_request_duration_seconds_count[5m])
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/c48675e6-51b2-4543-8353-7a23a2722003.png)

**応答時間分布の可視化**

VisualizationにHeatmapを選択して、以下のPromQLを入力しましょう。
※データが存在しない場合はエラーが表示されます。

```SQL
rate(http_request_duration_seconds_bucket[5m])
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/3711b4a8-be0f-4b11-afba-ea8243c17722.png)


**ユーザーアクションの可視化**

VisualizationにTime seriesを選択して、以下のPromQLを入力しましょう。

```SQL
sum(rate(user_actions_total[5m])) by (action_type)
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/6f618919-d989-4e73-889c-15df4eb6bef1.png)

ダッシュボードを保存しましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/29afe530-e788-447d-9f01-11b323d3d634.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/868d5877-0738-422d-a09a-930a9627ea18.png)

### アラート条件の設定

アラート条件の設定は`prometheus/alert_rules.yml`で定義しています。
今回は、以下のアラート条件を設定しました。

```YAML:prometheus/alert_rules.yml
      # === ユーザーアクション関連のアラート ===
      - alert: WebappLowUserActivity # ユーザーアクティビティが低い場合のアラート（5分間でリクエストが10未満）※監視系エンドポイントを除外
        expr: rate(user_actions_total{endpoint!~"/health|/schema|/metrics|/status"}[5m]) * 300 < 10
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Low user activity detected on webapp"
          description: "User activity is unusually low. Less than 10 requests in the past 5 minutes."
```

- rate : 1秒あたりの平均値を算出する。×300して5分換算している。
- user_actions_total : クライアントライブラリで集計しているエンドポイント毎のアクセス数。常時リクエストを送っている監視エンドポイントは除外している。

### クライアントライブラリでメトリクス収集

自作のアプリケーションには、独自にメトリクスを収集する処理を埋め込みました。

```javascript:app/app.js
// HTTPリクエストの応答時間を測定（ヒストグラム）
const httpRequestDuration = new clientProm.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.1, 0.3, 0.5, 0.7, 1, 3, 5, 7, 10] // レスポンス時間のバケット設定
});

// アプリケーション固有のビジネスメトリクス
const userActionsCounter = new clientProm.Counter({
  name: 'user_actions_total',
  help: 'Total number of user actions',
  labelNames: ['action_type', 'endpoint'], // ユーザーアクション種別を追跡
});
```

## クライアントライブラリのアラート通知検証

サービス自体は正常に起動しているけど、ユーザー操作がされない時間が続くと、ユーザーが離脱しているか、ネットワークに問題が発生している可能性があります。
今回は、5分間のうちにユーザー操作のリクエスト数が10回未満の場合にアラート通知する設定を入れています。

自作アプリケーションで会社一覧ボタンを押してリクエストを送ってみます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/7cdbd557-755e-404b-a88d-5c0d1a2a4af6.png)

Grafanaを確認してみます。
各パネルに反応がありました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/bfde54a1-5c7c-41c2-8075-e7b850043136.png)

5分間ユーザー操作を放置して、アラート通知を確認します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/0b389d40-9dac-4a8b-b288-11e2d81be92b.png)

PrometheusのAlertsから`WebappApplicationErrors`の詳細を開くと、過去5分間でユーザー操作が10回未満のエンドポイントを確認できます。
エンドポイントに対して10回以上リクエストするとアラート対象から外れます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2f615cfc-575f-4cd4-8877-de1a8dbf9ea7.png)

# まとめ

本記事では、Prometheusのクライアントライブラリを活用して、アプリケーション内のビジネスメトリクスをリアルタイムに収集・監視する方法を解説しました。

クライアントライブラリを使用することで、Exporterでは対応できない、ユーザー行動やKPIの可視化・監視が可能になります。

また、Grafanaによるダッシュボード可視化とPrometheus Alertmanagerによるアラート通知も組み合わせることで、プロダクトの健全性をビジネス視点で把握できる体制が構築できます。

なお、クライアントライブラリは便利ですが、アプリへの影響や保守性も考慮し、Exporterとの適切な使い分けが重要です。

一緒にOSSについて学んでいきたいなど、ご興味を持たれた方は、[弊社ホームページ](https://www.tech-ult.com/contact-8)からお問い合わせいただければ幸いです。
