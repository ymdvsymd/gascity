---
title: Tutorial 02 - Agents
sidebarTitle: 02 - Agents
description: エージェントを定義し、それを使って work を実行する。
---

[Tutorial 01](/tutorials/01-cities-and-rigs) では、city を作成し、暗黙のエージェントに work を sling し、rig を追加しました。暗黙のエージェント（`claude`、`codex` など）は便利ですが、カスタムプロンプトを持たない素のプロバイダにすぎません。このチュートリアルでは、特定のロールを持つ独自のエージェントを定義し、それを使って work を実行します。

Tutorial 01 の続きから始めます。`my-city` が動いていて `my-project` が rig 登録されているはずです。

## エージェントの定義

各カスタムエージェントは `agents/<name>/` 配下に独自のディレクトリを持ちます。まず rig スコープの reviewer を作成します:

```shell
~/my-city
$ gc agent add --name reviewer --dir my-project
Scaffolded agent 'reviewer'

~/my-city
$ cat > agents/reviewer/agent.toml << 'EOF'
dir = "my-project"
provider = "codex"
EOF
```

これで `agents/reviewer/prompt.template.md` が作成されます。エージェントごとのオーバーライドが欲しい場合は `agents/reviewer/agent.toml` を追加します。ここでは reviewer を `my-project` rig にスコープし、city のデフォルト `claude` プロバイダから `codex` に切り替えるために使っています。

新しいエージェント用のプロンプトを作成しましょう。プロンプトを指定しない場合のデフォルトの GC プロンプトを見てみます:

```shell
~/my-city
$ gc prime
# Gas City Agent

You are an agent in a Gas City workspace. Check for available work
and execute it.

## Your tools

- `bd ready` — see available work items
- `bd show <id>` — see details of a work item
- `bd close <id>` — mark work as done

## How to work

1. Check for available work: `bd ready`
2. Pick a bead and execute the work described in its title
3. When done, close it: `bd close <id>`
4. Check for more work. Repeat until the queue is empty.
```

`gc prime` コマンドは GC で動くエージェントに振る舞い方、特に自分に割り当てられた work の探し方を教えます。[tutorial 01](/tutorials/01-cities-and-rigs) ではエージェントに work を sling すると bead が作られることを学びました。ここでデフォルトプロンプトを見れば、エージェントが自分に投げられた work をどう拾うかが明確になるはずです。

ここでやりたいのは、GC のエージェントとして振る舞う指示を保ったまま、レビューエージェント特有の内容を追加することです。そのために、reviewer プロンプトを以下のように作ります:

```shell
~/my-city
$ cat > agents/reviewer/prompt.template.md << 'EOF'
# Code Reviewer Agent
You are an agent in a Gas City workspace. Check for available work and execute it.

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
EOF
$ gc prime my-project/reviewer
# Code Reviewer Agent
You are an agent in a Gas City workspace. Check for available work and execute it.
... # 上と同一のため省略
```

`gc prime <agent-name>` を使ってそのエージェント用のカスタムプロンプトの内容を取得していることに注意してください。組み込みエージェントや自分で作ったカスタムエージェントが時間と共に増えていくとき、それらがどのように設定されているかを確認するのに便利な方法です。

凝りたければ、モデルとパーミッションモードも設定できます:

```toml
dir = "my-project"
provider = "codex"
option_defaults = { model = "sonnet", permission_mode = "plan" }
```

このファイルは `agents/reviewer/agent.toml` に置きます。

エージェントが利用可能になったので、いよいよ work を sling します:

```shell
~/my-city
$ cd ~/my-project
~/my-project
$ gc sling my-project/reviewer "Review hello.py and write review.md with feedback"
Created mp-p956 — "Review hello.py and write review.md with feedback"
Auto-convoy mp-4wdl
Slung mp-p956 → my-project/reviewer
```

新しい reviewer エージェントは `my-project` rig にスコープされているので、そのディレクトリ内から `my-project/reviewer` として明示的にターゲットできます。Gas City は Codex セッションを開始し、`agents/reviewer/prompt.template.md` からプロンプトを読み込み、rig スコープの reviewer にタスクを配信しました。`bd show` で進捗を見られるのは既知の通りです。work が完了したら、要求したレビューがファイルシステムにあることを確認できます:

```shell
~/my-project
$ ls
hello.py  review.md

~/my-project
$ cat review.md
# Review
No findings.

`hello.py` is a single `print("Hello, World!")` statement and does not present a meaningful bug, security, or style issue in its current form.
```

これは fire-and-forget な作業に便利です。ただ、エージェントの動作を観察したり、直接やり取りしたい場合は session が必要になります。それは [次のチュートリアル](/tutorials/03-sessions) で見ていきます。

## 次は

カスタムプロンプトを持つエージェントを定義し、session 経由でやり取りし、異なるプロバイダで異なるエージェントを設定しました。ここから:

- **[Sessions](/tutorials/03-sessions)** — session のライフサイクル、sleep/wake、サスペンド、名前付き session
- **[Formulas](/tutorials/05-formulas)** — 依存関係と変数を持つマルチステップワークフローテンプレート
- **[Beads](/tutorials/06-beads)** — その下にある work トラッキングシステム
