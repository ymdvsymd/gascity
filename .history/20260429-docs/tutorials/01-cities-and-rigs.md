---
title: チュートリアル 01 - Cities と Rigs
sidebarTitle: 01 - Cities と Rigs
description: city を作成し、エージェントに作業を sling し、rig を追加し、複数のエージェントを設定します。
---

## セットアップ

まず、少なくとも 1 つの CLI コーディングエージェント（Gas City では「provider」と呼びます）をインストールし、PATH に通す必要があります。Gas City は Claude Code (`claude`)、Codex (`codex`)、Gemini (`gemini`) を含む多くの provider をサポートしています。選んだ provider それぞれに適切なトークンや API キーを設定し、それぞれが動作してあなたのために作業できるようにしてください（多ければ多いほどよいでしょう！）。

次に、Gas City の CLI をインストールして PATH に通す必要があります:

```shell
~
$ brew install gascity
...

~
$ gc version
0.13.4
```

> NOTE: gascity のインストールは適切な依存関係を整える優れた方法ですが、1.0 に向けて加えられている変更には追従しきれない場合があります。現時点でのベストプラクティスは、これらのチュートリアルを実行する前に、最新の bits を得るために [gascity リポジトリ](https://github.com/gastownhall/gascity) の `main` ブランチの HEAD から自分で `gc` バイナリをビルドすることです。

これで最初の city を作成する準備が整いました。

## city を作成する

city は、pack 定義、デプロイ設定、エージェントプロンプト、ワークフローを保持するディレクトリです。新しい city は `gc init` で作成します:

便利なメンタルモデルは次のとおりです:

- **city** は 1 つの Gas City 環境のための作業フォルダ全体です。エージェント、formula、rig、order、およびこのマシン上で Gas City にそれらをどう実行するか伝えるローカル設定をまとめたものです。
- **pack** はその city の再利用可能な部分です。可搬性があり、他の city や他の人と共有する価値のある Gas City の定義を保持します。

別の言い方をすれば、city とは pack にデプロイの詳細を加えたものです。

```shell

~
$ gc init ~/my-city
Welcome to Gas City SDK!

Choose a config template:
  1. minimal   — default coding agent (default)
  2. gastown   — multi-agent orchestration pack
  3. custom    — empty workspace, configure it yourself
Template [1]:

Choose your coding agent:
  1. Claude Code  (default)
  2. Codex CLI
  3. Gemini CLI
  4. Cursor Agent
  5. GitHub Copilot
  6. Sourcegraph AMP
  7. OpenCode
  8. Auggie CLI
  9. Pi Coding Agent
  10. Oh My Pi (OMP)
  11. Custom command
Agent [1]:
[1/8] Creating runtime scaffold
[2/8] Installing hooks (Claude Code)
[3/8] Scaffolding agent prompts
[4/8] Writing pack.toml
[5/8] Writing city configuration
Created minimal config (Level 1) in "my-city".
[6/8] Checking provider readiness
[7/8] Registering city with supervisor
Registered city 'my-city' (/Users/csells/my-city)
Installed launchd service: /Users/csells/Library/LaunchAgents/com.gascity.supervisor.plist
[8/8] Waiting for supervisor to start city
  Adopting sessions...
  Starting agents...

~
$ gc cities
NAME        PATH
my-city     /Users/csells/my-city
```

プロンプトを回避し、使いたい provider を直接指定することもできます。同じコマンドで provider を明示的に指定する例:

```shell
~
$ gc init ~/my-city --provider claude
```

Gas City は city ディレクトリを作成し、登録し、起動しました。`gc init` で作成された city には `pack.toml`、`city.toml` および標準のトップレベルディレクトリが含まれます。中身を見てみましょう:

```shell
~
$ cd ~/my-city

~/my-city
$ ls
agents  assets  city.toml  commands  doctor  formulas  orders  overlay  pack.toml  template-fragments
```

city ディレクトリのトップレベル:

- `pack.toml` — 可搬性のある pack 定義レイヤ
- `city.toml` — city ローカルのデプロイ・ランタイム設定

この city には組み込みの `mayor` エージェントが付属しています。mayor のプロンプトは `agents/mayor/prompt.template.md` にあり、`pack.toml` がそれを使用する常時稼働の mayor セッションを定義しています。デフォルトの `minimal` config テンプレートとデフォルトの provider を選んだ場合、`city.toml` は共有のランタイム設定を保持します:

```shell
~/my-city
$ cat city.toml
[workspace]
provider = "claude"
```

可搬性のある pack 定義はその隣にあります:

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
```

`city.toml` の `[workspace]` セクションは、provider などの共有ランタイムデフォルトを設定します。マシンローカルの workspace アイデンティティは代わりに `.gc/site.toml` にあり、これによって `gc cities`、`gc status`、その他のコマンドはこの city が `my-city` と名付けられていることを依然として認識します。

`pack.toml` の `[[agent]]` エントリは組み込みの `mayor` を定義し、`[[named_session]]` は `mayor` セッションを稼働させ続けるので、いつでも対話できます。後でエージェントを追加すると、Gas City は `agents/<name>/` を作成し、プロンプト用の `prompt.template.md` とエージェントごとのオーバーライド用の `agent.toml` を配置します。

Gas City はサポートされている各 provider に対する暗黙的なエージェントも提供します。そのため、`pack.toml` にリストされていなくても `claude`、`codex`、`gemini` がエージェント名として利用可能です。これらはカスタムプロンプトなしで provider のデフォルトを使用します。

city の状態を確認するには `gc status` を使います:

```shell
~/my-city
$ gc status
my-city  /Users/csells/my-city
  Controller: standalone-managed (PID 83621)
  Authority: standalone controller PID 83621
  Next: gc stop /Users/csells/my-city && gc start /Users/csells/my-city to hand ownership to the supervisor
  Suspended:  no

Agents:
  mayor                   pool (min=0, max=unlimited)
  claude                  pool (min=0, max=unlimited)

Sessions: 1 active, 0 suspended
```

## rig を追加する

Gas City では、city に登録されたプロジェクトディレクトリを「rig」と呼びます。プロジェクトのディレクトリを rig 化することで、エージェントがそこで作業できるようになります。

```shell
~/my-city
$ gc rig add ~/my-project
Adding rig 'my-project'...
  Prefix: mp
  Initialized beads database
  Generated routes.jsonl for cross-rig routing
Rig added.
```

Gas City はディレクトリのベース名（`my-project`）から rig 名を導出し、その中に作業追跡をセットアップしました。共有の rig 宣言は `city.toml` にあります:

```shell

~/my-city
$ cat city.toml
[workspace]
provider = "claude"

... # 内容は省略

[[rigs]]
name = "my-project"
```

マシンローカルの workspace アイデンティティとパスのバインディングは `.gc/site.toml` にあります:

```toml
workspace_name = "my-city"

[[rig]]
name = "my-project"
path = "/Users/csells/my-project"
```

city の rig は `gc rig list` でも確認できます:

```shell
~/my-project
$ gc rig list

Rigs in /Users/csells/my-city:

  my-city (HQ):
    Prefix: mc
    Beads:  initialized

  my-project:
    Path:   /Users/csells/my-project
    Prefix: mp
    Beads:  initialized
```

## 最初の作業を sling する

エージェントへの作業の割り当ては「sling」によって行います。やり方を知っている人にタスクを投げるイメージです。rig に対して作業を sling するには、rig ディレクトリの中から始めて、rig スコープのエージェントを明示的にターゲットにします:

```shell
~/my-city
$ cd ~/my-project

~/my-project
$ gc sling my-project/claude "Write hello world in python to the file hello.py"
Created mp-ff9 — "Write hello world in python to the file hello.py"
Attached wisp mp-6yh (default formula "mol-do-work") to mp-ff9
Auto-convoy mp-4tl
Slung mp-ff9 → my-project/claude
```

ターゲットが `my-project/claude` であるため、作業はこの rig にスコープされたままになります。

`gc sling` コマンドは city 内に作業項目（「bead」と呼ばれます）を作成し、`claude` エージェントにディスパッチしました。進行を観察できます:

```shell
~/my-city
$ gc bd show mp-ff9 --watch
✓ mp-ff9 · Write hello world in python to the file hello.py   [● P2 · CLOSED]
Owner: Chris Sells · Assignee: claude-mp-208 · Type: task
Created: 2026-04-07 · Updated: 2026-04-07

NOTES
Done: created hello.py

PARENT
  ↑ ○ mp-6yh: sling-mp-ff9 ● P2

Watching for changes... (Press Ctrl+C to exit)
```

bead が `CLOSED` に移ったら、結果を確認できます:

```shell
~/my-project
$ ls
hello.py
```

成功です！ AI エージェントに作業をディスパッチして、結果を受け取りました。

## 次のステップ

city を作成し、エージェントに作業を sling し、プロジェクトを rig として追加し、その rig に作業を sling しました。ここから先は:

- **[Agents](/tutorials/02-agents)** — エージェント設定をより深く理解する: プロンプト、セッション、スコープ、作業ディレクトリ
- **[Sessions](/tutorials/03-sessions)** — エージェントとの対話的な会話、polecat と crew
- **[Formulas](/tutorials/05-formulas)** — 依存関係と変数を持つ複数ステップのワークフローテンプレート
