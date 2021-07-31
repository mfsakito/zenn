---
title: "firebaseでのログはどんなものが取得されているのか観察してみた"
emoji: "🦅"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["firebase","データ分析"]
published: false
---

firebaseのログを分析に使っているサービスが多いと思います
GoogleAnalyticsやBigQueryと簡単に連携できるのがいいよね
非エンジニアで少しデータの一次情報にさわろうと思うと、一番手っ取り早い道

androidアプリに関しては、どんなアプリでも取得しているfirebaseログを閲覧できることがわかり、研究してみたことを書いておこうと思います。
ログの閲覧方法は後半に書いているので、自分でも見たい方はそっちをどうぞ。

## いろんなアプリを見てわかったこと

- 全体的に細かなログをとっているアプリは多くない。
- むしろ開発の動作確認用かな?と感じるログが入っていることが多いアプリもある(image_loadとか)
- が、よくできてるなあと感じるアプリは細かめに取得されてることも多いので、データ文化を測るとかでは役に立つかもしれない。
- screen_view系は `show` などイベント名を統一し、パラメータに画面名を入れるところが多い。
- 日本のアプリには高確率でfirebaseが入っているが、ワールド級サービスは内製しているのか、firebaseを入れてないことも多かった。

## ログの閲覧方法

詳しくはドキュメントにあるここのページ
> 詳細ロギングを有効にして SDK によるイベントのロギングをモニタリングし、イベントが正しくロギングされているかどうかを確認できます。対象となるのは、自動でロギングされるイベントと手動でロギングされるイベントの両方です。
https://firebase.google.com/docs/analytics/events?hl=ja&platform=android#view_events_in_the_android_studio_debug_log

ポイントとしては、
- デバッグのための仕組みではあるが、プロダクションビルドのログも見れる
- androidに関してはリリースビルドがあれば閲覧可能なので、開発中のものでなくても確認できる = 他社のアプリでもfirebaseのログが見れる

これがわかると
- どんなことを改善しようとしているか
- どんなKPIが置かれているか

などが想像できるようになります。
ライバルの動向や技術的にレベルの高いサービスの中身を少しだけ垣間見ることができそうです。

もうひとつ、これができるとログの動作確認を依頼側完結で実行できるので、何かととても重宝するようになるのがいいですね。

## 具体的なやり方

### 準備
1. android studioをインストール
2. adbの設定をする
    以下を順に入力する
    1. `echo "export PATH=$PATH:~/Library/Android/sdk/platform-tools/" >> ~/.zshrc`
    2. `source ~/.zshrc`
    3. `adb` と打って、エラーが出なければ完了 

インストールに成功しているときの画面


### 実践
1. アプリがインストールされた端末をPCにつなぐ
2. terminalを開く
3. 以下3つのコマンドを順に入力する
    - `adb shell setprop log.tag.FA VERBOSE`
    - `adb shell setprop log.tag.FA-SVC VERBOSE`
    - `adb logcat -v time -s FA FA-SVC`
4. 実機のアプリを操作してログを確認する

## 実際にみてみる

某サービスでとっているアプリログの要約

user_propertyに以下の項目を入れていることがわかりました(一部略)。
わりとデフォルトのものが多いですが、独自のものとしてコホートのフラグやDLの経路を残しているようです。

- 初回起動のタイムスタンプ (※)
- 累計起動時間 (※)
- セッションID (※)
- 累計起動回数 (※)
- 翌日起動したかどうか
- DL経路情報

※ 他アプリでも多く見られたのでfirebaseがデフォルトでとっているものと思われます。

実際にはこのように表示されます。
```
user_property {
  set_timestamp_millis: xxxxxxxxx
  name: first_open_time(_fot)
  string_value: 
  int_value: xxxxxxxx
}
user_property {
  set_timestamp_millis: xxxxxxxxx
  name: first_open_after_install(_fi)
  string_value: 
  int_value: xxxxxxxxx
}
user_property {
  set_timestamp_millis: xxxxxxxxx
  name: lifetime_user_engagement(_lte)
  string_value: 
  int_value: xxxxxxxxx
}
...
```

その他操作ごとのログなどは詳細にとってないようでしたが、特定の画面を開いたときに特別に発火されるログが見つかったので、その機能をKPIとして置いているor機能として力を入れていくのかな、という印象を受けました。

