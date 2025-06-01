---
title: Prometheus + Grafana で Webサービス監視環境を構築する - 短命ジョブ監視編 -
tags:
  - grafana
  - prometheus
  - pushgateway
  - Short-lived_jobs
private: false
updated_at: '2025-05-25T10:12:44+09:00'
id: 23381a9a555a44e9aeb6
organization_url_name: tech-ult
slide: false
ignorePublish: false
---
# はじめに

Webアプリケーションは、ユーザーがブラウザで操作することを前提としているため、ユーザーを待たせてしまうような実行時間が長い処理は敬遠されます。
画面のボタンを押した後、次の操作ができるまで数分待たされるようなサービスは、誰も使いたがらないですよね。

解決策として、メッセージキューを使って別のバッチジョブに重たい処理を任せて、すぐにユーザーへレスポンスを返し、操作を継続できるようにする手法がよく用いられます。

メッセージキューのリクエストを処理するバッチジョブの実行時間が、想定していたよりも長時間かかる場合、何かしらの問題が発生している可能性があります。
バッチジョブもスロークエリと同様、処理時間がPrometheusで定義した閾値を超えた場合にアラート通知して、速めに原因調査と対策をとっていく必要があります。

ここで一つ課題が発生します。
バッチジョブは常時起動しておらず、メッセージキューにリクエストが登録されることをトリガーに起動し、処理が終わるとジョブ自体もなくなります。
一方で、Prometheusは一定周期でExporterにリクエストを送ってメトリクスを収集しています。
Prometheusがメトリクスを収集するタイミングでバッチジョブが存在していることを前提とするため、すぐに処理終了するバッチジョブの場合はメトリクスを収集できず、異常を検知できません。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/abdec96f-72d5-44f6-b055-273cf150379b.png)

そこで登場するのが「Pushgateway」と呼ばれる、Prometheus向けのメトリクス中継サービスです。
今回は、Pushgatewayを使ったアラート通知について説明します。

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

本記事では、Prometheusによるモニタリングの中でも、**Pushgatewayを用いた監視**に焦点を当てて紹介します。
PrometheusやGrafana自体の説明や、外形監視、スロークエリやエラーレート監視、および、検証のための環境構築方法については以下を参照してください。

[Prometheus + Grafana で Webサービス監視環境を構築する - 外形監視編 -](https://qiita.com/rharuki-tech-ult/items/b2e4cdf81e2bc97c6007)

[Prometheus + Grafana で Webサービス監視環境を構築する - スロークエリ・エラーレート監視編 -](https://qiita.com/rharuki-tech-ult/items/5ef2200c414ce2b37396)

# 解説
## メッセージキューについて

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/f7edec5e-86e9-42bc-9213-13c10fe03b55.png)

メッセージキューについて、大まかな仕組みとしては以下の通りです。
ユーザーからのリクエストをWebアプリケーションが直接処理するのではなく、いったんメッセージキューに登録だけ行い、ユーザーには処理を受け付けたことをレスポンスして、ユーザー操作を継続可能にします。
その後、非同期で別のバッチジョブが起動し、メッセージキューからリクエストを取り出して処理を実行します。
バッチジョブは、以下に挙げるような方法で処理終了したことをWebアプリケーションに通知して、ジョブを停止します。
- Webアプリケーションとバッチジョブが共有するデータベースに処理ステータスを書き込む
- Webアプリケーションのエンドポイントにリクエストを送る

## Pushgatewayについて

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/21e52599-ffbd-42ed-8dcd-db0f920187a6.png)

Pushgatewayはメッセージキューのような役割を担い、Prometheusとバッチジョブの間でメトリクスの中継を行います。
バッチジョブが処理終了する際、Pushgatewayにメトリクスを送信します。
Prometheusは、Pushgatewayに蓄積されたバッチジョブのメトリクスを一定周期で収集します。

