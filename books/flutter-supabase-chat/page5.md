---
title: "チャットのロジック"
---

## メッセージモデルの定義

チャットページの作成に入る前に下準備です。アプリ内でチャットデータを扱う際に型を効かせられるようにモデルを定義しましょう。ここでは`messages`テーブル用のモデルを作ります。その際に`fromMap`コンストラクターを作って簡単にSupabaseから帰ってきたデータからインスタンスを作れるようにします。

```dart:lib/models/message.dart
class Message {
  Message({
    required this.id,
    required this.profileId,
    required this.content,
    required this.createdAt,
    required this.isMine,
  });

  /// メッセージのID
  final String id;

  /// メッセージを送信した人のユーザーID
  final String profileId;

  /// メッセージの内容
  final String content;

  /// メッセージの送信日時
  final DateTime createdAt;

  /// このメッセージを送ったのが自分かどうか
  final bool isMine;

  /// [map]にSupabaseからのデータを渡し、[myUserId]には自分のauthのユーザーIDを渡すと[Message]のインスタンスを作成できる。
  Message.fromMap({
    required Map<String, dynamic> map,
    required String myUserId,
  })  : id = map['id'],
        profileId = map['profile_id'],
        content = map['content'],
        createdAt = DateTime.parse(map['created_at']),
        isMine = myUserId == map['profile_id'];
}
```

## チャットページの作成

