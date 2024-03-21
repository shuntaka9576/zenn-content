---
title: "GitHub Projectsに紐づくIssue一覧を取得する"
emoji: "✨"
type: "tech"
topics: ["go", "githubcli", "githubprojects"]
published: true
---


## はじめに

GitHub Projectsを使っている人は、Board(カンバン)形式をよく使うと思います。Boardのカラム情報(正確にはCustom filedsのStatus)を紐づけたIssue一覧をcsvで作成します。以下のようなcsvが出力できます。

```
"type","number","title","status","url"
"ISSUE",48,"title2","In Progress","https://github.com/shuntaka9576/kanban-test/issues/48"
"DRAFT_ISSUE",0,"title4","In Progress",""
"ISSUE",49,"Fix bug",,"https://github.com/shuntaka9576/kanban-test/issues/49"
"DRAFT_ISSUE",0,"aaa",,""
"DRAFT_ISSUE",0,"aaa",,""
"ISSUE",50,"aaa[]",,"https://github.com/shuntaka9576/kanban-test/issues/50"
"ISSUE",51,"aaa",,"https://github.com/shuntaka9576/kanban-test/issues/51"
"ISSUE",52,"aaa",,"https://github.com/shuntaka9576/kanban-test/issues/52"
"ISSUE",58,"test",,"https://github.com/shuntaka9576/kanban-test/issues/58"
"ISSUE",89,"Fix bug","Todo","https://github.com/shuntaka9576/kanban-test/issues/89"
"ISSUE",90,"Fix bug","Todo","https://github.com/shuntaka9576/kanban-test/issues/90"
"DRAFT_ISSUE",0,"20240320",,""
```


以下の点注意してください。

:::message
* GitHub Projects(classic)では利用できません
* GitHub CLI拡張を利用します
:::



## 利用するツール
GitHub GraphQL APIを叩くだけで済ませたかったのですが、以下の理由で断念しました。

* GitHub GraphQL APIレスポンスそのものをjqで整形するにはつらみがあった
* GitHub Projects内部のIssueを全て取得するのにループを書く必要があった

重複する関数も多く、以前作成したgh-p2を拡張することにしました。GitHub ProjectsにIssueを追加するのがメインでしたが、雑に取得する機能を追加しました。

https://zenn.dev/shuntaka/articles/edfd9ce2c0ee52
https://github.com/shuntaka9576/gh-p2


## 利用方法

### 導入方法

```bash
gh auth login --scopes 'project'
gh extension install shuntaka9576/gh-p2
```

### Issue一覧取得方法

userの場合`-u`、orgの場合`-o`にowner名を指定してください。`-p`にはプロジェクトタイトルを指定してください。

```bash
# userの場合
gh p2 ls -u "shuntaka9576" -p "test"
# orgの場合
# gh p2 ls -o "aws" -p "AWS Toolkit for JetBrains"
```

こちらが実行結果です。

```json:gh p2 ls結果
{
  "title": "test",
  "items": [
    {
      "title": "title2",
      "singleSelectValues": {
        "Status": "In Progress"
      },
      "type": "ISSUE",
      "body": "",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/48",
      "number": 48
    },
    {
      "title": "title4",
      "singleSelectValues": {
        "Status": "In Progress"
      },
      "type": "DRAFT_ISSUE",
      "body": "",
      "url": "",
      "number": 0
    },
    {
      "title": "Fix bug",
      "singleSelectValues": {},
      "type": "ISSUE",
      "body": "",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/49",
      "number": 49
    },
    {
      "title": "aaa",
      "singleSelectValues": {},
      "type": "DRAFT_ISSUE",
      "body": "",
      "url": "",
      "number": 0
    },
    {
      "title": "aaa",
      "singleSelectValues": {},
      "type": "DRAFT_ISSUE",
      "body": "",
      "url": "",
      "number": 0
    },
    {
      "title": "aaa[]",
      "singleSelectValues": {},
      "type": "ISSUE",
      "body": "",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/50",
      "number": 50
    },
    {
      "title": "aaa",
      "singleSelectValues": {},
      "type": "ISSUE",
      "body": "",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/51",
      "number": 51
    },
    {
      "title": "aaa",
      "singleSelectValues": {},
      "type": "ISSUE",
      "body": "",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/52",
      "number": 52
    },
    {
      "title": "test",
      "singleSelectValues": {},
      "type": "ISSUE",
      "body": "aaaa",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/58",
      "number": 58
    },
    {
      "title": "Fix bug",
      "singleSelectValues": {
        "Status": "Todo"
      },
      "type": "ISSUE",
      "body": "テスト<img src=\"\" />",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/89",
      "number": 89
    },
    {
      "title": "Fix bug",
      "singleSelectValues": {
        "Status": "Todo"
      },
      "type": "ISSUE",
      "body": "test<img src=\"\" />",
      "url": "https://github.com/shuntaka9576/kanban-test/issues/90",
      "number": 90
    },
    {
      "title": "20240320",
      "singleSelectValues": {},
      "type": "DRAFT_ISSUE",
      "body": "",
      "url": "",
      "number": 0
    }
  ]
}
```


