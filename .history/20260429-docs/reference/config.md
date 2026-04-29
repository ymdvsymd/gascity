# Gas City Configuration

city.toml のスキーマ — Gas City インスタンスの PackV2 deployment ファイルです。Pack 定義は pack.toml と、agents/、formulas/、orders/、commands/ などの pack 規約ディレクトリに置かれます。PackV2 合成には `[imports.*]` を使用してください。レガシーな includes、`[packs.*]`、`[[agent]]` フィールドは移行互換性のために引き続き表示されます。

> **自動生成** — 編集しないでください。再生成するには `go run ./cmd/genschema` を実行してください。

## City

City は Gas City インスタンスのトップレベル設定です。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `include` | []string |  |  | Include は、この設定にマージする設定フラグメントファイルをリストします。LoadWithIncludes により処理され、再帰的ではありません (フラグメントは include できません)。 |
| `workspace` | Workspace | **yes** |  | Workspace は city レベルのメタデータ (name、デフォルト provider) を保持します。 |
| `providers` | map[string]ProviderSpec |  |  | Providers は agent 起動用の名前付き provider プリセットを定義します。 |
| `packs` | map[string]PackSource |  |  | Packs は git で取得される名前付きリモート pack ソースを定義します (V1 の仕組み)。 |
| `imports` | map[string]Import |  |  | Imports は名前付き pack imports を定義します (V2 の仕組み)。各キーは binding 名で、値は source とオプションの version、export、transitive コントロールを指定します。ExpandCityPacks 中に処理されます。 |
| `agent` | []Agent | **yes** |  | Agents はこの city に設定されたすべての agents をリストします。 |
| `named_session` | []NamedSession |  |  | NamedSessions は再利用可能な agent テンプレートから構築される正典的な alias-backed sessions をリストします。 |
| `rigs` | []Rig |  |  | Rigs はこの city に登録された外部プロジェクトをリストします。 |
| `patches` | Patches |  |  | Patches はフラグメントマージ後に適用される対象指向の変更を保持します。 |
| `beads` | BeadsConfig |  |  | Beads は bead store のバックエンドを設定します。 |
| `session` | SessionConfig |  |  | Session は session provider のバックエンドを設定します。 |
| `mail` | MailConfig |  |  | Mail は mail provider のバックエンドを設定します。 |
| `events` | EventsConfig |  |  | Events は events provider のバックエンドを設定します。 |
| `dolt` | DoltConfig |  |  | Dolt はオプションの dolt サーバー接続オーバーライドを設定します。 |
| `formulas` | FormulasConfig |  |  | Formulas は formula ディレクトリの設定を構成します。 |
| `daemon` | DaemonConfig |  |  | Daemon は controller daemon の設定を構成します。 |
| `orders` | OrdersConfig |  |  | Orders は order の設定 (skip リスト) を構成します。 |
| `api` | APIConfig |  |  | API はオプションの HTTP API サーバーを設定します。 |
| `chat_sessions` | ChatSessionsConfig |  |  | ChatSessions は chat session の動作 (自動 suspend) を設定します。 |
| `session_sleep` | SessionSleepConfig |  |  | SessionSleep は管理対象 session のアイドル sleep ポリシーのデフォルトを設定します。 |
| `convergence` | ConvergenceConfig |  |  | Convergence は収束ループの上限を設定します。 |
| `service` | []Service |  |  | Services は controller のエッジで /svc/&#123;name&#125; 配下にマウントされる workspace 所有の HTTP services を宣言します。 |
| `agent_defaults` | AgentDefaults |  |  | AgentDefaults は、上書きしない agent に対する city レベルのデフォルトを提供します (正典的な TOML キー: agent_defaults)。ランタイムは現在 default_sling_formula と append_fragments を適用します。アタッチメントリスト系のフィールドは tombstone のままで、その他のフィールドはパース・合成されますが自動継承はまだされません。 |

## ACPSessionConfig

ACPSessionConfig は ACP session provider の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `handshake_timeout` | string |  | `30s` | HandshakeTimeout は ACP ハンドシェイクの完了を待つ時間です。Duration 文字列 (例: "30s"、"1m")。デフォルトは "30s"。 |
| `nudge_busy_timeout` | string |  | `60s` | NudgeBusyTimeout は新しい prompt を送信する前に agent がアイドルになるのを待つ時間です。Duration 文字列。デフォルトは "60s"。 |
| `output_buffer_lines` | integer |  | `1000` | OutputBufferLines は Peek 用に循環バッファに保持する出力行数です。デフォルトは 1000。 |

## APIConfig

APIConfig は HTTP API サーバーを設定します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `port` | integer |  |  | Port は listen する TCP ポートです。デフォルトは 9443; 0 = 無効。 |
| `bind` | string |  |  | Bind は listener をバインドするアドレスです。デフォルトは "127.0.0.1"。 |
| `allow_mutations` | boolean |  |  | AllowMutations は bind が非 localhost のときのデフォルトの読み取り専用動作を上書きします。コンテナ環境で API がヘルスプローブのため 0.0.0.0 にバインドされる必要があるが mutations が依然として安全な場合に true に設定します。 |

## Agent

