---
title: "アプリサービスでfirebaseでのログはどんなものが取得されているのか観察してみた"
emoji: "🦅"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["firebase","データ分析"]
published: true
---

サービスデータの一次情報にさわろうと思うと、firebaseのログが一番手っ取り早い道ですね。

いろいろ見てたら、androidならどんなアプリでも取得しているfirebaseログが閲覧できることがわかり、研究してみたことを書いておこうと思います。
ログの閲覧方法は後半に書いているので、自分でも見たい方はそっちをどうぞ。

## firebaseのログとはなにか

任意の画面、タイミングでコードに1行追加するだけでfirebaseやBigQueryで分析可能なログを残せます。

例えば以下はとある開発環境で取得されたログです。`screen_view` というイベントはfirebaseが自動で取得しているログで、画面が表示されたときに落ちています。

![イベントログの例](https://storage.googleapis.com/zenn-user-upload/406e05923f11638356cbff6e.png)

ここからは確認できないですが、パラメータに`MainActivity`などのクラス名が入っているので、特定の画面について何人が何回表示しているか、などの分析ができます。

詳しくは[Google アナリティクスを使ってみる](https://firebase.google.com/docs/analytics/get-started?platform=ios&hl=ja)を読んでみてください。

## いろんなアプリを見てわかったこと

- 全体的に細かなログをとっているアプリは多くない。
- むしろ開発の動作確認用かな?と感じるログが入っていることが多いアプリもある(image_loadとか)
- が、よくできてるなあと感じるアプリは細かめに取得されてることも多いので、データ分析文化を測るとかでは役に立つかもしれない。
- screenView系は取らないか、 `show` などイベント名を統一しパラメータに画面名を入れるところが多い。(firebase自動でやってくれるもんね)
- 日本のアプリには高確率でfirebaseが入っているが、ワールド級サービスは内製しているのか、firebaseを入れてないことも多かった。
- ただ、google製のサービスは多くに入ってる(自社のだもんね)。そこまで複雑なものは多くないけど、アシスタントは設定アプリ経由でも送信してたりと特殊な動作のものもあり面白い。

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

インストールに成功していれば、こういう表示がずらっと表示されます。

![スクショ](https://storage.googleapis.com/zenn-user-upload/1efff03e022d00dc9939c7ef.png)

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

その他特定の画面を開いたときに特別に発火されるログが見つかったので、その機能をKPIとして置いているor機能として力を入れていくのかな、という印象を受けました。

## 見応えのあるログ取得サービス

今回の投稿をするのにあたり調査した中で面白かったサービスたちです。肌感ですが、ランキングの上位にあるようなサービスは意図を感じるログをとっていることが多いように思いました。

- Play ストア
- Google Assistant
- メルカリ
- ヤンジャン
- TVer
- AbemaTV

驚いたのはGoogleAssistantでした。普通に「今日の天気は」と聞いただけでも「その日初めての検索か」などのログが送信されていました。（内容までは送られていないようです）