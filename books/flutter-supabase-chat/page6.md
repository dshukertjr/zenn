---
title: "UI仕上げ"
---

今のままでは少しUIが寂しいのでもう少しチャットアプリっぽくしてあげましょう。まず最初にコードのリファクターがしやすいようにコードを整えます。

## チャットのメッセージ要素をウィジェットに切り抜き

まず、チャットページの`ListView`の中で色々と細々書くとコードが読みづらくなるので、チャットのメッセージを表示するためのウィジェットを切り出します。チャットページの一番下にこちらを追加しましょう。このウィジェットは、メッセージの情報を受け取り、それらを元にチャットのメッセージを表示するしてくれます。こうして抜き出してあげることで、チャットページの`ListView`の方をいじらずにUIを改善していけます。

一旦まずシンプルにチャットのテキストのみを表示してみます。

```dart:lib/pages/chat_page.dart
...
/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
  }) : super(key: key);

  /// メッセージの本文
  final Message message;

  @override
  Widget build(BuildContext context) {
    return Text(message.content);
  }
}
```

`ListView`のコードを書き換えて今作った`_ChatBubble`が表示されるようにしましょう。

```dart:lib/pages/chat_page.dart
...
ListView.builder(
    reverse: true,
    itemCount: messages.length,
    itemBuilder: (context, index) {
        final message = messages[index];

        return _ChatBubble(
            message: message,
        );
    },
),
...
```

これであとは`_ChatBubble`をいじるだけでUIの変更ができるようになり、身動きがとりやすくなりました。

## プロフィールアイコンの表示

ここからはまず、一つ一つのメッセージの横にユーザーのアイコンをつけてあげましょう。チャットを送信したユーザーのユーザー名の先頭に文字をアイコンのように表示します。

`profiles`テーブルに保存されているプロフィール情報を保持するためのモデルを定義します。

```dart:lib/models/profile.dart
/// ユーザーのプロフィール情報を保持するクラス
class Profile {
  Profile({
    required this.id,
    required this.username,
    required this.createdAt,
  });

  /// ユーザーのID
  final String id;

  /// ユーザー名
  final String username;

  /// ユーザーの作成日時
  final DateTime createdAt;

  Profile.fromMap(Map<String, dynamic> map)
      : id = map['id'],
        username = map['username'],
        createdAt = DateTime.parse(map['created_at']);
}
```

次に`_ChatBubble`クラスでプロフィール情報を受け取れるように書き換えます。

```dart:lib/pages/chat_page.dart
...
/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
    required this.profile, // 追加
  }) : super(key: key);

  /// メッセージの本文
  final Message message;

  /// 投稿者のプロフィール情報
  final Profile? profile; // 追加

  @override
  Widget build(BuildContext context) {
    return Text(message.content);
  }
}
```

`ChatPage`内で各メッセージの投稿者のプロフィールを取得するようにしましょう。

`ChatPage`をこのように書き換えましょう。


```dart:lib/pages/chat_page.dart
...
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

  /// プロフィール情報をメモリー内にキャッシュしておくための変数
  final Map<String, Profile> _profileCache = {};

  /// メッセージのサブスクリプション
  late final StreamSubscription<List<Message>> _messagesSubscription;

  @override
  void initState() {
    final myUserId = supabase.auth.currentUser!.id;
    _messagesStream = supabase
        .from('messages')
        .stream(primaryKey: ['id'])
        .order('created_at')
        .map((maps) => maps
            .map((map) => Message.fromMap(map: map, myUserId: myUserId))
            .toList());
    _messagesSubscription = _messagesStream.listen((messages) {
      for (final message in messages) {
        _loadProfileCache(message.profileId);
      }
    });
    super.initState();
  }

  @override
  void dispose() {
    // きちんとcancelしてメモリーリークを防ぐ
    _messagesSubscription.cancel();
    super.dispose();
  }

  /// 特定のユーザーのプロフィール情報をロードしてキャッシュする
  Future<void> _loadProfileCache(String profileId) async {
    if (_profileCache[profileId] != null) {
      return;
    }
    final data =
        await supabase.from('profiles').select().eq('id', profileId).single();
    final profile = Profile.fromMap(data);
    setState(() {
      _profileCache[profileId] = profile;
    });
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
                          reverse: true,
                          itemCount: messages.length,
                          itemBuilder: (context, index) {
                            final message = messages[index];
                            return _ChatBubble(
                              message: message,
                              profile: _profileCache[
                                  message.profileId], // キャッシュしたプロフィール情報を渡す
                            );
                          },
                        ),
                ),
                const _MessageBar(),
              ],
            );
          } else {
            return preloader;
          }
        },
      ),
    );
  }
}
...
```