Agent は city に設定された agent を定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name はこの agent の一意識別子です。 |
| `description` | string |  |  | Description は MC の session 作成 UI に表示される人間可読な説明です。 |
| `dir` | string |  |  | Dir は rig スコープ agent の identity prefix で、WorkDir が未設定のときのデフォルト working directory です。 |
| `work_dir` | string |  |  | WorkDir は agent の qualified identity を変えずに session の working directory を上書きします。相対パスは city root を基準に解決され、session_setup と同じテンプレートプレースホルダーを使用できます。 |
| `scope` | string |  |  | Scope はこの agent がどこにインスタンス化されるかを定義します: "city" (city ごとに 1 つ) または "rig" (rig ごとに 1 つ、デフォルト)。pack 定義 agent でのみ意味を持ちます; city.toml のインライン agent は Dir を直接使用します。Enum: `city`、`rig` |
| `suspended` | boolean |  |  | Suspended は reconciler がこの agent を spawn することを防ぎます。gc agent suspend/resume で切り替えます。 |
| `pre_start` | []string |  |  | PreStart は session 作成前に実行される shell コマンドのリストです。コマンドは対象ファイルシステム上で実行されます: tmux ではローカル、exec providers では pod/container 内部。テンプレート変数は session_setup と同じです。 |
| `prompt_template` | string |  |  | PromptTemplate はこの agent の prompt template ファイルへのパスです。相対パスは city ディレクトリを基準に解決されます。 |
| `nudge` | string |  |  | Nudge は起動後 agent の tmux session に入力されるテキストです。コマンドライン prompt を受け付けない CLI agent に使用されます。 |
| `session` | string |  |  | Session はこの agent の session トランスポートを上書きします。"" (デフォルト) は city レベルの session provider (通常は tmux) を使用します。"acp" は Agent Client Protocol (stdio 上の JSON-RPC) を使用します。Agent の解決された provider は supports_acp = true である必要があります。Enum: `acp` |
| `provider` | string |  |  | Provider はこの agent に使用する provider プリセット名を指定します。 |
| `start_command` | string |  |  | StartCommand はこの agent の provider のコマンドを上書きします。 |
| `args` | []string |  |  | Args は provider のデフォルト引数を上書きします。 |
| `prompt_mode` | string |  | `arg` | PromptMode は prompt の配信方法を制御します: "arg"、"flag"、または "none"。Enum: `arg`、`flag`、`none` |
| `prompt_flag` | string |  |  | PromptFlag は prompt_mode が "flag" のときに prompt を渡すために使用する CLI フラグです。 |
| `ready_delay_ms` | integer |  |  | ReadyDelayMs は launch 後に agent を ready とみなすまでの待機ミリ秒数です。 |
| `ready_prompt_prefix` | string |  |  | ReadyPromptPrefix は agent が入力可能であることを示す文字列の接頭辞です。 |
| `process_names` | []string |  |  | ProcessNames は agent が実行中かをチェックする際に探すプロセス名をリストします。 |
| `emits_permission_warning` | boolean |  |  | EmitsPermissionWarning は agent が抑制すべき permission prompt を発するかどうかを示します。 |
| `env` | map[string]string |  |  | Env は agent プロセスの追加環境変数を設定します。 |
| `option_defaults` | map[string]string |  |  | OptionDefaults はこの agent に対して provider の有効スキーマデフォルトを上書きします。キーはオプションキー、値は choice 値です。Provider の OptionDefaults の上に適用されます (agent のキーが優先)。例: option_defaults = &#123; permission_mode = "plan", model = "sonnet" &#125; |
| `max_active_sessions` | integer |  |  | MaxActiveSessions は同時 session に対する agent レベルの上限です。Nil は rig、次に workspace、最後に無制限から継承を意味します。pool.max を置き換えます。 |
| `min_active_sessions` | integer |  |  | MinActiveSessions は維持する session の最小数です。Agent レベルのみ。Rig/workspace の上限にカウントされます。pool.min を置き換えます。 |
| `scale_check` | string |  |  | ScaleCheck は新しい未割当 session 需要を出力で報告する shell コマンドテンプレートです。bead-backed reconciliation では加算的です: 割当済み work は別途 resume され、ScaleCheck はすべての上限レベルで制限される新しい汎用 session の起動数のみを報告します。Legacy no-store 評価は出力を希望 session 数として扱います。Go テンプレートプレースホルダーが含まれている場合、gc は work_dir や session_setup と同じ PathContext フィールド (Agent、AgentBase、Rig、RigRoot、CityRoot、CityName) で展開してからコマンドを実行します。 |
| `drain_timeout` | string |  | `5m` | DrainTimeout はスケールダウン中に session を強制終了する前に現在の作業を終わらせるための最大待機時間です。Duration 文字列 (例: "5m"、"30m"、"1h")。デフォルトは "5m"。 |
| `on_boot` | string |  |  | OnBoot はこの agent に対して controller 起動時に一度実行される shell コマンドテンプレートです。Go テンプレートプレースホルダーが含まれている場合、gc は work_dir や session_setup と同じ PathContext フィールド (Agent、AgentBase、Rig、RigRoot、CityRoot、CityName) で展開してから実行します。 |
| `on_death` | string |  |  | OnDeath は session が予期せず死んだときに実行される shell コマンドテンプレートです。Go テンプレートプレースホルダーが含まれている場合、gc は work_dir や session_setup と同じ PathContext フィールド (Agent、AgentBase、Rig、RigRoot、CityRoot、CityName) で展開してから実行します。 |
| `namepool` | string |  |  | Namepool は 1 行に 1 つの名前が記載された平文ファイルへのパスです。設定されている場合、session はそのファイルの名前を表示用 alias として使用します。 |
| `work_query` | string |  |  | WorkQuery はこの agent の利用可能な work を見つけるための shell コマンドテンプレートです。Go テンプレートプレースホルダーが含まれている場合、gc は work_dir や session_setup と同じ PathContext フィールド (Agent、AgentBase、Rig、RigRoot、CityRoot、CityName) で probe、hook、prompt-context 実行の前に展開します。gc hook で使用され、prompt template では &#123;&#123;.WorkQuery&#125;&#125; として利用可能です。未設定の場合、Gas City は 3 段のデフォルトクエリを使用します:   1. この session/alias に割り当てられた in_progress work (クラッシュ復旧)   2. この session/alias に割り当てられた ready work (事前割当 work)   3. gc.routed_to=&lt;qualified-name&gt; を持つ ready の未割当 work。Controller が session コンテキストなしで需要を probe するときは、routed_to の段のみ適用されます。外部 task システムと統合する場合に上書きします。 |
| `sling_query` | string |  |  | SlingQuery は bead をこの session 設定にルーティングするためのコマンドテンプレートです。Go テンプレートプレースホルダーが含まれている場合、gc は work_dir や session_setup と同じ PathContext フィールド (Agent、AgentBase、Rig、RigRoot、CityRoot、CityName) で &#123;&#125; を bead ID に置換する前に展開します。gc sling が bead をターゲットの work_query から見えるようにするために使用します。プレースホルダー &#123;&#125; はランタイム時に bead ID に置換されます。すべての agent のデフォルト: "bd update &#123;&#125; --set-metadata gc.routed_to=&lt;qualified-name&gt;"。ルーティングはメタデータベースで、sling はターゲットテンプレートを刻印し、reconciler/scale_check のパスがいつ session を作成するかを決定します。カスタムの sling_query と work_query は独立に上書きできます。 |
| `idle_timeout` | string |  |  | IdleTimeout は agent session が非アクティブでいられる最大時間で、controller がこれを超えると kill して再起動します。Duration 文字列 (例: "15m"、"1h")。空 (デフォルト) はアイドルチェックを無効化します。 |
| `sleep_after_idle` | string |  |  | SleepAfterIdle はこの agent のアイドル sleep ポリシーを上書きします。Duration 文字列 (例: "30s") または "off" を受け付けます。 |
| `install_agent_hooks` | []string |  |  | InstallAgentHooks はこの agent に対する workspace レベルの install_agent_hooks を上書きします。設定されると workspace のデフォルトに追加するのではなく置き換えます。 |
| `skills` | []string |  |  | Skills は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパース時に受け付けられますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `mcp` | []string |  |  | MCP は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパース時に受け付けられますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `hooks_installed` | boolean |  |  | HooksInstalled は自動 hook 検出を上書きします。Hooks が手動でインストールされている場合 (例: プロジェクト独自の hook 設定にマージされている場合) で、install_agent_hooks による自動インストールが望ましくない場合に true に設定します。True のとき、agent は起動動作のため hook 有効として扱われます: beacon に prime instruction なし、遅延 nudge なし。install_agent_hooks と相互作用します — hooks が事前インストール済みのときはこれを設定してください。 |
| `session_setup` | []string |  |  | SessionSetup は session 作成後に実行される shell コマンドのリストです。各コマンドは次のプレースホルダーをサポートするテンプレート文字列です: &#123;&#123;.Session&#125;&#125;、&#123;&#123;.Agent&#125;&#125;、&#123;&#123;.AgentBase&#125;&#125;、&#123;&#123;.Rig&#125;&#125;、&#123;&#123;.RigRoot&#125;&#125;、&#123;&#123;.CityRoot&#125;&#125;、&#123;&#123;.CityName&#125;&#125;、&#123;&#123;.WorkDir&#125;&#125;。コマンドは agent session 内ではなく gc のプロセスで sh -c 経由で実行されます。 |
| `session_setup_script` | string |  |  | SessionSetupScript は session_setup コマンドの後に実行される script へのパスです。相対パスは宣言する設定ファイルのディレクトリを基準に解決されます (pack-safe)。"//" 接頭辞のパスは city root を基準に解決されます。Script は環境変数 (GC_SESSION と既存の GC_* 変数) を介してコンテキストを受け取ります。 |
| `session_live` | []string |  |  | SessionLive は agent を再起動せずに再適用しても安全な shell コマンドのリストです。起動時 (session_setup の後) に実行され、再起動を引き起こさずに設定変更時に再適用されます。冪等である必要があります。典型的な用途: tmux のテーマ、キーバインド、ステータスバー。Session_setup と同じテンプレートプレースホルダー。 |
| `overlay_dir` | string |  |  | OverlayDir は起動時に内容が agent の working directory に再帰的にコピーされる (加算的) ディレクトリです。既存ファイルは上書きされません。相対パスは宣言する設定ファイルのディレクトリを基準に解決されます (pack-safe)。 |
| `default_sling_formula` | string |  |  | DefaultSlingFormula は --no-formula が設定されない限り、bead がこの agent に sling されたときに --on で自動適用される formula 名です。例: "mol-polecat-work" |
| `inject_fragments` | []string |  |  | InjectFragments はこの agent のレンダリング済み prompt に追加する名前付き template fragments をリストします。Fragments はロードされたすべての pack の共有 template ディレクトリから来ます。各名前は &#123;&#123; define "name" &#125;&#125; ブロックと一致する必要があります。 |
| `append_fragments` | []string |  |  | AppendFragments は prompt fragment 注入の V2 agent ごとの別名です。InjectFragments の後、継承/デフォルト fragments の前に重ねられます。 |
| `inject_assigned_skills` | boolean |  |  | InjectAssignedSkills は gc が agent のレンダリング済み prompt に "assigned skills" の付録を追加するかを制御します。付録はこの agent から見えるすべての skill を (assigned-to-you、shared-with-every-agent) に分けてリストし、scope-root sink を共有する agents が、自分の専門 skill と city ワイドのセットを区別できるようにします。  ポインタ三状態:   nil   -&gt; 継承: agent が vendor sink を持つ場合に注入   *true -&gt; 明示的に注入 (デフォルトと等価)   *false -&gt; 無効化; テンプレートが skill ガイダンスを自分でレンダリングする責任を負う |
| `attach` | boolean |  |  | Attach は agent の session が対話的接続 (例: tmux attach) をサポートするかを制御します。False のとき、agent はより軽量なランタイム (tmux ではなくサブプロセス) を使用できます。デフォルトは true。 |
| `fallback` | boolean |  |  | Fallback はこの agent を fallback 定義としてマークします。Pack 合成中、同じ名前の non-fallback agent が静かに優先されます。2 つの fallback が衝突した場合、最初にロードされたもの (深さ優先) が優先されます。 |
| `depends_on` | []string |  |  | DependsOn はこの agent が起動する前に起動している必要がある agent 名をリストします。依存関係順の起動とシャットダウンに使用されます。設定ロード時に循環がチェックされます。 |
| `resume_command` | string |  |  | ResumeCommand はこの agent を resume するときに実行する完全な shell コマンドです。&#123;&#123;.SessionKey&#125;&#125; テンプレート変数をサポートします。設定されている場合、provider の ResumeFlag/ResumeStyle より優先されます。例:   "claude --resume &#123;&#123;.SessionKey&#125;&#125; --dangerously-skip-permissions" |
| `wake_mode` | string |  |  | WakeMode は sleep/wake サイクルをまたいだコンテキストの新鮮さを制御します。"resume" (デフォルト): 会話の連続性のため provider session キーを再利用。"fresh": 起動のたびに新しい provider session を開始 (polecat パターン)。Enum: `resume`、`fresh` |

