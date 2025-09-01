---
title: "【BigQuery】古いテーブルを検出→GitHubにIssueを作成までを自動化してみる"
emoji: "🤖"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["BigQuery", "GitHub", "CloudScheduler", "Workflows"]
published: true
---

# これなに
- BigQuery上に、更新されなくなったテーブルが残ってたりしませんか？
- dbt等の便利ツールを使っていると、モデルは削除したがテーブルは削除されてなかった、なんてこともあったり。
- 単純にリストアップするだけであれば簡単なのですが、それを定期運用に乗せないと結局また同じことを繰り返してしまうことも。
- 今回はそんな課題を解決するために、最終更新日時が1週間以上前になっているテーブルを定期的にリストアップし、Issueを作成するところまで自動化してみようと思います。

# 本記事で得られるもの

- 不要テーブルを自動検知し、GitHubにIssueを作成する一連のフローを構築できる
- Cloud SchedulerとWorkflowsを組み合わせた定期実行パターンを理解できる
- BigQueryのINFORMATION_SCHEMAの活用例を学べる

# 全体像

- BigQuery前提のため、同じくGoogle Cloud製品であるCloud SchedulerとWorkflowsを使うことにしました。
- 最終的にIssueを作成するため、GitHub Actionsで定期実行することも検討したのですが、サービスアカウントにテーブルを操作できる権限を渡す必要があり、危険な気がしたため採用を見送りました。

![configuration_diagram](/images/bigquery-delete-expired-tables/configuration_diagram.png)

# テーブルごとの最終更新日時を管理するテーブルを作成する

