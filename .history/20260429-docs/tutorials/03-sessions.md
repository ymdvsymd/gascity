---
title: チュートリアル 03 - Sessions
sidebarTitle: 03 - Sessions
description: エージェントの出力を見て、エージェントと直接対話し、polecat と crew について学びます。
---

[チュートリアル 02](/tutorials/02-agents) では、エージェントと協力して作業を生み出しました。それによってエージェントとのセッションが作成されましたが、まだ見ていません。このチュートリアルでは、セッションを通じてエージェントとの様子を見たり対話したりする方法、エージェント同士がどのように対話するかを見ていきます。また、「polecat」（作業を処理するためにオンデマンドで起動されるエージェント）と「crew」（名前付きセッションを持つ永続的なエージェント）の違いも学びます。

このチュートリアルを続けるには、前の 2 つのチュートリアルが終わった状態から始めます: city ルートには `pack.toml` と `city.toml` があり、チュートリアル 02 で `agents/reviewer/` の下に reviewer を追加してあります:

```shell
~/my-city
$ cat pack.toml
[pack]
name = "my-city"
schema = 2

[[agent]]
name = "mayor"
prompt_template = "agents/mayor/prompt.template.md"

[[named_session]]
template = "mayor"
mode = "always"

~/my-city
$ cat city.toml
[workspace]
provider = "claude"

[[rigs]]
name = "my-project"

~/my-city
$ cat agents/reviewer/agent.toml
dir = "my-project"
provider = "codex"
```

city のマシンローカルアイデンティティと rig のパスバインディングは、代わりに `.gc/site.toml` にあります:

```toml
workspace_name = "my-city"

[[rig]]
name = "my-project"
path = "/Users/csells/my-project"
```

reviewer のプロンプトは `agents/reviewer/prompt.template.md` にあります。これが標準的な city の形です: ルートの設定ファイルと、`agents/` 配下のエージェントごとのディレクトリです。

## Polecat を覗く

provider はそれぞれ — Claude、Codex、Gemini など — 会話の管理方法に独自のやり方を持っています。Gas City はそれらすべてを **session** という単一の抽象の背後に正規化します。session はそれ自身のターミナル、状態、会話履歴を持つライブプロセスです。

bead を sling すると、セッションが作成されます。一時的な polecat セッションを覗き見るのに最も簡単な方法は、ライブのセッション ID を調べて `gc session peek` に渡すことです:

```shell
~/my-project
$ gc session list --template my-project/reviewer
ID       TEMPLATE  STATE     REASON  TITLE     AGE  LAST ACTIVE
mc-8sfd  my-project/reviewer  creating  create  reviewer  1s   -

~/my-project
$ gc session peek mc-8sfd
› [my-project] reviewer • 2026-04-07T11:56:59

  Run `gc prime` to initialize your context.

  # Code Reviewer Agent
  You are an agent in a Gas City workspace. Check for available work and
  execute it.

  ## Your tools
  - `bd ready` — see available work items
  - `bd show <id>` — see details of a work item
  - `bd close <id>` — mark work as done

  ## How to work
  1. Check for available work: `bd ready`
  2. Pick a bead and execute the work described in its title
  3. When done, close it: `bd close <id>`
  4. Check for more work. Repeat until the queue is empty.

  ## Reviewing Code
  Read the code and provide feedback on bugs, security issues, and style.

... # 内容は省略

• Ran bd ready --json
  └ warning: beads.role not configured (GH#2950).
      Fix: git config beads.role maintainer
    … +282 lines
      }
    ]

• The ready queue has one task that is explicitly routed to the reviewer agent:
  mp-p956, “Review hello.py and write review.md with feedback.” I’m pulling the
  bead details and the target file now, then I’ll write the review and close
  that item.

• Explored
  └ List rg --files -g hello.py -g review.md

• Ran bd show mp-p956 --json
  └ [
      {
    … +27 lines
      }
    ]

• Working (43s • esc to interrupt)
```

reviewer エージェントに対する `gc prime` の結果が、`codex` CLI への最初の入力になっていることに気付くでしょう。これが GC が Codex に振る舞い方を伝える方法です。次に、Codex がそれらの指示に従って、自分が処理すべき準備のできた bead を探していることに気付くでしょう。1 つ見つけて実行し、`review.md` ファイルが出力されます。

エージェントに作業がなくなると、idle になります。slung された作業を処理するために作成されたセッションで idle になっていると、そのセッションは GC supervisor プロセスによってきれいにシャットダウンされます。これらの一時的なセッションは「polecat」として知られる「単発」のエージェントによってよく使われます。対話的に話すこともできますが、bead を実行し、idle になり、できるだけ早くセッションをシャットダウンするように設定されています。

対話するエージェントが欲しい場合は、「crew」メンバーと呼ばれる、対話用に設定されたエージェントを使うことになります。

## Crew とチャットする

reviewer エージェントのプロンプトは、自分に割り当てられた作業を探してすぐに実行するように作られていることを思い出してください。その作業がアクティブな間は、セッションのリストに表示されます:

```shell
~/my-project
$ gc session list
2026/04/07 21:50:21 tmux state cache: refreshed 2 sessions in 3.82725ms
ID       TEMPLATE  STATE     REASON          TITLE     AGE  LAST ACTIVE
mc-8sfd  my-project/reviewer  creating  create          reviewer  1s   -
mc-5o1   mayor     active    session,config  mayor     10h  14m ago
```