なお、Pushgatewayに蓄積されたメトリクスは、APIやWebUIから明示的に削除を実行しない限り残り続けます。
Prometheusは、メトリクスを収集するだけで、Pushgateway上のデータを削除できません。

Prometheusは定期的にメトリクスを収集するため、すでに一度読み取ったメトリクスが何度も収集される可能性があります。
Prometheusが過去の異常値を繰り返し収集してしまうと、毎回アラートが発生する恐れがあります。これを防ぐには、Pushgatewayのメトリクスを定期的に削除する仕組みが必要です。

詳細は割愛しますが、クリーンアップ用のバッチを別途用意して、一定時間ごとにPushgateway上のメトリクスを削除していく仕組みを実行するなど、メトリクスの蓄積によるアラート誤発報を防ぐための、削除やクリーンアップの対策が必要です。
アラート通知が上がった際にGrafanaのダッシュボードを確認する時間を考慮すると、短すぎる周期ではなく、ある程度余裕を持たせた周期でクリーンアップを実行するとよいでしょう。

## 監視環境の構成図（再掲）

今回構築する監視環境の構成図は以下の通りです。

![overview.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/de90ecb5-52d7-4a92-90ca-3e212060839d.png)

## 環境構築手順

### Grafanaのダッシュボード作成

[Home > Dashboards > New dashboard > Edit panel] から、Pushgatewayのメトリクスを表示するダッシュボードを作りましょう。

Metrics browserにPromQLを入力して、VisualizationsにTime seriesを選択してパネルを作成します。

```SQL
batch_duration_seconds{exported_job=~"sample_.*"}
```

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/76f86ef0-378e-4def-8a4b-52edf97f3ba9.png)

続けて、直近のバッチ処理時間を一覧化するパネルを追加します。PromQLは先ほどと同様で、VisualizationsにTableを選択してパネルを作成します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/1551a313-dd0c-49d7-a0d6-ae40bef1c0e6.png)

Pushgatewayから明示的にメトリクスを削除するまでは、Prometheusは同じバッチジョブのメトリクスを何度も収集するため、バッチジョブ名でフィルタリングして最新の1レコードのみを表示させます。
TransformationからReduceを追加して、CalculationsにLast*を選択しましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/a4163e1d-5bcf-45b8-9ce5-a003ff3c4e3d.png)

exported_job名と実行時間だけ表示させたいので、TransformationからOrganize fields by nameを追加して、exported_jobとLast*以外は非表示にしましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/d51ca7e3-8a4b-4577-b74a-28255754017d.png)

バッチ処理時間の閾値の10秒を赤いラインで示したいので、Thresholdsで固定値10を赤色で設定します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/6692fd7e-d00a-4370-a42a-da4296f3bcb1.png)

バッチ処理時間の表示幅を0～10秒で出すため、AxisのSoft minに0、Soft maxに10を設定します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/b1b89fd7-9574-4517-8ada-f44c7fe9e9ca.png)

パネルの表示位置を調整して、ダッシュボードを保存しましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/31b6910e-50db-4cc7-ac87-844b6395f680.png)

### アラート条件の設定

アラート条件の設定は`prometheus/alert_rules.yml`で定義しています。
今回は、以下のアラート条件を設定しました。

```YAML:prometheus/alert_rules.yml
  - name: pushgateway_alerts
    rules:
      - alert: PushJobDurationTooLong
        expr: batch_duration_seconds{exported_job=~"sample_.*"} > 10 # ジョブ実行時間が10秒を超えたら
        for: 0s  # すぐにアラートを発火
        labels:
          severity: warning
        annotations:
          summary: "Pushgateway job duration too long"
          description: "A push job took more than 10 seconds to complete. Duration: {{ $value }}s, Job: {{ $labels.exported_job }}"
```

## 短命ジョブのアラート通知検証
### バッチジョブの作成