- まずは最終更新日時を取得してみましょう。
- 最終更新日時のカラムは、[INFORMATION_SCHEMA.PARTITIONSビュー](https://cloud.google.com/bigquery/docs/information-schema-partitions?hl=ja)の`last_modified_time`カラムをクエリすることで取得することができます。
- 例えば、プロジェクト名、データセット名、テーブル名、最終更新日時を抽出クエリはこんな感じ。

```sql
SELECT
  table_catalog AS project_id,
  table_schema AS dataset_id,
  table_name AS table_id,
  last_modified_time
FROM
  `project-id.dataset-id.INFORMATION_SCHEMA.PARTITIONS`
```

- データセット単位のビューになっているため、後続で処理しやすくするために蓄積用のテーブルを用意しておきましょう。
- テーブルによっては、一度作成してその後更新しないテーブルも存在しうるため、判定対象から除外する用のカラム(下記だと`exception_flg`)も用意しておくと便利かと思います。

```sql
CREATE TABLE utility.tables_last_modified (
  project_id STRING,
  dataset_id STRING,
  table_id STRING,
  last_modified_time TIMESTAMP,
  exception_flg BOOLEAN DEFAULT FALSE
)
PARTITION BY TIMESTAMP_TRUNC(last_modified_time, DAY);
```

- さて、取得用のクエリと蓄積用のテーブルができたため、実際にデータを蓄積するためのクエリを作っていきましょう。
- `INFORMATION_SCHEMA.PARTITIONS`はデータセット単位でクエリする必要があります。
- データセットの一覧は、[INFORMATION_SCHEMA.SCHEMATAビュー](https://cloud.google.com/bigquery/docs/information-schema-datasets-schemata?hl=ja)をクエリすることで取得することができます。
- FOR...INでデータセットごとにループ処理し、その中でMERGE文を実行することで`tables_last_modified`テーブルを更新していきましょう。

```sql
FOR record IN (
  SELECT 
    catalog_name AS project_id,
    schema_name AS dataset_id
  FROM 
    region-asia-northeast1.INFORMATION_SCHEMA.SCHEMATA
  WHERE
    catalog_name = @@project_id
) 
DO EXECUTE IMMEDIATE FORMAT("""
  MERGE utility.tables_last_modified
  USING(
    SELECT
      table_catalog AS project_id,
      table_schema AS dataset_id,
      table_name AS table_id,
      last_modified_time
    FROM
      `%s.%s.INFORMATION_SCHEMA.PARTITIONS`
  ) AS partitions
  ON tables_last_modified.project_id = partitions.project_id
  AND tables_last_modified.dataset_id = partitions.dataset_id
  AND tables_last_modified.table_id = partitions.table_id
  WHEN MATCHED THEN
    UPDATE SET
      last_modified_time = partitions.last_modified_time
  WHEN NOT MATCHED THEN
    INSERT (project_id, dataset_id, table_id, last_modified_time, exception_flg)
    VALUES (partitions.project_id, partitions.dataset_id, partitions.table_id, partitions.last_modified_time, FALSE)
""", record.project_id, record.dataset_id);
END FOR; 
```

- これで、テーブルごとの最終更新日時を管理するテーブルを作ることができました。
- ただし、このままではWorkflowsから扱いづらいため、上記処理をストアドプロシージャとして登録します。

```sql
CREATE OR REPLACE PROCEDURE `utility.sp_merge_table_master`()
BEGIN
  FOR record IN (
    SELECT 
      catalog_name AS project_id,
      schema_name AS dataset_id
    FROM 
      region-asia-northeast1.INFORMATION_SCHEMA.SCHEMATA
    WHERE
      catalog_name = @@project_id
    ) 
    DO EXECUTE IMMEDIATE FORMAT("""
      MERGE utility.tables_last_modified
      USING(
        SELECT
          table_catalog AS project_id,
          table_schema AS dataset_id,
          table_name AS table_id,
          last_modified_time
        FROM
          `%s.%s.INFORMATION_SCHEMA.PARTITIONS`
      ) AS partitions
      ON tables_last_modified.project_id = partitions.project_id
      AND tables_last_modified.dataset_id = partitions.dataset_id
      AND tables_last_modified.table_id = partitions.table_id
      WHEN MATCHED THEN
        UPDATE SET
          last_modified_time = partitions.last_modified_time
      WHEN NOT MATCHED THEN
        INSERT (project_id, dataset_id, table_id, last_modified_time, exception_flg)
        VALUES (partitions.project_id, partitions.dataset_id, partitions.table_id, partitions.last_modified_time, FALSE)
    """, record.project_id, record.dataset_id);
  END FOR; 
END;
```

- 併せて、今回は最終更新日時が1週間以上前のテーブルを検知したいため、該当テーブルを蓄積するテーブルを作るストアドプロシージャ、及びIssue本文に載せる文字列を加工するビューを作成しておきます。
- Workflows上で加工してもいいのですが、極力ワークフローのymlをシンプルにしたかったため、加工処理はクエリに寄せることにしました。

```sql
CREATE OR REPLACE PROCEDURE `utility.sp_build_stale_tables_weekly`()
BEGIN
  EXECUTE IMMEDIATE """
    CREATE OR REPLACE TABLE `utility.stale_tables_weekly` AS
    WITH stale_tables AS (
      SELECT
        project_id, 
        dataset_id, 
        table_id, 
        last_modified_time
      FROM 
        `utility.tables_last_modified`
      WHERE 
        exception_flg = FALSE
        AND last_modified_time < TIMESTAMP_SUB(CURRENT_TIMESTAMP(), INTERVAL 7 DAY)
    )
    SELECT
      dataset_id,
      COUNT(*) AS table_count,
      MIN(last_modified_time) AS oldest_last_modified_time,
      MAX(last_modified_time) AS newest_last_modified_time,
      ARRAY_AGG(
        STRUCT(
          table_id, 
          last_modified_time
        )
        ORDER BY last_modified_time
      ) AS tables
    FROM 
      stale_tables
    GROUP BY 
      dataset_id
  """;
END;
```

```sql
CREATE OR REPLACE VIEW `sample.v_stale_tables_issue_rows` AS
SELECT
  dataset_id,
  table_count,
  ARRAY_TO_STRING(
    ARRAY_AGG(
      CONCAT(
        "- ", table_id,
        " (last_modified_time: ",
        FORMAT_TIMESTAMP('%Y-%m-%d %H:%M:%SZ', last_modified_time, 'UTC'),
        ")"
      )
      ORDER BY 
        last_modified_time
    ),
    '\n'
  ) AS issue_body
FROM 
  `utility.stale_tables_weekly`,
  UNNEST(tables) AS t
WHERE 
  table_count > 0
GROUP BY 
  dataset_id, table_count;
```

# GitHubのアクセストークンを発行

- WorkflowsからGitHubのAPIを叩いてIssueを作成するために、アクセストークンを発行しましょう。([公式ドキュメント](https://docs.github.com/ja/authentication/keeping-your-account-and-data-secure/managing-your-personal-access-tokens))
- 今回はパーソナルアクセストークンを発行して実装を進めましたが、本格的な運用を考えるならGitHub Appsを作成して都度インストールトークンを発行して使うのが良さそうです。
- 作成したら、Workflowsから呼び出せるようにSecret Managerに登録しておきます。

# サービスアカウントの用意

- Cloud Scheduler用とWorkflows用に2つのサービスアカウントを作成しましょう。
- 最小権限の原則を意識しつつ、それぞれに割り当てる権限は次のとおりです。
  - Cloud Scheduler
    - ワークフロー起動元(`roles/workflows.invoker`)
  - Workflows
    - BigQueryジョブユーザー(`roles/bigquery.jobUser`)
    - BigQueryデータ編集者(`roles/bigquery.dataEditor`)
    - Secret Managerへのシークレットアクセサー(`roles/secretmanager.secretAccessor`)


# ワークフローの作成

- では、ワークフローの作成に移っていきましょう。
- いきなりですが、今回作成したワークフローのymlは以下の通りです。
- 実行時に、`{"project_id": "my-project", "location": "asia-northeast1"}`のようにパラメータを渡す前提で作っています。

```yml
main:
    params: [args]
    steps:
    # MERGE文を実行するストアドプロシージャを呼び出すステップ
    - call_merge:
        call: http.post
        args:
            url: ${"https://bigquery.googleapis.com/bigquery/v2/projects/" + args.project_id + "/queries"}
            auth:
                type: OAuth2
            headers:
                Content-Type: application/json
            body:
                useLegacySql: false
                location: ${args.location}
                query: ${"CALL `" + args.project_id + ".utility.sp_merge_table_master`();"}
        result: merge_res
    
    # 検知対象のテーブルの一覧を作成するストアドプロシージャを呼び出すステップ
    - call_build_weekly:
        call: http.post
        args:
            url: ${"https://bigquery.googleapis.com/bigquery/v2/projects/" + args.project_id + "/queries"}
            auth:
                type: OAuth2
            headers:
                Content-Type: application/json
            body:
                useLegacySql: false
                location: ${args.location}
                query: ${"CALL `" + args.project_id + ".utility.sp_build_stale_tables_weekly`();"}
        result: build_res

    # Issue作成用のデータを抽出するクエリを実行するステップ
    - query_weekly_rows:
        call: http.post
        args:
            url: ${"https://bigquery.googleapis.com/bigquery/v2/projects/" + args.project_id + "/queries"}
            auth:
                type: OAuth2
            headers:
                Content-Type: application/json
            body:
                useLegacySql: false
                location: ${args.location}
                query: ${"SELECT dataset_id, table_count, issue_body FROM `" + args.project_id + ".utility.v_stale_tables_issue_rows`"}
        result: weekly_res

    - init_rows:
        assign:
            - rows: ${default(weekly_res.body.rows, [])}
    
    # 検知対象が0件であれば終了    
    - check_empty:
        switch:
            - condition: ${len(rows) == 0}
              next: done

    # GitHubへアクセスするための準備
    - init:
        assign:
            - project_id: ${sys.get_env("GOOGLE_CLOUD_PROJECT_ID")}
            - gh_repo: "k-0120/data-platform-modeling"
            - gh_secret_name: "github-pat-issue-read-and-write"

    # GitHub PATをSecret Managerから取得するステップ
    - read_github_pat:
        call: googleapis.secretmanager.v1.projects.secrets.versions.access
        args:
            name: ${"projects/" + project_id + "/secrets/" + gh_secret_name + "/versions/latest"}
        result: gh_secret

    # PATを変数`gh_token`にセットするステップ
    - set_pat:
        assign:
            - gh_token: ${text.decode(base64.decode(gh_secret.payload.data))}

    # データセットごとにIssueを作成
    - loop_datasets:
        for:
            value: r
            in: ${rows}
            steps:
            - mapFields:
                assign:
                    - datasetId: ${r.f[0].v}
                    - tableCount: ${int(r.f[1].v)}
                    - tablesStr: ${r.f[2].v}
                    - title: ${"【BigQuery】dataset " + datasetId + " に削除漏れの可能性（週次チェック）"}
                    - body: ${"## 対象テーブル一覧（" + string(tableCount) + "件）\n" + tablesStr}

            - create_issue:
                call: http.post
                args:
                    url: ${"https://api.github.com/repos/" + gh_repo + "/issues"}
                    headers:
                        Authorization: ${"Bearer " + gh_token}
                        Accept: "application/vnd.github+json"
                    body:
                        title: ${title}
                        body: ${body}
                result: create_res

    - done:
        return: ${"done"}
```

# Cloud Schedulerで定期実行の設定

- 例えば毎週月曜日の午前9時に実行する場合は、頻度のところを`0 9 * * 1`と設定すればOKです。
- あとは、先程作ったWorkflowsとサービスアカウントを選択し、作成すれば完成です！
- 試しに強制実行し、ワークフローが起動するか、Issueが作られるかを確認してみましょう。

![issue_sample](/images/bigquery-delete-expired-tables/issue_sample.png)

# おわりに

- データ品質の観点でも、鮮度の古いテーブルを残しておくのは危険です。
- とは言え、中々優先度が上がらなかったり、つい放置してしまう気持ちもわかります。
- それでも重い腰をあげて対応していくために、こういった仕組みを通して少しでも楽に運用を回せると良いな、と思い書き起こしてみました。
- 読者にとって、少しでも役に立てば幸いです。