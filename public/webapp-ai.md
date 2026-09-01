---
title: GitHub CopilotエージェントでWebアプリを作ろう（非エンジニア向け）
tags:
  - ソフトウェア開発
  - AIエージェント
  - GitHub Copilot
private: false
updated_at: ''
id: null
organization_url_name: tech-ult
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---
# はじめに
コーディングエージェント界隈が盛況で、ソフトウェアエンジニアであれば日常業務でお世話になっている人たちも多いでしょう。

かく言う私も、もはやClaude Codeがなかった時代には戻れない状況になっており、コードの書き方を忘れつつあります。クリーンコードに躍起になっていた日々を懐かしく思います。

この度、非ソフトウェアエンジニア向けのコーディングエージェント勉強会を開催することになり、以下の点を考慮して内容を考えました。

- ソフトウェア開発未経験者向けなので、小難しい説明を省き、コーディングエージェントを使ってのアプリケーション開発体験を優先すること
- 開発ツールのインストールなど、開発の前段で詰まって脱落していく人をつくらないこと
- お試しで実施するので、ライセンスなどの費用が掛からないこと

# 前提条件
- 開発ツールのインストールにパッケージマネージャーの「WinGet」「npm」を利用しています。※WinGetはWindows10/11に標準で組み込まれています。
- コーディングエージェントには「GitHub Copilot」を利用します。GitHubへのアカウント登録が必要になるため、メールアドレス

# 解説
## 開発環境準備
社内勉強会でハンズオンを実施する前に、参加者のPCに必要な開発ツールをインストールしました。

開発ツールのインストールは、非エンジニアが最初に突き当たる壁で、ここがスムーズに進まないと、勉強する気も失せていきます。今回は、WinGetを使ったインストール用のスクリプトを用意して、なるべく手作業を省き、バッチを起動するだけですべてインストールされるように工夫しました。

なお、NotionなどインターネットからダウンロードしたPowerShellスクリプトを実行する場合、PowerShellのポリシーを変更する必要があります。

PowerShellターミナルを起動して、以下のコマンドを実行します。PowerShellのポリシー変更について聞かれるので、「Y」で進めてください。

```powershell
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### GitHubアカウントの登録
AIエージェントにGitHub Copilotを使用するために、GitHubアカウントを登録する必要があります。

Webから登録操作が必要で、途中でメール認証を挟む都合上、バッチにより自動化するのではなく、社内のNotionに [GitHubでのアカウントの作成](https://docs.github.com/ja/account-and-profile/how-tos/account-management/creating-an-account-on-github) をもとにした操作説明を用意して、参加者に手作業で登録してもらいました。

また、GitHubに登録する公開鍵について、キーペアを作成するスクリプトを用意して、 [GitHub アカウントへの新しい SSH キーの追加](https://docs.github.com/ja/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account) を参考にWeb画面から登録してもらいました。

<details><summary>GitHubキーペア作成用スクリプト</summary>
SSH接続用のキーペアと、configファイルを編集するためのps1スクリプトと、スクリプトを起動するbatファイルを用意しました。

ps1ファイルとbatファイルを同じフォルダに格納して、batファイルを右クリックして管理者として実行することで、キーペアの作成と `$env:USERPROFILE\.ssh\config` の編集が行われます。

※注意：configファイルに既存のGitHubの設定が存在した場合は上書きされます。

```dos:setup-github-ssh.bat
@echo off
setlocal

powershell.exe -NoProfile -ExecutionPolicy Bypass -File "%~dp0setup-github-ssh.ps1"

if %ERRORLEVEL% neq 0 (
    echo.
    echo ============================================
    echo SSH key setup failed.
    echo ============================================
    echo.
    pause
    exit /b %ERRORLEVEL%
)

echo.
echo ============================================
echo SSH key setup completed.
echo ============================================
echo.

pause
```

batから呼び出されるPowerShellのスクリプトは以下です。

```powershell:setup-github-ssh.ps1
$ErrorActionPreference = "Stop"

Write-Host ""
Write-Host "============================================"
Write-Host " GitHub SSH Key Setup"
Write-Host "============================================"
Write-Host ""

# メールアドレス入力
$email = Read-Host "GitHubに登録するメールアドレス"

if ([string]::IsNullOrWhiteSpace($email)) {
    Write-Error "メールアドレスが入力されていません。"
    exit 1
}