今回は、簡易的なシェルスクリプトを作成して、バッチジョブからPushgatewayにリクエストを送る動作検証を行います。
シェルのパラメータでジョブIDと処理時間（秒）をパラメータで受け取り、指定された秒数sleepして、バッチの処理時間を算出してPushgatewayにcurlコマンドでメトリクスをPushするだけのものです。

```bash:pushgateway/push_duration.sh
#!/bin/bash

# 引数チェック
if [ "$#" -ne 2 ]; then
  echo "Usage: $0 <request_id> <sleep_seconds>"
  exit 1
fi

REQUEST_ID="$1"
SLEEP_SECONDS="$2"

# バリデーション（sleep秒数は整数のみ許可）
if ! [[ "$SLEEP_SECONDS" =~ ^[0-9]+$ ]]; then
  echo "Error: sleep_seconds must be a positive integer"
  exit 1
fi

# 開始時刻（秒）
START_TIME=$(date +%s)

# ----------------------------------
# バッチ処理の代わりに sleep
sleep "$SLEEP_SECONDS"
# ----------------------------------

# 終了時刻（秒）
END_TIME=$(date +%s)

# duration 計算
DURATION=$((END_TIME - START_TIME))

# ジョブ名の生成
TIMESTAMP=$(date +"%Y%m%d%H%M%S")
JOB_NAME="sample_${REQUEST_ID}_${TIMESTAMP}"

# Pushgateway URL
PUSHGATEWAY_URL="http://localhost:9091"

# メトリクスをPush
cat <<EOF | curl --data-binary @- "${PUSHGATEWAY_URL}/metrics/job/${JOB_NAME}"
# TYPE batch_duration_seconds gauge
batch_duration_seconds ${DURATION}
EOF

echo "Pushed batch_duration_seconds=${DURATION} to job=${JOB_NAME}"
```

また、Pushgatewayに蓄積されたメトリクスを削除するためのシェルスクリプトも作成しておきましょう。
Pushgatewayからメトリクスの一覧を取得し、1件ずつDELETEするだけのものです。

```bash:pushgateway/clean_pushed_jobs.sh
#!/bin/bash

PUSHGATEWAY_URL="http://localhost:9091"
PROMETHEUS_URL="http://localhost:9090"
JOB_PREFIX="sample_"

echo "=== Fetch job list from Pushgateway ==="

job_list=$(curl -s "$PUSHGATEWAY_URL/metrics" | grep '^batch_duration_seconds{' | sed -E 's/.*job="([^"]+)".*/\1/' | sort | uniq)

echo "=== job_list ==="
echo "$job_list"
echo "==============="

for job in $job_list; do
  if [[ "$job" =~ ^$JOB_PREFIX ]]; then
    echo "🔍 Checking Prometheus for job: '$job'"

    query="batch_duration_seconds{exported_job=\"$job\"}"
    result=$(curl -s --get --data-urlencode "query=$query" "$PROMETHEUS_URL/api/v1/query" | jq -r '.data.result | length')

    echo "Result count: $result"

    if [[ "$result" -gt 0 ]]; then
      echo "✅ Job '$job' found in Prometheus. Deleting from Pushgateway..."
      curl -s -X DELETE "$PUSHGATEWAY_URL/metrics/job/$job"
      echo "Deleted job '$job' from Pushgateway."
    else
      echo "⏭️  Job '$job' not yet collected by Prometheus. Skipping deletion."
    fi
  fi
done
```

### メトリクスをPushし、アラート通知確認

事前に用意したシェルスクリプトを使って、アラート通知を検証します。
ジョブIDにid_0001、処理時間（秒）に3を指定してシェルスクリプトを実行します。
なお、閾値の10秒超を指定していないため、アラート通知はされません。

```bash
$ . pushgateway/push_duration.sh id_0001 3
Pushed batch_duration_seconds=3 to job=sample_id_0001_20250525092623
```

PushgatewayのWebUIで、Pushされたメトリクスを確認できます。

[http://localhost:9091/](http://localhost:9091/)
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2f00c433-3c3e-411d-80f8-0be04e114ea4.png)