## AgentDefaults

AgentDefaults は city.toml の `[agent_defaults]` で宣言される city レベルの agent デフォルトを提供します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `model` | string |  |  | Model は agent のパース・合成済みデフォルトモデル名 (例: "claude-sonnet-4-6") ですが、ランタイムでまだ自動適用されません。独自の model オーバーライドを持つ agent が優先されます。 |
| `wake_mode` | string |  |  | WakeMode はパース・合成済みデフォルトの wake mode ("resume" または "fresh") ですが、ランタイムでまだ自動適用されません。Enum: `resume`、`fresh` |
| `default_sling_formula` | string |  |  | DefaultSlingFormula は `[agent_defaults]` を継承する agent に使用される city レベルのデフォルト formula です。明示的な agent は agent_defaults.default_sling_formula が設定されているときのみこの値を受け取ります。暗黙的な multi-session 設定は明示的なデフォルトが設定されていないとき他の場所で "mol-do-work" が seed されます。 |
| `allow_overlay` | []string |  |  | AllowOverlay は session オーバーレイのための city レベル allowlist としてパース・合成されますが、ランタイムでまだ agent に自動継承されません。 |
| `allow_env_override` | []string |  |  | AllowEnvOverride は session 環境変数オーバーライドのための city レベル allowlist としてパース・合成されますが、ランタイムでまだ agent に自動継承されません。名前は ^[A-Z][A-Z0-9_]&#123;0,127&#125;$ にマッチする必要があります。 |
| `append_fragments` | []string |  |  | AppendFragments はレンダリング後に .template.md prompt に自動追加する名前付き template fragments をリストします。Legacy の .md.tmpl prompt は移行期間中もサポートされます; プレーンな .md は不活性のままです。V2 移行のための便利機能 — city ワイドデフォルトの global_fragments/inject_fragments を置き換えます。 |
| `skills` | []string |  |  | Skills は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパース・合成されますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `mcp` | []string |  |  | MCP は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパース・合成されますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |

## AgentOverride

AgentOverride は特定の rig に対して pack 刻印 agent を変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `agent` | string | **yes** |  | Agent はオーバーライドする pack agent の名前です (必須)。 |
| `dir` | string |  |  | Dir は刻印された dir を上書きします (デフォルト: rig 名)。 |
| `work_dir` | string |  |  | WorkDir は agent の qualified identity や rig 関連付けを変えずに working directory を上書きします。 |
| `scope` | string |  |  | Scope は agent の scope ("city" または "rig") を上書きします。 |
| `suspended` | boolean |  |  | Suspended は agent の suspended state を設定します。 |
| `pool` | PoolOverride |  |  | Pool は session スケーリングにマップされる legacy `[pool]` フィールドを上書きします。 |
| `env` | map[string]string |  |  | Env は環境変数を追加または上書きします。 |
| `env_remove` | []string |  |  | EnvRemove は削除する env 変数キーをリストします。 |
| `pre_start` | []string |  |  | PreStart は agent の pre_start コマンドを上書きします。 |
| `prompt_template` | string |  |  | PromptTemplate は prompt template のパスを上書きします。相対パスは city ディレクトリを基準に解決されます。 |
| `session` | string |  |  | Session は session トランスポート ("acp") を上書きします。 |
| `provider` | string |  |  | Provider は provider 名を上書きします。 |
| `start_command` | string |  |  | StartCommand は start command を上書きします。 |
| `nudge` | string |  |  | Nudge は nudge テキストを上書きします。 |
| `idle_timeout` | string |  |  | IdleTimeout はアイドルタイムアウト duration 文字列 (例: "30s"、"5m"、"1h") を上書きします。 |
| `sleep_after_idle` | string |  |  | SleepAfterIdle はこの agent のアイドル sleep ポリシーを上書きします。Duration 文字列 (例: "30s") または "off" を受け付けます。 |
| `install_agent_hooks` | []string |  |  | InstallAgentHooks は agent の install_agent_hooks リストを上書きします。 |
| `skills` | []string |  |  | Skills は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパースされますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `mcp` | []string |  |  | MCP は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパースされますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `hooks_installed` | boolean |  |  | HooksInstalled は自動 hook 検出を上書きします。 |
| `inject_assigned_skills` | boolean |  |  | InjectAssignedSkills は Agent.InjectAssignedSkills を上書きします (詳細はそのフィールド参照)。 |
| `session_setup` | []string |  |  | SessionSetup は agent の session_setup コマンドを上書きします。 |
| `session_setup_script` | string |  |  | SessionSetupScript は agent の session_setup_script のパスを上書きします。相対パスは宣言する設定ファイルのディレクトリを基準に解決されます (pack-safe)。"//" 接頭辞のパスは city root を基準に解決されます。 |
| `session_live` | []string |  |  | SessionLive は agent の session_live コマンドを上書きします。 |
| `overlay_dir` | string |  |  | OverlayDir は agent の overlay_dir パスを上書きします。起動時に内容を agent の working directory に加算的にコピーします。相対パスは city ディレクトリを基準に解決されます。 |
| `default_sling_formula` | string |  |  | DefaultSlingFormula はデフォルト sling formula を上書きします。 |
| `inject_fragments` | []string |  |  | InjectFragments は agent の inject_fragments リストを上書きします。 |
| `append_fragments` | []string |  |  | AppendFragments はこの agent のレンダリング済み prompt に名前付き template fragments を追加します。Agent ごとの fragment 選択の V2 表記です。 |
| `pre_start_append` | []string |  |  | PreStartAppend は agent の pre_start リストにコマンドを追加します (置換ではなく)。両方が設定されている場合 PreStart の後に適用されます。 |
| `session_setup_append` | []string |  |  | SessionSetupAppend は agent の session_setup リストにコマンドを追加します。 |
| `session_live_append` | []string |  |  | SessionLiveAppend は agent の session_live リストにコマンドを追加します。 |
| `install_agent_hooks_append` | []string |  |  | InstallAgentHooksAppend は agent の install_agent_hooks リストに追加します。 |
| `skills_append` | []string |  |  | SkillsAppend は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパースされますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `mcp_append` | []string |  |  | MCPAppend は v0.15.1 の後方互換のため保持される tombstone フィールドです。移行時の可視性のためにパースされますが、アタッチメントリスト系フィールドはアクティブな materializer に受け付けられても無視されます。 |
| `attach` | boolean |  |  | Attach は agent の attach 設定を上書きします。 |
| `depends_on` | []string |  |  | DependsOn は agent の依存関係リストを上書きします。 |
| `resume_command` | string |  |  | ResumeCommand は agent の resume_command テンプレートを上書きします。 |
| `wake_mode` | string |  |  | WakeMode は agent の wake mode ("resume" または "fresh") を上書きします。Enum: `resume`、`fresh` |
| `inject_fragments_append` | []string |  |  | InjectFragmentsAppend は agent の inject_fragments リストに追加します。 |
| `max_active_sessions` | integer |  |  | MaxActiveSessions は同時 session に対する agent レベルの上限を上書きします。 |
| `min_active_sessions` | integer |  |  | MinActiveSessions は維持する session の最小数を上書きします。 |
| `scale_check` | string |  |  | ScaleCheck は bead-backed reconciliation のための新しい未割当 session 需要を出力で報告する shell コマンドを上書きします。 |
| `option_defaults` | map[string]string |  |  | OptionDefaults はこの agent の provider オプションデフォルトを追加または上書きします。キーはオプションキー、値は choice 値です。加算的にマージされます (override キーが既存の agent キーより優先)。例: option_defaults = &#123; model = "sonnet" &#125; |