# .ssh ディレクトリ
$sshDir = Join-Path $env:USERPROFILE ".ssh"

if (!(Test-Path $sshDir)) {
    New-Item -ItemType Directory -Path $sshDir -Force | Out-Null
}

# 実行日 YYYYMMDD
$date = Get-Date -Format "yyyyMMdd"

# 鍵ファイル
$keyName = "id_ed25519_github_$date"
$keyPath = Join-Path $sshDir $keyName
$publicKeyPath = "$keyPath.pub"

# 同名鍵が存在する場合
if (Test-Path $keyPath) {
    Write-Error "秘密鍵が既に存在します: $keyPath"
    exit 1
}

if (Test-Path $publicKeyPath) {
    Write-Error "公開鍵が既に存在します: $publicKeyPath"
    exit 1
}

Write-Host ""
Write-Host "SSH key pair を作成します。"
Write-Host "  Email       : $email"
Write-Host "  Private key : $keyPath"
Write-Host "  Public key  : $publicKeyPath"
Write-Host ""

# SSH鍵生成
# -t ed25519 : Ed25519
# -C         : コメントとしてメールアドレスを設定
# -f         : 出力ファイルを指定
# -N ""      : パスフレーズなし
$sshArgs = @(
    "-t", "ed25519",
    "-C", $email,
    "-f", $keyPath
)

# パスフレーズと確認用パスフレーズを空で入力
$emptyPassphrase = "`n`n"

$emptyPassphrase | & ssh-keygen @sshArgs

if ($LASTEXITCODE -ne 0) {
    Write-Error "SSH鍵の生成に失敗しました。"
    exit 1
}

# SSH config に GitHub の定義を追加
$configPath = Join-Path $sshDir "config"
$hostName = "github.com"

$configEntry = @"
Host $hostName
    HostName github.com
    User git
    IdentityFile ~/.ssh/$keyName
    IdentitiesOnly yes
"@

Write-Host ""
Write-Host "SSH config を設定します。"

if (!(Test-Path $configPath)) {
    New-Item -ItemType File -Path $configPath -Force | Out-Null
}

$configContent = if (Test-Path $configPath) {
    Get-Content $configPath -Raw
} else {
    ""
}

# 既存の Host github.com 定義がある場合は、今回の鍵に更新
$escapedHost = [regex]::Escape($hostName)
$hostPattern = "(?ms)^Host\s+$escapedHost\s*$.*?(?=^Host\s+|\z)"

if ($configContent -match $hostPattern) {
    $configContent = [regex]::Replace(
        $configContent,
        $hostPattern,
        ($configEntry.TrimEnd() + "`r`n`r`n")
    )
} else {
    if ($configContent.Length -gt 0 -and -not $configContent.EndsWith("`r`n") -and -not $configContent.EndsWith("`n")) {
        $configContent += "`r`n"
    }

    if ($configContent.Length -gt 0) {
        $configContent += "`r`n"
    }

    $configContent += $configEntry.TrimEnd() + "`r`n"
}

Set-Content -Path $configPath -Value $configContent -Encoding utf8

# 公開鍵をクリップボードへコピー
Get-Content $publicKeyPath | Set-Clipboard

