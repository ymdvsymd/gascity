# CLI リファレンス

> **自動生成** — 編集しないでください。再生成するには `go run ./cmd/genschema` を実行します。

## グローバルフラグ

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--city` | string |  | city ディレクトリへのパス（デフォルト: cwd から上方向に探索） |
| `--rig` | string |  | rig 名またはパス（デフォルト: cwd から検出） |

## gc

Gas City CLI — マルチエージェントワークフローのための orchestration-builder

```
gc [flags]
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc agent](#gc-agent) | agent 設定を管理 |
| [gc bd](#gc-bd) | 適切な rig ディレクトリで bd を実行 |
| [gc beads](#gc-beads) | beads プロバイダを管理 |
| [gc build-image](#gc-build-image) | 事前焼き込み済みエージェントコンテナイメージをビルド |
| [gc cities](#gc-cities) | 登録済み city を一覧表示 |
| [gc completion](#gc-completion) | 指定したシェル向けの自動補完スクリプトを生成 |
| [gc config](#gc-config) | city 設定を確認・検証 |
| [gc converge](#gc-converge) | 収束ループ（範囲限定の反復精錬）を管理 |
| [gc convoy](#gc-convoy) | convoy — 関連作業のグラフを管理 |
| [gc dashboard](#gc-dashboard) | supervisor および管理対象 city を監視する Web ダッシュボード |
| [gc doctor](#gc-doctor) | ワークスペースの健全性を確認 |
| [gc event](#gc-event) | イベント操作 |
| [gc events](#gc-events) | GC API からイベントを表示 |
| [gc formula](#gc-formula) | formula を管理・確認 |
| [gc graph](#gc-graph) | bead の依存関係グラフを表示 |
| [gc handoff](#gc-handoff) | handoff メールを送信し controller 管理セッションを再起動 |
| [gc help](#gc-help) | 任意のコマンドのヘルプ |
| [gc hook](#gc-hook) | 利用可能な作業をチェック（Stop hook 出力には --inject を使用） |
| [gc import](#gc-import) | pack インポートを管理 |
| [gc init](#gc-init) | 新しい city を初期化 |
| [gc mail](#gc-mail) | エージェント・人間の間でメッセージを送受信 |
| [gc mcp](#gc-mcp) | 投影された MCP 設定を確認 |
| [gc nudge](#gc-nudge) | 遅延 nudge を確認・配信 |
| [gc order](#gc-order) | order（スケジュール・イベント駆動の dispatch）を管理 |
| [gc pack](#gc-pack) | リモート pack ソースを管理 |
| [gc prime](#gc-prime) | エージェント向けの振る舞いプロンプトを出力 |
| [gc register](#gc-register) | city をマシン全体の supervisor に登録 |
| [gc reload](#gc-reload) | city/controller を再起動せずに現在の city 設定を再読み込み |
| [gc restart](#gc-restart) | city 内のすべての agent セッションを再起動 |
| [gc resume](#gc-resume) | 一時停止中の city を再開 |
| [gc rig](#gc-rig) | rig（プロジェクト）を管理 |
| [gc runtime](#gc-runtime) | プロセス内在のランタイム操作 |
| [gc service](#gc-service) | ワークスペースサービスを確認 |
| [gc session](#gc-session) | 対話型チャットセッションを管理 |
| [gc shell](#gc-shell) | Gas City のシェル統合フックを管理 |
| [gc skill](#gc-skill) | 可視のスキル一覧を表示 |
| [gc sling](#gc-sling) | 作業をセッション設定または agent にルーティング |
| [gc start](#gc-start) | マシン全体の supervisor 配下で city を起動 |
| [gc status](#gc-status) | city 全体のステータス概要を表示 |
| [gc stop](#gc-stop) | city 内のすべての agent セッションを停止 |
| [gc supervisor](#gc-supervisor) | マシン全体の supervisor を管理 |
| [gc suspend](#gc-suspend) | city を一時停止（すべての agent が実質的に停止） |
| [gc trace](#gc-trace) | session reconciler のトレースを確認・制御 |
| [gc unregister](#gc-unregister) | マシン全体の supervisor から city を削除 |
| [gc version](#gc-version) | gc のバージョンを表示 |
| [gc wait](#gc-wait) | 永続的なセッション wait を確認・管理 |

## gc agent

city.toml 内の agent 設定を管理します。

ランタイム操作（attach、list、peek、nudge、kill、start、stop、destroy）は
"gc session" および "gc runtime" に移動しました。

```
gc agent
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc agent add](#gc-agent-add) | agent スキャフォルドを追加 |
| [gc agent resume](#gc-agent-resume) | 一時停止中の agent を再開 |
| [gc agent suspend](#gc-agent-suspend) | agent を一時停止（reconciler はスキップ） |

## gc agent add

agents/&lt;name&gt;/ 配下に新しい agent スキャフォルドを追加します。

agents/&lt;name&gt;/prompt.template.md と、必要に応じて
agents/&lt;name&gt;/agent.toml を作成します。これらのファイルは city ディレクトリに置かれ、
city.toml に [[agent]] ブロックを追加することはありません。

--prompt-template を使うと既存ファイルから prompt の内容を
正規の prompt.template.md にコピーできます。--dir を使うと rig や
作業ディレクトリのプレフィックスを agent.toml に記録できます。--suspended を使うと
一時停止状態でスキャフォルドを作成できます。

```
gc agent add --name <name> [flags]
```

**例:**

```
gc agent add --name mayor
  gc agent add --name polecat --dir my-project
  gc agent add --name worker --prompt-template ./worker.md --suspended
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--dir` | string |  | agent の作業ディレクトリ（city ルートからの相対パス） |
| `--name` | string |  | agent の名前 |
| `--prompt-template` | string |  | prompt テンプレートファイルへのパス（city ルートからの相対パス） |
| `--suspended` | bool |  | agent を一時停止状態で登録 |

## gc agent resume

永続設定の suspended をクリアして一時停止中の agent を再開します。

reconciler は次のティックで agent を起動します。短縮名（rig コンテキストから解決）と
完全修飾名（例: "myrig/worker"）の両方をサポートします。

```
gc agent resume <name>
```

## gc agent suspend

永続設定で suspended=true を設定して agent を一時停止します。

一時停止中の agent は reconciler によりスキップされ、セッションは
起動・再起動されません。既存セッションは動き続けますが、終了しても
置き換えられません。"gc agent resume" で復元します。

```
gc agent suspend <name>
```

## gc bd

適切な rig ディレクトリにルーティングされた bd コマンドを実行します。

beads が（city ルートではなく）rig に属している場合、bd は正しい
.beads データベースを見つけるために rig ディレクトリから実行する必要があります。
このコマンドは --rig フラグまたは引数中の bead プレフィックスから
rig を自動解決します。

"gc bd" 以降の引数はそのまま bd に転送されます。

```
gc bd [bd-args...]
```

**例:**

```
gc bd --rig my-project list
  gc bd --rig my-project create "New task"
  gc bd show my-project-abc          # bead プレフィックスから rig を自動検出
  gc bd list --rig my-project -s open
```

## gc beads

beads プロバイダ（issue 追跡のバックエンドストア）を管理します。

トポロジ操作、ヘルスチェック、診断のためのサブコマンドを提供します。

```
gc beads
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc beads city](#gc-beads-city) | 正規の city エンドポイントトポロジを管理 |
| [gc beads health](#gc-beads-health) | beads プロバイダのヘルスチェック |

## gc beads city

bd ベースの beads ストアに対する正規 city エンドポイントトポロジを管理します。

use-managed で city を再び GC 管理に戻します。use-external で city を
外部 Dolt エンドポイントに固定し、継承する rig ミラーを書き換えます。

```
gc beads city
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc beads city use-external](#gc-beads-city-use-external) | city エンドポイントを外部 Dolt サーバに設定 |
| [gc beads city use-managed](#gc-beads-city-use-managed) | city エンドポイントを GC 管理に設定 |

## gc beads city use-external

city エンドポイントを外部 Dolt サーバに設定します

```
gc beads city use-external [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--adopt-unverified` | bool |  | ライブ検証なしでエンドポイントを記録 |
| `--dry-run` | bool |  | ファイルを書き込まずに正規の変更内容を表示 |
| `--host` | string |  | 外部 Dolt ホスト |
| `--port` | string |  | 外部 Dolt ポート |
| `--user` | string |  | 外部 Dolt ユーザー |

## gc beads city use-managed

city エンドポイントを GC 管理に設定します

```
gc beads city use-managed [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--dry-run` | bool |  | ファイルを書き込まずに正規の変更内容を表示 |

## gc beads health

beads プロバイダの健全性を確認し、失敗時にはリカバリを試みます。

プロバイダのライフサイクル health 操作に委譲します。exec
プロバイダ（bd/dolt を含む）では、スクリプトが多階層チェックと
リカバリを内部で処理します。file プロバイダでは常に成功（no-op）です。

定期監視のための beads-health システム order からも使用されます。

```
gc beads health [flags]
```

**例:**

```
gc beads health
  gc beads health --quiet
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--quiet` | bool |  | 成功時は無音、失敗時のみ stderr 出力 |

## gc build-image

city 設定、prompt、formula、rig コンテンツから Docker ビルドコンテキストを
組み立て、すべてが事前ステージされたコンテナイメージをビルドします。

事前焼き込みイメージを使用する Pod は init コンテナとファイルステージングをスキップし、
起動時間を 30〜60 秒から数秒に短縮します。[session.k8s] に prebaked = true
を設定します。

シークレット（Claude 認証情報）は焼き込まれません — 実行時には
K8s Secret ボリュームマウントとして提供されます。

```
gc build-image [city-path] [flags]
```

**例:**

```
# ビルドコンテキストのみ（docker build なし）
  gc build-image ~/bright-lights --context-only

  # イメージをビルドしてタグ付け
  gc build-image ~/bright-lights --tag my-city:latest

  # rig コンテンツを焼き込んでビルド
  gc build-image ~/bright-lights --tag my-city:latest --rig-path demo:/path/to/demo

  # ビルドしてレジストリに push
  gc build-image ~/bright-lights --tag registry.io/my-city:latest --push
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--base-image` | string | `gc-agent:latest` | ベース Docker イメージ |
| `--context-only` | bool |  | docker build を実行せずにビルドコンテキストを書き出す |
| `--push` | bool |  | ビルド後にイメージを push |
| `--rig-path` | stringSlice |  | rig 名:パスの組（繰り返し可） |
| `--tag` | string |  | イメージタグ（--context-only 以外では必須） |

## gc cities

マシン全体の supervisor に登録されたすべての city を一覧表示します。

```
gc cities
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc cities list](#gc-cities-list) | 登録済み city を一覧表示 |

## gc cities list

登録済み city を一覧表示します

```
gc cities list
```

## gc completion

指定したシェル向けの gc 自動補完スクリプトを生成します。
生成されたスクリプトの使い方の詳細は各サブコマンドのヘルプを参照してください。

```
gc completion
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc completion bash](#gc-completion-bash) | bash 向けの自動補完スクリプトを生成 |
| [gc completion fish](#gc-completion-fish) | fish 向けの自動補完スクリプトを生成 |
| [gc completion powershell](#gc-completion-powershell) | powershell 向けの自動補完スクリプトを生成 |
| [gc completion zsh](#gc-completion-zsh) | zsh 向けの自動補完スクリプトを生成 |

## gc completion bash

bash シェル向けの自動補完スクリプトを生成します。

このスクリプトは 'bash-completion' パッケージに依存します。
未インストールの場合は OS のパッケージマネージャでインストールしてください。

現在のシェルセッションに補完を読み込むには:

	source &lt;(gc completion bash)

すべての新しいセッションに補完を読み込むには、一度だけ実行します:

#### Linux:

	gc completion bash &gt; /etc/bash_completion.d/gc

#### macOS:

	gc completion bash &gt; $(brew --prefix)/etc/bash_completion.d/gc

設定を反映させるには新しいシェルを起動する必要があります。

```
gc completion bash
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--no-descriptions` | bool |  | 補完の説明を無効化 |

## gc completion fish

fish シェル向けの自動補完スクリプトを生成します。

現在のシェルセッションに補完を読み込むには:

	gc completion fish | source

すべての新しいセッションに補完を読み込むには、一度だけ実行します:

	gc completion fish &gt; ~/.config/fish/completions/gc.fish

設定を反映させるには新しいシェルを起動する必要があります。

```
gc completion fish [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--no-descriptions` | bool |  | 補完の説明を無効化 |

## gc completion powershell

powershell 向けの自動補完スクリプトを生成します。

現在のシェルセッションに補完を読み込むには:

	gc completion powershell | Out-String | Invoke-Expression

すべての新しいセッションに補完を読み込むには、上記コマンドの出力を
あなたの powershell プロファイルに追加してください。

```
gc completion powershell [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--no-descriptions` | bool |  | 補完の説明を無効化 |

## gc completion zsh

zsh シェル向けの自動補完スクリプトを生成します。

シェル補完が環境で有効化されていない場合は、有効化する必要があります。
以下を一度だけ実行してください:

	echo "autoload -U compinit; compinit" &gt;&gt; ~/.zshrc

現在のシェルセッションに補完を読み込むには:

	source &lt;(gc completion zsh)

すべての新しいセッションに補完を読み込むには、一度だけ実行します:

#### Linux:

	gc completion zsh &gt; "$&#123;fpath[1]&#125;/_gc"

#### macOS:

	gc completion zsh &gt; $(brew --prefix)/share/zsh/site-functions/_gc

設定を反映させるには新しいシェルを起動する必要があります。

```
gc completion zsh [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--no-descriptions` | bool |  | 補完の説明を無効化 |

## gc config

解決された city 設定を確認・検証・デバッグします。

設定システムは include、pack、patch、override による複数ファイル合成を
サポートしています。"show" で解決済みの設定をダンプし、"explain" で各値の
出所を確認できます。

```
gc config
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc config explain](#gc-config-explain) | 出所の注釈付きで解決済み設定を表示 |
| [gc config show](#gc-config-show) | 解決済みの city 設定を TOML としてダンプ |

## gc config explain

解決された設定を出所付きで表示します。

agent（デフォルト）の場合: 値を提供した設定ファイルの注釈と共に
解決された全フィールドを表示します。--rig と --agent でフィルタできます。

provider（--provider）の場合: 各フィールド・各 map キーの帰属
（builtin:X か providers.Y のどのチェーンレイヤーが値を提供したか）と共に
解決された ProviderSpec を表示します。base チェーン継承のデバッグに有用です。

--json で機械可読出力を得られます（provider のみ）。

```
gc config explain [flags]
```

**例:**

```
gc config explain
  gc config explain --agent mayor
  gc config explain --rig my-project
  gc config explain --provider codex-max
  gc config explain --provider codex-max --json
  gc config explain -f overlay.toml --agent polecat
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--agent` | string |  | 特定の agent 名にフィルタ |
| `-f`, `--file` | stringArray |  | レイヤする追加設定ファイル（繰り返し可） |
| `--json` | bool |  | JSON で出力（--provider が必要） |
| `--provider` | string |  | agent ではなく provider の解決済みチェーンを説明 |
| `--rig` | string |  | この rig 内の agent にフィルタ |

## gc config show

完全に解決された city 設定を TOML としてダンプします。

すべての include、pack、patch、override を含めて city.toml を読み込み、
マージ結果を出力します。--validate でエラーの有無を確認するだけで出力しません。
--provenance で各設定要素の出所を表示します。-f で追加の設定ファイルを
レイヤできます。

```
gc config show [flags]
```

**例:**

```
gc config show
  gc config show --validate
  gc config show --provenance
  gc config show -f overlay.toml
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-f`, `--file` | stringArray |  | レイヤする追加設定ファイル（繰り返し可） |
| `--provenance` | bool |  | 各設定要素の出所を表示 |
| `--validate` | bool |  | 設定を検証して終了（0 = 有効、1 = エラー） |

## gc converge

収束ループは範囲限定の多段階精錬サイクルです。

ルート bead + formula + ゲート = ゲートが通過するか最大反復回数に達するまで
繰り返します。controller は wisp_closed イベントを処理してループを自動的に
進めます。

```
gc converge
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc converge approve](#gc-converge-approve) | 収束ループを承認・終了（手動ゲート） |
| [gc converge create](#gc-converge-create) | 収束ループを作成 |
| [gc converge iterate](#gc-converge-iterate) | 次の反復を強制（手動ゲート） |
| [gc converge list](#gc-converge-list) | 収束ループを一覧表示 |
| [gc converge retry](#gc-converge-retry) | 終了した収束ループを再試行 |
| [gc converge status](#gc-converge-status) | 収束ループのステータスを表示 |
| [gc converge stop](#gc-converge-stop) | 収束ループを停止 |
| [gc converge test-gate](#gc-converge-test-gate) | ゲート条件をドライラン（状態変更なし） |

## gc converge approve

収束ループを承認・終了します（手動ゲート）

```
gc converge approve <bead-id>
```

## gc converge create

収束ループを作成します

```
gc converge create [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--evaluate-prompt` | string |  | カスタム evaluate プロンプト（formula のデフォルトを上書き） |
| `--formula` | string |  | 使用する formula（必須） |
| `--gate` | string | `manual` | ゲートモード: manual、condition、hybrid |
| `--gate-condition` | string |  | ゲート条件スクリプトへのパス |
| `--gate-timeout` | string | `5m0s` | ゲート実行タイムアウト |
| `--gate-timeout-action` | string | `iterate` | ゲートタイムアウト時の動作: iterate、retry、manual、terminate |
| `--max-iterations` | int | `5` | 最大反復回数 |
| `--target` | string |  | ターゲット agent（必須） |
| `--title` | string |  | 収束ループのタイトル |
| `--var` | stringArray |  | テンプレート変数（key=value、繰り返し可） |

## gc converge iterate

次の反復を強制します（手動ゲート）

```
gc converge iterate <bead-id>
```

## gc converge list

収束ループを一覧表示します

```
gc converge list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--all` | bool |  | 終了済み・終結したループも含める |
| `--json` | bool |  | JSON で出力 |
| `--state` | string |  | 状態でフィルタ（active、waiting_manual、terminated） |

## gc converge retry

終了した収束ループを再試行します

```
gc converge retry <bead-id> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--max-iterations` | int |  | 最大反復回数を上書き（デフォルト: 元から継承） |

## gc converge status

収束ループのステータスを表示します

```
gc converge status <bead-id> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--json` | bool |  | JSON で出力 |

## gc converge stop

収束ループを停止します

```
gc converge stop <bead-id>
```

## gc converge test-gate

ゲート条件をドライラン実行します（状態変更なし）

```
gc converge test-gate <bead-id>
```

## gc convoy

convoy — 関連する作業 bead のグラフを管理します。

convoy は依存関係を持つ bead の名前付きグラフです。シンプルな convoy は
親子関係で関連 issue をグループ化します。複雑な convoy は orchestration 用の
control bead を持つ formula コンパイル済み DAG を使用します。

```
gc convoy
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc convoy add](#gc-convoy-add) | issue を convoy に追加 |
| [gc convoy check](#gc-convoy-check) | すべての issue が閉じている convoy を自動クローズ |
| [gc convoy close](#gc-convoy-close) | convoy をクローズ |
| [gc convoy control](#gc-convoy-control) | control bead を実行、または control-dispatcher ループを実行 |
| [gc convoy create](#gc-convoy-create) | convoy を作成し、必要に応じて issue を追跡 |
| [gc convoy delete](#gc-convoy-delete) | convoy とそのすべての bead をクローズまたは削除 |
| [gc convoy delete-source](#gc-convoy-delete-source) | bead を起点とするワークフローをクローズ |
| [gc convoy land](#gc-convoy-land) | 所有 convoy をランディング（終結 + クリーンアップ） |
| [gc convoy list](#gc-convoy-list) | 進捗付きでオープンな convoy を一覧表示 |
| [gc convoy reopen-source](#gc-convoy-reopen-source) | ワークフロークリーンアップ後に source bead を再オープン |
| [gc convoy status](#gc-convoy-status) | convoy の詳細ステータスを表示 |
| [gc convoy stranded](#gc-convoy-stranded) | 作業可能だが worker のいない convoy を見つける |
| [gc convoy target](#gc-convoy-target) | convoy のターゲットブランチを設定 |

## gc convoy add

既存の issue bead を convoy に紐付けます。

issue の親を convoy ID に設定し、convoy の進捗追跡に表示されるようにします。

```
gc convoy add <convoy-id> <issue-id>
```

## gc convoy check

オープンな convoy をスキャンし、すべての子 issue が解決済みのものを自動クローズします。

各オープン convoy の子を評価します。すべての子のステータスが
"closed" であれば、convoy は自動的にクローズされ、イベントが記録されます。

```
gc convoy check
```

## gc convoy close

convoy bead を手動でクローズします。

子 issue のステータスに関わらず convoy を closed としてマークします。
すべての issue が解決された convoy を自動クローズするには
"gc convoy check" を使用します。

```
gc convoy close <id>
```

## gc convoy control

単一の control bead を処理するか、--serve で control-dispatcher ループを実行して
ready 状態の control bead を継続的に処理します。
--follow &lt;agent&gt; で serve ループを特定の agent テンプレートにフィルタできます。

```
gc convoy control [bead-id] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--follow` | string |  | serve ループを特定の agent テンプレートにフィルタして実行 |
| `--serve` | bool |  | control-dispatcher ループを実行（継続実行） |

## gc convoy create

convoy を作成し、必要に応じて既存の issue を紐付けます。

convoy bead を作成し、指定された issue ID の親を新しい convoy に設定します。
issue は後から "gc convoy add" で追加することもできます。

```
gc convoy create <name> [issue-ids...] [flags]
```

**例:**

```
gc convoy create sprint-42
  gc convoy create sprint-42 issue-1 issue-2 issue-3
  gc convoy create deploy --owner mayor --notify mayor --merge mr
  gc convoy create auth-rewrite --owned --target integration/auth-rewrite
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--merge` | string |  | マージ戦略: direct、mr、local |
| `--notify` | string |  | 完了時の通知先 |
| `--owned` | bool |  | convoy を所有としてマーク（手動ライフサイクル、自動クローズなし） |
| `--owner` | string |  | convoy のオーナー（管理者） |
| `--target` | string |  | 子の作業 bead が継承するターゲットブランチ |

## gc convoy delete

convoy 内のすべてのオープン bead をクローズするか、削除します。

すべてのストア（city + rig）から convoy ルートおよび gc.root_bead_id
が一致するすべての bead を検索します。--force なしではプレビューを表示します。

デフォルトでは bead は gc.outcome=skipped でクローズされます。--delete を使うと
bd delete --cascade --force でストアから削除します。

```
gc convoy delete <convoy-id> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--delete` | bool |  | クローズではなくストアから削除 |
| `-f`, `--force` | bool |  | 実際にクローズ・削除する（指定しないとプレビューのみ） |

## gc convoy delete-source

指定された bead を起点とするすべてのライブワークフロールートを見つけ、
そのサブツリーをクローズします。デフォルトはプレビューです。--apply で実際に変更します。
--apply と共に --delete を使うとクローズ後の bead も削除します。

```
gc convoy delete-source <source-bead-id> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--apply` | bool |  | 一致したワークフローを実際にクローズ・削除 |
| `--delete` | bool |  | クローズ後にストアから bead も削除 |
| `--rig` | string |  | source bead 用に rig ストアを選択 |
| `--store-ref` | string |  | source bead ストアを選択（city:&lt;name&gt; または rig:&lt;name&gt;） |

## gc convoy land

所有 convoy をランディングし、すべての子がクローズされていることを確認します。

ランディングは "gc sling --owned" で作成された所有 convoy の自然な
ライフサイクル終結です。すべての子が閉じている（または --force を使う）ことを検証し、
convoy bead をクローズし、ConvoyClosed イベントを記録します。

```
gc convoy land <convoy-id> [flags]
```

**例:**

```
gc convoy land gc-42
  gc convoy land gc-42 --force
  gc convoy land gc-42 --dry-run
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--dry-run` | bool |  | 何が起こるかをプレビュー |
| `--force` | bool |  | オープンな子があってもランディング |

## gc convoy list

完了進捗付きですべてのオープン convoy を一覧表示します。

各 convoy の ID、タイトル、クローズ済み/全子 issue 数を表示します。

```
gc convoy list
```

## gc convoy reopen-source

ワークフロークリーンアップ後に source bead を再オープンします

```
gc convoy reopen-source <source-bead-id> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--rig` | string |  | source bead 用に rig ストアを選択 |
| `--store-ref` | string |  | source bead ストアを選択（city:&lt;name&gt; または rig:&lt;name&gt;） |

## gc convoy status

convoy とそのすべての子 issue の詳細ステータスを表示します。

convoy の ID、タイトル、ステータス、完了進捗、すべての子 issue の
ステータスとアサインの一覧を表示します。

```
gc convoy status <id>
```

## gc convoy stranded

担当者のいない convoy 内のオープンな issue を検索します。

作業可能だがどの agent にも引き取られていない issue を一覧表示します。
convoy 処理のボトルネックを特定するのに有用です。

```
gc convoy stranded
```

## gc convoy target

convoy のターゲットブランチメタデータを設定します。

子の作業 bead は mol-polecat-work のような feature-branch formula で
sling される際にこのターゲットブランチを継承できます。

```
gc convoy target <convoy-id> <branch>
```

## gc dashboard

マシン全体の supervisor API に対して静的な GC ダッシュボードを開きます。

city がスコープにない場合、ダッシュボードは supervisor レベルの状態と
管理対象 city タブを表示します。city ディレクトリ内、または --city 指定時には
その city 専用のパネルとアクションフォームが有効になります。

```
gc dashboard [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--api` | string |  | GC API サーバ URL の上書き（デフォルトは自動検出） |
| `--port` | int | `8080` | HTTP ポート |

| サブコマンド | 説明 |
|------------|-------------|
| [gc dashboard serve](#gc-dashboard-serve) | Web ダッシュボードを起動 |

## gc dashboard serve

マシン全体の supervisor API に対して静的な GC ダッシュボードを起動します。

city がスコープにない場合、ダッシュボードは supervisor レベルの状態と
管理対象 city タブを表示します。city ディレクトリ内、または --city 指定時には
その city 専用のパネルとアクションフォームが有効になります。

```
gc dashboard serve [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--api` | string |  | GC API サーバ URL の上書き（デフォルトは自動検出） |
| `--port` | int | `8080` | HTTP ポート |

## gc doctor

city ワークスペースに対して診断ヘルスチェックを実行します。

city 構造、設定の妥当性、バイナリ依存（tmux、git、bd、dolt）、
controller 状態、agent セッション、ゾンビ・孤立セッション、
bead ストア、Dolt サーバの健全性、イベントログの整合性、
rig ごとの健全性をチェックします。--fix で自動修復を試みます。

```
gc doctor [flags]
```

**例:**

```
gc doctor
  gc doctor --fix
  gc doctor --verbose
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--fix` | bool |  | 自動修復を試みる |
| `-v`, `--verbose` | bool |  | 追加の診断詳細を表示 |

## gc event

イベント操作

```
gc event
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc event emit](#gc-event-emit) | city イベントログにイベントを送出 |

## gc event emit

カスタムイベントを city イベントログに記録します。

ベストエフォート: bead hook が失敗しないよう常に終了コード 0 を返します。
任意の JSON ペイロードを添付できます。

```
gc event emit <type> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--actor` | string |  | actor 名（デフォルト: $GC_ALIAS、なければ $GC_AGENT、なければ $GC_SESSION_ID、なければ "human"） |
| `--message` | string |  | イベントメッセージ |
| `--payload` | string |  | イベントに添付する JSON ペイロード |
| `--subject` | string |  | イベントサブジェクト（例: bead ID） |

## gc events

GC API からイベントを表示し、必要に応じてフィルタします。

API は city スコープと supervisor スコープの両方のイベントの真実のソースです。
city ディレクトリ内（または --city 指定時）はこのコマンドは city の
/v0/city/&#123;cityName&#125;/events と /stream エンドポイントを反映します。
city がスコープにない場合は supervisor の /v0/events と /stream エンドポイントを反映します。

list、watch、follow の出力は常に JSON Lines です。各行は 1 つの API
DTO または SSE エンベロープです。

```
gc events [flags]
```

**例:**

```
gc events
  gc events --type bead.created --since 1h
  gc events --watch --type convoy.closed --timeout 5m
  gc events --follow
  gc events --seq
  gc events --follow --after-cursor city-a:12,city-b:9
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--after` | uint64 |  | この city イベントシーケンス番号から再開（city スコープのみ） |
| `--after-cursor` | string |  | この supervisor イベントカーソルから再開（supervisor スコープのみ） |
| `--api` | string |  | GC API サーバ URL の上書き（デフォルトは自動検出） |
| `--follow` | bool |  | イベント到着に合わせて継続的にストリーム |
| `--payload-match` | stringArray |  | ペイロードフィールドでフィルタ（key=value、繰り返し可） |
| `--seq` | bool |  | 現在のヘッドカーソルを表示して終了 |
| `--since` | string |  | 指定期間前以降のイベントを表示（例: 1h、30m） |
| `--timeout` | string | `30s` | --watch の最大待機時間（例: 30s、5m） |
| `--type` | string |  | イベントタイプでフィルタ（例: bead.created） |
| `--watch` | bool |  | 一致するイベントが到着するまでブロック（最初のマッチまたはバッファ済み再生で終了） |

## gc formula

formula を管理・確認します

```
gc formula
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc formula cook](#gc-formula-cook) | formula を現在の bead ストアにインスタンス化 |
| [gc formula list](#gc-formula-list) | 利用可能な formula を一覧表示 |
| [gc formula show](#gc-formula-show) | コンパイル済みの formula レシピを表示 |

## gc formula cook

formula をコンパイルし、現在のストアに実際の bead としてインスタンス化します。

これは低レベルなワークフロー構築ツールです。formula ルートと
コンパイル済みの全ステップ bead を作成しますが、作業のルーティングは行いません。

--attach=&lt;bead-id&gt; を指定すると、サブ DAG が指定 bead の子として作成されます。
bead はサブ DAG ルートへのブロッキング依存を獲得し、サブ DAG が完了するまで
クローズされません。これは遅延バインド DAG 拡張のコアプリミティブで、
任意の agent、スクリプト、ワークフローステップが実行時に bead を
サブワークフローへ展開するために呼び出せます。

```
gc formula cook <formula-name> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--attach` | string |  | 既存の bead にサブ DAG をアタッチ（bead はサブ DAG ルートへのブロッキング依存を獲得） |
| `--meta` | stringArray |  | cook 後にルート bead のメタデータを設定（key=value、繰り返し可） |
| `-t`, `--title` | string |  | ルート bead タイトルを上書き |
| `--var` | stringArray |  | formula の変数置換（key=value、繰り返し可） |

## gc formula list

city の formula 検索パスで利用可能なすべての formula を一覧表示します。

formula は pack や formulas_dir 設定で構成された city レベルおよび
rig レベルの formula ディレクトリから検出されます。

```
gc formula list
```

## gc formula show

formula レシピをコンパイルして表示します。

デフォルトでは &#123;&#123;variable&#125;&#125; プレースホルダを残したまま
レシピを表示します。--var で変数を置換し、解決後の出力をプレビューできます。

例:
  gc formula show mol-feature
  gc formula show mol-feature --var title="Auth system" --var branch=main

```
gc formula show <formula-name> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--var` | stringArray |  | プレビュー用の変数置換（key=value） |

## gc graph

bead 群または convoy の依存関係グラフを表示します。

bead ストアを通じて依存関係を解決し、各 bead をステータスとブロッカーと共に
出力します。convoy は子に自動的に展開されます。Readiness は表示されたセット内で
計算されます。

デフォルトはテーブル出力です。--tree で Unicode ツリー、--mermaid で
Markdown に貼り付け可能な Mermaid.js フローチャートを出力します。

```
gc graph <bead-ids|convoy-id...> [flags]
```

**例:**

```
gc graph gc-42               # convoy 子を展開
  gc graph gc-1 gc-2 gc-3     # 任意の bead
  gc graph gc-42 --tree        # 依存関係ツリー
  gc graph gc-42 --mermaid     # Mermaid.js 図
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--mermaid` | bool |  | Mermaid.js フローチャートを出力 |
| `--tree` | bool |  | Unicode 依存関係ツリーを出力 |

## gc handoff

コンテキスト handoff のための便利コマンドです。

セルフ handoff（デフォルト）: 自分宛にメールを送ります。現在のセッションが
controller により再起動可能な場合、再起動を要求し controller がセッションを停止するまで
ブロックします。オンデマンドで構成された名前付きセッションでは、controller が
ユーザーアテンドプロセスを再起動できないため、メールを送って再起動要求なしで戻ります。

controller により再起動可能なセッションでは、以下と等価です:

  gc mail send $GC_ALIAS &lt;subject&gt; [message]
  gc runtime request-restart

リモート handoff（--target）: ターゲットセッションへメールを送ります。ターゲットが
controller により再起動可能であれば、reconciler が再起動できるよう kill して
handoff メールが待機する状態にします。オンデマンドで構成された名前付きターゲットでは、
セッションを kill せずにメールを送って戻ります。

controller により再起動可能なターゲットでは、以下と等価です:

  gc mail send &lt;target&gt; &lt;subject&gt; [message]
  gc session kill &lt;target&gt;

セルフ handoff にはセッションコンテキスト（GC_ALIAS または GC_SESSION_ID、加えて
GC_SESSION_NAME と city コンテキストの環境変数）が必要です。リモート handoff は
セッションのエイリアスまたは ID を受け付けます。

```
gc handoff <subject> [message] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--target` | string |  | handoff 先のリモートセッションのエイリアスまたは ID（controller により再起動可能なセッションのみ kill） |

## gc help

Help はアプリケーションの任意のコマンドのヘルプを提供します。
詳細は gc help [path to command] と入力してください。

```
gc help [command]
```

## gc hook

agent の work_query 設定を使って利用可能な作業をチェックします。

--inject なし: 生の出力を表示し、作業があれば終了コード 0、なければ 1。
--inject あり: hook 注入用に &lt;system-reminder&gt; で出力をラップし、常に 0 で終了。

		agent は $GC_AGENT または位置引数から決定されます。

```
gc hook [agent] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--hook-format` | string |  | provider 向けに hook 出力をフォーマット |
| `--inject` | bool |  | hook 注入用の &lt;system-reminder&gt; ブロックを出力 |

## gc import

pack インポートを管理します

```
gc import
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc import add](#gc-import-add) | pack インポートを追加 |
| [gc import check](#gc-import-check) | インストール済み pack インポートの状態を検証 |
| [gc import install](#gc-import-install) | pack.toml と packs.lock からインポートをインストール |
| [gc import list](#gc-import-list) | インポート済み pack を一覧表示 |
| [gc import migrate](#gc-import-migrate) | V1 city レイアウトを V2 pack 形式へ移行 |
| [gc import remove](#gc-import-remove) | pack インポートを削除 |
| [gc import upgrade](#gc-import-upgrade) | 制約内でインポート済み pack をアップグレード |
| [gc import why](#gc-import-why) | インポートが存在する理由を説明 |

## gc import add

pack インポートを追加します

```
gc import add <source> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--name` | string |  | ローカルバインディング名の上書き |
| `--version` | string |  | git ベースインポートのバージョン制約 |

## gc import check

インストール済み pack インポートの状態を検証します

```
gc import check
```

## gc import install

pack.toml と packs.lock からインポートをインストールします

```
gc import install
```

## gc import list

インポート済み pack を一覧表示します

```
gc import list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--tree` | bool |  | インポート依存関係ツリーを表示 |

## gc import migrate

レガシー city を V2 移行形式へ書き換えます。

workspace.includes を pack インポートへ移し、[[agent]] テーブルを
agents/&lt;name&gt;/ ディレクトリへ変換し、prompt/overlay/namepool
アセットを V2 配置にステージします。

```
gc import migrate [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--dry-run` | bool |  | 書き込みせずに変更内容を表示 |

## gc import remove

pack インポートを削除します

```
gc import remove <name>
```

## gc import upgrade

制約内でインポート済み pack をアップグレードします

```
gc import upgrade [name]
```

## gc import why

インポートが存在する理由を説明します

```
gc import why <name-or-source>
```

## gc init

指定ディレクトリ（または cwd）に新しい Gas City ワークスペースを作成します。

設定テンプレートとコーディング agent provider を選択する対話型ウィザードを実行します。
.gc/ ランタイムディレクトリ、pack.toml、city.toml、標準のトップレベルディレクトリ群、
.template.md prompt テンプレートを作成し、組み込み pack を
.gc/system/packs 配下にマテリアライズします。--provider でデフォルトの最小 city を
非対話的に作成、--file で既存の TOML 設定ファイルから初期化できます。

```
gc init [path] [flags]
```

**例:**

```
gc init
  gc init ~/my-city
  gc init --provider codex ~/my-city
  gc init --provider codex --bootstrap-profile k8s-cell /city
  gc init --name my-city
  gc init --from ~/elan --name elan /city
  gc init --file examples/gastown.toml ~/bright-lights
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--bootstrap-profile` | string |  | ホスト/コンテナ向けデフォルトに適用する bootstrap プロファイル |
| `--file` | string |  | city.toml として使用する TOML ファイルへのパス |
| `--from` | string |  | コピー元となるサンプル city ディレクトリへのパス |
| `--name` | string |  | ワークスペース名（デフォルト: ターゲットディレクトリの basename） |
| `--provider` | string |  | デフォルトの mayor 設定で使用する組み込みワークスペース provider |
| `--skip-provider-readiness` | bool |  | init 時の provider ログイン/レディネスチェックをスキップして起動を継続 |

## gc mail

agent と人間の間でメッセージを送受信します。

mail は type="message" の bead として実装されます。メッセージは
sender、recipient、subject、body を持ちます。"gc mail check --inject" を
agent hook で使うと、mail 通知を agent prompt に配信できます。

```
gc mail
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc mail archive](#gc-mail-archive) | メッセージを読まずにアーカイブ |
| [gc mail check](#gc-mail-check) | 未読メールをチェック（hook 出力には --inject を使用） |
| [gc mail count](#gc-mail-count) | 全件・未読件数を表示 |
| [gc mail delete](#gc-mail-delete) | メッセージを削除（bead をクローズ） |
| [gc mail inbox](#gc-mail-inbox) | 未読メッセージを一覧表示（デフォルトは自分の inbox） |
| [gc mail mark-read](#gc-mail-mark-read) | メッセージを既読にする |
| [gc mail mark-unread](#gc-mail-mark-unread) | メッセージを未読にする |
| [gc mail peek](#gc-mail-peek) | 既読にせずにメッセージを表示 |
| [gc mail read](#gc-mail-read) | メッセージを読み既読にする |
| [gc mail reply](#gc-mail-reply) | メッセージに返信 |
| [gc mail send](#gc-mail-send) | セッションエイリアスまたは人間にメッセージを送信 |
| [gc mail thread](#gc-mail-thread) | スレッド内のすべてのメッセージを一覧表示 |

## gc mail archive

メッセージ bead を内容を表示せずにクローズします。

メッセージを読まずに却下するために使います。メッセージは閉じられたとして
マークされ、mail check や inbox の結果に表示されなくなります。

```
gc mail archive <id>
```

## gc mail check

セッションエイリアスまたはメールボックス宛の未読メールを確認します。

--inject なし: 件数を表示し、メールがあれば終了コード 0、なければ 1。
--inject あり: hook 注入に適した &lt;system-reminder&gt; ブロックを出力します（常に 0 で終了）。
受信者は $GC_SESSION_ID、$GC_ALIAS、$GC_AGENT、または "human" がデフォルトです。

```
gc mail check [session] [flags]
```

**例:**

```
gc mail check
  gc mail check --inject
  gc mail check mayor
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--hook-format` | string |  | provider 向けに hook 出力をフォーマット |
| `--inject` | bool |  | hook 注入用の &lt;system-reminder&gt; ブロックを出力 |

## gc mail count

セッションエイリアスまたは人間の全件・未読メッセージ件数を表示します。
受信者は $GC_SESSION_ID、$GC_ALIAS、$GC_AGENT、または "human" がデフォルトです。

```
gc mail count [session]
```

## gc mail delete

bead をクローズしてメッセージを削除します。archive と同じ効果ですがユーザー意図が異なります。

```
gc mail delete <id>
```

## gc mail inbox

セッションエイリアスまたは人間のすべての未読メッセージを一覧表示します。

メッセージ ID、送信者、件名、本文をテーブルで表示します。受信者は
$GC_SESSION_ID、$GC_ALIAS、$GC_AGENT、または "human" がデフォルトです。
他のセッションの inbox を見るにはセッションエイリアスを渡します。

```
gc mail inbox [session]
```

## gc mail mark-read

表示せずにメッセージを既読にします。inbox の結果に表示されなくなります。

```
gc mail mark-read <id>
```

## gc mail mark-unread

メッセージを未読にします。inbox の結果に再び表示されます。

```
gc mail mark-unread <id>
```

## gc mail peek

メッセージを既読にせずに表示します。

"gc mail read" と同じ出力ですが、メッセージの既読状態は変わりません。
inbox の結果には引き続き表示されます。

```
gc mail peek <id>
```

## gc mail read

メッセージを表示し既読にします。

メッセージの全詳細（ID、送信者、受信者、件名、日付、本文）を表示します。
メッセージはストアに残ります — 完全にクローズするには "gc mail archive" を使います。

```
gc mail read <id>
```

## gc mail reply

メッセージに返信します。返信は元の送信者に宛てられます。

会話追跡のために元メッセージのスレッド ID を継承します。
返信後に受信者に nudge を送るには --notify を使います。
返信件名には -s/--subject、本文には -m/--message を使います。

```
gc mail reply <id> [-s subject] [-m body] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-m`, `--message` | string |  | 返信本文 |
| `--notify` | bool |  | 返信後に受信者に nudge |
| `-s`, `--subject` | string |  | 返信件名 |

## gc mail send

セッションエイリアスまたは人間にメッセージを送信します。

受信者宛のメッセージ bead を作成します。送信者は
$GC_SESSION_ID、$GC_ALIAS、$GC_AGENT、または "human" がデフォルトです。
送信後に受信者に nudge するには --notify、送信者識別を上書きするには --from を使います。
位置引数 &lt;to&gt; の代わりに --to を使うこともできます。
要約行には -s/--subject、本文には -m/--message を使います。
すべてのライブセッションへ（送信者と "human" を除いて）ブロードキャストするには --all を使います。

```
gc mail send [<to>] [<body>] [flags]
```

**例:**

```
gc mail send mayor "Build is green"
  gc mail send mayor -s "Build is green"
  gc mail send myrig/witness -s "Need investigation" -m "Attach logs from the last failed run"
  gc mail send --to mayor "Build is green"
  gc mail send human "Review needed for PR #42"
  gc mail send polecat "Priority task" --notify
  gc mail send --all "Status update: tests passing"
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--all` | bool |  | すべてのライブセッションへブロードキャスト（送信者と human を除く） |
| `--from` | string |  | 送信者識別（デフォルト: $GC_SESSION_ID、$GC_ALIAS、$GC_AGENT、または "human"） |
| `-m`, `--message` | string |  | メッセージ本文 |
| `--notify` | bool |  | 送信後に受信者に nudge |
| `-s`, `--subject` | string |  | メッセージ件名 |
| `--to` | string |  | 受信者アドレス（位置引数の代替） |

## gc mail thread

同じスレッド ID を共有するすべてのメッセージを時系列順に表示します。

```
gc mail thread <thread-id>
```

## gc mcp

具体的なターゲットに対する投影済み MCP カタログを確認します。

投影された MCP はターゲット固有です。agent が設定から決定論的に
1 つの投影ターゲットを持つ場合は "gc mcp list --agent &lt;name&gt;" を、
ライブセッションのターゲットには "gc mcp list --session &lt;id&gt;" を使います。

```
gc mcp
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc mcp list](#gc-mcp-list) | 投影済み MCP サーバを表示 |

## gc mcp list

ある agent またはセッションターゲットに対し、Gas City が provider ネイティブ設定へ投影する
優先順位解決済みの MCP サーバを表示します。

```
gc mcp list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--agent` | string |  | この agent の投影済み MCP 設定を表示 |
| `--session` | string |  | このセッションの投影済み MCP 設定を表示 |

## gc nudge

遅延 nudge を確認・配信します。

遅延 nudge は、ターゲット agent が休眠中、または安全な対話境界に
まだ到達していない理由でキューイングされたリマインダーです。

```
gc nudge
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc nudge status](#gc-nudge-status) | セッションのキュー済み・dead-letter nudge を表示 |

## gc nudge status

セッションのキュー済み・dead-letter nudge を表示します。

セッション内で実行された場合は $GC_ALIAS または $GC_SESSION_ID がデフォルトになります。

```
gc nudge status [session]
```

## gc order

order — formula およびスクリプトのスケジュール・イベント駆動 dispatch を管理します。

order はフラットな orders/&lt;name&gt;.toml ファイルにあります。各 order はトリガー条件
（cooldown、cron、condition、event、または manual）とアクション
（formula または exec スクリプト）を組み合わせます。controller は各ティックで
トリガーを評価し、トリガーが開いたときに作業を dispatch します。

```
gc order
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc order check](#gc-order-check) | 実行予定の order を確認 |
| [gc order history](#gc-order-history) | order の実行履歴を表示 |
| [gc order list](#gc-order-list) | 利用可能な order を一覧表示 |
| [gc order run](#gc-order-run) | order を手動実行 |
| [gc order show](#gc-order-show) | order の詳細を表示 |

## gc order check

すべての order のトリガー条件を評価し、実行対象を表示します。

各 order のトリガー、due 状態、理由を含むテーブルを表示します。
いずれかの order が due であれば終了コード 0、なければ 1 を返します。

```
gc order check
```

## gc order history

order の実行履歴を表示します。

過去の order 実行を bead 履歴から照会します。order 名でフィルタ可能、
--rig で rig フィルタができます。

```
gc order history [name] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--rig` | string |  | order 履歴をフィルタする rig 名 |

## gc order list

利用可能なすべての order をトリガータイプ、スケジュール、ターゲットと共に一覧表示します。

トリガー条件、スケジュールパラメータ、ターゲットプールを定義する
フラットな .toml ファイルを orders/ ディレクトリからスキャンします。

```
gc order list
```

## gc order run

トリガー条件をバイパスして order を手動実行します。

order の formula から wisp をインスタンス化し、構成済みターゲット（あれば）に
ルーティングします。order のテストや通常スケジュール外でのトリガーに有用です。
異なる rig 内の同名 order を識別するには --rig を使います。

```
gc order run <name> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--rig` | string |  | 同名 order を識別する rig 名 |

## gc order show

名前付き order の詳細情報を表示します。

order 名、説明、formula 参照、トリガータイプ、スケジュールパラメータ、
check コマンド、ターゲット、ソースファイルを表示します。
異なる rig 内の同名 order を識別するには --rig を使います。

```
gc order show <name> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--rig` | string |  | 同名 order を識別する rig 名 |

## gc pack

agent 設定を提供するリモート pack ソースを管理します。

pack は rig 用の agent 設定を定義する pack.toml ファイルを含む
git リポジトリです。ローカルにキャッシュされ、特定の git ref に固定できます。

```
gc pack
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc pack fetch](#gc-pack-fetch) | 不足分をクローンし既存リモート pack を更新 |
| [gc pack list](#gc-pack-list) | リモート pack ソースとキャッシュ状態を表示 |

## gc pack fetch

不足しているリモート pack キャッシュをクローンし、既存のものを更新します。

すべての構成済み pack ソースを git リポジトリから取得し、
ローカルキャッシュを更新し、再現性のためコミットハッシュ付きの
ロックファイルを書き出します。"gc start" 中に自動的に呼び出されます。

```
gc pack fetch
```

## gc pack list

構成済みの pack ソースをキャッシュ状態と共に表示します。

各 pack の名前、ソース URL、git ref、キャッシュ状態、固定済みコミットハッシュ
（あれば）を表示します。

```
gc pack list
```

## gc prime

agent の振る舞いプロンプトを出力します。

任意の CLI コーディング agent を city 対応の指示で prime するために使います:
  claude "$(gc prime mayor)"
  codex --prompt "$(gc prime worker)"

ランタイム hook プロファイルは `gc prime --hook` を呼び出すことがあります。
agent-name が省略されると `GC_ALIAS` が使われます（フォールバックとして `GC_AGENT`）。

agent-name が prompt_template を持つ構成済み agent と一致する場合、
そのテンプレートが出力されます。それ以外の場合はデフォルトの worker prompt が出力されます。

デバッグ上のミスでデフォルト prompt に静かにフォールバックする代わりに失敗させるには
--strict を渡します。Strict は以下の場合にエラーを出します:

  - city 設定が見つからない
  - city 設定の読み込みに失敗
  - agent 名が指定されていない（引数、GC_ALIAS、GC_AGENT のいずれからも）
  - agent 名が city 設定にない（タイポ検出 — 主な用途）
  - agent の prompt_template が読めないファイルを指している

Strict は意図的に prompt_template を欠いた agent（サポートされた最小構成）、
有効な条件ロジックの結果として空出力にレンダリングされるテンプレート、
または一時停止状態（city または agent）ではエラーを出しません — それらは
ミスではなく正当な静止状態です。

```
gc prime [agent-name] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--hook` | bool |  | ランタイム hook 呼び出し向けの互換モード |
| `--hook-format` | string |  | provider 向けに hook 出力をフォーマット |
| `--strict` | bool |  | city 不在、agent 不在/不明、prompt_template 読み込み不能の場合にデフォルト prompt にフォールバックせず失敗させる |

## gc register

city ディレクトリをマシン全体の supervisor に登録します。

パスを指定しない場合、現在の city（cwd から検出）を登録します。
--name でマシンローカルの登録エイリアスを設定します。エイリアスは
マシンローカルの supervisor レジストリに保存され、city.toml には書き戻されません。
--name を省略すると、現在有効な city 識別が使用されます
（site バインドされたワークスペース名があればそれ、なければ legacy workspace.name、
それもなければディレクトリ basename）— いずれの場合も city.toml は変更されません。
登録は冪等です — 同じ city を 2 回登録しても no-op です。
必要に応じて supervisor が起動され、city が直ちに調停されます。

```
gc register [path] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--name` | string |  | この city 登録のためのマシンローカルエイリアス |

## gc reload

city/controller を再起動せずに、現在の city controller に有効な設定を再読み込みさせ、
1 回の reload ティックを処理させます。

reload は有効な設定の再計算前に構成済みリモート pack を取得することがあります。
通常の設定ドリフトルールが要求すれば、既存セッションごとの再起動が起こることもあります。

```
gc reload [path] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--async` | bool |  | controller が reload 要求を受理した時点で復帰 |
| `--timeout` | string | `5m` | reload 完了を待つ時間 |

## gc restart

city を停止してから再起動します。

"gc stop" の後 "gc start" を実行するのと同等です。supervisor モードでは
city の登録解除、再登録、即時調停をトリガーします。

```
gc restart [path]
```

## gc resume

city.toml の workspace.suspended をクリアして一時停止中の city を再開します。

通常運用を復元します: reconciler は再び agent を起動し、gc hook/prime は
作業を返します。個別の agent を再開するには "gc agent resume"、rig には
"gc rig resume" を使います。

```
gc resume [path]
```

## gc rig

city に登録された rig（外部プロジェクトディレクトリ）を管理します。

rig は city が orchestration するプロジェクトディレクトリです。各 rig は
独自の beads データベース、agent hook、rig 横断ルーティングを持ちます。
agent は "dir" フィールドで rig にスコープされます。

```
gc rig
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc rig add](#gc-rig-add) | プロジェクトを rig として登録 |
| [gc rig list](#gc-rig-list) | 登録済み rig を一覧表示 |
| [gc rig remove](#gc-rig-remove) | rig を city から削除 |
| [gc rig restart](#gc-rig-restart) | rig 内のすべての agent を再起動 |
| [gc rig resume](#gc-rig-resume) | 一時停止中の rig を再開 |
| [gc rig set-endpoint](#gc-rig-set-endpoint) | rig の正規エンドポイント所有を設定 |
| [gc rig status](#gc-rig-status) | rig のステータスと agent 起動状態を表示 |
| [gc rig suspend](#gc-rig-suspend) | rig を一時停止（reconciler は agent をスキップ） |

## gc rig add

外部プロジェクトディレクトリを rig として登録します。

beads データベースを初期化し、構成されていれば agent hook をインストールし、
rig 横断ルートを生成し、rig を city.toml に追記します。
ターゲットディレクトリが存在しない場合は作成されます。rig の agent 設定を定義する
pack ディレクトリを適用するには --include を使います。1 つの rig に複数 pack を
合成するにはフラグを繰り返します。

rig 名を明示するには --name（デフォルト: ディレクトリ basename）を使います。
bead ID プレフィックスを明示するには --prefix（デフォルト: 名前から派生）を使います。
一時停止状態（デフォルト休眠）で rig を追加するには --start-suspended を使います。
rig の agent は "gc rig resume" で明示的に再開されるまで起動しません。

完全に初期化された .beads/ ディレクトリ（metadata.json と config.yaml の両方を含む必要あり）を
すでに持つディレクトリを登録するには --adopt を使います。
beads init をスキップしますが、git リポジトリチェックは情報提供として残ります。

```
gc rig add <path> [flags]
```

**例:**

```
gc rig add /path/to/project
  gc rig add /path/to/project --name myrig
  gc rig add /path/to/project --prefix r1
  gc rig add ./my-project --include packs/gastown
  gc rig add ./my-project --include packs/planner --include packs/architect
  gc rig add ./my-project --include packs/gastown --start-suspended
  gc rig add /path/to/existing --adopt
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--adopt` | bool |  | 既存の .beads/ ディレクトリを採用（init をスキップ） |
| `--include` | stringArray |  | rig agent 用の pack ディレクトリ（繰り返し可） |
| `--name` | string |  | rig 名（デフォルト: ディレクトリ basename） |
| `--prefix` | string |  | bead ID プレフィックス（デフォルト: 名前から派生） |
| `--start-suspended` | bool |  | 一時停止状態（デフォルト休眠）で rig を追加 |

## gc rig list

すべての登録済み rig をパス、プレフィックス、beads ステータスと共に一覧表示します。

HQ rig（city 自身）とすべての構成済み rig を表示します。各 rig は
bead ID プレフィックスと beads データベース初期化状態を表示します。

```
gc rig list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--json` | bool |  | JSON 形式で出力 |

## gc rig remove

現在の city 設定から rig を削除します。

city.toml から rig エントリを削除し、.gc/site.toml からマシンローカルの
パスバインディングも削除します。

```
gc rig remove <name>
```

**例:**

```
gc rig remove myrig
```

## gc rig restart

rig に属するすべての agent セッションを kill します。

reconciler は次のティックで agent を再起動します。特定プロジェクトで作業中の
全 agent を強制リフレッシュする手早い方法です。

```
gc rig restart [name]
```

## gc rig resume

city.toml の suspended をクリアして一時停止中の rig を再開します。

reconciler は次のティックで rig の agent を起動します。

```
gc rig resume [name]
```

## gc rig set-endpoint

rig の正規エンドポイント所有を設定します。

現在の city トポロジから rig がエンドポイントを派生させるには --inherit、
独自の外部 Dolt エンドポイントに rig を固定するには --external を使います。

このコマンドは rig の正規 .beads/config.yaml トポロジ状態を所有します。

```
gc rig set-endpoint <rig> [flags]
```

**例:**

```
gc rig set-endpoint frontend --inherit
  gc rig set-endpoint frontend --external --host db.example.com --port 3307
  gc rig set-endpoint frontend --external --host db.example.com --port 3307 --user agent --adopt-unverified
  gc rig set-endpoint frontend --inherit --dry-run
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--adopt-unverified` | bool |  | ライブ検証なしでエンドポイントを記録 |
| `--dry-run` | bool |  | ファイルを書き込まずに正規の変更内容を表示 |
| `--external` | bool |  | rig 用の明示的な外部エンドポイントを設定 |
| `--host` | string |  | 外部 Dolt ホスト |
| `--inherit` | bool |  | city エンドポイントを継承 |
| `--port` | string |  | 外部 Dolt ポート |
| `--user` | string |  | 外部 Dolt ユーザー |

## gc rig status

rig のステータスと agent 起動状態を表示します

```
gc rig status [name]
```

## gc rig suspend

city.toml で suspended=true を設定して rig を一時停止します。

一時停止中の rig にスコープされたすべての agent は実質的に停止します —
reconciler はそれらをスキップし、gc hook は空を返します。rig の beads
データベースは引き続きアクセス可能です。"gc rig resume" で復元します。

```
gc rig suspend [name]
```

## gc runtime

セッション内から agent コードによって呼び出される、プロセス内在のランタイム操作です。

これらのコマンドはセッションメタデータを読み書きして、agent と controller の間で
ライフサイクルイベント（drain、restart）を調停します。これらは実行中の agent セッション内から
呼び出されることを想定しており、人間が呼び出すものではありません。

```
gc runtime
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc runtime drain](#gc-runtime-drain) | セッションに drain（穏やかな終結）を信号 |
| [gc runtime drain-ack](#gc-runtime-drain-ack) | drain を確認 — このセッションを停止するよう controller に信号 |
| [gc runtime drain-check](#gc-runtime-drain-check) | セッションが draining 中か確認（終了コード 0 = draining） |
| [gc runtime request-restart](#gc-runtime-request-restart) | controller にこのセッションの再起動を要求（kill されるまでブロック） |
| [gc runtime undrain](#gc-runtime-undrain) | セッションの drain をキャンセル |

## gc runtime drain

セッションに drain — 現在の作業を穏やかに終結させるよう信号を送ります。

セッションに GC_DRAIN メタデータフラグを立てます。agent は定期的に
drain 状態を確認（"gc runtime drain-check" 経由で）し、現在のタスクを
完了してから終了する必要があります。セッションのエイリアスまたは ID を渡します。
キャンセルには "gc runtime undrain" を使います。

```
gc runtime drain <name>
```

## gc runtime drain-ack

drain 信号を確認 — controller にこのセッションを停止するよう伝えます。

セッションに GC_DRAIN_ACK メタデータを立てます。controller は次の調停ティックで
セッションを停止します。drain 信号への応答として現在の作業を完了した後にこれを
呼び出します。

```
gc runtime drain-ack [name]
```

## gc runtime drain-check

セッションが現在 draining 中か確認します。

draining なら終了コード 0、そうでなければ 1 を返します。条件分岐での使用を想定:
"if gc runtime drain-check; then finish-up; fi"。引数なしの場合は現在のセッション
コンテキストを使います。

```
gc runtime drain-check [name]
```

## gc runtime request-restart

controller にこのセッションを停止して再起動するよう信号を送ります。

セッションに GC_RESTART_REQUESTED メタデータを立て、永遠にブロックします。
controller は次の調停ティックでセッションを停止し、新規に再起動します。
ブロックすることで agent が待機中にコンテキストを消費するのを防ぎます。

オンデマンドで構成された名前付きセッションでは、controller がユーザーアテンドプロセスを
再起動できません。その場合このコマンドは再起動がスキップされたと報告して
ブロックせずに戻ります。再起動がスキップされたとき session.draining イベントは送出されません。

このコマンドはセッションコンテキスト内から呼ばれることを想定しています。
ブロック前に session.draining イベントを送出します。

```
gc runtime request-restart
```

## gc runtime undrain

セッションの保留中の drain 信号をキャンセルします。

GC_DRAIN と GC_DRAIN_ACK メタデータフラグをクリアし、セッションが通常運用を
継続できるようにします。セッションのエイリアスまたは ID を渡します。

```
gc runtime undrain <name>
```

## gc service

ワークスペースサービスを確認します

```
gc service
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc service doctor](#gc-service-doctor) | ワークスペースサービスの詳細ステータスを表示 |
| [gc service list](#gc-service-list) | ワークスペースサービスを一覧表示 |
| [gc service restart](#gc-service-restart) | ワークスペースサービスを再起動 |

## gc service doctor

ワークスペースサービスの詳細ステータスを表示します

```
gc service doctor <name>
```

## gc service list

ワークスペースサービスを一覧表示します

```
gc service list
```

## gc service restart

名前でワークスペースサービスを停止して再起動します。

controller は現在のサービスプロセスを閉じ、新しいプロセスを起動します。
city 全体の再起動なしで pack スクリプトを更新した後に有用です。

```
gc service restart <name>
```

## gc session

agent との永続的な会話を作成、再開、停止、終了します。

セッションは agent テンプレートに支えられた会話です。リソースを解放するために
一時停止し、後から完全な会話継続性で再開できます。

```
gc session
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc session attach](#gc-session-attach) | チャットセッションにアタッチ（または再開） |
| [gc session close](#gc-session-close) | セッションを永久に終了 |
| [gc session kill](#gc-session-kill) | セッションランタイムを強制 kill（reconciler が再起動） |
| [gc session list](#gc-session-list) | チャットセッションを一覧表示 |
| [gc session logs](#gc-session-logs) | セッションのログを表示 |
| [gc session new](#gc-session-new) | agent テンプレートから新しいチャットセッションを作成 |
| [gc session nudge](#gc-session-nudge) | 実行中セッションにテキストメッセージを送信 |
| [gc session peek](#gc-session-peek) | アタッチせずにセッション出力を確認 |
| [gc session pin](#gc-session-pin) | セッションを起きた状態に保つ |
| [gc session prune](#gc-session-prune) | 古い一時停止セッションを終了 |
| [gc session rename](#gc-session-rename) | セッションを改名 |
| [gc session reset](#gc-session-reset) | bead を保ったままセッションを新規再起動 |
| [gc session submit](#gc-session-submit) | セマンティックな配信意図を伴うメッセージ送信 |
| [gc session suspend](#gc-session-suspend) | セッションを一時停止（状態保存・リソース解放） |
| [gc session unpin](#gc-session-unpin) | セッションの awake pin を解除 |
| [gc session wait](#gc-session-wait) | セッションに依存 wait を登録 |
| [gc session wake](#gc-session-wake) | セッションを起こす（起動を要求し hold をクリア） |

## gc session attach

実行中のセッションにアタッチするか、一時停止中のセッションを再開します。

セッションがアクティブで tmux セッションがライブの場合は再アタッチします。
一時停止中、または tmux セッションが死んでいる場合は、provider の再開メカニズム
（サポートされていれば）で再開、または再起動します。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session attach <session-id-or-alias>
```

## gc session close

会話を終了します。アクティブならランタイムを停止し、bead を閉じます。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session close <session-id-or-alias>
```

## gc session kill

bead 状態を変更せずにセッションのランタイムプロセスを強制 kill します。

セッションはアクティブとしてマークされたままなので、reconciler は死んだプロセスを検出し、
セッションのライフサイクルルールに従って再起動します。会話履歴を失わずに
セッションのスタックを解除するのに有用です。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session kill <session-id-or-alias>
```

## gc session list

すべてのチャットセッションを一覧表示します。デフォルトではアクティブと一時停止中のセッションを表示します。

```
gc session list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--json` | bool |  | JSON 出力 |
| `--state` | string |  | 状態でフィルタ: "active"、"suspended"、"closed"、"all" |
| `--template` | string |  | テンプレート名でフィルタ |

## gc session logs

セッションの JSONL ファイルから構造化されたセッションログメッセージを表示します。

セッションログを読み、会話 DAG を解決し、メッセージを時系列順に出力します。
デフォルトのパス（~/.claude/projects/）と city.toml の [daemon] observe_paths に
あるすべての追加パスを検索します。

最後の N トランスクリプトエントリのみを出力するには --tail を使います（0 = 全件）。
セマンティクスは Unix の 'tail -n' に一致: '--tail 5' は最後の 5 件を出力し、
最初の 5 件ではありません。複数の tool-use ブロックを持つ単一の assistant ターンも
1 エントリとしてカウントされます。Compact 境界の区切りも、最終ウィンドウに含まれる場合は
エントリとしてカウントされます。

互換性メモ: 1.0 より前は --tail は compaction セグメントにマップされていました。
1.0 以降、--tail は表示されるトランスクリプトエントリウィンドウをトリムします。
HTTP API の tail クエリパラメータは引き続き compaction セグメントセマンティクスを使用します。
新しいメッセージを到着次第追従するには -f を使います。

```
gc session logs <session> [flags]
```

**例:**

```
gc session logs mayor
  gc session logs mayor --tail 2
  gc session logs gc-123 --tail 20
  gc session logs gc-123 --tail 0
  gc session logs s-gc-123 -f
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-f`, `--follow` | bool |  | 新しいメッセージを到着次第追従 |
| `--tail` | int | `10` | 表示する直近のトランスクリプトエントリ数（0 = 全件、compact 区切りもエントリとしてカウント） |

## gc session new

ロード済みの city 設定で定義された agent テンプレートから、新しい永続会話を作成します。
デフォルトでは作成後にターミナルにアタッチします。

--title なしで --title-hint が指定されると、セッションタイトルはヒントテキストから
自動生成されます: 短いバージョンが即座に設定され、バックグラウンドでタイトルモデルが
精錬します。

```
gc session new <template> [flags]
```

**例:**

```
gc session new helper
  gc session new helper --alias sky
  gc session new helper --title "debugging auth"
  gc session new helper --title-hint "fix the login redirect loop"
  gc session new helper --no-attach
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--alias` | string |  | コマンドおよびメール用の人間にやさしいセッション識別子 |
| `--no-attach` | bool |  | アタッチせずにセッションを作成 |
| `--title` | string |  | 人間が読めるセッションタイトル |
| `--title-hint` | string |  | セッションタイトルを自動生成するためのテキスト |

## gc session nudge

ランタイム provider 経由で実行中のセッションにテキスト入力を送ります。

メッセージはセッションの入力にテキストコンテンツとして配信されます。これは
セッションのターミナルにメッセージを入力するのと等価です。

セッション ID またはセッションエイリアスを受け付けます。複数語のメッセージは
自動的に結合されます。

```
gc session nudge <id-or-alias> <message...> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--delivery` | string | `wait-idle` | 配信モード: immediate、wait-idle、または queue |

## gc session peek

アタッチせずにセッション出力を確認します

```
gc session peek <session-id-or-alias> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--lines` | int | `50` | キャプチャする行数 |

## gc session pin

セッションの永続 pin オーバーライドを設定して起きた状態に保ちます。

Pin は suspend hold やその他のハードブロッカーをクリアしません。ターゲットが
まだマテリアライズされていない構成済み名前付きセッションの場合、pin はその正規 bead を
作成し、ブロックが解除されたときに reconciler が起動できるようにします。

```
gc session pin <session-id-or-alias>
```

## gc session prune

指定された経過時間より古い一時停止中セッションを終了します。一時停止中の
セッションのみが対象 — アクティブセッションは決して prune されません。

```
gc session prune [flags]
```

**例:**

```
gc session prune --before 7d
  gc session prune --before 24h
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--before` | string | `7d` | この期間より古いセッションを prune（例: 7d、24h） |

## gc session rename

セッションを改名します

```
gc session rename <session-id-or-alias> <title>
```

## gc session reset

bead を閉じずに既存セッションの新規再起動を要求します。

controller は現在のランタイムを停止し、provider の会話状態を新規にして
同じセッションを再起動します。セッション識別、エイリアス、メール、キュー済みの作業は
既存のセッション bead に紐付いたままです。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session reset <session-id-or-alias>
```

## gc session submit

provider トランスポートの詳細を選ばずに、ユーザーメッセージをセッションに送信します。

ランタイムは選択されたセマンティック意図に従って、起こす・即時注入する・キューイングのどれを
行うかを決定します。

```
gc session submit <id-or-alias> <message...> [flags]
```

**例:**

```
gc session submit mayor "status update"
  gc session submit mayor "after this run, handle docs" --intent follow_up
  gc session submit mayor "stop and do this instead" --intent interrupt_now
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--intent` | string | `default` | submit 意図: default、follow_up、または interrupt_now |

## gc session suspend

ランタイムプロセスを停止してアクティブなセッションを一時停止します。
セッション bead は永続化され、後から再開できます。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session suspend <session-id-or-alias>
```

## gc session unpin

セッションの永続 pin オーバーライドのみを解除します。

unpin は即時停止を強制しません。reconciler は次のパスで通常の wake/sleep ルールを
適用します。

```
gc session unpin <session-id-or-alias>
```

## gc session wait

セッションに依存 wait を登録します

```
gc session wait [session-id-or-alias] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--any` | bool |  | 監視中の任意の bead が閉じたら起こす（デフォルト: 全件） |
| `--note` | string |  | wait が満たされたときに配信するリマインダーテキスト |
| `--on-beads` | stringSlice |  | 監視する bead ID |
| `--sleep` | bool |  | セッションが眠りに drain できるよう wait hold を設定 |

## gc session wake

セッションの起動を要求し、ユーザー hold やクラッシュループ隔離メタデータを解放します。

起動後、reconciler は次のティックで起動理由（例: 一致する config agent）があれば
セッションを起動します。起動理由がなければセッションは眠ったままです。

セッション ID（例: gc-42）またはセッションエイリアス（例: mayor）を受け付けます。

```
gc session wake <session-id-or-alias>
```

**例:**

```
gc session wake gc-42
  gc session wake mayor
```

## gc shell

シェル統合は gc コマンドおよびフラグのタブ補完を提供する補完 hook を
シェル RC ファイルに追加します。

サブコマンド: install、remove、status。

```
gc shell
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc shell install](#gc-shell-install) | シェル統合をインストール・更新 |
| [gc shell remove](#gc-shell-remove) | シェル統合を削除 |
| [gc shell status](#gc-shell-status) | シェル統合の状態を表示 |

## gc shell install

gc シェル補完 hook をインストールまたは更新します。

シェルが指定されない場合、$SHELL から検出されます。
補完スクリプトは ~/.gc/completions/ に書き出され、シェル RC ファイルに source 行が
追加されます。

```
gc shell install [bash|zsh|fish]
```

## gc shell remove

シェル RC ファイルから gc シェル補完 hook を削除し、補完スクリプトを削除します。

```
gc shell remove
```

## gc shell status

シェル統合の状態を表示します

```
gc shell status
```

## gc skill

現在の city から見えるスキルを一覧表示します。

出力には以下が含まれます:
  - city pack スキル（city ルート配下の skills/&lt;name&gt;/SKILL.md）
  - インポート済み pack の共有スキル（バインディング修飾、例: ops.code-review）
  - レガシー暗黙インポートが残る場合の互換 bootstrap スキル
  - --agent/--session 指定時: その agent の agents/&lt;name&gt;/skills/ カタログ

一覧は *利用可能* な内容の診断ビューです。優先順位を畳み込んだり、
provider にベンダーシンクがある agent にフィルタしたり、名前衝突時に
materializer が選ぶエントリを正確に予測したりはしません。マテリアライズ済みの
セットを確認するには、"gc start" 後に &lt;scope-root&gt;/.&lt;vendor&gt;/skills/ シンクを確認するか、
"gc doctor" で衝突を表面化させます。

```
gc skill
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc skill list](#gc-skill-list) | 可視のスキルを一覧表示 |

## gc skill list

現在の共有スキルおよび agent ローカルの可視スキルを一覧表示します。任意で agent または
セッションにスコープできます。

```
gc skill list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--agent` | string |  | この agent の有効なスキルビューを表示 |
| `--session` | string |  | このセッションの有効なスキルビューを表示 |

## gc sling

ターゲットの sling_query を使用して bead をセッション設定または agent にルーティングします。

ターゲットは agent の完全修飾名（例: "mayor" や "hello-world/polecat"）です。
2 番目の引数は bead ID、--formula が設定された場合の formula 名、
または任意のテキスト（自動的に task bead が作成される）です。

ターゲットが省略された場合、bead の rig プレフィックスが使われ、設定からその rig の
default_sling_target が参照されます。明示的なターゲットを与えるには --formula が必要です。
インラインテキストにも明示的なターゲットが必要です。

--formula を指定すると、formula から wisp（一時的な molecule）がインスタンス化され、
そのルート bead がターゲットへルーティングされます。

例:
  gc sling my-rig/claude BL-42              # 既存の bead をルーティング
  gc sling my-rig/claude "write a README"   # テキストから bead を作成してルーティング
  gc sling mayor code-review --formula      # formula をインスタンス化し wisp をルーティング
  echo "fix login" | gc sling mayor --stdin # stdin から bead テキストを読み込み

```
gc sling [target] <bead-or-formula-or-text> [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-n`, `--dry-run` | bool |  | 実行せずに何が行われるかを表示 |
| `--force` | bool |  | 警告を抑止し、rig 横断ルーティング、グラフワークフロー置換を許可。直接 bead ルートでは bead がローカルストアで解決できなくても dispatch する |
| `-f`, `--formula` | bool |  | 引数を formula 名として扱う |
| `--merge` | string |  | マージ戦略: direct、mr、または local |
| `--no-convoy` | bool |  | 自動 convoy 作成をスキップ |
| `--no-formula` | bool |  | デフォルト formula を抑制（生 bead をルーティング） |
| `--nudge` | bool |  | ルーティング後にターゲットを nudge |
| `--on` | string |  | ルーティング前に formula から wisp を bead にアタッチ |
| `--owned` | bool |  | 自動 convoy を所有としてマーク（自動クローズをスキップ） |
| `--scope-kind` | string |  | graph.v2 起動の論理ワークフロースコープ kind |
| `--scope-ref` | string |  | graph.v2 起動の論理ワークフロースコープ ref |
| `--stdin` | bool |  | stdin から bead テキストを読み込む（最初の行 = タイトル、残り = 説明） |
| `-t`, `--title` | string |  | wisp ルート bead タイトル（--formula または --on と共に） |
| `--var` | stringArray |  | formula の変数置換（key=value、繰り返し可） |

## gc start

マシン全体の supervisor 配下で city を起動します。

"gc init" でブートストラップされた既存の city が必要です。必要に応じてリモート pack を
取得し、city をマシン全体の supervisor に登録し、supervisor の起動を確認し、
即時調停をトリガーします。フォアグラウンド運用には "gc supervisor run" を使います。

```
gc start [path] [flags]
```

**例:**

```
gc start
  gc start ~/my-city
  gc start --dry-run
  gc supervisor run
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-n`, `--dry-run` | bool |  | 起動せずにどの agent が起動するかをプレビュー |

## gc status

city 全体の概要を表示: controller の状態、suspend、
すべての agent の起動状態、rig、サマリ件数。

```
gc status [path] [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--json` | bool |  | JSON 形式で出力 |

## gc stop

city 内のすべての agent セッションを穏やかなシャットダウンで停止します。

実行中の agent に割り込みシグナルを送り、構成済みのシャットダウンタイムアウトまで
待機し、残ったセッションを強制 kill します。Dolt サーバも停止し、孤立セッションを
クリーンアップします。controller が実行中の場合はシャットダウンを controller に委譲します。

```
gc stop [path]
```

## gc supervisor

マシン全体の supervisor を管理します。

supervisor は単一プロセスからすべての登録済み city を管理し、統一された API サーバを
ホストします。city を追加するには "gc init"、"gc start"、または "gc register" を使います。

```
gc supervisor
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc supervisor install](#gc-supervisor-install) | supervisor をプラットフォームサービスとしてインストール |
| [gc supervisor logs](#gc-supervisor-logs) | supervisor ログファイルを tail |
| [gc supervisor reload](#gc-supervisor-reload) | すべての city の即時調停をトリガー |
| [gc supervisor run](#gc-supervisor-run) | マシン全体の supervisor をフォアグラウンドで実行 |
| [gc supervisor start](#gc-supervisor-start) | マシン全体の supervisor をバックグラウンドで起動 |
| [gc supervisor status](#gc-supervisor-status) | supervisor が実行中か確認 |
| [gc supervisor stop](#gc-supervisor-stop) | マシン全体の supervisor を停止 |
| [gc supervisor uninstall](#gc-supervisor-uninstall) | プラットフォームサービスを削除 |

## gc supervisor install

ログイン時に起動するプラットフォームサービスとしてマシン全体の supervisor を
インストールします。

```
gc supervisor install
```

## gc supervisor logs

マシン全体の supervisor ログファイルを tail します。

バックグラウンドおよびサービス管理 supervisor 実行の最近のログ出力を表示します。

```
gc supervisor logs [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-f`, `--follow` | bool |  | ログ出力を追従 |
| `-n`, `--lines` | int | `50` | 表示する行数 |

## gc supervisor reload

実行中の supervisor に reload 信号を送り、レジストリの再読み込みと
すべての city の即時調停を行わせます。子プロセスを kill した後、
次のパトロールティックを待たずに supervisor に変更を検出させ
再起動させたいときに使います。

```
gc supervisor reload
```

## gc supervisor run

マシン全体の supervisor をフォアグラウンドで実行します。

これは正規の長時間実行制御ループです。~/.gc/cities.toml から登録済み city を読み、
1 つのプロセスから管理し、共有 API サーバをホストします。

```
gc supervisor run
```

## gc supervisor start

マシン全体の supervisor をバックグラウンドで起動します。

"gc supervisor run" を fork し、起動完了を確認してから戻ります。

```
gc supervisor start
```

## gc supervisor status

supervisor が実行中か確認します

```
gc supervisor status
```

## gc supervisor stop

実行中のマシン全体 supervisor とそのすべての city を停止します。

デフォルトでは、supervisor が停止要求を受理した時点で戻ります — シャットダウンは
非同期に継続します。supervisor ソケットが応答しなくなるまでブロックするには --wait を
渡します。これは決定論的なクリーンアップが必要な呼び出し元（例: 一時ディレクトリ削除を
すぐに行う統合テストで、残留 supervisor/controller サブプロセスとの競合を避けたい場合）が
通常欲しい動作です。

```
gc supervisor stop [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--wait` | bool |  | 戻る前に、supervisor がすべての管理 city の停止を完了しソケットを解放するのを待つ |
| `--wait-timeout` | duration | `30s` | --wait 設定時の最大待機時間 |

## gc supervisor uninstall

プラットフォームサービスを削除し、マシン全体の supervisor を停止します。

```
gc supervisor uninstall
```

## gc suspend

city.toml の workspace.suspended を true に設定して city を一時停止します。

これは下方向に継承されます — city が一時停止されると、各 agent の個別 suspended
フィールドに関わらずすべての agent が実質的に一時停止します。
reconciler は agent を起動せず、gc hook/prime は空を返します。

復元には "gc resume" を使います。

```
gc suspend [path]
```

## gc trace

session reconciler のトレースストリームを確認・制御します。

トレース状態は .gc/runtime/session-reconciler-trace 配下にローカルに永続化され、
controller がオフラインでも管理できます。

```
gc trace
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc trace cycle](#gc-trace-cycle) | tick id でサイクルを表示 |
| [gc trace reasons](#gc-trace-reasons) | トレースレコード内で観測された理由コードを表示 |
| [gc trace show](#gc-trace-show) | トレースレコードを表示 |
| [gc trace start](#gc-trace-start) | テンプレートのトレースを開始または延長 |
| [gc trace status](#gc-trace-status) | トレースアームとストリーム状態を表示 |
| [gc trace stop](#gc-trace-stop) | テンプレートのトレースを停止 |
| [gc trace tail](#gc-trace-tail) | トレースレコードを追従 |

## gc trace cycle

tick id でサイクルを表示します

```
gc trace cycle [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--tick` | string |  | 表示する tick id |

## gc trace reasons

トレースレコード内で観測された理由コードを表示します

```
gc trace reasons [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--since` | string |  | 指定期間前以降の理由を表示 |
| `--template` | string |  | 厳密に正規化されたテンプレートセレクタ |

## gc trace show

トレースレコードを表示します

```
gc trace show [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--json` | bool | `true` | JSON 配列を出力 |
| `--reason` | string |  | 理由コードでフィルタ |
| `--since` | string |  | 指定期間前以降のレコードを表示 |
| `--template` | string |  | 厳密に正規化されたテンプレートセレクタ |
| `--tick` | string |  | tick id でフィルタ |
| `--trace-id` | string |  | trace id でフィルタ |
| `--type` | string |  | レコードタイプでフィルタ |

## gc trace start

テンプレートのトレースを開始または延長します

```
gc trace start [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--auto` | bool |  | アームを自動トリガとしてマーク |
| `--for` | string | `15m` | トレースアームの期間（例: 15m） |
| `--level` | string | `detail` | トレースレベル: baseline または detail |
| `--template` | string |  | 厳密に正規化されたテンプレートセレクタ |

## gc trace status

トレースアームとストリーム状態を表示します

```
gc trace status
```

## gc trace stop

テンプレートのトレースを停止します

```
gc trace stop [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--all` | bool |  | manual と auto のアームを両方削除 |
| `--template` | string |  | 厳密に正規化されたテンプレートセレクタ |

## gc trace tail

トレースレコードを追従します

```
gc trace tail [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--since` | string |  | 指定期間前から追従 |
| `--template` | string |  | 厳密に正規化されたテンプレートセレクタ |

## gc unregister

マシン全体の supervisor レジストリから city を削除します。

パスを指定しない場合、現在の city（cwd から検出）の登録を解除します。
supervisor が実行中の場合、その city の管理を直ちに停止します。

```
gc unregister [path]
```

## gc version

gc のバージョン文字列を表示します。

git commit とビルド日時のメタデータを含めるには --long を使います。

```
gc version [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `-l`, `--long` | bool |  | git commit とビルド日時のメタデータを含める |

## gc wait

永続的なセッション wait を確認・管理します

```
gc wait
```

| サブコマンド | 説明 |
|------------|-------------|
| [gc wait cancel](#gc-wait-cancel) | wait をキャンセル |
| [gc wait inspect](#gc-wait-inspect) | wait の詳細を表示 |
| [gc wait list](#gc-wait-list) | 永続的な wait を一覧表示 |
| [gc wait ready](#gc-wait-ready) | wait を手動で ready に設定 |

## gc wait cancel

wait をキャンセルします

```
gc wait cancel <wait-id>
```

## gc wait inspect

wait の詳細を表示します

```
gc wait inspect <wait-id>
```

## gc wait list

永続的な wait を一覧表示します

```
gc wait list [flags]
```

| フラグ | 型 | デフォルト | 説明 |
|------|------|---------|-------------|
| `--session` | string |  | セッション ID でフィルタ |
| `--state` | string |  | wait 状態でフィルタ |

## gc wait ready

wait を手動で ready に設定します

```
gc wait ready <wait-id>
```
