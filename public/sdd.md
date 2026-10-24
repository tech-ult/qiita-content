---
title: Spec-Driven Development(SDD：仕様駆動開発)でAIに的確に仕様を伝える
tags:
  - ソフトウェア開発
  - AIエージェント
  - github-spec-kit
private: false
updated_at: '2026-10-24T23:40:07+09:00'
id: bf2243eb20e877899cda
organization_url_name: tech-ult
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/ee9d50e9-9f5b-42ba-a673-d951408467fa.png)

AIエージェントの開発を行う際、要求仕様を的確に伝えることで、希望に近いアプリケーションを開発させることができます。

仕様（ゴール）を決めず、バイブコーディングでAIエージェントに都度指示をして開発を進めていく方法は、プロトタイプや小規模のアプリケーションの場合はスピード感を持って開発に臨めるメリットがあります。

デメリットは、ゴールを決めていないので、誰も正解や完了を判断できずレビューのしようもなく、品質面でも粗が出るため、業務向けのアプリケーション開発には不向きです。

AIと一緒に仕様を決めていけば、最初からガチガチに要求仕様のドキュメントを作成する時間もない場合に便利です。今回は仕様駆動開発ツール「GitHub Spec Kit」を紹介します。

# 前提条件
AIエージェントにはGitHub Copilotを使用します。

また、GitHub Spec Kitのランタイムとして、Python 3.11以上とuv（Pythonのパッケージ管理ツール）が必要になるため、事前にWindowsにインストールしておく必要があります。

<details><summary>Pythonとuvのインストール</summary>
WindowsにPython 3.11以上、uvがインストールされていない場合のインストール手順は以下の通りです。

### Pythonインストール

#### 1. WinGetでPythonイメージの検索

```powershell
winget search Python.Python
```

#### 2. Pythonのインストール（例：Python 3.13をインストールする場合）

```powershell
winget install --id Python.Python.3.13 -e
```

#### 3. インストールしたバージョンの確認
Python 3.13.x と出力されていればOK。

```powershell
python --version
```

### uvインストール

#### 1. WinGetでuvのインストール

```powershell
winget install --id astral-sh.uv -e
```

#### 2. インストールしたバージョンの確認
`uv 0.x.x`と出力されていればOK。

```powershell
uv --version
```

</details>

# 解説
## GitHub Spec Kit について
### Spec Kitの概要
#### AIエージェントに使わせる開発ワークフローツール
![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/d1cf0313-cd84-4621-ad71-015a8ff8a51c.png)

GitHub Spec Kitは、単体では使えません。Claude Code、GitHub Copilot、Gemini CLIなどの既存エージェントに、構造化されたプロセスを与えるツールキットです。AIエージェントと組み合わせることで、初めて力を発揮します。

Spec Kitは、AIコーディングエージェントに開発ワークフロー「**要件定義→設計→タスク分割→実装→設計との整合性確認**」という規律あるSDD（Spec-Driven Development）の型を与える、GitHub公式のオープンソース・ツールキットです。

#### テンプレートをもとに決められた構成でドキュメント作成
Spec Kitは、コード生成そのものより、**エージェントが勝手に実装しないように、仕様と根拠を先にMarkdown（MD）ファイルを成果物として作成する**点が特徴です。

Spec Kitを使わなくても、途中の意思決定をドキュメントに残せば、別の開発者に設計の根拠を示すことはできますが、Spec Kitでは決められたディレクトリ構造に、決められた順にドキュメントを残していくので、どこにどのドキュメントがあるのか把握しやすく、ドキュメントをどの順に読めば良いのかもわかります。