Write-Host ""
Write-Host "============================================"
Write-Host " SSH Key Setup Complete"
Write-Host "============================================"
Write-Host ""
Write-Host "秘密鍵:"
Write-Host "  $keyPath"
Write-Host ""
Write-Host "公開鍵:"
Write-Host "  $publicKeyPath"
Write-Host ""
Write-Host "SSH config:"
Write-Host "  $configPath"
Write-Host ""
Write-Host "GitHub 用の SSH 設定を追加しました。"
Write-Host "公開鍵の内容をクリップボードにコピーしました。"
Write-Host ""
Write-Host "GitHub:"
Write-Host "  Settings"
Write-Host "    -> SSH and GPG keys"
Write-Host "    -> New SSH key"
Write-Host ""
Write-Host "クリップボードから公開鍵を貼り付けてください。"
Write-Host ""
```

</details>

上記のスクリプト実行後、Windowsのクリップボードに公開鍵のテキストファイルがコピーされているので、GitHubの公開鍵入力欄にペーストします。


なお、社内の同じグローバルIPアドレスから、同時に複数のアカウント登録操作が行われたため、GitHubのレート制限にかかって一定時間作業が進まなくなった人が数名いました。

可能なら、GitHubアカウント登録は勉強会前に時間を分けて実施してもらうか、社用携帯があればテザリングしてIPアドレスを変えてサインアップしてもらう方が良いかもしれません。

### 開発ツールのインストール
社内のNotionにPowerShellのスクリプトとWindowsのバッチファイルを用意して、参加者がダウンロードしたバッチを起動するだけで必要なツール、Webアプリケーション開発の環境が整うようにしました。

<details><summary>セットアップ用スクリプト</summary>

後述するPowerShellのスクリプトを起動するだけのバッチを用意しました。

ps1ファイルとbatファイルを同じフォルダに格納して、batファイルを右クリックして管理者権限で実行してもらえば、自動でインストールが始まります。

```dos:setup.bat
@echo off
net session >nul 2>&1
if %errorlevel% neq 0 (
 powershell -Command "Start-Process '%~f0' -Verb RunAs"
 exit /b
)
powershell -ExecutionPolicy Bypass -File "%~dp0setup.ps1"
pause
```

セットアップスクリプトは、大きく以下の処理を行います。

1. 開発ツールのインストール
2. VS Code拡張機能のインストール
3. Git設定
4. GitHub CLIログイン
5. `project-root` の作成
6. Frontendの作成
7. Backendの作成
8. OpenAPI / Swagger / Prism環境の作成

※途中で、Git用のメールアドレスやGitHubに登録したキーペアの入力を促されます。

```powershell:setup.ps1
$ErrorActionPreference = "Stop"

# npm 等の native コマンドの終了コードで自動停止しないようにする（PowerShell 7.3+）
# ※ cmdlet のエラーでは従来どおり停止する
$PSNativeCommandUseErrorActionPreference = $false

# 管理者として実行するとカレントディレクトリが System32 になるため、
# スクリプト自身の場所へ移動してから処理を行う
$scriptDir = if ($PSScriptRoot) {
    $PSScriptRoot
}
else {
    Split-Path -Parent $MyInvocation.MyCommand.Path
}

Set-Location -Path $scriptDir


# =====================================================================
# ヘルパー関数
# =====================================================================

# winget インストール直後、同一セッションの PATH を更新する
function Update-SessionPath {
    $machine = [System.Environment]::GetEnvironmentVariable("Path", "Machine")
    $user    = [System.Environment]::GetEnvironmentVariable("Path", "User")
    $env:Path = @($machine, $user | Where-Object { $_ }) -join ";"
}

# コマンドの存在確認
function Test-CommandExists {
    param([string]$Name)

    return [bool](Get-Command $Name -ErrorAction SilentlyContinue)
}

# winget パッケージをインストール
# 既にインストール済みでも処理を継続する
function Install-WingetPackage {
    param([string]$Id)

    Write-Host "  - $Id"

    winget install `
        --id $Id `
        -e `
        --accept-package-agreements `
        --accept-source-agreements

    $code = $LASTEXITCODE

    # 0 = 成功
    # -1978335189 = 該当更新なし（インストール済み）
    if ($code -ne 0 -and $code -ne -1978335189) {
        Write-Warning "    winget が終了コード $code を返しました（$Id）。処理は継続します。"
    }
}

# UTF-8（BOMなし）でファイルを書き出す
# .env に BOM が付くと dotenv が最初のキーを読み違えることがあるため
function Write-Utf8NoBom {
    param(
        [string]$Path,
        [string]$Content
    )

    if ([System.IO.Path]::IsPathRooted($Path)) {
        $full = $Path
    }
    else {
        $full = Join-Path (Get-Location).Path $Path
    }

    $full = [System.IO.Path]::GetFullPath($full)

    $dir = Split-Path -Parent $full

    if (!(Test-Path $dir)) {
        New-Item -ItemType Directory -Path $dir -Force | Out-Null
    }

    [System.IO.File]::WriteAllText(
        $full,
        $Content,
        (New-Object System.Text.UTF8Encoding($false))
    )
}


# =====================================================================
# 1. ソフトウェアのインストール
# =====================================================================

