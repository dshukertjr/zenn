---
title: "UI仕上げ"
---

今のままでは少しUIが寂しいのでもう少しチャットアプリっぽくしてあげましょう。まず最初にコードのリファクターがしやすいようにコードを整えます。

## チャットをウィジェットに切り抜き

メッセージを読み込んだ際に、そのメッセージの送信者情報を`profiles`テーブルから適宜ロードしてきています。その際、ロードされたメッセージはすぐにUI上に表示させ、後から遅れてロードされてくるプロフィール情報は、一旦くるくる回るローダーを表示させたのちにプロフィール情報がロードされ次第ユーザー名の最初の二文字を表示したプロフィール画像的なものを表示させている形になります。

まず、チャットページの`ListView`の中で色々と細々書くとコードが読みづらくなるので、チャットのメッセージを表示するためのウィジェットを切り出します。チャットページの一番下にこちらを追加しましょう。このウィジェットは、メッセージの本文と投稿者のプロフィール情報を受け取り、それらを元にチャットのメッセージを表示するしてくれます。こうして抜き出してあげることで、チャットページの`ListView`の方をいじらずにUIを改善していけます。

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
    return Text(message);
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
            profile: null, // 一旦nullにしておいて後でプロフィールを表示するようにする。
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
    return Text(message);
  }
}
```

`ChatPage`内で各メッセージの投稿者のプロフィールを取得するようにしましょう。`ChatPage`クラスの編集はこれが最後です！

`initState`の中でメッセージストリームに対してリスナーを張り、メッセージを取得した際にメッセージ一覧をループします。その中でまだメモリー内にキャッシュしていないユーザーがいない場合はSupabaseから取得してキャッシュします。

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
        _loadProfileCache(message.userId);
      }
    });
    super.initState();
  }

  @override
  void dispose() {
    // きちんとcancelしてメモリーリークを防ぐ
    _messagsSubscription.cancel();
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
      appBar: AppBar(title: const Text('チャット')),
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
                              profile: _profileCache[message.profileId], // キャッシュしたプロフィール情報を渡す
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

ここまできたらあとは`_ChatBubble`を編集してチャットの見た目を整えていくだけです！まずはプロフィールアイコンを表示してみましょう。メッセージのテキストを`Row`で囲い`CircleAvatar`でプロフィールアイコンを表示します。この時、自分のメッセージの場合はプロフィールアイコンを表示しないようにします。

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
    return Padding(
      padding: const EdgeInsets.symmetric(horizontal: 8, vertical: 18),
      child: Row(
        mainAxisAlignment:
            message.isMine ? MainAxisAlignment.end : MainAxisAlignment.start,
        children: [
            if (!message.isMine) // message.isMineがfalseの場合のみ表示
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

![プロフィールアイコンの表示]()

いい感じに表示できていますね。続けてメッセージの時間を表示してみましょう。`timeago`というパッケージを使ってみます。`pubspec.yaml`に追加して`pub get`を実行しましょう。

```yaml
timeago: ^3.1.0
```

timeagoは`DateTime`を`1分前`や`1時間前`のような文字列に変換してくれる便利なパッケージです。`_ChatBubble`の`build`メソッドの中で`timeago.format`を使って`DateTime`を文字列に変換して表示してみます。

ついでに細かな修正もしておきます。まず、メインのテキスト部分を`Flexible`で囲います。これによりテキストが長い場合は画面幅が許す限りマックスの横幅を取り、逆にメッセージが短い時はそれに応じて縮んでくれます。あとはテキストに自分のメッセージか相手のメッセージかで色を変えてみます。

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
        mainAxisAlignment:
            message.isMine ? MainAxisAlignment.end : MainAxisAlignment.start,
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

![色をつけ時間を表示]()

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

色々いじりましたが、最終的に`chat_page.dart`はこんな感じになります。

```dart:lib/pages/chat_page.dart
import 'dart:async';

import 'package:flutter/material.dart';

import 'package:my_chat_app/models/message.dart';
import 'package:my_chat_app/models/profile.dart';
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
    super.initState();
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
      appBar: AppBar(title: const Text('チャット')),
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

                            // 本当はbuildメソッド内で非同期処理を行うべきではないが、
                            // 今回は簡単のためにここでプロフィール情報をロードしている。
                            _loadProfileCache(message.profileId);

                            return _ChatBubble(
                              message: message,
                              profile: _profileCache[message.profileId],
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
  late final TextEditingController _textController;

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
                  maxLines: null,
                  autofocus: true,
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
  void initState() {
    _textController = TextEditingController();
    super.initState();
  }

  @override
  void dispose() {
    _textController.dispose();
    super.dispose();
  }

  void _submitMessage() async {
    final text = _textController.text;
    final myUserId = supabase.auth.currentUser!.id;
    if (text.isEmpty) {
      return;
    }
    _textController.clear();
    try {
      await supabase.from('messages').insert({
        'profile_id': myUserId,
        'content': text,
      });
    } on PostgrestException catch (error) {
      context.showErrorSnackBar(message: error.message);
    } catch (_) {
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

この状態でアプリを動かしてみると綺麗なチャットUIが完成していると思います。