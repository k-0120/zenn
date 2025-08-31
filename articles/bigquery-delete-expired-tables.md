---
title: "【BigQuery】不要になったテーブルを検知して是正する運用について考えてみた"
emoji: "🐡"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: []
published: false
---

- 最終更新から1週間以上経過しているテーブルをリストアップ。
- 削除判断を行う。
  - クエリ状況が一緒に確認できるとベスト。
  - 最悪、確認用のクエリを自動発行するでもOK。
- マスタデータなど、定期更新されないが削除する必要のないものを管理できる方法も必要。
- ビュー、マテビューは検知対象外とする。
  - 参照元のテーブルが検知で削除されたら、自然とエラーになるため。
- 最終更新日時のカラムは、INFORMATION_SCHEMAを参照して取得する

```sql
SELECT
  table_catalog AS project_id,
  table_schema AS dataset_id,
  table_name AS table_id,
  last_modified_time
FROM
  `hey-data-pf-production.source_ec.INFORMATION_SCHEMA.PARTITIONS`
```

- テーブルのマスタデータを管理するテーブルを用意する。
  - 都度PARTITIONSから取得してもいいが、削除対象から外したい場合のフラグの管理を一緒にしたいため。

```sql
CREATE TABLE tmp_koumura.table_master (
  project_id STRING,
  dataset_id STRING,
  table_id STRING,
  last_modified_time TIMESTAMP,
  ecxeption_flg BOOLEAN DEFAULT FALSE
)
PARTITION BY TIMESTAMP_TRUNC(last_modified_time, DAY);
```

- table_masterにUPSERTするクエリが必要。


```sql
FOR record IN (
  SELECT 
    schema_name
  FROM 
    region-asia-northeast1.INFORMATION_SCHEMA.SCHEMATA
  WHERE
    schema_name = 'source_crm'
  ) 
  DO EXECUTE IMMEDIATE FORMAT("""
    MERGE tmp_koumura.table_master AS table_master
    USING(
      SELECT
        table_catalog AS project_id,
        table_schema AS dataset_id,
        table_name AS table_id,
        last_modified_time
      FROM
        `hey-data-pf-production.%s.INFORMATION_SCHEMA.PARTITIONS`
    ) AS partitions
    ON table_master.project_id = partitions.project_id
    AND table_master.dataset_id = partitions.dataset_id
    AND table_master.table_id = partitions.table_id
    WHEN MATCHED THEN
      UPDATE SET
        last_modified_time = partitions.last_modified_time
    WHEN NOT MATCHED THEN
      INSERT (project_id, dataset_id, table_id, last_modified_time, ecxeption_flg)
      VALUES (partitions.project_id, partitions.dataset_id, partitions.table_id, partitions.last_modified_time, FALSE)
  """, record.schema_name);
END FOR; 
```

- exception_flgをTRUEに変えるスクリプトも必要。

- exception_flgがFALSE、かつlast_modified_timeが1週間以上経過しているテーブルをどう扱うか。
- データセット単位でIssueを切る
- LookerStudioで可視化するのも良さそう。
- 「運用に載せる」の視点だとどう？
  - 可視化するだけでは弱い。
  - Issueの作成まで必要。
    - やることが明記されていること。
    - ジョブの履歴の確認。
    - アーカイブクエリの発行。
- テーブルオーナー(削除するかどうかの意思決定を下す人？)を明記しておけるとよい。
  - これは結構重たいので、今後のアップデートとして。
- Issue作成までをゴールとしよう。
