> -_- coding: utf-8 -_-
> First Edit: 2020-09-24
> Last Change: 2020-09-24
> Author: @anosillus
---
title: PythonでSlack Botを作る。(BoltとGAEを使用する)
emoji: "👁️‍🗨️"
type: "tech"
topics: ["Slack", "Bolt", "GAE", "Google App Engine", "Python", "Bot", "初心者向け", "入門", "自動化"]
published: false
---

# GAE,Boltを使用しPythonでSlack Botを作る。

## 概要
### この記事に書いてあること
- Bolt for Pythonとクラウド(GAE)を用いたSlack botの導入方法、及び注意点の紹介。
(動作確認日:2020年9月20日)

### 記事のゴール
- 大まかな仕組みを理解した上でBotを動かせるようになる。
- 読者がPythonを用いてSlack Botを簡単に導入できるようになり、かつ自由に機能の追加も出来るようになる。

### 対象
- Slack bot、及びBolt for Python の導入方法に関心がある方。
- PythonでBotを動かすことに関心がある初心者。
- GAEについて関心がある初心者。

### この記事を読んで再現するために必要な要素
- 1時間程度の作業時間 (個人差があります。)
- Google SDKを導入でき、Webブラウザを使える環境があること。(管理者権限がないと厳しい気がします)^[[公式ドキュメント](https://cloud.google.com/sdk/install)]
- クレッジットカードを所有していること。
- GoogleアカウントとSlackのWorkSpaceを保有していること。
- Slackの用語を理解でき、簡単なPythonのコードが読め、Shellが立ち上げられてcdなどのコマンドを使えること。
- 英語の技術ドキュメントと英語の仕様説明が読めること。

## イントロとモチベーション

先日、Slackを運営するSlack社から`Bolt for Python`がリリースされた。^[執筆時(2020年9月20日)においてはアルファ版であった。]
`Bolt`, `Slack Bot`とは何か、ということについてこの記事では取り扱わないが、作者の解説動画へのリンクを記載しておく。
@[youtube](X8YRevU_fD8)

このリリースによって、これまでJavaやJavaScriptからしか使えなかったBoltをPythonから呼べるようになった。そのため、Pythonのライブラリを呼ぶ処理とBoltを呼ぶ二つの処理を、単一言語、単一サーバーで作れるようになった。^[以前からSlack社はpython-slackclientやpython-slack-events-apiなどをリリースしていたが、私はBoltの設計に興味があった。これまではBoltを叩くTypeScriptと処理系を書くPythonの間の連携処理を作る過程で面倒になっていた。]

Slackをスクリプトで操作する方法には「外からの単方向の通信」^[ローカルマシンから`Incoming Webhooks`などの機能を用いてた操作]と「双方向の連携」^[WorkSpace上で起きたイベントに対応して動く操作(Bot)]の2種類があり、この記事はBoltの機能を活用出来る後者を採用する。

## 実装工程の概要

必要な実装工程を5つのタスクに分類します。(括弧内は予測作業時間です。)
1. Slackの`Workspace`上に操作可能なBotを準備する。(5~20分)
2. Slackからのイベント通知に対して、サーバーが実行する処理、Botに対して発する命令を定義する。(10分)
3. Slackからのイベント通知を受信するサーバーを構築する。
4. Workspace上で発生したイベントを外部に通知する機能を有効化する。
5. 動作テストを行う。(5分)


:::details フロー図に使ったコード
``` mermaid.js
sequenceDiagram
    participant local as ローカル環境
    participant gae   as App Engine
    participant slack as Slack
    participant user  as ユーザー
    alt 準備フェイズ
        Note over slack: Botの準備をする。
        slack ->> local: BotのTokenを取得する。
        Note over local: ファイルを用意する
        local ->>+ gae: デプロイする。
        Note over local: URLを確認する。
        Note over slack: サーバーのURLを入力する
        slack -->> gae: 認証する
    end

    loop 運用フェイズ
        user -->>+ slack: 文字を入力する。
        slack -->>+ gae: イベントを通知する。
        deactivate slack
        gae -->>+ slack:Botに命令を出す。
        deactivate gae
        slack -->> user: Botが返信する。
        deactivate slack
    end
    local ->> gae:停止させる。
    deactivate gae
```
:::

-----

## 1. SlackのWorkSpace上に操作可能なBotを準備する。
### 作業概要(1)
- **Slackのウェブサイト上でボタンをクリックして設定をオンにする作業です。**

### 作業手順(1)
1. Botを導入したいWorkSpaceにログインする。
2. [Slack API: Applications | Slack](https://api.slack.com/apps?new_app=1)にアクセスします。
3. [Slack APIページ](https://api.slack.com/apps?new_app=1)内の、`Create New App`を選択(クリック)して、アプリ名と導入先のWorkspaceを指定し、`Create App`を作る。
4. [Slack APIページ](https://api.slack.com/apps?new_app=1)から作成したアプリ名を選択し、`Basic Information`のページに飛ぶ。
5. [Applications](https://api.slack.com/apps?new_app=1)」 > {アプリ名} > `Features` > `OAuth & Permissions`の場所にある`Bot Token Scopes`を変更し、Botに`chat:write`, `chat:read`の権限を与えてから、`Install App to Workspace`(もしくは`Reinstall App`)を押してBotをインストールする。
6. [Applications](https://api.slack.com/apps?new_app=1)」 > {アプリ名} > `Features` > `OAuth & Permissions` > `Tokens for Your Workspace` の階層にある`Bot User OAuth Access Token`の値をメモします。
7. [Applications](https://api.slack.com/apps?new_app=1)」 > {アプリ名} > `Settings`> `Basic Information` > `Credentials` の階層にある `Signing Secret` の値をメモします。

### 注意点(1)
- Slackの仕様は、公式ドキュメントを参照するとよいです。  
- 個人的なハマりどころ、感想としては、Slackは機能が多くサイト構成が複雑なため、目的達成に何の機能が必要で何を設定するべきなのか理解していないと時間がかかりそうでした。
- 私はこの作業を手探りで20分くらいで終わらせられましたが、開くべきページや設定を間違え続けると数時間溶けることもありそうです。
- 機能を追加していく場合、安定版と開発版の2種類のBotを作っておくと、クラウドとngrokの設定を分離でき、運用しながらのデバッグもしやすくなりました。

### 参考(1)
- [Slack | Bolt for JavaScript](https://slack.dev/bolt-js/tutorial/getting-started)
- [日本語版ページ | Slack](https://api.slack.com/lang/ja-jp)

## 2. Slackからのイベント通知に対して、サーバーが実行する処理、Botに対して発する命令を定義する。
### 作業概要(2)
- **サーバーの処理を実装します。**
- 今回はBolt for PythonのSampleディレクトリ内のサンプルコードを使用します。
- サーバーがSlackと通信出来るようにSlackのトークンを用意する必要があり、今回はGAEの環境変数の設定ファイル(`env_vaiables.yaml`)内にトークンを記入します。
- 時前で命令を書く場合は、Bolt for Pythonのドキュメントの他に、JavaScriptのBoltのドキュメント、実装が参考になります。
- 私が動かしたサンプルコードは[こちら](https://github.com/slackapi/bolt-python/tree/d2a6e36e407f12ec960e443056e4aab66bb066c0/samples/google_app_engine/flask)です。

:::message alert
私が参照したのはアルファ版(2020年9月20日の最新版)です。サンプルコード、ライブラリの仕様、依存関係、GAEの仕様に変更が入る可能性があるため、最新版を確認することをお勧めします。
:::
### 作業手順(2)
1. 必要なファイル郡(サンプルコード)をGitHubからダウンロードする。
2. サンプルコードにSlackのトークンを記載する。
::: コードで表すと以下のようになります。
``` sh
cd $HOME  # 再現性のために書いています。
git clone https://github.com/slackapi/bolt-python.git    # Git が使えない人はwgetやブラウザを使って頑張って下さい。
cd bolt-python/samples/google_app_engine/flask    # サンプルファイル郡の場所まで移動
cp env_variables.yaml.sample env_variables.yaml    # サンプルを下書きにして記入出来るようにする。
code env_variables.yaml    # SLACK_BOT_TOKENとSLACK_SIGNIN_SECRETを記入例に従って記入して下さい。code は Visual Studio Code の起動例です。
```
:::

### 注意点(2) 
- 複数あるTOKENのうち、`SLACK_BOT_TOKEN`に該当するのは`Bot User OAuth Access Token`です。
- TOKENには権限の更新、再インストールによって値が更新されるものがあります。

## 3. Slackからのイベント通知を受信するサーバー(App Engine)を構築する。
### 作業概要(3)
- **GAEを使って、サーバーを起動します。**
- GAEの入門、概要は[公式チュートリアル](https://console.cloud.google.com/appengine/start)を見て下さい。
- チュートリアル、アプリの作成を完遂するには、環境選択とクレジットカードの登録とリージョン選択^[us-central1を選べばよいと思います。]が必要になります。
- 私はGAE入門チュートリアルの後、[「App Engine で Python 3 アプリをビルドする」](https://cloud.google.com/appengine/docs/standard/python3/building-app?hl=ja)(以下、ガイド)を流し読みしました。
- このガイドを流し読みし真似て、ガイドがdeployするソースファイルをガイド指定のものから時前で用意したものに置換するだけで、Slack botを動かすサーバーを用意できます。この記事ではソースファイルにPython for BoltのGAEサンプルを使います。

### 作業手順(3)
具体的に必要になるのは以下の5つの手順です。(詳細は作業概要(3)のリンク先を見て下さい)
1. Googleアカウントにログインする。
2. Google Cloudにクレジットカードを登録する。
3. Google SDKをインストールする。
4. Google App Engine上にアプリを作る。(スタンダード環境をお勧めします。)
5. ローカルマシン上で以下のコマンド、作業を実行する。
::: 実行する処理一覧
``` sh
######  5.1 サーバーにアップロードするファイルがある場所に移動する。  ######
cd $HOME/bolt-python/samples/google_app_engine/flask

######  5.2 Google SDKのセットアップ  ######
gcloud components update    # gcloud コマンドが使えない場合はSDKがダウンロードされていないかPathが通っていません。
gcloud auth application-default login
gcloud components install app-engine-python

######  5.3 デプロイ設定  ######
gcloud init    # 質問されるのでブラウザで設定したリージョンやプロジェクト名を答える。
######  5.4 ファイルアップロードとサーバーの起動  ######
gcloud app deploy    # サーバー起動後にデプロイ先のURLが表示される。
```
:::

### 注意点(3) 
- もしガイドを実行していくなら、ダウンロードを勧められるzipファイルには触らず、チュートリアルのコードを自分で書くか、参照元のGitHubを見て必要な分だけ落としてくる方が早いです。^[中身はGoogleクラウドコンピューティング全体のチュートリアルであり、動画も含まれていてダウンロード、ファイルアップロードに無駄に時間がかかる。]
- デプロイしたアプリは、ブラウザから停止させることが出来る。

::: Google App Engineの選定理由について
Bolt for Pythonのチュートリアルでは`ngrok`がサーバー構築に用いられているが、私は手軽に運用出来るようにクラウドコンピューティングを採用した。具体的には`GAE`(Google App Engine)を用いた。^[デバッグや検証はngrokで行う方が簡単で早いです。]
他に`AWS Lambda`や`ECS`(Amazon Elastic Container Service)、`GCE`(Google Computer Engine)、`Heroku`を検討したが『趣味なので、無料であること、簡潔さを重視する』という基準から、無料内の自由度が高く、ファイルアップロードが簡単なGAEを選択した。(個人の感想による比較です。)^[なお、Bolt for Pythonのsampleディレクトリに他の環境の用の動作スクリプトも用意されているので、それを見ればGAE以外も簡単に動かせる。]^[有料だと管理コストが増える。]
:::

::: GAEのリージョン選択と無料枠(Always Free)について
リージョンについては、私は東京リージョン(asia-northeast)を選びました。
ローカルからサーバーまでの実距離が短い方が、botのレスポンス速度が向上する、と思ったからですが、よく考えると通信相手のSlackのホスティングサーバーは米国のAWSサーバー内にある^「[Slack Case Study](https://aws.amazon.com/solutions/case-studies/slack/)]のでアイオワリージョン(us-central1）などを選択した方がよいと思います。
リージョン毎にインスタンス使用料は異なり、例えば東京リージョンは高めの価格設定になっています。
しかし、現在のGAEの無料枠(`Always Free`)の仕様ではスタンダード環境だと、使用した金額ではなく、一日あたりのインスタンスの総起動時間とインスタンススペックの組合せで決まるため、組み合わせが無料枠に収まっていれば東京リージョンでも何処でも使用料は請求されません。
私は単体の「F」インスタンスを常時稼動させていますが、無料枠に収まっています。^[火力が足りなくなれば課金する予定ですが、現状のBot運用には今のところ差し支えないと思っています。]
:::

### 参考リンク(3)
- [App Engine の料金  |  App Engine ドキュメント  |  Google Cloud](https://cloud.google.com/appengine/pricing?hl=ja)
- [App Engine の概要  |  Python 3 の App Engine スタンダード環境  |  Google Cloud](https://cloud.google.com/appengine/docs/standard/python3/an-overview-of-app-engine#services)
[slackapi/bolt-python: A framework to build Slack apps using Python (still in beta)](https://github.com/slackapi/bolt-python)

## 4. Workspace上で発生したイベントを外部に通知する機能を有効化する。
### 作業概要(4)
- 今回は`slash command`に関連した機能を有効化します。
- 特定のコマンドの入力イベントが発生したとき、サーバーにその通知を投げるようにSlackに設定します。

### 作業内容(4) 
1. [Applications](https://api.slack.com/apps?new_app=1)」 > {アプリ名} > `Settings` の下にある
 `Slash Commands`を選択し、続けて`Create New Commands`を選択します。
2. 入力フォームが表示されるので `Command`欄には `/hey-google-app-engine`を入力します。
3. `Request URL`には`gcloud app deploy`の後に表示されたサーバーURL(`http://{app-name}-{app-id}.an.r.appspot.com`)を `https://{app-name}-{app-id}.appspot.com/slack/events`の形式に直して入力します。
4. `Short Description`の欄はなんでも良いですが、'hello, bot`などの任意の文字を入力します。
5. `Save`を選択します。

### 注意点(4)
- 作業内容4の4についてですが、`http`形式のPostをSlackは受け付けないので、`gcloud app deploy`が表示したURLを`https`形式に変更しておく必要があります。

## 5.テスト
- Workspaceにアクセスして任意の`channel`にbotを招待した上で `/hey-google-app-engine` と入力すると、botが返事を返してくれます。
- デバッグはGUIや、SDKから実行出来ます。
- GAEのサーバーはブラウザから止められます。^[CUIでも出来るとは思いますが。]

## 感想とこの次
- ここまでで想定以上に記事が長くなってしまったのでBoltのAPIの仕様説明などはボツにしてしまった。
- 導入記事をちゃんと書くと、ここまで面倒になるとは想定していなかった。
- 最近、機能増えたし、Lambdaの方が良かったのでは？と思わなくはない。
- もう書いてしまったけど、Bolt for Pythonは未だalfa版^[執筆時の版、記事公開時にはbeta版になっていた。]であり、一般向けに書くような記事ではなかった気も若干する。
- Boltの機能を活用したアプリを他にも作って遊んでいたので、そちらの機能紹介、作り方を私の元気と需要があれば書きます。

This article is written by anosillus(anosillus@gmail.com) <i class="cc cc-by-sa"></i>