## AgentPatch

AgentPatch は (Dir, Name) で識別される既存の agent を変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `dir` | string | **yes** |  | Dir はターゲット指定キーです (Name と共に必須)。Agent の working directory スコープを識別します。City スコープ agent の場合は空。 |
| `name` | string | **yes** |  | Name はターゲット指定キーです (必須)。既存の agent の名前と一致する必要があります。 |
| `work_dir` | string |  |  | WorkDir は agent の session working directory を上書きします。 |
| `scope` | string |  |  | Scope は agent の scope ("city" または "rig") を上書きします。 |
| `suspended` | boolean |  |  | Suspended は agent の suspended state を上書きします。 |
| `pool` | PoolOverride |  |  | Pool は session スケーリングにマップされる legacy `[pool]` フィールドを上書きします。 |
| `env` | map[string]string |  |  | Env は環境変数を追加または上書きします。 |
| `env_remove` | []string |  |  | EnvRemove はマージ後に削除する env 変数キーをリストします。 |
| `pre_start` | []string |  |  | PreStart は agent の pre_start コマンドを上書きします。 |
| `prompt_template` | string |  |  | PromptTemplate は prompt template のパスを上書きします。相対パスは city ディレクトリを基準に解決されます。 |
| `session` | string |  |  | Session は session トランスポート ("acp") を上書きします。 |
| `provider` | string |  |  | Provider は provider 名を上書きします。 |
| `start_command` | string |  |  | StartCommand は start command を上書きします。 |
| `nudge` | string |  |  | Nudge は nudge テキストを上書きします。 |
| `idle_timeout` | string |  |  | IdleTimeout はアイドルタイムアウトを上書きします。Duration 文字列 (例: "30s"、"5m"、"1h")。 |
| `sleep_after_idle` | string |  |  | SleepAfterIdle はこの agent のアイドル sleep ポリシーを上書きします。Duration 文字列または "off" を受け付けます。 |
| `install_agent_hooks` | []string |  |  | InstallAgentHooks は agent の install_agent_hooks リストを上書きします。 |
| `skills` | []string |  |  | Skills は v0.15.1 の後方互換のため保持される tombstone フィールドです。  Deprecated: v0.16 で削除。Tombstone — 受け付けるが無視されます。engdocs/proposals/skill-materialization.md 参照。 |
| `mcp` | []string |  |  | MCP は v0.15.1 の後方互換のため保持される tombstone フィールドです。  Deprecated: v0.16 で削除。Tombstone — 受け付けるが無視されます。engdocs/proposals/skill-materialization.md 参照。 |
| `skills_append` | []string |  |  | SkillsAppend は v0.15.1 の後方互換のため保持される tombstone フィールドです。  Deprecated: v0.16 で削除。Tombstone — 受け付けるが無視されます。engdocs/proposals/skill-materialization.md 参照。 |
| `mcp_append` | []string |  |  | MCPAppend は v0.15.1 の後方互換のため保持される tombstone フィールドです。  Deprecated: v0.16 で削除。Tombstone — 受け付けるが無視されます。engdocs/proposals/skill-materialization.md 参照。 |
| `hooks_installed` | boolean |  |  | HooksInstalled は自動 hook 検出を上書きします。 |
| `inject_assigned_skills` | boolean |  |  | InjectAssignedSkills は agent ごとの付録注入を上書きします (Agent.InjectAssignedSkills 参照)。 |
| `session_setup` | []string |  |  | SessionSetup は agent の session_setup コマンドを上書きします。 |
| `session_setup_script` | string |  |  | SessionSetupScript は agent の session_setup_script のパスを上書きします。相対パスは宣言する設定ファイルのディレクトリを基準に解決されます (pack-safe)。"//" 接頭辞のパスは city root を基準に解決されます。 |
| `session_live` | []string |  |  | SessionLive は agent の session_live コマンドを上書きします。 |
| `overlay_dir` | string |  |  | OverlayDir は agent の overlay_dir パスを上書きします。起動時に内容を agent の working directory に加算的にコピーします。相対パスは city ディレクトリを基準に解決されます。 |
| `default_sling_formula` | string |  |  | DefaultSlingFormula はデフォルト sling formula を上書きします。 |
| `inject_fragments` | []string |  |  | InjectFragments は agent の inject_fragments リストを上書きします。 |
| `append_fragments` | []string |  |  | AppendFragments は agent の append_fragments リストを上書きします。 |
| `attach` | boolean |  |  | Attach は agent の attach 設定を上書きします。 |
| `depends_on` | []string |  |  | DependsOn は agent の依存関係リストを上書きします。 |
| `resume_command` | string |  |  | ResumeCommand は agent の resume_command テンプレートを上書きします。 |
| `wake_mode` | string |  |  | WakeMode は agent の wake mode ("resume" または "fresh") を上書きします。Enum: `resume`、`fresh` |
| `pre_start_append` | []string |  |  | PreStartAppend は agent の pre_start リストにコマンドを追加します (置換ではなく)。両方が設定されている場合 PreStart の後に適用されます。 |
| `session_setup_append` | []string |  |  | SessionSetupAppend は agent の session_setup リストにコマンドを追加します。 |
| `session_live_append` | []string |  |  | SessionLiveAppend は agent の session_live リストにコマンドを追加します。 |
| `install_agent_hooks_append` | []string |  |  | InstallAgentHooksAppend は agent の install_agent_hooks リストに追加します。 |
| `inject_fragments_append` | []string |  |  | InjectFragmentsAppend は agent の inject_fragments リストに追加します。 |
| `max_active_sessions` | integer |  |  | MaxActiveSessions は同時 session に対する agent レベルの上限を上書きします。 |
| `min_active_sessions` | integer |  |  | MinActiveSessions は維持する session の最小数を上書きします。 |
| `scale_check` | string |  |  | ScaleCheck は bead-backed reconciliation のための新しい未割当 session 需要を出力で報告するコマンドテンプレートを上書きします。Agent.scale_check と同じ Go テンプレートプレースホルダーをサポートします。 |
| `option_defaults` | map[string]string |  |  | OptionDefaults はこの agent の provider オプションデフォルトを追加または上書きします。キーはオプションキー、値は choice 値です。加算的にマージされます (patch キーが既存の agent キーより優先)。例: option_defaults = &#123; model = "sonnet" &#125; |

## BeadsConfig

BeadsConfig は bead store の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `provider` | string |  | `bd` | Provider は bead store のバックエンドを選択します: "bd" (デフォルト)、"file"、または user 提供スクリプトの "exec:&lt;script&gt;"。 |

## ChatSessionsConfig

ChatSessionsConfig は chat session の動作を設定します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `idle_timeout` | string |  |  | IdleTimeout はデタッチされた chat session が自動 suspend されるまでの時間です。Duration 文字列 (例: "30m"、"1h")。0 = 無効。 |

## ConvergenceConfig

