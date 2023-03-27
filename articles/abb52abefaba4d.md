---
title: "ChatGPTをターミナルから使うコマンドラインツール(CLI)を作った"
emoji: "💥"
type: "tech"
topics: ["chatgpt", "openai", "openaiapi", "go"]
published: true
---

## はじめに

ChatGPT流行っていますね。私はGPT-4が使えるようになってから触るようになりました。触っている過程でよりストレスなく使うにはターミナルから使えた方が良いと感じコマンドラインツールを作ることにしました。

Web版のChatGPT(https://chat.openai.com)は便利ですが、以下の点でもやもやがありました

* ブラウザを開く必要がある
* New Chatしすぎると整理が難しい
* チャットの履歴を横断的に見ることが難しい(あいまい検索など)
* ブラウザのエディタが使いにくい
* 障害で2日程度チャットの履歴が見れない期間が存在した
* API以外(ChatGPTとか)からの入力は学習用途に利用される^[[OpenAI の利用規約改定！！2023年3月1日更新 / API 経由は学習・訓練には使用されない。](https://note.com/tyaperujp01/n/nf35eef98039f)]
* 使い方によっては、ChatGPT Plusのサブスクより、従量の方が安い場合がありえる

ただこれらの点は一部今後改善されると思いますので、要は**趣味**です。

## リポジトリ

リポジトリは、[github.com/shuntaka9576/oax](https://github.com/shuntaka9576/oax)です。ツール名は、`oax`(OpenAI eXecutor)です。以下の画像が名付け親です。

![image](https://user-images.githubusercontent.com/12817245/226498673-90482fea-6ddc-4057-8ebe-7191f171cc4d.png)

## 必要なもの

* OpenAI APIのAPIキー
* Vim/Neovimのような同期的に実行可能なエディタ(試していませんが多分Nanoも使えると思います)



## 導入方法

以下のコマンドで導入可能です。

```bash
brew tap shuntaka9576/tap
brew install shuntaka9576/tap/oax
```

導入後APIキーと利用エディタの設定をしますが、本記事では割愛します。README.mdの[QuickStart](https://github.com/shuntaka9576/oax/tree/main#quick-start)を参照ください。

## サポート機能

* chat.openai.comのような対話機能(oax chat)
* プロファイル機能(複数のアカウントAPIキーがあっても、指定してリクエスト可能。`--profile`オプションで指定可能)
  * Organization IDを指定したリクエストも可

詳細は[README.md](https://github.com/shuntaka9576/oax#readme)を参照してください。

## 使用感

### 新規でチャットをする場合

チャット履歴は`~/.config/oax/chat-log`に保存されます。チャット履歴のファイルはTOMLファイルです。
![gif](https://res.cloudinary.com/dkerzyk09/image/upload/v1679860503/tools/oax/oax-chat.gif)

チャット履歴ファイルは以下のような形式で、role="user"が質問者でrole="assistant"がChatGPTからの回答です。messages配列が全てOpenAIのAPIに送信されるので、コンテキストに基づいた対話が可能です。

```toml:チャット履歴ファイル
[[messages]]
  sequentialNumber = 0 # 順番
  role = "user"
  content = '''
しりとりしよう
'''

[[messages]]
  sequentialNumber = 1
  role = "assistant"
  content = '''
りんご
'''

[[messages]]
  sequentialNumber = 2
  role = "user"
  content = '''
ゴリら
'''

[[messages]]
  sequentialNumber = 3
  role = "assistant"
  content = '''
らくだ
'''

[[messages]]
  sequentialNumber = 4
  role = "user"
  content = '''
だんご
'''

[[messages]]
  sequentialNumber = 5
  role = "assistant"
  content = '''
ごま油
'''

```

チャット履歴を保存するパスを変更することも可能です。例えば`ghq`を使っている場合は、`~/repos/github.com/{OWNER名}/chat-log`を指定することでGit管理下にチャット履歴を管理できます。変更方法は以下の通りです。

```bash:設定ファイルを開く
oax config --settings
```

```toml:履歴を保存するパスを変更
[setting]
  editor = "nvim"
  chatLogDir = "~/repos/github.com/shuntaka9576/chat-log" # <-- 追加
```

詳細は[こちら](https://github.com/shuntaka9576/oax#setting)を参照ください。

`-m`オプションでモデルの指定が可能です。デフォルトは`gpt-3.5-turbo`です。

```bash
$ oax chat --help
Usage: oax chat

Provides a dialogue function like chat.openai.com.

Flags:
  -h, --help                     Show context-sensitive help.
  -p, --profile=STRING           Specify the profile.
  -v, --version                  Print the version.

  -m, --model="gpt-3.5-turbo"    Specify the ID of the model to use(gpt-4, gpt-4-0314, gpt-4-32k, gpt-4-32k-0314, gpt-3.5-turbo,
                                 gpt-3.5-turbo-0301).
  -f, --file=FILE                Specify the chat history file with the full path.
```


### 途中からチャットを再開する場合

`-f`オプションでチャット履歴ファイルをフルパスで指定します。

![gif](https://res.cloudinary.com/dkerzyk09/image/upload/v1679860850/tools/oax/chat-resume.gif)


## 技術

### Go

* シングルバイナリで配布しやすい
* チャンネルを使って簡単にマルチスレッドが可能
* goreleaserのバイナリリリース配布機能が強力(クロスビルド、brew tap連携など)

今回ですと、後述するSSEのイベントをサブスクライブし、イベント取得したらチャンネルに送信して差分更新をし、Web版ChatGPTのような表示を実現しています。

https://github.com/shuntaka9576/oax/blob/1f254be80695c9a82a53c489d8c657c9ab7ff7c8/openai/chat.go#L45-L81

`io.MultiWriter`でbytes.Bufferと標準出力に書き込み、回答が返却されたタイミングでtomlファイルに出力しています。

https://github.com/shuntaka9576/oax/blob/main/cli/chat.go#L105-L194

Go経験が乏しく結構辛みのあるコードになっていますが、今後リファクタリングしていく所存です。

### TOML

設定ファイル2種類とチャット履歴のファイルにTOMLを採用しています。これは`'''`で括ると改行にエスケープシーケンスを利用しなくて済むので可読性が上がり、採用しました。
ただTOMLにChatGPTの返信内容をデシリアライズして書き込むと`'''`表記にはならないので、ループで文字列のを繋げて出力しています。

https://github.com/shuntaka9576/oax/blob/1f254be80695c9a82a53c489d8c657c9ab7ff7c8/chat.go#L104-L110

### OpenAI API

OpenAIのAPIの[/v1/chat/completions](https://platform.openai.com/docs/api-reference/chat)は、[SSE(Sever-sent events)](https://developer.mozilla.org/en-US/docs/Web/API/Server-sent_events/Using_server-sent_events#event_stream_format)に対応しています。

GoでSSEのイベント取り回すライブラリでPOSTのSSEに対応しておらず、少しハマりました。詳しくは[ChatGPTのストリーミング(SSE)APIを試してみた(Go実装)](https://dev.classmethod.jp/articles/shuntaka-open-ai-api-sse/)を参照ください。[Feature Request: Support non-GET method and request body in the sse client](https://github.com/r3labs/sse/issues/153)でForkされたsseライブラリをgit submoduleで参照しています。

go.modでreplaceディレクティブを利用することで、インポートパスを自分のGitHubのOwner名に書き換えないですみます。

https://github.com/shuntaka9576/oax/blob/1f254be80695c9a82a53c489d8c657c9ab7ff7c8/go.mod#L16

git submoduleを使う場合は、`actions/checkout@v3`のsubmodulesオプションをtrueにする必要があります。
https://github.com/shuntaka9576/oax/blob/1f254be80695c9a82a53c489d8c657c9ab7ff7c8/.github/workflows/release.yml#L12-L16
### [bubbletea](https://github.com/charmbracelet/bubbletea)の採用検討と見送り

[bubbles](https://github.com/charmbracelet/bubbles)のtextareaとviewportを使って、[bubblesのサンプル](https://github.com/charmbracelet/bubbletea/blob/master/examples/chat/chat.gif)のようなUIを作る予定でしたが、実力不足で見送りになりました。。

* viewport
  * 1行の横幅が長い場合に改行されない
  * 自動スクロールを別途実装する必要がある
* 標準出力をbubbletea側で握られており、今回の要件では逆に普通の標準出力できることをやろうとすると別途実装が必要だった

より研究して使いこなせるようにしていきたいです。

## 今後

個人的なユースケースとしては現状で満足していますが、以下の内容を今後対応したいと思います。

* モデル指定のデフォルト設定を設定ファイルに外だしする
* chat.openai.comのような会話履歴内容からタイトル要約し会話履歴ファイル名として保存する
* Windows対応(やりかけになっている、、)
* 他のOpen AIのAPIで便利そうな機能の導入

多分公式からもっとすごいやつが出る気がしますが、薄く簡単に使えるツールとして差別化して改善しようと思います、、



