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

いい感じにチャットが機能していることが確認できました！このままでは見た目が少し寂しいので、次のページでは少しデザインを整えてあげましょう。