ConvergenceConfig は収束ループの上限を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `max_per_agent` | integer |  | `2` | MaxPerAgent は agent ごとのアクティブな収束ループの最大数です。0 はデフォルト (2) を使用することを意味します。 |
| `max_total` | integer |  | `10` | MaxTotal はアクティブな収束ループの総数の最大値です。0 はデフォルト (10) を使用することを意味します。 |

## DaemonConfig

DaemonConfig は controller daemon の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `formula_v2` | boolean |  |  | FormulaV2 は formula v2 graph workflow インフラストラクチャを有効化します: control-dispatcher implicit agent、graph.v2 formula コンパイル、batch graph-apply bead 作成。--graph サポート付きの bd が必要です。デフォルト: false (機能の安定化中はオプトイン)。 |
| `graph_workflows` | boolean |  |  | GraphWorkflows は FormulaV2 の非推奨の前身です。後方互換のために保持されています: TOML で graph_workflows が true で formula_v2 が未設定の場合、パース中に FormulaV2 が自動的に昇格されます。 |
| `patrol_interval` | string |  | `30s` | PatrolInterval はヘルス patrol 間隔です。Duration 文字列 (例: "30s"、"5m"、"1h")。デフォルトは "30s"。 |
| `max_restarts` | integer |  | `5` | MaxRestarts は agent が隔離されるまでの RestartWindow 内の agent 再起動の最大数です。0 は無制限を意味します (クラッシュループ検出なし)。デフォルトは 5。 |
| `restart_window` | string |  | `1h` | RestartWindow は再起動をカウントするスライディング時間ウィンドウです。Duration 文字列 (例: "30s"、"5m"、"1h")。デフォルトは "1h"。 |
| `shutdown_timeout` | string |  | `5s` | ShutdownTimeout はシャットダウン中に agent を強制終了する前に Ctrl-C を送信した後の待機時間です。Duration 文字列 (例: "5s"、"30s")。即時 kill の場合は "0s" に設定。デフォルトは "5s"。 |
| `wisp_gc_interval` | string |  |  | WispGCInterval は wisp GC が実行される頻度です。Duration 文字列 (例: "5m"、"1h")。WispGCInterval と WispTTL の両方が設定されない限り、wisp GC は無効です。 |
| `wisp_ttl` | string |  |  | WispTTL は閉じた molecule が purge されるまで残存する時間です。Duration 文字列 (例: "24h"、"7d")。WispGCInterval と WispTTL の両方が設定されない限り、wisp GC は無効です。 |
| `drift_drain_timeout` | string |  | `2m` | DriftDrainTimeout は config-drift 再起動中に agent が drain シグナルを ack するまでの最大待機時間です。このウィンドウ内で agent が ack しない場合、controller は強制終了して再起動します。Duration 文字列 (例: "2m"、"5m")。デフォルトは "2m"。 |
| `observe_paths` | []string |  |  | ObservePaths は Claude JSONL session ファイル (例: aimux session パス) を検索する追加ディレクトリをリストします。デフォルトの検索パス (~/.claude/projects/) は常に含まれます。 |
| `probe_concurrency` | integer |  | `8` | ProbeConcurrency は pool scale_check と work_query パスで発行される bd サブプロセス probe の同時実行数を制限します。bd は共有 dolt sql-server 上でシリアライズされるため、無制限の並列性は競合を引き起こします。Nil (未設定) はデフォルト 8 です。専用の高速 dolt サーバーがある workspace では高めに、低速ストレージで競合を減らしたい場合は低めに設定します。 |
| `max_wakes_per_tick` | integer |  | `5` | MaxWakesPerTick は reconciler が単一 tick で起動できる session 数を制限します。Nil (未設定) はデフォルト 5 です。&lt;= 0 の値はデフォルトとして扱われます — 上書きするには正の整数を設定してください。 |

## DoltConfig

DoltConfig はオプションの dolt サーバーオーバーライドを保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `port` | integer |  | `0` | Port は dolt サーバーのポートです。0 は ephemeral ポート割り当てを使用 (city パスからハッシュ化) を意味します。明示的に設定すると上書きします。 |
| `host` | string |  | `localhost` | Host は dolt サーバーのホスト名です。デフォルトは localhost。 |

## EventsConfig

EventsConfig は events provider の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `provider` | string |  |  | Provider は events のバックエンドを選択します: "fake"、"fail"、"exec:&lt;script&gt;"、または "" (デフォルト: file-backed JSONL)。 |

## FormulasConfig

FormulasConfig は formula ディレクトリの設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `dir` | string |  | `formulas` | Dir は formulas ディレクトリへのパスです。デフォルトは "formulas"。 |

## Import

Import は別の pack の名前付き import を定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `source` | string | **yes** |  | Source は pack の場所です: ローカル相対パス (例: "./assets/imports/gastown") またはリモート URL (例: "github.com/gastownhall/gastown")。ローカルパスにはバージョンがありません。 |
| `version` | string |  |  | Version はリモート import の semver 制約です (例: "^1.2")。ローカルパスでは空。"sha:&lt;hex&gt;" でコミットを固定。 |
| `export` | boolean |  |  | Export はこの import の内容を親 pack の名前空間に再エクスポートします。親の consumer はこの import の agent を親の binding 名の下にフラット化して取得します。 |
| `transitive` | boolean |  |  | Transitive はこの import 自身の imports が consumer に見えるかを制御します。デフォルトは true (transitive)。この特定の import の transitive 解決を抑制するには false に設定。 |
| `shadow` | string |  |  | Shadow は importer がこの import の agent と同じ名前の agent を定義したときの shadow 警告を制御します。"warn" (デフォルト) は警告を発し、"silent" は抑制します。Enum: `warn`、`silent` |

## K8sConfig

K8sConfig はネイティブ K8s session provider の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `namespace` | string |  | `gc` | Namespace は agent pod の K8s namespace です。デフォルト: "gc"。 |
| `image` | string |  |  | Image は agent のコンテナイメージです。 |
| `context` | string |  |  | Context は kubectl/kubeconfig コンテキストです。デフォルト: 現在のもの。 |
| `cpu_request` | string |  | `500m` | CPURequest は pod の CPU リクエストです。デフォルト: "500m"。 |
| `mem_request` | string |  | `1Gi` | MemRequest は pod のメモリリクエストです。デフォルト: "1Gi"。 |
| `cpu_limit` | string |  | `2` | CPULimit は pod の CPU 制限です。デフォルト: "2"。 |
| `mem_limit` | string |  | `4Gi` | MemLimit は pod のメモリ制限です。デフォルト: "4Gi"。 |
| `prebaked` | boolean |  |  | Prebaked は true のとき init コンテナのステージングと EmptyDir ボリュームをスキップします。City content がベイクされた `gc build-image` で構築されたイメージで使用します。 |

## MailConfig

MailConfig は mail provider の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `provider` | string |  |  | Provider は mail のバックエンドを選択します: "fake"、"fail"、"exec:&lt;script&gt;"、または "" (デフォルト: beadmail)。 |

## NamedSession

NamedSession は agent テンプレートに支えられた正典的な永続 session を定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string |  |  | Name は設定された公開 session identity です。省略時は Template が互換性 identity として残ります。 |
| `template` | string | **yes** |  | Template は参照される agent テンプレート名です。Root 宣言は "binding.agent" を介して import された PackV2 agent をターゲットにできます。 |
| `scope` | string |  |  | Scope は pack 展開でこの named session がインスタンス化される場所を定義します: "city" (city ごとに 1 つ) または "rig" (rig ごとに 1 つ)。Enum: `city`、`rig` |
| `dir` | string |  |  | Dir は pack 展開後の rig スコープ named session の identity prefix です。空は city スコープを意味します。 |
| `mode` | string |  |  | Mode はこの named session の controller 動作を制御します。"on_demand" (デフォルト): identity を予約し、work または明示的参照が必要としたときに materialize します。"always": 正典的な session を controller 管理下に保ちます。Enum: `on_demand`、`always` |

## OptionChoice