```
my-app/
├─ .specify/
│  ├─ memory/
│  │  └─ constitution.md          # プロジェクト横断の原則・品質基準
│  ├─ templates/                  # spec / plan / tasksなどの雛形
│  ├─ scripts/                    # 補助スクリプト
│  └─ feature.json                # アクティブな機能ディレクトリ情報など
│
├─ specs/
│  ├─ 001-user-auth/              # 初回開発：ユーザー認証
│  │  ├─ spec.md                  # 何を、なぜ作るか
│  │  ├─ plan.md                  # どう作るか
│  │  ├─ tasks.md                 # 何を順に実装するか
│  │  ├─ data-model.md            # データモデルを定義
│  │  ├─ research.md              # 技術的な調査・意思決定記録
│  │  ├─ quickstart.md            # 実装後に機能を検証するための手順書
│  │  └─ contracts/               # API契約など
│  │     ├─ auth-api.openapi.yaml
│  │     └─ auth-events.md
│  │
│  ├─ 002-password-reset/         # 追加開発①：パスワードリセット
│  │  ├─ spec.md
│  │  ├─ plan.md
│  │  ├─ tasks.md
│  │  └─ contracts/
│  │     └─ password-reset-api.openapi.yaml
│  │
│  ├─ 003-mfa/                    # 追加開発②：多要素認証
│  │  ├─ spec.md
│  │  ├─ plan.md
│  │  ├─ tasks.md
│  │  └─ contracts/
│  │     └─ mfa-api.openapi.yaml
│  │
│  └─ 004-audit-log/              # 追加開発③：監査ログ強化
│     ├─ spec.md
│     ├─ plan.md
│     ├─ tasks.md
│     └─ data-model.md
│
├─ src/ または infra/ など         # 実装コード・Terraformなど
└─ README.md
```

| ファイル | 主な問い | 主な読者 | タイミング |
| --- | --- | --- | --- |
| `spec.md` | 何を、なぜ作るか | プロダクト・開発・レビュアー | `/speckit.specify` |
| `research.md` | 技術的にどう解決するか。なぜその選択か | 技術リード・実装者 | `/speckit.plan`の初期段階 |
| `plan.md` | 全体としてどう実装するか | 実装者・レビュアー | `/speckit.plan` |
| `data-model.md` | データをどう保持・遷移させるか | 実装者・DB設計者 | `/speckit.plan` |
| `contracts/` | 外部インターフェースは何か | API利用者・フロントエンド・他チーム | `/speckit.plan` |
| `tasks.md` | 何をどの順で実装するか | 実装者 | `/speckit.tasks` |
| `quickstart.md` | 完成をどう検証するか | 実装者・QA・レビュアー | `/speckit.plan` |

#### 新規システム開発が使いやすいが、既存システム改修でも使える
Spec Kitは、0からつくる新規開発だけではなく、既存システムへの機能追加やモダン化、技術選定やUIの比較探索にも使えます。

- 既存システムでの利用例
    - オンプレミスのシステムをAWSのサービスに置き換える
    - 既存のWebシステムのUIに、レスポンシブ対応やアクセシビリティ改善を入れる

#### その他のSDDツール
SDDツールには、Spec Kitの他にも、AWS提供の「Kiro」や、Kiroをもとに作られた日本産の「cc-sdd」などがあります。

Spec Kitが小〜中規模で、人間がレビューしながら開発を進める案件に向いているのに対し、Kiroやcc-sddでは中〜大規模で、最初に決めた仕様をもとに途中のレビューなしで並列・長時間でAIエージェントが自立開発する案件に向いています。


### Spec Driven Developmentとは
Spec-Driven Development（SDD）とは、「コードを先に書き、仕様を後から整える」のではなく、作りたいものの意図・要件・設計・タスクを仕様として先に確定し、その仕様を実装と検証の基準にする開発手法です。

AIコーディングでは、曖昧な指示による暴走や意図とのズレを防ぐために、特に有効になります。

#### バイブコーディングとSDDの違い
バイブコーディングは、「バイブス」という言葉がその場や人が放つ **「雰囲気」「ノリ」「テンション」「気分」** を示す通り、要件を先に決めず、都度その場のノリで決めていく開発手法で、ドキュメントは後付けのサブ的な成果物として扱います。このため、ドキュメントとコードの整合性が合わなくなるケースもあります。