if (
    (Test-CommandExists node) -and
    (Test-CommandExists git) -and
    (Test-CommandExists gh)
) {
    Write-Host "Node.js / Git / GitHub CLI は導入済みです。インストールをスキップします。"
}
else {
    Write-Host "Installing software..."

    Install-WingetPackage "OpenJS.NodeJS.22"
    Install-WingetPackage "Git.Git"
    Install-WingetPackage "Microsoft.VisualStudioCode"
    Install-WingetPackage "GitHub.cli"
}

# インストールした実行ファイルを同一セッションで使えるように PATH を更新
Update-SessionPath


# =====================================================================
# 2. VS Code 拡張機能
# =====================================================================

$code = "$Env:LOCALAPPDATA\Programs\Microsoft VS Code\bin\code.cmd"

if (!(Test-Path $code)) {
    $code = "code"
}

$extensions = @(
    "dbaeumer.vscode-eslint",
    "esbenp.prettier-vscode",
    "qwtel.sqlite-viewer",
    "rangav.vscode-thunder-client",
    "usernamehw.errorlens",
    "eamodio.gitlens",
    "EditorConfig.EditorConfig",
    "mhutchie.git-graph",
    "ritwickdey.LiveServer"
)

Write-Host ""
Write-Host "Installing VS Code extensions..."

foreach ($e in $extensions) {
    Write-Host "  - $e"

    & $code --install-extension $e --force

    if ($LASTEXITCODE -ne 0) {
        Write-Warning "拡張機能のインストールに失敗しました: $e"
    }
}


# =====================================================================
# 3. Git 設定
# =====================================================================

Write-Host ""
Write-Host "Git Configuration"

$existingName = & git config --global user.name 2>$null

if ([string]::IsNullOrWhiteSpace($existingName)) {
    $name  = Read-Host "Git User Name"
    $email = Read-Host "Git Email"

    git config --global user.name "$name"
    git config --global user.email "$email"
}
else {
    Write-Host "  Git は既に設定済みです（$existingName）。スキップします。"
}


# =====================================================================
# 4. GitHub ログイン
# =====================================================================

Write-Host ""
Write-Host "GitHub Login"

& gh auth status 2>$null | Out-Null

if ($LASTEXITCODE -ne 0) {
    gh auth login
}
else {
    Write-Host "  GitHub CLI は既にログイン済みです。スキップします。"
}


# =====================================================================
# 5. プロジェクト生成
# =====================================================================

Write-Host ""
Write-Host "Creating project structure..."

# npm create / npx の確認プロンプトを自動承認
$env:npm_config_yes = "true"

$root = Join-Path $scriptDir "project-root"

if (!(Test-Path $root)) {
    New-Item -ItemType Directory -Path $root | Out-Null
}


# -----------------------------------------------------------------
# ルート .gitignore
# -----------------------------------------------------------------

$gitignore = Join-Path $root ".gitignore"

if (!(Test-Path $gitignore)) {
    Write-Utf8NoBom $gitignore @'
node_modules/
dist/
*.log
.env
database.db
database.db-*
'@
}


# =====================================================================
# Frontend : React + TypeScript + Vite
# =====================================================================

$frontend = Join-Path $root "frontend"

