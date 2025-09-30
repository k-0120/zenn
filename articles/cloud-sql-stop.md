---
title: "【Google Cloud】特定時刻になったらCloud SQLのインスタンスを停止する"
emoji: "🕑️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["GoogleCloud", "CloudScheduler", "CloudFunctions", "CloudSQL", "PubSub"]
published: true
---

# これなに

- Google CloudのCloud SQL便利ですよね！
- 個人の学習用で使ったりするんですが、つい停止し忘れて無駄に課金されてしまったり…。
- 今回は、特定の時刻になったらCloud SQLのインスタンスを停止する方法について解説してみようと思います。

# 構成

- Cloud Schedulerで定期的にPub/Subから特定のトピックにメッセージを送信し、それをトリガーにCloud Run functionsでCloud SQLのインスタンスを停止するスクリプトを実行する流れになっています。
![configuration_diagram](/images/cloud-sql-stop/configration_diagram.png)

# Pub/Subでトピックを作成

- Cloud Schedulerから送信されたメッセージを受信するためです。

```sh
export TOPIC_ID="patch-cloud-sql-topic"
gcloud pubsub topics create "${TOPIC_ID}"
```

# Cloud Schedulerでスケジュールを作成

- 毎日日本時間の19時に、先程作成したトピックに対してメッセージを送信するためです。
  - メッセージの`instance_id`には、利用されているCloud SQLのインスタンスIDを指定してください。
  - 今回は停止させるため`activation_policy`には"NEVER"という文字列を指定しました。
  - "ALWAYS"を渡すことで起動させることも可能です。

```sh
export INSTANCE_ID="your cloud sql instance_id"
export SCHEDULE_ID="stop-cloud_sql-schedule"
gcloud scheduler jobs create pubsub ${SCHEDULE_ID} \
  --schedule="0 19 * * *" \
  --time-zone="Asia/Tokyo" \
  --topic="${TOPIC_ID}" \
  --location=us-central1 \
  --message-body="{\"instance_id\":\"${INSTANCE_ID}\",\"activation_policy\":\"NEVER\"}"
```

# Cloud Run functions用のサービスアカウントの作成と権限付与

- Cloud SQLを停止するためにサービスアカウントを作成し権限を付与します。
  - 念の為、ログも出力できるようにしておきましょう。

```sh
export RUN_SA="patch-cloud-sql-run-sa"
gcloud iam service-accounts create "${RUN_SA}" 

export PROJECT_ID=$(gcloud config get-value project)
gcloud projects add-iam-policy-binding "${PROJECT_ID}" \
  --member="serviceAccount:${RUN_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/cloudsql.admin"
gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member="serviceAccount:${RUN_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/logging.logWriter"
```

# Pythonスクリプトを作成

- Pub/Subのメッセージをもとに、Cloud SQLインスタンスを停止するスクリプトファイル(`main.py`)と、モジュールのインストール用ファイル(`requirements.txt`)を作成します。
- カレントディレクトリに`patch-cloud-sql`ディレクトリを作成し、その中に上記2ファイルを配置しましょう。

```python:main.py
import base64
import json
import functions_framework
import google.auth
from googleapiclient import discovery

credentials, project_id = google.auth.default()
service = discovery.build("sqladmin", "v1", credentials=credentials)

PROJECT_ID = project_id

@functions_framework.cloud_event
def patched_cloud_sql(cloud_event):
  try:
    msg = cloud_event.data.get("message", {})
    data_b64 = msg.get("data", "")
    payload = json.loads(base64.b64decode(data_b64).decode("utf-8")) if data_b64 else {}
        
    instance_id = payload.get("instance_id")
    activation_policy = payload.get("activation_policy")
    body = {"settings": {"activationPolicy": activation_policy}}
    op = service.instances().patch(
      project=PROJECT_ID,
      instance=instance_id,
      body=body
    ).execute()

    print("success")
    return {"status": "success", "operation": op.get("name")}
    
  except Exception as e:
    print("error")
    print(str(e))
    return {"status": "error", "message": str(e)}, 500
```

```txt:requirements.txt
functions-framework==3.*
google-api-python-client
google-auth
google-auth-httplib2
google-auth-oauthlib
```

# Cloud Run functionsに関数をデプロイ

- 事前に作成しておいたPub/Subのトピックをトリガーに設定した上で、関数をデプロイします。

```sh
export SERVICE_ID="patch-cloud-sql-function"
gcloud functions deploy "${SERVICE_ID}" \
  --gen2 \
  --region=us-central1 \
  --source=./patch-cloud-sql \
  --entry-point=patched_cloud_sql \
  --runtime=python311 \
  --ingress-settings=all \
  --no-allow-unauthenticated \
  --trigger-topic=${TOPIC_ID} \
  --service-account="${RUN_SA}@${PROJECT_ID}.iam.gserviceaccount.com"
```

# 自動生成されたEventArcに紐づくサービスアカウントに権限を付与

- `--trigger-topic`を指定して関数をデプロイした場合、自動的にEventArcが作られます。
- EventArcが関数を実行できるようにするために、実行するサービスアカウントに権限を付与しましょう。

```sh
export PROJECT_NUMBER="$(gcloud projects describe ${PROJECT_ID} --format='value(projectNumber)')"

gcloud run services add-iam-policy-binding "${SERVICE_ID}" \
  --region=us-central1 \
  --member="serviceAccount:${RUN_SA}@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role="roles/run.invoker"
```


# 動作検証

- Cloud SQLインスタンスが起動していることを確認した上で、以下のコマンドを実行し停止されるかどうかを確認しましょう。

```sh
gcloud scheduler jobs run ${SCHEDULE_ID} \
  --location=us-central1 \
  --project=${PROJECT_ID}
```

# おわりに

- ここまで読んでくださりありがとうございました。
- 実装してみればそこまで難しいことはしていないのですが、Pub/SubやCloud Run functionsなど初めて触るサービスもあり、とてもいい勉強になりました。
- 今回はCloud SQLのインスタンスを停止する、というお題で実装しましたが、スクリプトの内容を変えれば色んなことができそうですね！
- この記事の内容が少しでもお役に立てば幸いです。