一方、SDDは、ドキュメントをメインの成果物として扱い、機能追加や仕様変更があった場合、先にドキュメントを修正してコード実装を行うため、常にドキュメントとコードの整合性が合った状態になります。

| 観点 | バイブコーディング | SDD |
| --- | --- | --- |
| 出発点 | 「〇〇を作って」という曖昧なプロンプト | ユーザーストーリー、受け入れ条件、制約、非機能要件 |
| AIへの指示 | 会話の中で都度修正 | 仕様・設計・タスクという構造化された文脈 |
| 実装の根拠 | モデルの推測、その場の会話 | 承認済みのspec、plan、tasks |
| レビュー対象 | 主にコード差分 | 要件との整合、設計との整合、タスクの完了状況、コード差分 |
| 変更時 | 影響範囲が不明瞭になりがち | 仕様を先に更新し、影響する設計・タスク・テストを導出できる |
| ドキュメント | 実装後に残ったり、古くなったりする | 実装と並行して更新される「生きた仕様」 |

#### SDDの５原則
SDDは、以下の５原則をもとに開発が行われます。

1. **Intent first**
    
    「どう作るか」の前に、「何を・なぜ作るか」を確定する。

2. **Executable specification**
    
    仕様は読むだけの資料ではなく、実装・テスト・レビューの基準にする。

3. **Multi-step refinement**
    
    一発のプロンプトで完成させず、仕様→計画→タスク→実装→検証と段階的に詰める。

4. **Living specification**
    
    要件が変われば、コードだけでなく仕様も更新する。

5. **Human-in-the-loop**
    
    AIに丸投げせず、要件・設計・タスク・実装結果を人間がレビューする。

#### SDDの向く案件・向かない案件

SDDによる開発手法が向く案件、向かない案件は以下の通りです。

| 向く案件 | 向かない案件 |
| --- | --- |
| 新規機能開発、既存システムへの機能追加 | 単発の小さなバグ修正、試作や技術検証（PoC）の初期段階 |
| 複数人・複数AIエージェントでの開発 | 個人の即席修正 |
| 業務アプリやSaaSなど、大規模で中長期的にメンテナンスが発生するサービスの開発 | 一回きりのスクリプト作成、完全に使い捨てのプロトタイプ |
| 要件がある程度決まっていて、仕様の変動性が少ない場合 | 要件が流動的で、仕様を固定する意味がない場合 |


### Spec Kitが解決する課題
Spec Kitを使えば、バイブコーディングによる行き当たりばったりな開発で仕様が抜け漏れていたり、仕様が属人化・ブラックボックスになっていたり、プロジェクトにより開発手法が異なり生産性が安定しないなどの課題を解決できます。

#### AI開発プロセスにおけるよくある失敗への解決策
AIエージェントを使った開発で、よく耳にする失敗に対して、Spec Kitを導入すれば改善する可能性があります。

