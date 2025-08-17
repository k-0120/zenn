---
title: "【BigQuery】一定時間実行されているジョブを強制キャンセルする"
emoji: "⏰️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["BigQuery"]
published: true
---

# これなに
- BigQueryにクエリを発行する上で、意図せず多対多のクエリを発行してしまったり、パーティションを効かせずに全範囲クエリしてしまったりすると、長時間ジョブが走り続けることによる無駄なコストの発生や、過剰にスロットを消費して他のクエリ実行に影響を与えてしまうことがあります。
- BigQueryコンソール上であればキャンセルボタンがありますが、BIツール等だとキャンセルできなかったり、そもそも料金形態のことを知らないため、キャンセルの必要性を把握せずに利用していることもあるかもしれません。
- 今回はBigQueryの`スケジュールされたクエリ`を用いて、実行されてから一定時間経過しているジョブを強制キャンセルする仕組みについて紹介してみようと思います。

# 結論

- 以下のクエリを`スケジュールされたクエリ`で5分間隔で実行することで、実行開始から10分以上経過したジョブを強制キャンセルさせることができます。
- 実行間隔やキャンセルのしきい値は、要件に合わせて適宜変更可能です。

```sql
CREATE TEMP TABLE cancelled_job_list AS ( 
  SELECT 
    project_id || "." || job_id AS full_job_id
  FROM 
    `project-id`.`region-asia-northeast1`.INFORMATION_SCHEMA.JOBS 
  WHERE 
    state = 'RUNNING' 
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 MINUTE) 
    AND DATETIME_DIFF(CURRENT_DATETIME('Asia/Tokyo'), DATETIME(creation_time, 'Asia/Tokyo'), MINUTE) >= 10 
    AND statement_type = 'SELECT' 
); 

FOR job IN (
  SELECT 
    full_job_id 
  FROM 
    cancelled_job_list
  ) 
  DO CALL BQ.JOBS.CANCEL(job.full_job_id); 
END FOR; 
```


# 解説

```sql
state = 'RUNNING'
AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 MINUTE)
```

- 実行されたジョブは[INFORMATION_SCHEMA.JOBS](https://cloud.google.com/bigquery/docs/information-schema-jobs?hl=ja)で抽出することができ、`state`が"RUNNING"のレコードに絞ることで実行中のジョブのみを取得することができます。
- また、INFORMATION_SCHEMA.JOBSは`creation_time`でパーティション分割されているため、直近30分に発行されたジョブに絞ることで抽出にかかるコストを抑制しています。

```sql
AND DATETIME_DIFF(CURRENT_DATETIME('Asia/Tokyo'), DATETIME(creation_time, 'Asia/Tokyo'), MINUTE) >= 10
```

- この絞り込み条件で、実行開始からどれぐらい経過したジョブをキャンセル対象にするかを制御しています。

```sql
AND statement_type = 'SELECT' 
```

- バッチ処理等でCREATE文やUPDATE文等が長時間実行されることを考慮して、SELECT文のみを対象としています。
- また、例えばBIツールからサービスアカウント経由で実行しているクエリのみを監視対象としたい場合は、`user_email = 'bi-sa@project-id.iam.gserviceaccount.com'`の条件を加えることで更に絞り込むこともできます。


```sql
SELECT
  project_id || "." || job_id AS full_job_id
```

- キャンセルするにはプロジェクトID付きのジョブIDが必要となるため、`project_id`と`job_id`を文字列結合しています。

```sql
FOR job IN (
  SELECT 
    full_job_id 
  FROM 
    cancelled_job_list
  ) 
  DO CALL BQ.JOBS.CANCEL(job.full_job_id); 
END FOR; 
```

- この仕組みの肝になる部分です。
- 手続き型言語のループ関数の1つである[FOR..IN](https://cloud.google.com/bigquery/docs/reference/standard-sql/procedural-language#for-in)を用いることで、キャンセル対象のジョブのレコードを1つ抽出し、`CALL BQ.JOBS.CANCEL(job.full_job_id);`を実行しています。
- `CALL`も手続き型言語の1つで、ジョブをキャンセルする[システムプロシージャ](https://cloud.google.com/bigquery/docs/managing-jobs?hl=ja#cancel_jobs)を呼び出すことで、対象のジョブをキャンセルしています。

# 拡張

- 「いつ」「誰が」「どんなクエリを」発行したかを特定し再発防止に繋げるためにも、キャンセルしたクエリを蓄積してみましょう。
- と言ってもそんなに難しいことはなく、事前に`cancelled_jobs`のようなテーブルを作っておき、キャンセル処理を実行した後、INSERT文を実行するだけです。

```sql
CREATE TEMP TABLE cancelled_job_list AS ( 
  SELECT 
    project_id || "." || job_id AS full_job_id, 
    query, 
    user_email, 
    DATETIME(creation_time, 'Asia/Tokyo') AS creation_time 
  FROM 
    `project-id`.`region-asia-northeast1`.INFORMATION_SCHEMA.JOBS 
  WHERE 
    state = 'RUNNING' 
    AND creation_time >= TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 30 MINUTE) 
    AND DATETIME_DIFF(CURRENT_DATETIME('Asia/Tokyo'), DATETIME(creation_time, 'Asia/Tokyo'), MINUTE) >= 10 
    AND statement_type = 'SELECT' 
); 

FOR job IN (
  SELECT 
    full_job_id 
  FROM 
    cancelled_job_list
  ) 
  DO CALL BQ.JOBS.CANCEL(job.full_job_id); 
END FOR; 

INSERT INTO 
  `project-id.bigquery_job_monitoring.cancelled_jobs`
SELECT 
  full_job_id, 
  query, 
  user_email, 
  creation_time 
FROM 
  cancelled_job_list;
```