`initState`の中でメッセージストリームに対してリスナーを張り、メッセージを取得した際にメッセージ一覧をループします。その中でまだメモリー内にキャッシュしていないユーザーがいない場合はSupabaseから取得してキャッシュします。そうして適宜プロフィール情報を取得・キャッシュしつつ`_ChatBubble`クラスにそのキャッシュからメッセージの送信者のプロフィール情報を渡している形になります。

ここまできたらあとは`_ChatBubble`を編集してチャットの見た目を整えていくだけです！まずはプロフィールアイコンを表示してみましょう。メッセージのテキストを`Row`で囲い`CircleAvatar`でプロフィールアイコンを表示します。この時、自分のメッセージの場合はプロフィールアイコンを表示しないようにします。

```dart:lib/pages/chat_page.dart
...
/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
    required this.profile,
  }) : super(key: key);

  final Message message;
  final Profile? profile;

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 18),
      child: Row(
        children: [
          if (!message.isMine) // 自分のメッセージでない時のみプロフィールアイコンを表示
            CircleAvatar(
              child: profile == null
                  ? preloader
                  : Text(profile!.username.substring(0, 2)),
            ),
          const SizedBox(width: 12),
          Text(message.content),
        ],
      ),
    );
  }
}
```

ここで一つ問題が！よくあるチャットアプリでは自分のプロフィールアイコンは表示されないことが多いため、今回のアプリもそのような仕様にしているのですが、今のところこのアプリには自分しかいないためプロフィールアイコンを確認することができません！一度右上のログアウトボタンからログアウトし、新しくアカウントをもう一つ作って確認してみましょう！

その際、まだ登録ページとログインページの登録・ログインアクション完了後のナビゲーション部分がまだコメントアウトされたままだったので、こちらのコメントアウトをはずしてあげましょう。`lib/pages/register_page.dart`と`lib/pages/login_page.dart`の中の`TODO: チャットページ実装後に下記コードを追加`という箇所を探してナビゲーション部分のコメントを外してください。

![プロフィールアイコンの表示](/images/flutter-supabase-chat/chat-page-with-profile.png)

いい感じに表示できていますね。続けてメッセージの時間を表示してみましょう。`timeago`というパッケージを使ってみます。`pubspec.yaml`に追加して`pub get`を実行しましょう。

```yaml:pubspec.yaml
timeago: ^3.1.0
```

`timeago`は`DateTime`型の値を渡すと自動的に現在時刻と比較して「1d」みたいにどれくらい前に投稿されたかのテキストを出してくれます。`_ChatBubble`の`build`メソッドの中で`timeago.format`を使って`DateTime`を文字列に変換して表示してみます。

ついでに細かな修正もしておきます。まず、メインのテキスト部分を`Flexible`で囲います。これによりテキストが長い場合は画面幅が許す限りマックスの横幅を取り、逆にメッセージが短い時はそれに応じて縮んでくれます。