| 開発のステップ | 従来のAI活用での失敗 | Spec Kit 導入後の変化 |
| --- | --- | --- |
| **指示出し** | **「動くけど意図と違う」手戻り** <br>「ログイン機能を作って」と曖昧に指示したら、バリデーションやセキュリティ要件が抜け落ちたコードが出力され、修正に時間がかかる。 | **仕様に合意してから実装をスタート** <br>`/spec` で要件やエッジケース（何を作らないか含む）を事前に定義するため、最初の段階で意図のズレを完全に防止できる。 |
| **設計・構築** | **勝手に複雑なアーキテクチャにされる** <br>既存のディレクトリ構成や設計思想を無視され、AI独自の突飛なライブラリやコード構成で実装されてしまう。 | **プロジェクトの設計指針を強制遵守**<br>`/plan` で技術選定や構成案を事前にチェック・修正できるため、既存コードと調和したアーキテクチャが保たれる。 |
| **タスク分解** | **巨大なコードが一発生成されて破綻する**<br>一度に広範囲のファイルを書き換えられ、レビュー不能な巨大な差分（Diff）や、修正途中の動かないコードが発生する。 | **小さくテスト可能な単位に自動分解**<br>`/tasks` で安全なステップ（タスク）に細分化されるため、1ステップずつ動作確認しながら確実に進められる。 |
| **ナレッジ共有** | **チャット履歴が流れて仕様が迷子になる**<br>なぜその実装になったのかがAIとのチャットの中に埋もれ、後から参画したメンバーが理由を追えない。 | **仕様と計画がコードと一緒にバージョン管理される**<br>すべての工程が `docs/specs` などに Markdown ドキュメントとして残るため、チーム全員で仕様を共有・追跡できる。 |
| **チーム連携** | **プロンプトの上手い人しか使いこなせない**<br>「指示の書き方のコツ」が個人のノウハウ（属人化）になり、チーム全体でAI活用の品質や成果にバラつきが出る。 | **標準化されたフローで品質が平準化される**<br>全員がスラッシュコマンド（/spec, /plan, /tasks）に沿って進めるだけで、誰もが一定レベル以上の指示出しを行える。 |
| **修正・改修** | **ちょっとした変更で既存機能が壊れる**<br>追加実装を頼んだら、関係のない既存機能のコードまでAIが書き換えてしまい、不具合（デグレ）が発生する。 | **仕様ドキュメントを起点に影響範囲を特定**<br>既存の Spec を参照・更新してから修正作業に入るため、AIがコンテキストを正しく把握しデグレを最小限に抑えられる。 |

## 基本的な使い方
GitHub Spec Kitの使い方について、「ToDoアプリ」を作りながら説明します。

### 1. プロジェクト初期化
uvxを使って、Speck Kitを実行し、プロジェクトを初期化します。

途中で `Choose your coding agent integration` のプロンプトが出て、利用するAIエージェントを聞かれます。今回は、GitHub Copilotを選択します。

また、続けて `Choose script type` のプロンプトが出て、利用するスクリプトを聞かれます。今回は、PowerShellを選択します。

```
uvx --from git+https://github.com/github/spec-kit.git specify init <project名>
```

<details><summary>実行例</summary>