いよいよメインのチャットページの作成に入ります。このページはリアルタイムにメッセージがロードされ、さらに誰でも他の人に向けてメッセージを送信することができるページになります。ここではsupabase-flutterの[stream()](https://supabase.com/docs/reference/dart/stream)メソッドを使ってメッセージテーブルからデータをロードしてきています。

```dart:lib/pages/chat_page.dart
import 'dart:async';

import 'package:flutter/material.dart';

import 'package:my_chat_app/models/message.dart';
import 'package:my_chat_app/utils/constants.dart';

/// 他のユーザーとチャットができるページ
///
/// `ListView`内にチャットが表示され、下の`TextField`から他のユーザーへチャットを送信できる。
class ChatPage extends StatefulWidget {
  const ChatPage({Key? key}) : super(key: key);

  static Route<void> route() {
    return MaterialPageRoute(
      builder: (context) => const ChatPage(),
    );
  }

  @override
  State<ChatPage> createState() => _ChatPageState();
}

class _ChatPageState extends State<ChatPage> {
  /// メッセージをロードするためのストリーム
  late final Stream<List<Message>> _messagesStream;

  @override
  void initState() {
    final myUserId = supabase.auth.currentUser!.id;
    _messagesStream = supabase
        .from('messages')
        .stream(primaryKey: ['id'])
        .order('created_at') // 送信日時が新しいものが先に来るようにソート
        .map((maps) => maps
            .map((map) => Message.fromMap(map: map, myUserId: myUserId))
            .toList());
    super.initState();
  }

  @override
  Widget build(BuildContext context) {
    return Scaffold(
      appBar: AppBar(
        title: const Text('チャット'),
        actions: [
          TextButton(
            onPressed: () {
              supabase.auth.signOut();
              Navigator.of(context)
                  .pushAndRemoveUntil(RegisterPage.route(), (route) => false);
            },
            child: Text(
              'ログアウト',
              style: TextStyle(color: Theme.of(context).colorScheme.onPrimary),
            ),
          )
        ],
      ),
      body: StreamBuilder<List<Message>>(
        stream: _messagesStream,
        builder: (context, snapshot) {
          if (snapshot.hasData) {
            final messages = snapshot.data!;
            return Column(
              children: [
                Expanded(
                  child: messages.isEmpty
                      ? const Center(
                          child: Text('早速メッセージを送ってみよう！'),
                        )
                      : ListView.builder(
                          reverse: true, // 新しいメッセージが下に来るように表示順を上下逆にする
                          itemCount: messages.length,
                          itemBuilder: (context, index) {
                            final message = messages[index];
                            return Text(message.content);
                          },
                        ),
                ),
                // ここに後でメッセージ送信ウィジェットを追加
              ],
            );
          } else {
            // ローディング中はローダーを表示
            return preloader;
          }
        },
      ),
    );
  }
}
```

ここまできたらぜひ一度動きを確認しましょう。今回はmain.dartを上書きして`ChatPage`を強制的に表示させるのではなく、きちんと場合分けをし、アプリを開いた際にログイン済みの場合に`ChatPage`が表示され、ログインしていない場合は`RegisterPage`が表示されるようにしましょう。

### SplashPageでリダイレクト

ユーザーがアプリを立ち上げたときにそのユーザーがログインしているかどうかに応じて適切なページにリダイレクトしてあげましょう。これをするにはSplashPageというページを作り、その中でログイン状態を判別し適切なページにリダイレクトしてあげます。UIはただ単に真ん中でローダーがくるくる回っているだけのものになります。

```dart:lib/pages/splash_page.dart
// ignore_for_file: use_build_context_synchronously

import 'package:flutter/material.dart';
import 'package:my_chat_app/pages/chat_page.dart';
import 'package:my_chat_app/pages/register_page.dart';
import 'package:my_chat_app/utils/constants.dart';

/// ログイン状態に応じてユーザーをリダイレクトするページ
class SplashPage extends StatefulWidget {
  const SplashPage({Key? key}) : super(key: key);

  @override
  SplashPageState createState() => SplashPageState();
}

class SplashPageState extends State<SplashPage> {
  @override
  void initState() {
    super.initState();
    _redirect();
  }

  Future<void> _redirect() async {
    // widgetがmountするのを待つ
    await Future.delayed(Duration.zero);

    /// ログイン状態に応じて適切なページにリダイレクト
    final session = supabase.auth.currentSession;
    if (session == null) {
      Navigator.of(context)
          .pushAndRemoveUntil(RegisterPage.route(), (route) => false);
    } else {
      Navigator.of(context)
          .pushAndRemoveUntil(ChatPage.route(), (route) => false);
    }
  }

  @override
  Widget build(BuildContext context) {
    return const Scaffold(body: preloader);
  }
}
```

main.dartの`MyApp`を書き換えてアプリロード時に`SplashPage`が表示されるようにします。

```dart:lib/main.dart
class MyApp extends StatelessWidget {
  const MyApp({Key? key}) : super(key: key);

  @override
  Widget build(BuildContext context) {
    return const MaterialApp(
      debugShowCheckedModeBanner: false,
      title: 'チャットアプリ',
      home: SplashPage(),
    );
  }
}
```

ここまで作れたらまた一度挙動を確認してみましょう。アプリを一度Refreshしてみてください。うまくいけばチャット画面が表示されるはずです。

![チャットがまだ入力されていないチャット画面](/images/flutter-supabase-chat/empty-chat-page.png)

また、実際にチャットが表示されるかを確認するために、Supabaseのダッシュボードから手動でメッセージを作成してみます。ダッシュボードの`Table Editor -> messages`でメッセージテーブルを開き、Insertボタンから新しくrowを追加します。

![Supabaseのダッシュボードとチャット画面](/images/flutter-supabase-chat/add-a-row.png)

## チャットの送信

データベースのメッセージ情報がアプリの方に正しく同期されていることが確認できたところでアプリにもチャットの送信ロジックを実装していきましょう。
チャットページの下の方にこちらのテキストボックスと送信ボタンのセットのウィジェットを追加します。

```dart:lib/pages/chat_page.dart
...
/// チャット入力用のテキストフィールドと送信ボタンを持つウィジェット
class _MessageBar extends StatefulWidget {
  const _MessageBar({
    Key? key,
  }) : super(key: key);

  @override
  State<_MessageBar> createState() => _MessageBarState();
}

class _MessageBarState extends State<_MessageBar> {
  late final TextEditingController _textController = TextEditingController();

  @override
  Widget build(BuildContext context) {
    return Material(
      color: Colors.grey[200],
      child: SafeArea(
        child: Padding(
          padding: const EdgeInsets.all(8.0),
          child: Row(
            children: [
              Expanded(
                child: TextFormField(
                  keyboardType: TextInputType.text,
                  maxLines: null, // 複数行入力可能にする
                  autofocus: true, // ページを開いた際に自動的にフォーカスする
                  controller: _textController,
                  decoration: const InputDecoration(
                    hintText: 'メッセージを入力',
                    border: InputBorder.none,
                    focusedBorder: InputBorder.none,
                    contentPadding: EdgeInsets.all(8),
                  ),
                ),
              ),
              TextButton(
                onPressed: () => _submitMessage(),
                child: const Text('送信'),
              ),
            ],
          ),
        ),
      ),
    );
  }

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  /// メッセージを送信する
  void _submitMessage() async {
    final text = _textController.text;
    final myUserId = supabase.auth.currentUser!.id;
    if (text.isEmpty) {
      // 入力された文字がなければ何もしない
      return;
    }
    _textController.clear();
    try {
      await supabase.from('messages').insert({
        'profile_id': myUserId,
        'content': text,
      });
    } on PostgrestException catch (error) {
      // エラーが発生した場合はエラーメッセージを表示
      context.showErrorSnackBar(message: error.message);
    } catch (_) {
      // 予期せぬエラーが起きた際は予期せぬエラー用のメッセージを表示
      context.showErrorSnackBar(message: unexpectedErrorMessage);
    }
  }
}
```

あとはこれをチャットページのUIに追加してあげます。チャットページの`ここに後でメッセージ送信ウィジェットを追加`と書いてある箇所を`_MessageBar()`に書き換えてあげましょう。

```dart:lib/pages/chat_page.dart
...
return Column(
  children: [
    Expanded(
      child: messages.isEmpty
          ? const Center(
              child: Text('早速メッセージを送ってみよう！'),
            )
          : ListView.builder(
              reverse: true,
              itemCount: messages.length,
              itemBuilder: (context, index) {
                final message = messages[index];
                return Text(message.content);
              },
            ),
    ),
    _MessageBar(), //ここに追加！
  ],
);
...
```

これでアプリ内からもチャットを送信することができるようになったはずです！

![アプリからチャットを送信](/images/flutter-supabase-chat/chat-page-with-form.png)

いい感じにチャットが機能していることが確認できました！

## RLSを使ってアプリをセキュアにする

これでも問題なくチャットとして機能するのですが、今の状態だと実はAPIに特に何のアクセス制限もかけていない状態で、誰でも好きにデータの改ざんができてしまう状態です。なので、SupabaseのRow Level Security (行レベルセキュリティ)、略してRLSを使ってアプリをセキュアにしてあげましょう。

RLSとはテーブルごとに設定できる「誰がCRUDのどのアクションを実行する権限があるか」を設定する機能です。これが設定されていないと、少しSupabaseの知識がある人であれば簡単に他のユーザーになりすましてチャットを投稿したり、データの全削除なんかもできたりします。

設定方法は簡単！まず`Authentication -> Policies`のページに移動します。ここでは今アプリで設定されているRLSを閲覧することができます。

![RLSが設定されていない](/images/flutter-supabase-chat/no-rls.png)

今はそもそもRLSが有効化されていない状態なので、まず`messages`と`profiles`テーブル両方の`Enable RLS`ボタンを押してRLSを有効化してあげましょう。

RLSが有効化されるとデフォルトで`Select`, `Insert`, `Update`, `Delete`全てのアクセスを弾きます。そこから、`true`か`false`を返すSQLを含んだポリシー定義してあげて特定のアクションを許可するか拒否するかを決めることができます。

まずは`messages`テーブルの`Insert`ポリシーを設定していきましょう。ここでは他のユーザーへのなりすまし投稿を防ぐようなポリシーを設定します。一番下の`With Check Expression`の箇所に、特定の行ごとに`true`か`false`になるようなSQLの`where`文にあたる部分を書き、その行が`true`の時に`Insert`が許可され、`false`の時に`Insert`が拒否されるようになります。

今回は`With Check Expression`の箇所に`auth.uid() = profile_id`と記述します。

![メッセージテーブルのInsertポリシー](/images/flutter-supabase-chat/messages-insert-policy.png)

ここで出てくる`auth.uid()`はSupabaseが用意してくれている関数で、アクセスしようとしてきたユーザーのIDを返してくれます。今回のポリシーではそのユーザーのIDと、実際に挿入しようとしているprofile_idの値を比較して、一致していれば`Insert`を許可するようにしています。これで他人になりすましてチャットデータの挿入を防ぐことができました。

では続いて`Select`ポリシーを設定しましょう。`messages`テーブルは特に制限なく誰でも閲覧できる形になっているので、`Using Expression`のところに`true`と入力してあげます。これで誰でもメッセージを閲覧することができるようになりました。

![メッセージテーブルのSelectポリシー](/images/flutter-supabase-chat/messages-select-policy.png)

ここで注意深い方は気づいたかもしれませんが、`Insert`の時は`With Check Expression`を設定し、`Select`の時は`Using Expression`を設定しましたね。`With Check`というのは新しく挿入・上書きしようとしているデータに対してチェックを行うときに使うもので、`Using`は既存のデータに対してチェックを行うときに使うものです。今回の例では特に`Update`のポリシーは書きませんが、`Update`のポリシーを設定するときは`With Check`と`Using`の両方を使います。

さて、続いて`profiles`テーブルのポリシー設定に進みましょう。

`profiles`テーブルは挿入に関してはユーザー登録時にSupabaseのトリガーを使って自動的に行ってくれるので特にPolicyを設定する必要はありません。なので今回は全ユーザーが自由に他のユーザーのプロフィールを閲覧できるようなポリシーを設定してあげます。

![プロフィールのSelectポリシー](/images/flutter-supabase-chat/profiles-select-policy.png)

以上でRLSの設定は完了です。たったこれだけのステップで不正アクセスされないセキュアなアプリができました。

次のページでは、このままではアプリの見た目が少し寂しいので、少しデザインを整えてあげましょう。