if (!(Test-Path $frontend)) {

    Write-Host ""
    Write-Host "Setting up frontend (Vite + React + TS)..."

    Push-Location $root

    # --no-interactive : 対話プロンプトを出さない
    # --no-immediate   : スキャフォールドのみ
    npm create vite@latest frontend -- `
        --template react-ts `
        --no-interactive `
        --no-immediate

    Pop-Location
}
else {
    Write-Host "frontend は既に存在します。スキャフォールドをスキップします。"
}


# -----------------------------------------------------------------
# Frontend の依存を整える
# -----------------------------------------------------------------

if (Test-Path $frontend) {

    Push-Location $frontend

    npm install axios react-router-dom

    # 既知の脆弱性を、破壊的なメジャーアップデートなしで修正
    Write-Host ""
    Write-Host "Running npm audit fix (frontend)..."
    npm audit fix

    if ($LASTEXITCODE -ne 0) {
        Write-Warning "frontend の npm audit fix で未修正の脆弱性が残っている可能性があります。"
    }

    $feDirs = @(
        "api",
        "components",
        "pages",
        "hooks",
        "layouts",
        "routes"
    )

    foreach ($d in $feDirs) {

        $p = Join-Path "src" $d

        New-Item `
            -ItemType Directory `
            -Path $p `
            -Force |
            Out-Null

        $gitkeep = Join-Path $p ".gitkeep"

        if (!(Test-Path $gitkeep)) {
            New-Item `
                -ItemType File `
                -Path $gitkeep `
                -Force |
                Out-Null
        }
    }

    Pop-Location
}


# =====================================================================
# Backend : Express + SQLite + TypeScript + Swagger UI
# =====================================================================

$backend = Join-Path $root "backend"

if (!(Test-Path $backend)) {

    Write-Host ""
    Write-Host "Setting up backend (Express + SQLite + TS + Swagger)..."

    New-Item `
        -ItemType Directory `
        -Path $backend `
        -Force |
        Out-Null
}
else {
    Write-Host ""
    Write-Host "backend は既に存在します。既存環境に不足している設定を追加します。"
}


# -----------------------------------------------------------------
# Backend npm
# -----------------------------------------------------------------

Push-Location $backend

if (!(Test-Path "package.json")) {
    npm init -y
}

Write-Host ""
Write-Host "Installing backend dependencies..."

npm install `
    express `
    better-sqlite3 `
    dotenv `
    cors  `
    swagger-ui-express `
    yaml

npm install -D `
    typescript `
    tsx `
    @stoplight/prism-cli `
    @types/node `
    @types/express `
    @types/better-sqlite3 `
    @types/cors `
    @types/swagger-ui-express

# 既知の脆弱性を、破壊的なメジャーアップデートなしで修正
Write-Host ""
Write-Host "Running npm audit fix (backend)..."
npm audit fix

if ($LASTEXITCODE -ne 0) {
    Write-Warning "backend の npm audit fix で未修正の脆弱性が残っている可能性があります。"
}

if ($LASTEXITCODE -ne 0) {
    Write-Warning "npm install が失敗した可能性があります。"
    Write-Warning "better-sqlite3 のビルドには Visual Studio Build Tools / Python が必要になる場合があります。"
}


# -----------------------------------------------------------------
# npm scripts
# -----------------------------------------------------------------

npm pkg set scripts.dev="tsx watch src/server.ts"
npm pkg set scripts.swagger="tsx swagger/server.ts"
npm pkg set scripts.mock="prism mock openapi.yaml"
npm pkg set scripts.build="tsc"
npm pkg set scripts.start="node dist/server.js"


# -----------------------------------------------------------------
# Backend ディレクトリ
# -----------------------------------------------------------------

$beDirs = @(
    "controllers",
    "services",
    "repositories",
    "routes",
    "middleware",
    "db"
)

foreach ($d in $beDirs) {

    $p = Join-Path "src" $d

    New-Item `
        -ItemType Directory `
        -Path $p `
        -Force |
        Out-Null

    $gitkeep = Join-Path $p ".gitkeep"

    if (!(Test-Path $gitkeep)) {
        New-Item `
            -ItemType File `
            -Path $gitkeep `
            -Force |
            Out-Null
    }
}


# -----------------------------------------------------------------
# Swagger ディレクトリ
# -----------------------------------------------------------------

$swaggerDir = Join-Path $backend "swagger"

New-Item `
    -ItemType Directory `
    -Path $swaggerDir `
    -Force |
    Out-Null


# -----------------------------------------------------------------
# tsconfig.json
#
# Swagger は本番ビルド対象外。
# include は src/**/* のみ。
# -----------------------------------------------------------------

if (!(Test-Path "tsconfig.json")) {

    Write-Utf8NoBom "tsconfig.json" @'
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "CommonJS",
    "moduleResolution": "Node",
    "outDir": "./dist",
    "rootDir": "./src",
    "strict": true,
    "esModuleInterop": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true
  },
  "include": ["src/**/*"]
}
'@
}


# -----------------------------------------------------------------
# .env
# -----------------------------------------------------------------

if (!(Test-Path ".env")) {

    Write-Utf8NoBom ".env" @'
PORT=3000
DATABASE_PATH=./database.db
'@
}


# -----------------------------------------------------------------
# src/db/index.ts
# -----------------------------------------------------------------

if (!(Test-Path "src/db/index.ts")) {

    Write-Utf8NoBom "src/db/index.ts" @'
import Database from "better-sqlite3";

const dbPath = process.env.DATABASE_PATH || "./database.db";

const db = new Database(dbPath);

db.pragma("journal_mode = WAL");

export default db;
'@
}


# -----------------------------------------------------------------
# src/server.ts
#
# 本番API。
# Swagger関連コードは入れない。
# -----------------------------------------------------------------

if (!(Test-Path "src/server.ts")) {

    Write-Utf8NoBom "src/server.ts" @'
import "dotenv/config";
import express from "express";
import cors from "cors";
import db from "./db";

const app = express();
const PORT = process.env.PORT || 3000;

app.use(cors());
app.use(express.json());

// DB ファイルを初期化時に生成する（動作確認用テーブル）
db.exec(`
  CREATE TABLE IF NOT EXISTS health_check (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    checked_at TEXT NOT NULL
  )