```
PS C:\tech-ult> uvx --from git+https://github.com/github/spec-kit.git specify init todo-app
                                            ███████╗██████╗ ███████╗ ██████╗██╗███████╗██╗   ██╗                                            
                                            ██╔════╝██╔══██╗██╔════╝██╔════╝██║██╔════╝╚██╗ ██╔╝                                            
                                            ███████╗██████╔╝█████╗  ██║     ██║█████╗   ╚████╔╝                                             
                                            ╚════██║██╔═══╝ ██╔══╝  ██║     ██║██╔══╝    ╚██╔╝                                              
                                            ███████║██║     ███████╗╚██████╗██║██║        ██║                                               
                                            ╚══════╝╚═╝     ╚══════╝ ╚═════╝╚═╝╚═╝        ╚═╝                                               
                                                                                                                                            
                                             GitHub Spec Kit - Spec-Driven Development Toolkit                                              


╭───────────────────────────────────────────────── Choose your coding agent integration: ──────────────────────────────────────────────────╮
│                                                                                                                                          │
│       agy (Antigravity)                                                                                                                  │
│       alquimia (Alquimia AI)                                                                                                             │
│       amp (Amp)                                                                                                                          │
│       auggie (Auggie CLI)                                                                                                                │
│       bob (IBM Bob)                                                                                                                      │
│       claude (Claude Code)                                                                                                               │
│       cline (Cline)                                                                                                                      │
│       codebuddy (CodeBuddy)                                                                                                              │
│       codex (Codex CLI)                                                                                                                  │
│       command-code (Command Code)                                                                                                        │
│  ▶    copilot (GitHub Copilot)                                                                                                           │
│       cursor-agent (Cursor)                                                                                                              │
│       devin (Devin for Terminal)                                                                                                         │
│       docker-agent (Docker Agent)                                                                                                        │
│       droid (Factory Droid)                                                                                                              │
│       dsh (DeepSeek Harness)                                                                                                             │
│       firebender (Firebender)                                                                                                            │
│       forge (Forge)                                                                                                                      │
│       gemini (Gemini CLI)                                                                                                                │
│       generic (Generic (bring your own agent))                                                                                           │
│       goose (Goose)                                                                                                                      │
│       grok (Grok Build)                                                                                                                  │
│       hermes (Hermes Agent)                                                                                                              │
│       junie (Junie)                                                                                                                      │
│       kilocode (Kilo Code)                                                                                                               │
│       kimi (Kimi Code)                                                                                                                   │
│       kiro-cli (Kiro CLI)                                                                                                                │
│       lingma (Lingma)                                                                                                                    │
│       mcode (MiniMax Code)                                                                                                               │
│       muse (Muse Code)                                                                                                                   │
│       omp (Oh My Pi)                                                                                                                     │
│       opencode (opencode)                                                                                                                │
│       pi (Pi Coding Agent)                                                                                                               │
│       qodercli (Qoder CLI)                                                                                                               │
│       qwen (Qwen Code)                                                                                                                   │
│       rovodev (RovoDev ACLI)                                                                                                             │
│       shai (SHAI)                                                                                                                        │
│       tabnine (Tabnine CLI)                                                                                                              │
│       trae (Trae)                                                                                                                        │
│       vibe (Mistral Vibe)                                                                                                                │
│       zcode (ZCode)                                                                                                                      │
│       zed (Zed)                                                                                                                          │
│                                                                                                                                          │
│       Use ↑/↓ to navigate, Enter to select, Esc to cancel                                                                                │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
╭──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╮
│                                                                                                                                          │
│  Specify Project Setup                                                                                                                   │
│                                                                                                                                          │
│  Project         todo-app                                                                                                                │
│  Working Path    C:\tech-ult                                                                                                             │
│  Target Path     C:\tech-ult\todo-app                                                                                                    │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

╭────────────────────────────────────────────────── Choose script type (or press Enter) ───────────────────────────────────────────────────╮
│                                                                                                                                          │
│       sh (POSIX Shell (bash/zsh))                                                                                                        │
│  ▶    ps (PowerShell)                                                                                                                    │
│       py (Python)                                                                                                                        │
│                                                                                                                                          │
│       Use ↑/↓ to navigate, Enter to select, Esc to cancel                                                                                │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯                                                                    ...                                                                     
Selected coding agent integration: copilot
Selected script type: ps
Initialize Specify Project
├── ● Check required tools (ok)
├── ● Select coding agent integration (copilot)
├── ● Select script type (ps)
├── ● Install integration (GitHub Copilot)
├── ● Install shared infrastructure (scripts (ps) + templates)
├── ○ Ensure scripts executable
├── ● Constitution setup (copied from template)
├── ● Install bundled workflow (speckit installed)
└── ● Finalize (project ready)

Project ready.

╭───────────────────────────────────────────────────────── Agent Folder Security ──────────────────────────────────────────────────────────╮
│                                                                                                                                          │
│  Some agents may store credentials, auth tokens, or other identifying and private artifacts in the agent folder within your project.     │
│  Consider adding .github/ (or parts of it) to .gitignore to prevent accidental credential leakage.                                       │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

╭─────────────────────────────────────────────────────────────── Next Steps ───────────────────────────────────────────────────────────────╮
│                                                                                                                                          │
│  1. Go to the project folder: cd todo-app                                                                                                │
│  2. Start using skills with your coding agent:                                                                                           │
│     2.1 /speckit-constitution - Establish project principles                                                                             │
│     2.2 /speckit-specify - Create baseline specification                                                                                 │
│     2.3 /speckit-plan - Create implementation plan                                                                                       │
│     2.4 /speckit-tasks - Generate actionable tasks                                                                                       │
│     2.5 /speckit-implement - Execute implementation                                                                                      │
│     2.6 /speckit-converge - Assess the codebase and append remaining work as tasks                                                       │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯

╭─────────────────────────────────────────────────────────── Enhancement Skills ───────────────────────────────────────────────────────────╮
│                                                                                                                                          │
│  Optional skills that you can use for your specs (improve quality & confidence)                                                          │
│                                                                                                                                          │
│  ○ /speckit-clarify (optional) - Ask structured questions to de-risk ambiguous areas before planning (run before /speckit-plan if used)  │
│  ○ /speckit-analyze (optional) - Cross-artifact consistency & alignment report (after /speckit-tasks, before /speckit-implement)         │
│  ○ /speckit-checklist (optional) - Generate quality checklists to validate requirements completeness, clarity, and consistency (after    │
│  /speckit-plan)                                                                                                                          │
│                                                                                                                                          │
╰──────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────╯
```

