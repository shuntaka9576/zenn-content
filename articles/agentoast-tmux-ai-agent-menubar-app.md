---
title: "tmuxとAIエージェント(CLI)を管理するメニューバーアプリを作った話"
emoji: "🍞"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["tmux", "claudecode", "opencode", "codex"]
published: false
---

## はじめに

Claude Code/Codex/opencode などのCLI型AIコーディングエージェント(以降コーディングエージェント)の普及に伴い、並列でエージェントを動かす開発スタイルがスタンダードになり表題のようなツールを作成しました。

動作の様子は以下で、リポジトリのフォルダアイコンの中に、コーディングエージェントが動作中のtmuxペインを閲覧、選択して遷移できます。ショートカットで全て飛べますのでマウス操作は必要ないようにしています。

![img](https://raw.githubusercontent.com/shuntaka9576/agentoast/refs/heads/main/docs/assets/main.gif)

リポジトリは以下です。

https://github.com/shuntaka9576/agentoast

自分はWezterm + tmux + lazygit + neovimでコードベースごとにtmuxセッションを立て、そこでエージェントを動かしています。エージェントの終了を見逃すケースがあったため、tmuxセッションとコーディングエージェントを管理するメニューバーアプリを作るきっかけになりました。

インストールはbrewが推奨です。CLIとmacOSアプリ(menubarアプリ)があり、両方必要です。インストール方法の詳細は[Installation](https://github.com/shuntaka9576/agentoast?tab=readme-ov-file#installation)に記載しています。簡単に説明すると以下です。

```bash
# CLI
brew install shuntaka9576/tap/agentoast-cli
# menubar アプリ
brew install --cask shuntaka9576/tap/agentoast

# 起動時GUIは出ず、メニューバーにトーストにお化けがいるアイコンが表示されます。右クリック > Quit で停止できます。
open /Applications/Agentoast.app
```

[GitHub Releases](https://github.com/shuntaka9576/agentoast/releases/latest)からも導入可能です。

機能概要は以下です。

* コーディングエージェント一覧表示
  * 終了、権限確認の場合はオレンジ色
  * Idle状態は灰色
* (オプトイン形式) 通知機能
  * 各コーディングエージェントのライフサイクルに応じた通知
  * 一覧表示と統合し通知のバッジを統合表示

以下でもう少し詳しく説明します。

## 機能

### コーディングエージェント一覧画面

コードベース上で稼働しているコーディングエージェント一覧が確認でき、そのtmuxペインに遷移できます。デフォルトでは`Cmd+Ctrl+N`でtoggle表示が可能です。`agentoast config` で設定変更可能です。

![img](https://raw.githubusercontent.com/shuntaka9576/agentoast/refs/heads/main/docs/assets/menubar.png)

ブラウザやSlackを開いている状態でも、セッション一覧からtmuxペインに少ないショートカットで遷移可能です。
![img](https://raw.githubusercontent.com/shuntaka9576/agentoast/refs/heads/main/docs/assets/menubar.gif)

### 通知

通知機能は**オプトイン方式**です。各種コーディングエージェント設定に書く必要があります。設定方法は[こちら](https://github.com/shuntaka9576/agentoast?tab=readme-ov-file#claude-code)です。

コーディングエージェントの通知やHooks機能と連携し、osascriptよりリッチな通知表示をしています。
![img](https://raw.githubusercontent.com/shuntaka9576/agentoast/refs/heads/main/docs/assets/toast.png)

オプトイン形式をとった理由は、各種Hooksと連携する必要性の他に、**通知文脈の切り替えコストや、自分の深い思考を邪魔させない目的もあります**。ただ終了したら**すぐ通知が欲しいケース**もあると感じ、用意しました。

またハイブリッドも可能です。**通知不要だが自分が注力したい作業が終わり、コーディングエージェントの状況にすぐ気づけるバッジ表示**があると便利なケースがあると感じこの方式をとっています。

赤枠のmuteボタンで実現でき、前述のような通知がOFFになります。コーディングエージェントからHookが呼ばれると、メニューバーアイコンとエージェント一覧画面の対象tmuxペインに🟠バッジが表示されます。これにより通知で作業を妨害されることなく、自分のペースでコーディングエージェントが待ちになっているか気づくことができます。
![img](https://storage.googleapis.com/zenn-user-upload/f10cfc42d39e-20260226.png)

### サポートしているCLI

サポートしているツールは以下です。Kiro CLIは対応予定で、Gemini CLIも要望があれば対応しようと思っています。

* Claude Code
* Codex
* opencode

### コンフィグ

`agentoast config` コマンドで設定ファイル (`~/.config/agentoast/config.toml`) を開けます。toml形式で以下のような形です。

https://github.com/shuntaka9576/agentoast/blob/50114579a77816d62293c62eaa12fdae22995161/README.md#L42-L114

トースト通知が自動で消えるまでの時間は `[toast]` の `duration_ms` で変更できます。デフォルトは 4000ms（4秒）です。クリックするまで消えないようにしたい場合は `persistent` を `true` に設定してください。

通知を一括で止めたいときは `[notification]` の `muted` を `true` にすると、すべての通知が無効になります。こちらはUIからも設定可能です。

エージェントごとの通知設定は `[notification.agents.*]` で行います。`events` で通知を出すイベントを選べます。Claude Code は `Stop`, `permission_prompt`, `idle_prompt`, `auth_success`, `elicitation_dialog`、Codex は `agent-turn-complete`、opencode は `session.status`, `session.error`, `permission.asked` が指定可能です。`focus_events` を設定すると、該当イベント発生時にトーストを出さずにターミナルを自動で前面に持ってきます。Codex では `include_body` でエージェントの応答をトーストに含めるかどうかも設定できます（デフォルトは `true`、200文字まで）。

パネルを開閉するグローバルショートカットは `[keybinding]` の `toggle_panel` で変更できます。デフォルトは `Cmd+Ctrl+N` です。`""` を指定すると無効化できます。

エディタは設定ファイルの `editor` フィールドで指定できます。未設定の場合は `$EDITOR` 環境変数、それもなければ `vim` が使われます。

## 技術面

### macOSアプリ(Tauri)

Tauriを使っています。

https://v2.tauri.app/

macOSのコード署名・公証をGitHub Actions上で行っており、Apple側で発行された公証チケットをstapleしてGitHub Release経由で配信しています。そのためbrew経由でインストールしてもすぐに起動できるようになっています。

コード署名、公証を受けるための技術的なワークフローはこちらで記載していますので参考にしてください。

https://dev.classmethod.jp/articles/shuntaka-tauri-macos-codesign-notarization-updater-github-releases/

TauriのUpdaterプラグインによる自動アップデート機能があり、ユーザーが許可した場合にアップデートが走るようになっています。Updaterはアップデートバイナリの改ざん検知にminisign署名を使っています。

新しいバージョンが作成されると裏側でダウンロードされ、更新マークで自動更新可能です。

![](https://storage.googleapis.com/zenn-user-upload/c26b5b83aa04-20260226.png)
![](https://storage.googleapis.com/zenn-user-upload/3143d5027949-20260226.png)

### tmuxステータス表示

本アプリはtmuxの画面をcapture-paneして内容の一部分をマッチさせてコーディングエージェントの状態を判断させています。なのでtmuxのラッパーアプリと言えます。アプリのアップデートによって壊れやすいので、AIを使って実装効率化できるような仕組みを整えたいです。

### 通知表示

Toast通知をWebView(Tauri ウィンドウ)からNSPanel + NSView(objc2)のネイティブ実装に途中で置き換えました。これはTauriで WebKit レンダラープロセス1つを消費し、30 MBほど使っていることがわかったためです。

https://github.com/shuntaka9576/agentoast/pull/31


## さいごに

この手のツールはN番煎じかつかなり癖が強いのですが、せっかく作ったので公開することにしました。ベストエフォートの対応になりますが、フィードバック大歓迎です！(日本語、英語どちらでも全然問題ないです！)

https://github.com/shuntaka9576/agentoast/issues


## 感謝

毛色は全然違うのですが、こちらのアプリをみて着想を得ましたし、参考にさせて頂いています！ありがとうございます🙇

https://github.com/robinebers/openusage
