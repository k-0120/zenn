---
title: "【git】mainブランチとの差分がない作業ブランチを全て削除する"
emoji: "🧹"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["git"]
published: true
---

# これなに
- ローカルリポジトリに作業ブランチが溜まってきたときに、1つ1つ削除するのは面倒くさいですよね。
- 複数の作業を並行していると、どれがMerge済みかわからなくもなったり。
- 今回はシェルスクリプト一発でMerge済みの作業ブランチを全て削除してみようと思います。

# 結論

```bash
# ベースにしたいブランチ名
BASE=main

# 念のため最新状態に更新し、リモートの消えたブランチを掃除
git fetch --all --prune

# ベースに切り替え（安全のため）
git switch "$BASE"

# ベースにマージ済みのローカルブランチをまとめて削除
git branch --merged "$BASE" \
  | grep -v '^\*' \
  | sed -E 's/^[[:space:]]+//' \
  | egrep -v "^($BASE|main|staging|experimental)$" \
  | xargs -n1 git branch -d
```

# 解説

```bash
git branch --merged "$BASE" \
  | grep -v '^\*' \
  | sed -E 's/^[[:space:]]+//' \
  | egrep -v "^($BASE|main|staging|experimental)$" \
  | xargs -n1 git branch -d
```

- 肝になるのはこの処理。
- `git branch --merged <branch_name>`は、指定したブランチにMerge済みのブランチの一覧を出します。
- カレントブランチの前には`*`がつくため、`grep -v '^\*'`で該当行を除外しています。
- `sed -E 's/^[[:space:]]+//'`で、先頭のスペースやタブを削除し、ブランチ名のみにしています。
- `egrep -v "^($BASE|main|staging|experimental)$"`で、削除したくないブランチを念の為除外しています。
- `xargs -n1 git branch -d`で、一行ずつ取り出し削除します。