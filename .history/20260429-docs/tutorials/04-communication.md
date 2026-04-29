---
title: Tutorial 04 - Agent-to-Agent Communication
sidebarTitle: 04 - Communication
description: エージェントが直接接続せずに、mail、slung work、フックを通じてどう協調するか。
---

[Tutorial 03](/tutorials/03-sessions) では、polecat session でエージェントの出力を覗き見たり、crew session にアタッチしたり、メッセージで nudge したりする方法を見ました。それらはすべて、あなたがエージェントに話しかけることでした。このチュートリアルでは、エージェントが _お互い_ に話す方法を扱います。

Tutorial 03 の続きから始めます。`my-city` が動いていて、`my-project` が rig 登録されており、`mayor` と `reviewer` のエージェントがあるはずです。

## エージェント同士の対話

ここまでは、session を一度に 1 つずつ管理してきました — polecat はオンデマンドで作成し、crew は名前付き session として生かし続ける。しかし、city は孤立して働く独立したエージェントの集合ではありません。互いに話し合えるエージェントのシステムです。

city 内のエージェントは互いを直接呼び出しません。エージェント間の関数呼び出しもなく、共有メモリも、直接参照もありません。各 session はそれ自身のプロセスで、独自のターミナル、独自の会話履歴、独自のプロバイダを持ちます。mayor は polecat へのハンドルを持っておらず、その逆もしかりです。

それでも、**mail** と **slung work** を通じてお互いに協調できます。どちらも間接的で、送信者はどの session がメッセージを受け取るか、どのインスタンスがタスクを拾うかを知る必要がありません。Gas City がルーティングを処理します。

この間接性は意図的です。エージェントが互いの参照を持たないため、独立に動いたり、アイドルになったり、再起動したり、スケールしたりできます。mayor は `my-project/reviewer` に作業をディスパッチするとき、その rig の reviewer session が 1 つなのか 5 つなのか、Claude なのか Codex なのか、現在アクティブなのかアイドルなのかを知る必要がありません。work とメッセージはストアに永続化されます。session は来ては去ります。

mail はエージェント同士が話す主要な方法です。slung work — `gc sling` — はタスクを委譲する方法です。両方を見てみましょう。

## Mail

mail は永続的でトラックされたメッセージを作成し、受信者は次のターンでそれを拾います。nudge（一時的なターミナル入力）と異なり、mail はクラッシュに耐え、件名を持ち、エージェントが処理するまで未読のまま残ります。mail 自体は受信者を起こしません。

mayor に mail を送ります:

```shell
~/my-city
$ gc mail send mayor -s "Review needed" -m "Please look at the auth module changes in my-project"
Sent message mc-wisp-8t8 to mayor
```

`gc mail send` は受信者を位置引数として取り、件名/本文を `-s`/`-m` フラグで取ります。（件名なしで `<to> <body>` だけを渡すこともできます。）

未読 mail を確認します:

```shell
~/my-city
$ gc mail check mayor
1 unread message(s) for mayor
```

inbox を見ます:

```shell
~/my-city
$ gc mail inbox mayor
ID           FROM   SUBJECT        BODY
mc-wisp-8t8  human  Review needed  Please look at the auth module changes in my-project
```

`gc mail inbox` はデフォルトで未読メッセージを表示するので、STATE 列はありません — リストされているものはすべて定義上未読です。

`peek` や `logs` で mayor がすぐ反応するのを見たい場合は、ターンを与えます:

```shell
~/my-city
$ gc session nudge mayor "Check mail and hook status, then act accordingly"
Nudged mayor
```

mayor は inbox を手動で確認する必要はありません。Gas City は未読 mail を自動的に表面化するプロバイダフックをインストールします — 各ターンで `gc mail check --inject` を実行するフックが走り、未読 mail があればエージェントのコンテキストにシステムリマインダーとして表示されます。エージェントは何もせずに mail を見られます。

その nudge は単独で mail を配信するわけではなく、mayor を起こして新しいターンを始めさせるだけです。mayor が起きたり新しいターンを始めたりすると、フックが保留中の mail を配信し、nudge はそれに対して行動するよう伝えます。

## エージェントの協調のために bead を sling する

実際の協調がどのように見えるかを示します。mayor がターンを取ると、あなたが送った mail メッセージを読みます。reviewer に処理させるべきだと判断し、work を sling します:

```shell
~/my-city
$ gc session peek mayor --lines 6
[mayor] Got mail: "Review needed" — auth module changes in my-project
[mayor] Routing to my-project/reviewer...
[mayor] Running: gc sling my-project/reviewer "Review the auth module changes"
```

（上は説明用です。`peek` は session の実際のターミナル内容を返すため、Gas City がフォーマットした行ではなく、エージェントがレンダリングしたものを目にします。）

mayor は reviewer に直接話したわけではありません。`my-project/reviewer` エージェントテンプレートに bead を sling し、Gas City がどの session がそれを拾うかを判断しました。reviewer が眠っていれば、Gas City が起こしました。その rig に複数の reviewer session があれば、Gas City が利用可能なものに work をルーティングしました。mayor はそのいずれも知らないし気にしません — work を記述して sling するだけです。

これがスケールするパターンです。人間が mayor に mail を送る。mayor がそれを読み、work を計画し、エージェントにタスクを sling する。それらのエージェントが work を行い、bead を閉じる。全員がストアを通じて通信し、直接接続では通信しません。session は来ては去り、work は永続化されます。

## フック

フックは、これらすべてを裏で動かす仕組みです。フックがなければ、session は素のプロバイダプロセスにすぎません — Claude がターミナルで動いているだけで、Gas City の認識はありません。フックはプロバイダのイベントシステムを Gas City に配線し、エージェントが mail を受け取り、slung work を拾い、キューに入った nudge を自動的にドレインできるようにします。

最小テンプレートはワークスペースレベルでフックを設定するため、すべてのエージェントにすでにフックが入っています:

```toml
[workspace]
install_agent_hooks = ["claude"]
```

エージェントごとに設定することもできます:

```toml
# agents/mayor/agent.toml
install_agent_hooks = ["claude"]
```

このようなエージェントローカルのオーバーライドは `agents/<name>/agent.toml` に置きます。

session が開始されると、Gas City はプロバイダが読むフック設定をインストールします。Claude では、新しい city が管理対象の `.gc/settings.json` 設定を書き出し、これが要所で Gas City コマンドを発火します — session 開始時、各ターンの前、シャットダウン時。これらのコマンドは mail を配信し、nudge をドレインし、保留中の work を表面化します。

フックがなければ、各エージェントに手動で `gc mail check` と `gc prime` を実行させる必要があります。フックがあれば、毎ターン自動で起こります。

## 次は

2 つの協調メカニズム — メッセージのための mail と work のための slung bead — および全部を結び付けるフックインフラを見ました。ここから:

- **[Formulas](/tutorials/05-formulas)** — 依存関係と変数を持つマルチステップワークフローテンプレート
- **[Beads](/tutorials/06-beads)** — その下にある work トラッキングシステム
