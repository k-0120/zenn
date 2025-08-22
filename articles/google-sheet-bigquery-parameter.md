---
title: "【スプシ】コネクテッドシートでセルの値をクエリに渡して動的に絞り込む"
emoji: "📄"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["Google Sheets", "BigQuery"]
published: true
---

# これなに
- コネクテッドシートは、GoogleスプレッドシートからBigQueryに対してクエリを実行し、データを出力できる機能です。
- 今回はスプレッドシート上のセルの値をクエリに渡し、動的にデータを絞り込む方法について紹介してみようと思います。

# 具体例1

- BigQueryに以下のような`users`テーブルがあるとします。

![usersテーブル](/images/google-sheet-bigquery-parameter/users.png)

- スプシ上のB1セルにユーザーIDが入力されているので、その値をクエリに渡して対象ユーザーの情報をスプシに出力してみましょう。

![spreadsheet1](/images/google-sheet-bigquery-parameter/spreadsheet1.png)

- データ>データコネクタ>BigQueryに接続を選択し、GoogleCloudプロジェクトを選択した上で、保存したクエリとクエリエディタを選択します。

![spreadsheet2](/images/google-sheet-bigquery-parameter/spreadsheet2.png)

- クエリエディタが表示されたら、パラメータから追加を選択し、`USER_ID`という名前でユーザーIDが入力されているB1セルを選択してパラメータを追加してみましょう。
- するとクエリエディタ上に`@USER_ID`が出力されます。
- 以下のように`@USER_ID`を含めた上でクエリを実行すると、`@USER_ID`にB1セルの値が代入され実行される仕組みになっています。

```sql
SELECT
  *
FROM
  sample.users
WHERE
  id = @USER_ID
```

![spreadsheet3](/images/google-sheet-bigquery-parameter/spreadsheet3.png)

# 複数セルを条件に含める方法

- コネクテッドシートのパラメータは特定の1セルしか参照できないため、TEXTJOIN()関数を使って1つの文字列にした上で、パラメータとして渡していきます。

# 具体例2
- 以下のように、A2〜A4セルにメールアドレスが入力されているとします。
- パラメータで参照できるようにするために、B1セルに`=TEXTJOIN(",", TRUE, A2:A4)`を入力し、カンマ区切りで1つの文字列にしてみましょう。

![spreadsheet4](/images/google-sheet-bigquery-parameter/spreadsheet4.png)

- あとは先程と同じように、コネクテッドシートでパラメータを追加してクエリするだけです。
- この時、複数のメールアドレスが1つの文字列になっているため、SPLIT()で個々のメールアドレスに分解し、それを配列に入れた上で再度UNNEST()するのがポイントです。

```sql
SELECT
  *
FROM
  sample.users
WHERE
  email IN UNNEST(
    ARRAY(
      SELECT 
        TRIM(email, "'")
      FROM 
        UNNEST(SPLIT(@EMAIL, ',')) email
    )
  )
ORDER BY 
  id
```

![spreadsheet5](/images/google-sheet-bigquery-parameter/spreadsheet5.png)