OptionChoice は "select" オプションに許可される 1 つの値です。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `value` | string | **yes** |  |  |
| `label` | string | **yes** |  |  |
| `flag_args` | []string | **yes** |  | FlagArgs はこの choice が選択されたときに注入される CLI 引数です。json:"-" は意図的です: FlagArgs は公開 API DTO に決して現れてはなりません (セキュリティ境界 — クライアントが内部 CLI フラグを見ることを防ぎます)。 |
| `flag_aliases` | []array |  |  | FlagAliases はレガシー provider の args から削除される等価な CLI 引数シーケンスです。FlagArgs と同様、サーバー側のみに留まります。 |

## OrderOverride

OrderOverride はスキャンされた order のスケジューリングフィールドを変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name はターゲットとする order 名です (必須)。 |
| `rig` | string |  |  | Rig はオーバーライドを特定の rig の order にスコープします。空は city レベルの order にマッチします。 |
| `enabled` | boolean |  |  | Enabled は order がアクティブかを上書きします。 |
| `trigger` | string |  |  | Trigger は trigger タイプを上書きします。 |
| `gate` | string |  |  | Gate は gate→trigger 移行中に受け付けられる Trigger の非推奨別名です。パースされた入力は Trigger に正規化されます。 |
| `interval` | string |  |  | Interval はクールダウン間隔を上書きします。Go duration 文字列。 |
| `schedule` | string |  |  | Schedule は cron 式を上書きします。 |
| `check` | string |  |  | Check は条件 trigger チェックコマンドを上書きします。 |
| `on` | string |  |  | On はイベント trigger のイベントタイプを上書きします。 |
| `pool` | string |  |  | Pool はターゲットの session 設定を上書きします。 |
| `timeout` | string |  |  | Timeout は order ごとのタイムアウトを上書きします。Go duration 文字列。 |

## OrdersConfig

OrdersConfig は order の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `skip` | []string |  |  | Skip はスキャンから除外する order 名をリストします。 |
| `max_timeout` | string |  |  | MaxTimeout は order ごとのタイムアウトに対するオペレータのハードキャップです。どの order もこの duration を超えません。Go duration 文字列 (例: "60s")。空は上限なし (上書きなし) を意味します。 |
| `overrides` | []OrderOverride |  |  | Overrides はスキャン後に order ごとのフィールドオーバーライドを適用します。各オーバーライドは order を名前で、オプションで rig でターゲットします。 |

## PackSource

PackSource はリモート pack リポジトリを定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `source` | string | **yes** |  | Source は git リポジトリの URL です。 |
| `ref` | string |  |  | Ref はチェックアウトする git ref (branch、tag、commit) です。デフォルトは HEAD。 |
| `path` | string |  |  | Path は pack ファイルを含むリポジトリ内のサブディレクトリです。 |

## Patches

Patches は合成からのすべての patch ブロックを保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `agent` | []AgentPatch |  |  | Agents は (dir、name) で agents をターゲットします。 |
| `rigs` | []RigPatch |  |  | Rigs は名前で rigs をターゲットします。 |
| `providers` | []ProviderPatch |  |  | Providers は名前で providers をターゲットします。 |

## PoolOverride

PoolOverride は session スケーリングにマップされる legacy `[pool]` フィールドを変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `min` | integer |  |  | Min は session の最小数を上書きします。 |
| `max` | integer |  |  | Max は session の最大数を上書きします。0 はルーティングされた work をクレームできる session がないことを意味します。 |
| `check` | string |  |  | Check は session スケールチェックコマンドテンプレートを上書きします。Agent.scale_check と同じ Go テンプレートプレースホルダーをサポートします。 |
| `drain_timeout` | string |  |  | DrainTimeout は drain タイムアウトを上書きします。Duration 文字列 (例: "5m"、"30m"、"1h")。 |
| `on_death` | string |  |  | OnDeath は on_death コマンドテンプレートを上書きします。Agent.on_death と同じ Go テンプレートプレースホルダーをサポートします。 |
| `on_boot` | string |  |  | OnBoot は on_boot コマンドテンプレートを上書きします。Agent.on_boot と同じ Go テンプレートプレースホルダーをサポートします。 |

## ProviderOption

ProviderOption は provider に対する単一の設定可能オプションを宣言します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `key` | string | **yes** |  |  |
| `label` | string | **yes** |  |  |
| `type` | string | **yes** |  | "select" のみ (v1) |
| `default` | string | **yes** |  |  |
| `choices` | []OptionChoice | **yes** |  |  |
| `omit` | boolean |  |  | Omit は options_schema_merge = "by_key" の削除センチネルです。子レイヤーのエントリで設定された場合、親レイヤーから継承された一致する Key は解決スキーマから削除されます。 |

## ProviderPatch

ProviderPatch は Name で識別される既存の provider を変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name はターゲット指定キーです (必須)。既存の provider の名前と一致する必要があります。 |
| `base` | string |  |  | Base は provider の継承親を上書きします (presence-aware)。Patch が "変更なし" (double-nil)、"継承デフォルトにクリア" (外側ポインタの single-nil 値)、"明示的な空 opt-out に設定" (内側ポインタの値 "")、"&lt;name&gt; に設定" を区別できるように、ポインタへのポインタです。呼び出し側は次を使用します:   nil          = patch は Base に触れない   &(*string)(nil) = patch は Base を absent にクリア   &(&"")       = patch は Base = "" に設定 (明示的 opt-out)   &(&"builtin:codex") = patch は Base をその値に設定 |
| `command` | string |  |  | Command は provider command を上書きします。 |
| `acp_command` | string |  |  | ACPCommand は ACP トランスポート session の provider command を上書きします。 |
| `args` | []string |  |  | Args は provider の args を上書きします。 |
| `acp_args` | []string |  |  | ACPArgs は ACP トランスポート session の provider の args を上書きします。 |
| `args_append` | []string |  |  | ArgsAppend は provider の args_append リストを上書きします。 |
| `options_schema_merge` | string |  |  | OptionsSchemaMerge は options_schema のマージモードを上書きします。 |
| `prompt_mode` | string |  |  | PromptMode は prompt 配信モードを上書きします。Enum: `arg`、`flag`、`none` |
| `prompt_flag` | string |  |  | PromptFlag は prompt フラグを上書きします。 |
| `ready_delay_ms` | integer |  |  | ReadyDelayMs はミリ秒単位の ready 遅延を上書きします。 |
| `env` | map[string]string |  |  | Env は環境変数を追加または上書きします。 |
| `env_remove` | []string |  |  | EnvRemove は削除する env 変数キーをリストします。 |
| `_replace` | boolean |  |  | Replace は深いマージではなく provider ブロック全体を置換します。 |

## ProviderSpec