Grafanaのダッシュボードでもメトリクスを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/4c943a2c-5c33-4fca-9d8b-523538dd6c92.png)

それでは、シェルスクリプトで11秒を指定して、アラート通知を確認してみましょう。

```bash
$ . pushgateway/push_duration.sh id_0002 11
Pushed batch_duration_seconds=11 to job=sample_id_0002_20250525093138
```

Pushgatewayにメトリクスが送られます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/2726bcb6-898a-41d3-97f5-0d929f34e785.png)

Prometheusでアラートを検知しました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/08131fb9-b337-4a39-8374-1acf5394922d.png)

Slackにもアラートが通知されました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/e32825af-439d-4e32-9b4d-9ae3b3b18db5.png)

Grafanaのダッシュボードでも閾値を超えたことを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/f0dce4c8-cd34-466a-beca-069b136387b7.png)

最後に、クリーンアップのシェルスクリプトでPushgatewayのメトリクスを削除しておきましょう。

```bash
$ . pushgateway/clean_pushed_jobs.sh 
=== Fetch job list from Pushgateway ===
=== job_list ===
sample_id_0001_20250525092623
sample_id_0002_20250525093138
===============
🔍 Checking Prometheus for job: 'sample_id_0001_20250525092623'
Result count: 1
✅ Job 'sample_id_0001_20250525092623' found in Prometheus. Deleting from Pushgateway...
Deleted job 'sample_id_0001_20250525092623' from Pushgateway.
🔍 Checking Prometheus for job: 'sample_id_0002_20250525093138'
Result count: 1
✅ Job 'sample_id_0002_20250525093138' found in Prometheus. Deleting from Pushgateway...
Deleted job 'sample_id_0002_20250525093138' from Pushgateway.
```

PushgatewayのWebUIからメトリクスが削除されたことを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/3317599d-61ab-4991-a92a-15f4a4adcb50.png)

Prometheusのアラートも正常に戻りました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/efeceeb6-82b0-4bee-87bc-86dfdb8124c9.png)

Slackにもアラート状態が回復されたことが通知されました。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/dd4e7daa-8e7c-4e12-9faa-4b6a06554c5e.png)

Grafanaのダッシュボードでもメトリクスが削除されたことを確認できます。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/7a69c5e7-0656-490e-870e-6741fcc70213.png)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/80d42b2d-4e66-40d0-8a38-d0ef3ba63b5d.png)

# まとめ

本記事では、短命なバッチジョブの実行時間を監視し、一定時間を超えた処理に対してアラート通知を行う方法として、Pushgatewayを用いたPrometheus監視の手法を紹介しました。

Webアプリケーションにおいて、ユーザー体験を損なわないために非同期バッチ処理を採用するケースは多くありますが、それに伴い即時起動・即時終了するジョブの監視の難しさが生まれます。

Pushgatewayはその課題を解決する手段として非常に有効です。ジョブ終了時にメトリクスをPushすることで、Prometheusが定期収集するモデルと両立しながら、ジョブの実行時間を確実に監視できます。また、Grafanaを用いた可視化やアラートルールの設定によって、異常の早期発見と対処が可能になります。

ただし、Pushされたメトリクスは明示的に削除しない限り残り続けるため、不要なアラートの再発防止のためのクリーンアップ処理は必須です。

この手法を導入することで、ユーザーに見えないバッチ処理の遅延を素早くキャッチでき、より信頼性の高いWebアプリケーションの運用につながるでしょう。

一緒にOSSについて学んでいきたいなど、ご興味を持たれた方は、[弊社ホームページ](https://www.tech-ult.com/contact-8)からお問い合わせいただければ幸いです。

# 参考

[ウィキペディア：メッセージキュー](https://ja.wikipedia.org/wiki/%E3%83%A1%E3%83%83%E3%82%BB%E3%83%BC%E3%82%B8%E3%82%AD%E3%83%A5%E3%83%BC)