`);

app.get("/api/health", (_req, res) => {
  res.json({ status: "ok" });
});

app.listen(PORT, () => {
  console.log(`Server running on http://localhost:${PORT}`);
});
'@
}


# -----------------------------------------------------------------
# openapi.yaml
#
# API仕様。
# -----------------------------------------------------------------

$openapiFile = Join-Path $backend "openapi.yaml"

if (!(Test-Path $openapiFile)) {

    Write-Host ""
    Write-Host "Creating openapi.yaml..."

    Write-Utf8NoBom $openapiFile @'
openapi: 3.0.3

info:
  title: User API
  version: 1.0.0
  description: User management API

servers:
  - url: http://localhost:3000
  - url: http://localhost:4010

paths:

  /api/health:
    get:
      summary: ヘルスチェック
      responses:
        '200':
          description: OK
          content:
            application/json:
              schema:
                type: object
                properties:
                  status:
                    type: string
                    example: ok

  /api/users:
    get:
      summary: ユーザー一覧取得
      responses:
        '200':
          description: ユーザー一覧取得成功
          content:
            application/json:
              schema:
                type: object
                properties:
                  users:
                    type: array
                    items:
                      type: object
                      properties:
                        id:
                          type: integer
                        name:
                          type: string
                        email:
                          type: string
              example:
                users:
                  - id: 1
                    name: 山田太郎
                    email: yamada@example.com
                  - id: 2
                    name: 鈴木花子
                    email: suzuki@example.com
'@
}
else {
    Write-Host "openapi.yaml は既に存在します。既存ファイルを保持します。"
}


# -----------------------------------------------------------------
# swagger/server.ts
#
# Swagger UI専用サーバー。
# 本番APIの server.ts とは完全分離。
# -----------------------------------------------------------------

$swaggerServer = Join-Path $swaggerDir "server.ts"

if (!(Test-Path $swaggerServer)) {

    Write-Host ""
    Write-Host "Creating swagger/server.ts..."

    Write-Utf8NoBom $swaggerServer @'
import fs from "fs";
import path from "path";
import express from "express";
import swaggerUi from "swagger-ui-express";
import YAML from "yaml";

const app = express();

const PORT = 8080;

// backend/openapi.yaml を読み込む
const openapiPath = path.resolve(__dirname, "../openapi.yaml");

const openapiYaml = fs.readFileSync(
  openapiPath,
  "utf8"
);

const openapiDocument = YAML.parse(openapiYaml);

// Swagger UI
app.use(
  "/",
  swaggerUi.serve,
  swaggerUi.setup(openapiDocument)
);

app.listen(PORT, () => {
  console.log(`Swagger UI: http://localhost:${PORT}`);
});
'@
}
else {
    Write-Host "swagger/server.ts は既に存在します。既存ファイルを保持します。"
}


Pop-Location


# =====================================================================
# 完了
# =====================================================================

Write-Host ""
Write-Host "============================================================"
Write-Host "Setup Complete!"
Write-Host "============================================================"
Write-Host ""

Write-Host "Backend:"
Write-Host "  cd project-root/backend"
Write-Host "  npm run dev"
Write-Host "  -> http://localhost:3000/api/health"
Write-Host ""

Write-Host "Swagger UI:"
Write-Host "  cd project-root/backend"
Write-Host "  npm run swagger"
Write-Host "  -> http://localhost:8080"
Write-Host ""

Write-Host "Frontend:"
Write-Host "  cd project-root/frontend"
Write-Host "  npm run dev"
Write-Host "  -> http://localhost:5173"
Write-Host ""


# =====================================================================
# npm セキュリティ監査
# =====================================================================