ProviderSpec は名前付き provider の起動パラメータを定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `base` | string |  |  | Base はこの spec が継承する親 provider を指名します。サポートされる形式:   "&lt;name&gt;"          - カスタム優先 (自身を除外)、次に組み込み   "builtin:&lt;name&gt;"  - 組み込みルックアップを強制   "provider:&lt;name&gt;" - カスタムルックアップを強制   ""                - 明示的にスタンドアロン opt-out   nil               - フィールド不在; 明示的宣言なし |
| `args_append` | []string |  |  | ArgsAppend は各レイヤーの Args 置換後に追加 args を蓄積します。 |
| `options_schema_merge` | string |  |  | OptionsSchemaMerge はチェーン全体の OptionsSchema マージモードを制御します: "replace" (デフォルト) または "by_key"。Enum: `replace`、`by_key` |
| `display_name` | string |  |  | DisplayName は UI とログに表示される人間可読名です。 |
| `command` | string |  |  | Command はこの provider に対して実行する実行可能ファイルです。 |
| `args` | []string |  |  | Args は provider に渡されるデフォルトのコマンドライン引数です。 |
| `prompt_mode` | string |  | `arg` | PromptMode は prompt の配信方法を制御します: "arg"、"flag"、または "none"。Enum: `arg`、`flag`、`none` |
| `prompt_flag` | string |  |  | PromptFlag は prompt_mode が "flag" のときに使用される CLI フラグです (例: "--prompt")。 |
| `ready_delay_ms` | integer |  |  | ReadyDelayMs は launch 後 provider が ready とみなされるまでの待機ミリ秒数です。 |
| `ready_prompt_prefix` | string |  |  | ReadyPromptPrefix は provider が入力可能であることを示す文字列の接頭辞です。 |
| `process_names` | []string |  |  | ProcessNames は provider が実行中かをチェックする際に探すプロセス名をリストします。 |
| `emits_permission_warning` | boolean |  |  | EmitsPermissionWarning は三状態です: nil = 継承、&true = 有効、&false = 明示的に無効。 |
| `env` | map[string]string |  |  | Env は provider プロセスの追加環境変数を設定します。 |
| `path_check` | string |  |  | PathCheck は PATH 検出で使用されるバイナリ名を上書きします。設定されている場合、lookupProvider と detectProviderName は exec.LookPath チェックで Command の代わりにこれを使用します。Command が shell ラッパー (例: sh -c '...') だが実際のバイナリがインストールされていることを検証する必要がある場合に有用です。 |
| `supports_acp` | boolean |  |  | SupportsACP はバイナリが Agent Client Protocol (stdio 上の JSON-RPC 2.0) を話すことを示します。Agent が session = "acp" を設定する場合、解決された provider は SupportsACP = true である必要があります。 |
| `supports_hooks` | boolean |  |  | SupportsHooks は provider がライフサイクルイベント用の実行可能 hook 機構 (settings.json、plugins など) を持つことを示します。 |
| `instructions_file` | string |  |  | InstructionsFile は provider がプロジェクト指示として読むファイル名です (例: "CLAUDE.md"、"AGENTS.md")。空はデフォルトで "AGENTS.md" になります。 |
| `resume_flag` | string |  |  | ResumeFlag は ID で session を resume するための CLI フラグです。空は provider が resume をサポートしないことを意味します。例: "--resume" (claude)、"resume" (codex) |
| `resume_style` | string |  |  | ResumeStyle は ResumeFlag がどのように適用されるかを制御します:   "flag"       → command --resume &lt;key&gt;              (デフォルト)   "subcommand" → command resume &lt;key&gt; |
| `resume_command` | string |  |  | ResumeCommand は session を resume するときに実行する完全な shell コマンドです。&#123;&#123;.SessionKey&#125;&#125; テンプレート変数をサポートします。設定されている場合、ResumeFlag/ResumeStyle より優先されます。例:   "claude --resume &#123;&#123;.SessionKey&#125;&#125; --dangerously-skip-permissions" |
| `session_id_flag` | string |  |  | SessionIDFlag は特定の ID で session を作成するための CLI フラグです。Session キー管理の Generate & Pass 戦略を有効にします。例: "--session-id" (claude) |
| `permission_modes` | map[string]string |  |  | PermissionModes は permission モード名を CLI フラグにマップします。例: &#123;"unrestricted": "--dangerously-skip-permissions", "plan": "--permission-mode plan"&#125;。これは外部クライアント (例: Mission Control) が permission モードドロップダウンを populate するために使用する設定専用ルックアップテーブルです。Launch 時のフラグ置換はフォローアップ PR で計画されています — 現在ランタイムコードはこのフィールドを読みません。 |
| `option_defaults` | map[string]string |  |  | OptionDefaults は OptionsSchema エントリの Default 値をスキーマ自体を再定義せずに上書きします。キーはオプションキー (例: "permission_mode")、値は choice 値 (例: "unrestricted") です。city.toml ユーザーは Args や OptionsSchema に触れずに provider 動作をカスタマイズするためにこれを設定します。 |
| `options_schema` | []ProviderOption |  |  | OptionsSchema はこの provider がサポートする設定可能オプションを宣言します。各オプションは Choices[].FlagArgs フィールドを介して CLI 引数にマップされます。FlagArgs がサーバー側のままであるよう、専用 DTO 経由でシリアライズされます (JSON に直接ではなく)。 |
| `print_args` | []string |  |  | PrintArgs はワンショット非対話モードを有効にする CLI 引数です。Provider はレスポンスを stdout に表示して終了します。空のとき、provider はワンショット呼び出しをサポートしません。例: ["-p"] (claude、gemini)、["exec"] (codex) |
| `title_model` | string |  |  | TitleModel はタイトル生成に使用される OptionsSchema の model キーです。OptionsSchema の "model" オプションを介して FlagArgs を取得するように解決されます。各 provider のデフォルトは最も安価/高速なモデルです。例: "haiku" (claude)、"o4-mini" (codex)、"gemini-2.5-flash" (gemini) |
| `acp_command` | string |  |  | ACPCommand は session トランスポートが ACP のとき Command を上書きします。空のとき、Command は tmux と ACP の両トランスポートに使用されます。 |
| `acp_args` | []string |  |  | ACPArgs は session トランスポートが ACP のとき Args を上書きします。Nil のとき、Args は tmux と ACP の両トランスポートに使用されます。 |

## Rig

Rig は city に登録された外部プロジェクトを定義します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name はこの rig の一意識別子です。 |
| `path` | string |  |  | Path は rig のリポジトリへの絶対ファイルシステムパスです。 |
| `prefix` | string |  |  | Prefix はこの rig の自動派生される bead ID prefix を上書きします。 |
| `suspended` | boolean |  |  | Suspended は reconciler がこの rig で agent を spawn することを防ぎます。gc rig suspend/resume で切り替えます。 |
| `formulas_dir` | string |  |  | FormulasDir は rig ローカルな formula ディレクトリです (Layer 4)。この rig に対して pack の formulas をファイル名で上書きします。相対パスは city ディレクトリを基準に解決されます。 |
| `includes` | []string |  |  | Includes はこの rig の pack ディレクトリまたは URL をリストします (V1 の仕組み)。各エントリはローカルパス、git source//sub#ref URL、または GitHub tree URL です。 |
| `imports` | map[string]Import |  |  | Imports はこの rig の名前付き pack imports を定義します (V2 の仕組み)。各キーは binding 名で、これらの imports からの agent は "rigName/bindingName.agentName" のような qualified 名を取得します。 |
| `max_active_sessions` | integer |  |  | MaxActiveSessions はこの rig 内のすべての agent にわたる総同時 session 数に対する rig レベルの上限です。Nil は workspace から継承 (または無制限) を意味します。 |
| `overrides` | []AgentOverride |  |  | Overrides は pack 展開後に適用される agent ごとの patch です。V2 では `[[patches.agent]]` との一貫性のために "patches" にリネームされます。両方の TOML キーが移行中は受け付けられます。 |
| `patches` | []AgentOverride |  |  | Patches は rig レベルの agent オーバーライドの V2 名です。両方が設定されている場合 Overrides より優先されます。 |
| `default_sling_target` | string |  |  | DefaultSlingTarget は gc sling が bead ID のみで (明示ターゲットなしで) 呼び出されたときに使用される agent qualified 名です。resolveAgentIdentity 経由で解決されます。例: "rig/polecat" |
| `session_sleep` | SessionSleepConfig |  |  | SessionSleep はこの rig の agent に対して workspace レベルのアイドル sleep デフォルトを上書きします。 |
| `dolt_host` | string |  |  | DoltHost はこの rig の bead に対して city レベルの Dolt host を上書きします。Rig のデータベースが別の Dolt サーバー上に存在する場合 (例: 別の city から共有される場合) に使用します。 |
| `dolt_port` | string |  |  | DoltPort はこの rig の bead に対して city レベルの Dolt ポートを上書きします。設定されている場合、controller コマンド (scale_check、work_query) は shell 呼び出しに `BEADS_DOLT_SERVER_PORT=<port>` を前置するので、bd は city レベルのデフォルトではなく正しいサーバーに接続します。 |

## RigPatch

RigPatch は Name で識別される既存の rig を変更します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name はターゲット指定キーです (必須)。既存の rig の名前と一致する必要があります。 |
| `path` | string |  |  | Path は rig のファイルシステムパスを上書きします。 |
| `prefix` | string |  |  | Prefix は bead ID prefix を上書きします。 |
| `suspended` | boolean |  |  | Suspended は rig の suspended state を上書きします。 |

## Service