```dart:lib/pages/chat_page.dart
...
/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
    required this.profile,
  }) : super(key: key);

  final Message message;
  final Profile? profile;

  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 18),
      child: Row(
        children: [
            if (!message.isMine)
                CircleAvatar(
                child: profile == null
                    ? preloader
                    : Text(profile!.username.substring(0, 2)),
                ),
            const SizedBox(width: 12),
            Flexible(
                child: Container(
                padding: const EdgeInsets.symmetric(
                    vertical: 8,
                    horizontal: 12,
                ),
                decoration: BoxDecoration(
                    color: message.isMine
                        ? Theme.of(context).primaryColor
                        : Colors.grey[300],
                    borderRadius: BorderRadius.circular(8),
                ),
                child: Text(message.content),
                ),
            ),
            const SizedBox(width: 12),
            Text(format(message.createdAt, locale: 'en_short')), // 時間を表示
            const SizedBox(width: 60),
        ],
      ),
    );
  }
}
```

![色をつけ時間を表示](/images/flutter-supabase-chat/chat-with-color.png)

最後に、自分のチャットと他のユーザーのチャットとで画面の右と左どちらに寄せるかを変えましょう。これが地味にトリッキー利で、ただ単に右よせ、左寄せをするだけでなく、各要素の左右の並び順を変えないと自然なレイアウトが作れなかったりします。そんな、場合わけで左右の並び順を変えるようなレイアウトも変数に一度各要素を入れて、`List`の`reversed`を使って逆順にすることで実現できます。

```dart:lib/pages/chat_page.dart
/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
    required this.profile,
  }) : super(key: key);

  final Message message;
  final Profile? profile;

  @override
  Widget build(BuildContext context) {
    List<Widget> chatContents = [
      if (!message.isMine)
        CircleAvatar(
          child: profile == null
              ? preloader
              : Text(profile!.username.substring(0, 2)),
        ),
      const SizedBox(width: 12),
      Flexible(
        child: Container(
          padding: const EdgeInsets.symmetric(
            vertical: 8,
            horizontal: 12,
          ),
          decoration: BoxDecoration(
            color: message.isMine
                ? Theme.of(context).primaryColor
                : Colors.grey[300],
            borderRadius: BorderRadius.circular(8),
          ),
          child: Text(message.content),
        ),
      ),
      const SizedBox(width: 12),
      Text(format(message.createdAt, locale: 'en_short')),
      const SizedBox(width: 60),
    ];
    if (message.isMine) {
      chatContents = chatContents.reversed.toList();
    }
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 18),
      child: Row(
        mainAxisAlignment:
            message.isMine ? MainAxisAlignment.end : MainAxisAlignment.start,
        children: chatContents,
      ),
    );
  }
}
```

![最終的なUI](/images/flutter-supabase-chat/natural-layout.png)

だいぶ自然なレイアウトになりましたね。

色々いじりましたが、最終的に`chat_page.dart`はこんな感じになります。