Write-Host ""
Write-Host "Running final npm security audit..."

$projects = @(
    @{ Name = "frontend"; Path = $frontend },
    @{ Name = "backend";  Path = $backend }
)

foreach ($project in $projects) {
    if (Test-Path (Join-Path $project.Path "package.json")) {
        Push-Location $project.Path

        Write-Host ""
        Write-Host "[$($project.Name)] npm audit"

        npm audit

        # npm audit の終了コードは脆弱性が残っている場合もあるため、
        # セットアップ全体はここでは停止しない。
        $auditCode = $LASTEXITCODE

        if ($auditCode -ne 0) {
            Write-Warning "$($project.Name) に未修正の脆弱性が残っています。"
            Write-Warning "npm audit で詳細を確認してください。"
        }

        Pop-Location
    }
}

```

</details>


セットアップスクリプト実行後、同一フォルダに以下のプロジェクトが作成されます。

```text
project-root/
│
├── .gitignore
│
├── frontend/
│   ├── src/
│   │   ├── api/
│   │   ├── components/
│   │   ├── pages/
│   │   ├── hooks/
│   │   ├── layouts/
│   │   ├── routes/
│   │   ├── App.tsx
│   │   └── ...
│   ├── package.json
│   └── ...
│
└── backend/
    ├── src/
    │   ├── controllers/
    │   ├── services/
    │   ├── repositories/
    │   ├── routes/
    │   ├── middleware/
    │   ├── db/
    │   │   └── index.ts
    │   └── server.ts
    │
    ├── swagger/
    │   └── server.ts
    ├── openapi.yaml
    ├── tsconfig.json
    ├── .env
    ├── package.json
    └── database.db

※ `database.db` はBackendを起動してSQLiteへ接続した際に生成されます。
```

### フロントエンドの起動方法
開発ツールのインストールが終わった後、PowerShellのターミナルから、以下を実行します。

```powershell
cd project-root/frontend
npm run dev
```

Webブラウザから以下のURLへアクセスすると、フロントエンドのUIを参照可能です。

```
http://localhost:5173
```

### バックエンドの起動方法
以下のコマンドを実行します。

```powershell
cd project-root/backend
npm run dev
```

Webブラウザから以下のURLへアクセスすると、バックエンドのレスポンス（{"status":"ok"}）を確認可能です。

```
http://localhost:3000/api/health
```

## エージェントの設定

セットアップ用スクリプトでインストールしたVS Code上でGitHub Copilotと連携し、エージェントを起動して、Skill等の設定を行います。

### GitHub Copilotとの連携
VS Codeの左メニューアイコンからアカウント（人型）アイコンを選択して、GitHubアカウントと紐づいて、サインインしていることを確認します。

※下記例では、「rharuki-tech-ult(GitHub)」と表示されているので、サインインしている。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/0a48ac8f-658a-400a-b38e-96224fc5af38.png)

サインインしていなければ、「クラウドの変更を有効にします」を選択すると、画面上部に「GitHubでサインイン」と表示されるので、クリックしてブラウザを立ち上げ、GitHubとの連携を許可する設定を行います。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/3ecaad84-56ee-4b73-8993-e616bccd124c.png)

GitHubでサインイン

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/4243a2a7-8be3-4840-aa58-aa562f26b123.png)

Continue

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/308fb3f4-80f3-4625-9e67-50d0c44ada04.png)

許可(A)

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/e752dc52-d360-4f4f-8f75-245d4fac0acd.png)

画面左下のステータスバーにCopilotのアイコンで「サインイン」となっている場合は、クリックしてAI機能の使用を開始します。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/75b35296-f6ef-499d-b8a9-b81941bae7f8.png)

AI機能を使用する

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/f1ea2cc0-bc9e-49da-afc1-c119d76a6416.png)

使用済みクレジットの量を確認可能

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/83287b64-30f6-45d5-b68b-4112a7963d4f.png)

### Skillなどのカスタマイズ
VS Codeのプロジェクトで、画面右上の「エージェントで開く」でプロジェクトを開きましょう。

![image.png](https://qiita-image-store.s3.ap-northeast-1.amazonaws.com/0/4072121/c23c19e8-ef0e-4850-8d02-7725bb84ca83.png)



## Webアプリケーション開発のハンズオン
### バックエンドのAPI開発
### フロントエンドのUI開発

# まとめ