</details>


プロジェクト初期化後、以下のフォルダが作成されます。

- `.specify/memory/`
    
    プロジェクト全体の共通ルール（後でやる Constitution など）を格納する場所
    
- `.specify/templates/`
    
    後続の `/plan` や `/tasks` で使われるMarkdownのテンプレートファイルを格納する場所

### 2. Constitution
プロジェクトの環境構築後は、**AIに絶対守らせる基本原則（技術選定、コード品質、デザインポリシー、禁止事項など）** を『憲法』として最初に定義します。

人間でいう『チームの開発ガイドライン』を、AIにも理解できる形で渡しておくイメージです。

どんなアプリケーションを作りたいか等の目的は次の「Specify」で会話するため、品質・セキュリティ・テスト・保守性などの不変原則を定めるだけにします。

```powershell
/speckit-constitution <プロジェクトの規定など>
```

上記コマンドをGitHub Copilotのプロンプトで実行すると、AIと対話が始まり、どのようなルールをプロジェクトの基本原則にするかを決めていきます。

作成した憲法は、 `.specify/memory/constitution.md` として作成されます。

`constitution.md` は、これ以降に実行するすべてのステップ（仕様決定、設計、タスク分解、コード実装）において、AIが常時参照する『最上位の前提条件』になります。

これにより、プロンプトのたびに「Reactで書いて」「TypeScriptで書いて」「テストも書いて」と何度も同じ指示を繰り返す必要がなくなります。

※以降のドキュメントは英語で作成されるため、日本語にしてレビューしたい場合は、「ドキュメントはすべて日本語で記載すること」などのルールを決めます。


### 3. Specify
憲法が決まったら、次はアプリで『何を作るか』の仕様を固めます。

これまでは『xxアプリを作って』とAIに丸投げしていましたが、Spec Kitでは **`/speckit-specify`** というコマンドを使って、人間とAIが対話しながら『仕様書（Specification）』を一緒に作り上げていきます。

ここでは、**作る機能・対象ユーザー・できること・受け入れ条件**を自然言語で書きます。技術選定（Vite、Reactなど）は通常この段階では書かず、次の `/speckit-plan` に回します。

```powershell
/speckit-specify <作る機能など>
```

ざっくりとした対話をした場合でも、AIが不足している要件や、考慮すべき『エッジケース（例外的な状況）』を逆質問してくれたり、きれいな仕様書のドラフトを自動生成してくれます。

「何を作るか（What）」だけでなく、「何を作らないか」「どんなエラーを考慮すべきか（Edge Cases）」などをこの段階で網羅的に言語化します。これにより、**AI開発にありがちな“後からの仕様漏れ・手戻り”を削減**できます。

作成されたドキュメントは `specs/.../spec.md` に格納されます。