### jqで整形しcsv出力

```bash
gh p2 ls -u "shuntaka9576" -p "test" |\
  jq -r '["type", "number", "title", "status", "url"], (.items[] |[.type, .number, .title, .singleFiledValues.Status, .url]) | @csv'
```

```csv:実行結果
"type","number","title","status","url"
"ISSUE",48,"title2","In Progress","https://github.com/shuntaka9576/kanban-test/issues/48"
"DRAFT_ISSUE",0,"title4","In Progress",""
"ISSUE",49,"Fix bug",,"https://github.com/shuntaka9576/kanban-test/issues/49"
"DRAFT_ISSUE",0,"aaa",,""
"DRAFT_ISSUE",0,"aaa",,""
"ISSUE",50,"aaa[]",,"https://github.com/shuntaka9576/kanban-test/issues/50"
"ISSUE",51,"aaa",,"https://github.com/shuntaka9576/kanban-test/issues/51"
"ISSUE",52,"aaa",,"https://github.com/shuntaka9576/kanban-test/issues/52"
"ISSUE",58,"test",,"https://github.com/shuntaka9576/kanban-test/issues/58"
"ISSUE",89,"Fix bug","Todo","https://github.com/shuntaka9576/kanban-test/issues/89"
"ISSUE",90,"Fix bug","Todo","https://github.com/shuntaka9576/kanban-test/issues/90"
"DRAFT_ISSUE",0,"20240320",,""
```


## 内部動作解説

### はじめに

GitHub ProjectsはCustom fieldsと呼ばれるIssueに紐づけられるフィールドを作成することができます。Boardのカラムは、基本的にCustom fieldsのStatus属性を参照しています。

### プロジェクト一覧の取得

```graphql:ユーザーに紐づくプロジェクトの場合
{
  user(login: "shuntaka9576") {
    projectsV2(first: 20) {
      nodes {
        id
        title
      }
    }
  }
}
```

```graphql:orgに紐づくプロジェクトの場合
{
  organization(login: "aws") {
    projectsV2(first: 20) {
      nodes {
        id
        title
      }
    }
  }
}
```

### プロジェクト単体の取得

[AWS Toolkit for JetBrains](https://github.com/orgs/aws/projects/18)の単体を取得します。itemsの部分は20件になっていますが、無くなるまで再起ループを回す必要があります。


```graphql
{
  node(id: "PVT_kwDOACIPmc4ABJ33") {
    ... on ProjectV2 {
      title
      items(first: 20) {
        pageInfo {
          hasNextPage
          endCursor
        }
        nodes {
          id
          createdAt
          updatedAt
          isArchived
          content {
            __typename
          }
          fieldValues(first: 20) {
            nodes {
              __typename
              ... on ProjectV2ItemFieldSingleSelectValue {
                name
                optionId
                field {
                  ... on ProjectV2SingleSelectField {
                    id
                    name
                  }
                }
              }
            }
          }
          content {
            ... on DraftIssue {
              title
              body
            }
            ... on Issue {
              url
              number
              title
              state
              url
              body
            }
          }
        }
      }
    }
  }
}
```

## さいごに

個人的にどのカラムに何があるかスプシにまとめられれば良かったで、細かいところの作り込みはしていません。Custom filedsはsingleSelectValuesしか対応してないので、要望があればお気軽にIssueください！
