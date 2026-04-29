---
title: Quickstart
description: city を作成し、rig を追加し、work をルーティングするまでを数分で。
---

<Note>
このガイドは Gas City とその前提条件をすでにインストール済みであることを前提としています。まだの場合は [Installation](/getting-started/installation) ページから始めてください。
</Note>

`gc`、`tmux`、`git`、`jq`、および beads プロバイダ（デフォルトでは `bd` + `dolt`、または `GC_BEADS=file` を設定してスキップ）が必要です。

<Tip>
Oh My Zsh の `git` プラグインは `git commit --verbose` を `gc` というエイリアスで定義しています。`gc version` や `gc init` を実行したときに Gas City ではなく git commit が開く場合は、一時的に `command gc ...` を使い、Oh My Zsh のロード後にエイリアスを削除してください。
[Troubleshooting](/getting-started/troubleshooting#oh-my-zsh-git-plugin-hides-gc) を参照してください。
</Tip>

## 1. city を作成する

```bash
gc init ~/bright-lights
cd ~/bright-lights
```

`gc init` は city ディレクトリをブートストラップし、supervisor に登録し、コントローラを起動します。init が完了した時点で city は実行中です。

## 2. rig を追加する

```bash
mkdir ~/hello-world && cd ~/hello-world && git init
gc rig add ~/hello-world
```

rig は city に登録された外部プロジェクトディレクトリです。独自の beads データベース、フックインストール、ルーティングコンテキストを持ちます。

## 3. work を sling する

```bash
cd ~/hello-world
gc sling claude "Create a script that prints hello world"
```

`gc sling` は work アイテム (bead) を作成し、エージェントにルーティングします。Gas City はセッションを開始してタスクを配信し、エージェントがそれを実行します。

## 4. エージェントの作業を見る

```bash
bd show <bead-id> --watch
```

同じパスをより詳しく辿るには [Tutorial 01](/tutorials/01-cities-and-rigs) に進んでください。