また、作成した仕様書の曖昧さや未決事項をAIが最大5問の質問に絞り、その回答を `spec.md` に反映するためのステップをオプションで実行することが可能です。以下のコマンドを実行します。

```powershell
/speckit-clarify
```

### 4. Plan
『何を作るか（仕様）』が決まったら、次は **『どう作るか（技術設計）』** を決める Plan フェーズです。

仕様書（`spec.md`）と憲法（`constitution.md`）をインプットにして、**『コンポーネント構成はどうするか』『データ構造はどうするか』** といった建築でいう『青写真（設計図）』をAIと一緒に作成します。

ここでは、**技術スタック・アーキテクチャ・データ保存方式・テスト方針**など、「どのように実装するか」を自然言語で渡します。

```powershell
/speckit-plan <技術スタックなど>
```

上記コマンドを実行すると、AIは先ほど作った仕様書（`spec.md`）と憲法（`constitution.md`）を読み込み、具体的なフォルダ構成やデータ構造案を提案してくれます。

作成されたドキュメントは `specs/.../plan.md` に格納されます。


### 5. Tasks
仕様（What）と設計（How）が決まったら、次は実装に向けた「ロードマップ（タスクリスト）」**を作るフェーズに入ります。**

**ここでのポイントは、AIに巨大な実装を一気に行わせず、**「小さな検証可能ステップ」に切り分けることです。

AI開発で一番よくある失敗は、設計図があるからといって『全部作って』と一括で頼んでしまうことです。広範囲のコードを一度に変更させると、途中でAIが混乱したり、テストできないコードが出力されたり、巨大な差分（Diff）が発生して人間がレビューできなくなります。

Spec Kitでは **`/speckit-tasks`** コマンドを使い、設計図を『人間とAIが1つずつ確実に確認しながら進められる最小単位のタスク』へ自動分解します。

コマンドの引数に追加指示のコメントをオプションで追加可能です。

```powershell
/speckit-tasks <タスクの粒度など詳細に指定したい場合>
```

タスク一覧のタスクひとつひとつにはチェックボックスが設けられ、完了したタスクは次のImplementフェーズでAIエージェントがチェックをつけていき、どのタスクをどこまで実行したかを判別可能にします。これにより、タスク中断した後でも、どこから再開すれば良いかが明確になります。

作成したドキュメントは `specs/.../tasks.md` に格納されます。

### 6. Implement
最後のステップ、**『Implement（実装・テスト）』** です。

ここまでの準備（Constitution, Spec, Plan, Tasks）がすべてリポジトリのドキュメントとして整っているため、生成されたタスクリスト（`tasks.md`）に沿って、AIに『次のタスクを実装して』と指示するだけで実装が進んでいきます。

このフェーズではドキュメントではなく、ソースコードを成果物として作成します。

```powershell
/speckit-implement <実行指示など>
```

仕様通りに実装できているかテストを実施して確認後、`tasks.md` の該当タスクにチェック `[x]` を入れ、次のタスクに進みます。

一気に実装させず、基盤部分だけ先に作らせたいなどの要望があれば、オプションで指定することが可能です。

# まとめ
ここまで、Spec Kitを使ったToDoアプリ開発の流れを見てきました。

1. **Initialize**（環境を整える）
2. **Constitution**（憲法でルールを縛る）
3. **Specify**（仕様で何を作るか決める）
4. **Plan**（設計でどう作るか決める）
5. **Tasks**（タスクで小さく分解する）
6. **Implement**（確実に実装・テストする）

この一連のステップ（型）を踏むことで、AIコーディングは『ガチャ（一か八か）』から『エンジニアリング（確実なプロセス）』に変わります。ぜひ皆さんの開発現場でも試してみてください。

一緒にAIエージェント開発について学んでいきたいなど、ご興味を持たれた方は、[弊社ホームページ](https://www.tech-ult.com/contact-8)からお問い合わせいただければ幸いです。