しかし、作業が終わると reviewer は idle になり、そのセッションは GC によってシャットダウンされます。一方、このサンプル出力からは、mayor が過去 10 時間 — city が起動して以来 — 動作しているのに、私たちは一度も話しかけていないことがわかります。ずっとトークンを消費しているのでしょうか？ 見てみましょう:

```shell
~/my-project
$ gc session peek mayor --lines 3

City is up and idle. No pending work, no agents running besides me. What would
  you like to do?
```

mayor は明らかに idle ですが、シャットダウンされていません。なぜでしょうか？ `pack.toml` ファイルをもう一度見ればわかります:

```toml
...
[[agent]]
name = "mayor"
prompt_template = "agents/mayor/prompt.template.md"

[[named_session]]
template = "mayor"
mode = "always"
...
```

mayor には "mayor" という特別な名前付きセッションがあり、常時稼働しています。チャットや計画など、すばやくアクセスできるようにシステムが起動状態を保ちます。polecat は一時的に設計されていますが、エージェントが（city 全体または rig 固有の）「crew」のメンバーであるならば、対話的にチャットしたり作業を受け取ったりするために常にそばにいます。

mayor（または実行中のセッションを持つ任意のエージェント）と話すには「アタッチ」します:

```shell
~/my-project
$ gc session attach mayor
2026/04/07 22:03:26 tmux state cache: refreshed 1 sessions in 3.828541ms
Attaching to session mc-5o1 (mayor)...
```

そうするとすぐに [tmux セッション](https://github.com/tmux/tmux/wiki/Getting-Started)に入ります:

![mayor セッションのスクリーンショット](mayor-session.png)

ライブの会話に参加しています。エージェントは他のチャットベースのコーディングアシスタントと同じように応答しますが、プロンプトテンプレートのフルコンテキストを持っています。

セッションを終了せずに detach するには、`Ctrl-b d` を押します（標準的な tmux detach）。セッションはバックグラウンドで実行され続けます。いつでも再アタッチできます。

attach せずに実行中のセッションと対話することもできます。peek の様子はすでに見ました。「nudge」もできます。これはセッションのターミナルに新しいメッセージを入力します:

```shell
~/my-city
$ gc session nudge mayor "What's the current city status?"
2026/04/07 22:07:28 tmux state cache: refreshed 2 sessions in 3.765375ms
```

Gas City は `Nudged mayor` または `Queued nudge for mayor` のいずれかで nudge を確認します。

![mayor nudge のスクリーンショット](mayor-nudge.png)

city で何が起きているかの感触をつかむために、すべての実行中セッションを見ることができます:

```shell
~/my-city
$ gc session list
ID      ALIAS  TEMPLATE  STATE
my-4    —      mayor     active
```

## セッションログ

Peek はターミナル出力の最後の数行を表示します。Logs は完全な会話履歴を表示します:

```shell
~/my-city
$ gc session logs mayor --tail 2
07:22:29 [USER] [my-city] mayor • 2026-04-08T00:22:24
Check the status of mc-wisp-8t8

07:22:31 [ASSISTANT] [my-city] mayor • 2026-04-08T00:22:31
mc-wisp-8t8 is a review request for the auth module. I've routed it to
my-project/reviewer.
```

`--tail N` は最後の N 件のトランスクリプトエントリを表示します（`tail -n` と同じ慣例）。上の `--tail 2` は、もっとも最近のユーザープロンプトと mayor の応答を表示します。コンパクト境界の区切りは、その最終ウィンドウ内に入る場合にエントリとしてカウントされます。会話全体を表示するには `--tail 0` を使います。互換性メモ: 1.0 以前は `--tail` はコンパクションセグメントをカウントしていました。1.0 では代わりに表示されるトランスクリプトエントリをカウントします。HTTP API の `tail` クエリパラメータは依然としてコンパクションセグメントをカウントします。`-f` でライブ出力を追跡します:

```shell
~/my-city
$ gc session logs mayor -f
```

別のターミナルで mayor を nudge し、follow ストリームが会話の進行を表示するのを観察します:

```shell
~/my-city
$ gc session nudge mayor "What's the current city status?"
```

繰り返しになりますが、Gas City は `Nudged mayor` または `Queued nudge for mayor` のいずれかで nudge を確認します。

attach して中断する可能性なしに、バックグラウンドエージェントが何をしているかを観察するのに便利です。Peek はターミナルを表示し、logs は新しいユーザーやアシスタントのメッセージが届いた際の会話を表示します。

## 次のステップ

slung された作業のためにセッションがオンデマンドで作成される様子、名前付きセッションが crew エージェントを生かし続ける様子、peek、attach、nudge、ログ読み取りの方法を見てきました。ここから先は:

- **[Agent-to-Agent Communication](/tutorials/04-communication)** — メール、slung 作業、フックを通じてエージェントが連携する方法
- **[Formulas](/tutorials/05-formulas)** — 依存関係と変数を持つ複数ステップのワークフローテンプレート
- **[Beads](/tutorials/06-beads)** — その下にある作業追跡システム
