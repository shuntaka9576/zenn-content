---
title: "GitHub Projectsへ簡単にissueを作成、追加するGitHub CLI拡張を作った"
emoji: "✨"
type: "tech"
topics: ["go", "githubcli", "githubprojects"]
published: true
---

:::message
本記事のGitHub Projectsは、リニューアルされたProjectV2を指します。GitHub Projects(classic)は対象外となります。
:::

# はじめに

業務やプライベートで、少し特殊な方法でGitHub Projectsを運用しており、GitHub Projectsに直接issueを追加したいと思うことが多々ありました。

詳細は割愛しますが、運用を簡単に説明しますと、プランニングでGoogle Sheetsを用いてタスクや見積もりを整理/調整し、issueを作成、GitHub Projectsへ追加を行なっていました。これらの作業を簡単にするために、GitHub CLI拡張を作成しました。

# GitHub CLIの拡張について

詳細は[公式ドキュメント](https://docs.github.com/ja/github-cli/github-cli/using-github-cli-extensions)を参考にしてください。GitHub CLIには任意のユーザーが作成した拡張機能を導入することが可能です。

```bash
github extensions install {オーナー名}/{リポジトリ名}
```

拡張はGitHub CLIラッパーで、デフォルトでは足りない権限もGitHub CLIの設定で適宜scopeを追加することが可能です。
メリットとして、拡張導入側はツール毎に設定ファイルを管理する必要がなく、拡張作成側は、トークン設定周りの実装をGitHub CLI側に任せることが可能です。


# 動作イメージ

動作概要は以下の画像の通りです。issue及びdraft issue両方とも作成可能で、ProjectV2で導入されたカスタムフィールドも設定可能です。(※ iterationは未対応)

![img](https://github.com/shuntaka9576/gh-p2/blob/main/doc/gif/p2.gif?raw=true)

# 利用方法

ツールは以下に公開しております。

https://github.com/shuntaka9576/gh-p2

## インストール方法

### GitHub CLIの導入

GitHub CLIを導入してない方は、[公式ドキュメント](https://cli.github.com/manual/installation)を参考に導入してください。

### projectsのscopeの追加

GitHub CLIのデフォルトのscopeだと権限が足りないため、権限を追加します。

```bash
gh auth login --scopes 'project'
```

### gh-p2の導入

拡張を導入します。

```bash
gh extension install shuntaka9576/gh-p2
```

※ 初回実行時は、バイナリインストールため遅い場合はがあります。

## 使い方

### 必須オプション解説

詳細は[README](https://github.com/shuntaka9576/gh-p2/blob/main/README.md#usage)を参考にしてください。本記事ではQuick start的な内容を書いていきます。

利用するGitHub Projectがuserに紐づくかorganizationに紐づくかを確認して頂き、どちらかを指定します。
```bash
# userに紐づく場合
gh p2 create -u "<user名>"

# organizationに紐づく場合
gh p2 create -o "<organization名>"
```

今回はuserに紐づくとし、ユーザー名は`shuntaka9576`とします。


次に作成するissue or draft issueタイトルを指定します。`Fix bug`とします。
```bash
gh p2 create -u "shuntaka9576" \
 -t "Fix bug"
```

次にプロジェクトタイトルを指定します。`testProject`とします。

```bash
gh p2 create -u "shuntaka9576" \
 -t "Fix bug" \
 -p "testProject"
```

存在しないプロジェクトタイトルだった場合、以下のように利用可能なプロジェクトタイトル一覧が出力されるようにしています。タイトル名を拾うのが面倒な場合は、適当な文字列を指定するのもありです。

```bash
$ gh p2 create -u "shuntaka9576"
 -t "Fix bug" \
 -p "testProject"

Not found project name: testProject
The following project names are available and can be specified in the project-title(-p) flag.
  * test
```

実際に存在するプロジェクトタイトルがtestということがわかったので、プロジェクトタイトルに`test`を指定します。

```bash
gh p2 create -u "shuntaka9576" \
 -t "Fix bug" \
 -p "test"
```

ここから先は、issueを作成するかdraft issueを作成するかで指定するオプションが異なります。

### issueを作成する

issueを作成したいリポジトリを`-r`オプションで指定します。

```bash
gh p2 create -u "shuntaka9576" \
 -t "Fix bug" \
 -p "test" \
 -r "repoName"
```

これで一先ず、repoNameリポジトリへissue発行しつつProjectへ登録が可能です。

`label` `Projectのカスタムフィールド` `assignees` を設定する場合は以下の通りです。labelは、存在しない場合にランダムな色で新規作成します。カスタムフィールドはiteration以外は対応しています。date型のカスタムフィールドを設定する場合は`ISO 8601`形式で指定するようにしてください。

```bash
gh p2 create -u "shuntaka9576" \
  -t "Fix bug" \
  -p "test" \
  -r "repoName" \
  -a "shuntaka9576" \
  -l "label1" \
  -l "label2" \
  -f "Status:Todo" \
  -f "Point:1" \
  -f "deadline:2022-08-11"
```

指定可能なオプションの詳細は、helpをご確認ください。

```bash
gh p2 create --help
Usage: p2 create --project-title=STRING --title=STRING

Option to create an issue or draft issue directly in Project V2.

Flags:
  -h, --help                       Show context-sensitive help.
  -o, --org=STRING                 Specify an organization name.
  -u, --user=STRING                Specify a user name.
  -v, --version                    print the version.

  -p, --project-title=STRING       Specify the title of ProjectV2.
  -t, --title=STRING               Specify issue title.
  -r, --repo=STRING                Specify the repository name. Owner name is not required. This flag is not available when creating draft issues.
  -f, --fields=FIELDS,...          Specify ProjectV2 custom fields in the format {keyName}:{valueName}. e.g. Status:Todo, Point:3, date:2022-08-29. See https://docs.github.com/ja/graphql/reference/input-objects#projectv2fieldvalue for data types. Iteration is not currently supported.
  -l, --labels=LABELS,...          Specify the label name to be set for the issue. If a label with the target name does not exist, a new one will be created with a random color. This flag is not available when creating draft issues.
  -d, --draft                      Due to GitHub specifications, the --label and --repo options cannot be used together.
  -a, --assignees=ASSIGNEES,...    Specify the GitHub account ID to be assigned.
```


## draft issueを作成する

draft issueは、GitHubの仕様上labelを指定することが不可能です。また所属するのはProjectであるため、リポジトリの指定も不可能です。最後に`-d`オプションをつければ作成できます。需要はなさそうです。

```bash
gh p2 create -u "shuntaka9576" \
  -t "Fix bug" \
  -p "test" \
  -a "shuntaka9576" \
  -f "Status:In Progress" \
  -f "Point:4" \
  -f "deadline:2022-08-14" \
  -d
```

# 利用技術 
Go言語で書きました。CLIライブラリは、[alecthomas/kong](https://github.com/alecthomas/kong)を利用しています。他にも気になるところがあればリポジトリの中身を参照してください。

https://github.com/shuntaka9576/gh-p2

yusukebe様の`gh-markdown-preview`のshell実装や配布周りをかなり参考にさせて頂きました。。ありがとうございます。。

https://github.com/yusukebe/gh-markdown-preview


ProjectV2のAPIがあることは知らなかったのですが、以下の記事で存在を知りました。ありがとうございます、、

https://zenn.dev/hsaki/articles/github-graphql

Projects(classic)は[TUI](https://github.com/shuntaka9576/kanban)を作ったことがあるのですが、ProjectV2はカスタムフィールドのStausがカンバンの列に該当するため、かなり取り回しが楽でした。

# 今後

* issueのbodyを設定可能にする
* カスタムフィールドのiterationタイプ対応
* GitHub API呼び出し周りの最適化(少しissue作成が遅い)

# 最後に

ニッチなツールですが、同じような問題がある方がいれば使ってみてください。