Service は /svc/{name} 配下にマウントされる workspace 所有の HTTP service を宣言します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string | **yes** |  | Name は workspace 内での一意な service 識別子です。 |
| `kind` | string |  |  | Kind は service の実装方法を選択します。Enum: `workflow`、`proxy_process` |
| `publish_mode` | string |  |  | PublishMode は service の発行意図を宣言します。v0 はプライベート service と API listener の直接再利用をサポートします。Enum: `private`、`direct` |
| `state_root` | string |  |  | StateRoot は管理対象 service の state root を上書きします。デフォルトは .gc/services/&#123;name&#125;。パスは .gc/services/ 内に留まる必要があります。 |
| `publication` | ServicePublicationConfig |  |  | Publication は汎用的な発行意図を宣言します。プラットフォームがその意図を公開ルートにするかどうか、どのようにするかを決定します。 |
| `workflow` | ServiceWorkflowConfig |  |  | Workflow は controller 所有の workflow service を設定します。 |
| `process` | ServiceProcessConfig |  |  | Process は controller 監視のプロキシ service を設定します。 |

## ServiceProcessConfig

ServiceProcessConfig は /svc/{name} 配下でリバースプロキシされる controller 監視のローカルプロセスを設定します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `command` | []string |  |  | Command はローカル service プロセスを起動するために使用される argv です。 |
| `health_path` | string |  |  | HealthPath は設定されているとき、service が ready とマークされる前にローカル listener で probe されます。 |

## ServicePublicationConfig

ServicePublicationConfig はプラットフォーム中立な発行意図を宣言します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `visibility` | string |  |  | Visibility は service が workspace に対してプライベート、公開可能、またはプラットフォームエッジで tenant 認証によりゲートされるかを選択します。Enum: `private`、`public`、`tenant` |
| `hostname` | string |  |  | Hostname は service.name から派生されたデフォルトホスト名ラベルを上書きします。 |
| `allow_websockets` | boolean |  |  | AllowWebSockets は発行ルート上での websocket アップグレードを許可します。 |

## ServiceWorkflowConfig

ServiceWorkflowConfig は controller 所有の workflow service を設定します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `contract` | string |  |  | Contract は組み込み workflow ハンドラを選択します。 |

## SessionConfig

SessionConfig は session provider の設定を保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `provider` | string |  |  | Provider は session のバックエンドを選択します: "fake"、"fail"、"subprocess"、"acp"、"exec:&lt;script&gt;"、"k8s"、または "" (デフォルト: tmux)。 |
| `k8s` | K8sConfig |  |  | K8s はネイティブ K8s provider 用の Kubernetes 固有設定を保持します。 |
| `acp` | ACPSessionConfig |  |  | ACP は ACP (Agent Client Protocol) session provider の設定を保持します。 |
| `setup_timeout` | string |  | `10s` | SetupTimeout は session セットアップと pre_start コマンドのコマンド/script ごとのタイムアウトです。Duration 文字列 (例: "10s"、"30s")。デフォルトは "10s"。 |
| `nudge_ready_timeout` | string |  | `10s` | NudgeReadyTimeout は nudge テキストを送信する前に agent が ready になるのを待つ時間です。Duration 文字列。デフォルトは "10s"。 |
| `nudge_retry_interval` | string |  | `500ms` | NudgeRetryInterval は nudge readiness ポーリング間のリトライ間隔です。Duration 文字列。デフォルトは "500ms"。 |
| `nudge_lock_timeout` | string |  | `30s` | NudgeLockTimeout は session ごとの nudge ロックを取得するための待機時間です。Duration 文字列。デフォルトは "30s"。 |
| `debounce_ms` | integer |  | `500` | DebounceMs は send-keys のデフォルト debounce 間隔 (ミリ秒) です。デフォルトは 500。 |
| `display_ms` | integer |  | `5000` | DisplayMs は status メッセージのデフォルト表示時間 (ミリ秒) です。デフォルトは 5000。 |
| `startup_timeout` | string |  | `60s` | StartupTimeout は各 agent の Start() 呼び出しを失敗とみなす前の待機時間です。Duration 文字列 (例: "60s"、"2m")。デフォルトは "60s"。 |
| `socket` | string |  |  | Socket は per-city 隔離のための tmux ソケット名を指定します。設定されている場合、すべての tmux コマンドは "tmux -L <socket>" を使用して専用サーバーに接続します。空のとき、デフォルトは city 名 (workspace.name) — すべての city が自動的に独自の tmux サーバーを持つようになります。明示的に設定して上書きします。 |
| `remote_match` | string |  |  | RemoteMatch は hybrid provider が session をリモート (K8s) バックエンドにルーティングするための部分文字列パターンです。名前にこのパターンを含む session は K8s に行き、それ以外はローカル (tmux) に留まります。設定されている場合 GC_HYBRID_REMOTE_MATCH 環境変数で上書きされます。 |

## SessionSleepConfig

SessionSleepConfig は session クラスごとのデフォルトアイドル sleep ポリシーを設定します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `interactive_resume` | string |  |  | InteractiveResume は wake_mode=resume を使用するアタッチ可能な session に適用されます。Duration 文字列または "off" を受け付けます。 |
| `interactive_fresh` | string |  |  | InteractiveFresh は wake_mode=fresh を使用するアタッチ可能な session に適用されます。Duration 文字列または "off" を受け付けます。 |
| `noninteractive` | string |  |  | NonInteractive は attach=false の session に適用されます。Duration 文字列または "off" を受け付けます。 |

## Workspace

Workspace は city レベルのメタデータと、agent ごとに上書きされない限りすべての agent に適用されるオプションのデフォルトを保持します。

| フィールド | 型 | 必須 | デフォルト | 説明 |
|-------|------|----------|---------|-------------|
| `name` | string |  |  | Name はレガシーのチェックイン済み city 名です。ランタイム identity は今や代わりに site binding (.gc/site.toml workspace_name)、宣言された設定、basename の優先順位から解決されます; gc init は machine-local 名を site.toml に書き込み、city.toml からは省略します。 |
| `prefix` | string |  |  | Prefix は自動派生される HQ bead ID prefix を上書きします。空のとき、prefix は DeriveBeadsPrefix を介して city Name から派生されます。 |
| `provider` | string |  |  | Provider は指定しない agent が使用するデフォルトの provider 名です。 |
| `start_command` | string |  |  | StartCommand はすべての agent に対して provider のコマンドを上書きします。 |
| `suspended` | boolean |  |  | Suspended は city が suspend されているかを制御します。True のとき、すべての agent は事実上 suspend されます: reconciler はそれらを spawn せず、gc hook/prime は空を返します。下方向に継承されます — 個別の agent/rig の suspended フィールドは独立にチェックされます。 |
| `max_active_sessions` | integer |  |  | MaxActiveSessions は総同時 session 数に対する workspace レベルの上限です。Nil は無制限を意味します。Agent と rig は独自に設定しない場合これを継承します。 |
| `session_template` | string |  |  | SessionTemplate は次のプレースホルダーをサポートするテンプレート文字列です: &#123;&#123;.City&#125;&#125;、&#123;&#123;.Agent&#125;&#125; (sanitize 済み)、&#123;&#123;.Dir&#125;&#125;、&#123;&#123;.Name&#125;&#125;。Tmux session の命名を制御します。デフォルト (空): "&#123;&#123;.Agent&#125;&#125;" — sanitize 済み agent 名のみ。Per-city な tmux ソケット隔離により city prefix は不要です。 |
| `install_agent_hooks` | []string |  |  | InstallAgentHooks は hook を agent の working directory にインストールすべき provider 名をリストします。Agent レベルが workspace レベルを上書きします (置換、非加算)。サポート: "claude"、"codex"、"gemini"、"opencode"、"copilot"、"cursor"、"pi"、"omp"。 |
| `global_fragments` | []string |  |  | GlobalFragments はすべての agent のレンダリング済み prompt に注入される名前付き template fragments をリストします。Agent ごとの InjectFragments の前に適用されます。各名前は pack の prompts/shared/ ディレクトリの &#123;&#123; define "name" &#125;&#125; ブロックと一致する必要があります。 |
| `includes` | []string |  |  | Includes はこの workspace に合成する pack ディレクトリまたは URL をリストします。古い pack/packs フィールドを置き換えます。各エントリはローカルパス、git source//sub#ref URL、または GitHub tree URL です。 |
| `default_rig_includes` | []string |  |  | DefaultRigIncludes は --include なしで "gc rig add" が呼び出されたときに新しい rig に適用される pack ディレクトリをリストします。City がすべての rig に対するデフォルト pack を定義できるようにします。 |