```dart:lib/pages/chat_page.dart
import 'dart:async';

import 'package:flutter/material.dart';

import 'package:my_chat_app/models/message.dart';
import 'package:my_chat_app/models/profile.dart';
import 'package:my_chat_app/pages/register_page.dart';
import 'package:my_chat_app/utils/constants.dart';
import 'package:supabase_flutter/supabase_flutter.dart';
import 'package:timeago/timeago.dart';

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

  /// プロフィール情報をメモリー内にキャッシュしておくための変数
  final Map<String, Profile> _profileCache = {};

  /// メッセージのサブスクリプション
  late final StreamSubscription<List<Message>> _messagesSubscription;

  @override
  void initState() {
    final myUserId = supabase.auth.currentUser!.id;
    _messagesStream = supabase
        .from('messages')
        .stream(primaryKey: ['id'])
        .order('created_at')
        .map((maps) => maps
            .map((map) => Message.fromMap(map: map, myUserId: myUserId))
            .toList());
    _messagesSubscription = _messagesStream.listen((messages) {
      for (final message in messages) {
        _loadProfileCache(message.profileId);
      }
    });
    super.initState();
  }

  @override
  void dispose() {
    // きちんとcancelしてメモリーリークを防ぐ
    _messagesSubscription.cancel();
    super.dispose();
  }

  /// 特定のユーザーのプロフィール情報をロードしてキャッシュする
  Future<void> _loadProfileCache(String profileId) async {
    if (_profileCache[profileId] != null) {
      return;
    }
    final data =
        await supabase.from('profiles').select().eq('id', profileId).single();
    final profile = Profile.fromMap(data);
    setState(() {
      _profileCache[profileId] = profile;
    });
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
                          reverse: true,
                          itemCount: messages.length,
                          itemBuilder: (context, index) {
                            final message = messages[index];
                            return _ChatBubble(
                              message: message,
                              profile: _profileCache[
                                  message.profileId], // キャッシュしたプロフィール情報を渡す
                            );
                          },
                        ),
                ),
                const _MessageBar(),
              ],
            );
          } else {
            return preloader;
          }
        },
      ),
    );
  }
}

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

/// チャットのメッセージを表示するためのウィジェット
class _ChatBubble extends StatelessWidget {
  const _ChatBubble({
    Key? key,
    required this.message,
    required this.profile,
  }) : super(key: key);

  final Message message;
  final Profile? profile;

  @override
  Widget build(BuildContext context) {
    List<Widget> chatContents = [
      if (!message.isMine)
        CircleAvatar(
          child: profile == null
              ? preloader
              : Text(profile!.username.substring(0, 2)),
        ),
      const SizedBox(width: 12),
      Flexible(
        child: Container(
          padding: const EdgeInsets.symmetric(
            vertical: 8,
            horizontal: 12,
          ),
          decoration: BoxDecoration(
            color: message.isMine
                ? Theme.of(context).primaryColor
                : Colors.grey[300],
            borderRadius: BorderRadius.circular(8),
          ),
          child: Text(message.content),
        ),
      ),
      const SizedBox(width: 12),
      Text(format(message.createdAt, locale: 'en_short')),
      const SizedBox(width: 60),
    ];
    if (message.isMine) {
      chatContents = chatContents.reversed.toList();
    }
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 18),
      child: Row(
        mainAxisAlignment:
            message.isMine ? MainAxisAlignment.end : MainAxisAlignment.start,
        children: chatContents,
      ),
    );
  }
}
```

## おまけ

Flutterの強みはThemeを使って簡単にアプリのテーマを変更できることです。例えば下記コードでオレンジを基調としたテーマに模様替えができます！テーマで遊ぶのも楽しいのでぜひ色々試してみてください！

```dart:lib/main.dart
...
return MaterialApp(
    debugShowCheckedModeBanner: false,
    title: 'チャットアプリ',
    theme: ThemeData.light().copyWith(
    primaryColorDark: Colors.orange,
    appBarTheme: const AppBarTheme(
        elevation: 1,
        backgroundColor: Colors.white,
        iconTheme: IconThemeData(color: Colors.black),
        titleTextStyle: TextStyle(
        color: Colors.black,
        fontSize: 18,
        ),
    ),
    primaryColor: Colors.orange,
    textButtonTheme: TextButtonThemeData(
        style: TextButton.styleFrom(
        foregroundColor: Colors.orange,
        ),
    ),
    elevatedButtonTheme: ElevatedButtonThemeData(
        style: ElevatedButton.styleFrom(
        foregroundColor: Colors.white,
        backgroundColor: Colors.orange,
        ),
    ),
    inputDecorationTheme: InputDecorationTheme(
        floatingLabelStyle: const TextStyle(
        color: Colors.orange,
        ),
        border: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: const BorderSide(
            color: Colors.grey,
            width: 2,
        ),
        ),
        focusColor: Colors.orange,
        focusedBorder: OutlineInputBorder(
        borderRadius: BorderRadius.circular(12),
        borderSide: const BorderSide(
            color: Colors.orange,
            width: 2,
        ),
        ),
    ),
    ),
    home: const SplashPage(),
);
```

![オレンジテーマのチャット](/images/flutter-supabase-chat/orange-chat.png)