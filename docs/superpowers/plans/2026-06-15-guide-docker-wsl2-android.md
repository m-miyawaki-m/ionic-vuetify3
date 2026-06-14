# Docker/WSL2/Android 入門ガイド群 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `guide/` ディレクトリに4本の入門ガイドを作成し、Docker概念からWSL2環境構築、AndroidエミュレータによるCapacitorアプリ実行までをエンドツーエンドで説明する。

**Architecture:** `guide/00-overview.md` を入口として4本を順番に読む設計。`container/` は詳細手順、`guide/` は概念理解・考え方・全体像という住み分け。各ファイルは独立して参照できるよう自己完結させつつ、相互参照リンクを置く。

**Tech Stack:** Markdown, テキスト図（ASCII）, 既存 container/ ドキュメントへの相互参照

---

## ファイル一覧

| ファイル | 役割 |
|---|---|
| `guide/00-overview.md` | 全体ロードマップ・ゴールイメージ |
| `guide/01-docker-basics.md` | Docker の概念（仮想マシンとの違い、3要素、使い捨て設計） |
| `guide/02-wsl2-dev-env.md` | WSL2+Ubuntu の仕組みとセットアップ概念 |
| `guide/03-android-emulator.md` | エミュレータ制約の理由・Studio 利用・CLI 利用・Capacitor 接続 |

---

### Task 1: `guide/00-overview.md` を作成する

**Files:**
- Create: `guide/00-overview.md`

- [ ] **Step 1: ファイルを作成する**

以下の内容で `guide/00-overview.md` を作成する:

```markdown
# 開発環境セットアップ ガイド — ロードマップ

## このガイド群について

`container/` にある詳細な手順書を読む前に、全体の構造と目的を理解するためのガイド群です。

| # | ファイル | 学べること |
|---|---|---|
| 1 | [01-docker-basics.md](01-docker-basics.md) | Docker とは何か・コンテナと仮想マシンの違い |
| 2 | [02-wsl2-dev-env.md](02-wsl2-dev-env.md) | WSL2 の仕組みと開発環境の全体像 |
| 3 | [03-android-emulator.md](03-android-emulator.md) | Android エミュレータの起動方法と Capacitor 接続 |

## 最終的に何が動く状態になるか

```
物理PC (Windows)
├─ Hyper-V
│   └─ WSL2 (Ubuntu)
│       └─ Docker Engine
│           └─ 開発コンテナ (Oracle Linux 9)
│               ├─ Node.js / npm
│               └─ Capacitor CLI  ← ここで npm run dev を実行
│
└─ Android エミュレータ (AVD)  ← ここでアプリを表示
    ↑
    localhost でつながる (adb reverse / server.url)
```

## 既存ドキュメントとの関係

- **このガイド群 (`guide/`)** — 概念・考え方・全体像（入門）
- **`container/` の手順書** — 詳細コマンド・設定値のリファレンス

| このガイド | 詳細は |
|---|---|
| 02-wsl2-dev-env.md | [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) |
| 02-wsl2-dev-env.md | [container/5. prerequisite-concepts-guide.md](../container/5.%20prerequisite-concepts-guide.md) |
| 03-android-emulator.md | [container/3. vue-vuetify-capacitor-dev-environment.md](../container/3.%20vue-vuetify-capacitor-dev-environment.md) |
```

- [ ] **Step 2: コミット**

```bash
git add guide/00-overview.md
git commit -m "docs: add guide/00-overview.md — ロードマップ"
```

---

### Task 2: `guide/01-docker-basics.md` を作成する

**Files:**
- Create: `guide/01-docker-basics.md`

- [ ] **Step 1: ファイルを作成する**

以下の内容で `guide/01-docker-basics.md` を作成する:

````markdown
# Docker とは何か

## 1. 仮想マシンとコンテナの違い

「Docker = 軽い仮想マシン」というイメージは **誤り**。根本的に異なる仕組みです。

### 仮想マシン (VM)

```
物理PC
└─ ハイパーバイザ (Hyper-V など)
    ├─ VM-A: OS丸ごと (カーネル + ライブラリ + アプリ)
    └─ VM-B: OS丸ごと (カーネル + ライブラリ + アプリ)
```

- OS丸ごとを模倣するため、起動に数十秒〜数分かかる
- ディスク使用量が数GB単位になる
- カーネルを自前で持つため、強い隔離性がある

### コンテナ

```
物理PC
└─ OS (Linux カーネル)
    └─ Docker Engine
        ├─ コンテナA: ライブラリ + アプリのみ  ← カーネルは共有
        └─ コンテナB: ライブラリ + アプリのみ  ← カーネルは共有
```

- カーネルはホストと共有するため、起動が数秒以下
- ディスク使用量が数百MB〜数GB（カーネル分が不要）
- **コンテナは「隔離されたプロセス」であり、OSではない**

### 今回の構成

```
物理PC (Windows)
└─ Hyper-V (ハイパーバイザ)
    └─ WSL2 (Linux 仮想マシン) ← ここが「本物のLinuxカーネル」
        └─ Docker Engine
            └─ 開発コンテナ (Oracle Linux 9) ← WSL2のカーネルを共有
```

Docker が動くのは **Linux カーネルが必要** なため、Windows上では WSL2 経由になります。

---

## 2. Docker の3要素

### イメージ (Image)

- コンテナの「設計図」
- `Dockerfile` というテキストファイルに「どんな環境を作るか」を記述してビルドする
- 一度ビルドしたイメージは変更不可（イミュータブル）
- Docker Hub などのレジストリで公開・共有できる

### コンテナ (Container)

- イメージから起動した「実行中の環境」
- 同じイメージから何個でも起動できる
- **コンテナの中で行った変更（ファイル作成など）はコンテナを削除すると消える**

### レジストリ (Registry)

- イメージを保存・配布する場所
- Docker Hub (公式)、GitHub Container Registry、自社プライベートレジストリなど
- `docker pull イメージ名` でレジストリからイメージを取得する

---

## 3. なぜ開発環境に Docker を使うのか

### 問題: 「自分のPCでは動く」

チームメンバーごとにインストールしたツールのバージョンが違うと、同じコードでも動作が変わります。

### Dockerによる解決策

```
Dockerfile に「Node.js 22.x を使う」と書く
    ↓
誰がビルドしても同じイメージができる
    ↓
「自分のPCだけ動く」問題が起きない
```

### ホスト環境の汚染を防ぐ

Node.js や Java をホスト(WSL2)に直接インストールすると、バージョン管理が複雑になります。
Docker を使えば、ツールチェーンはコンテナの中にだけ存在し、ホストを汚しません。

---

## 4. 使い捨て設計

コンテナは **壊して作り直すことを前提** に設計します。

```
# コンテナを起動
docker compose up -d

# 開発作業（コンテナ内でコマンド実行）
docker compose exec app npm install

# コンテナを削除（中のファイルは消える）
docker compose down

# 同じイメージで再作成（すぐに同じ状態になる）
docker compose up -d
```

**「コンテナに手作業で設定を入れない」** が原則。
すべての設定は `Dockerfile` または `docker-compose.yml` に記述します。

> 詳細な手順は [container/2. oraclelinux9-jdk21-node22-update.md](../container/2.%20oraclelinux9-jdk21-node22-update.md) を参照。

---

## 次のステップ

→ [02-wsl2-dev-env.md](02-wsl2-dev-env.md) — WSL2 の仕組みと開発環境のセットアップ
````

- [ ] **Step 2: コミット**

```bash
git add guide/01-docker-basics.md
git commit -m "docs: add guide/01-docker-basics.md — Dockerとは何か"
```

---

### Task 3: `guide/02-wsl2-dev-env.md` を作成する

**Files:**
- Create: `guide/02-wsl2-dev-env.md`

- [ ] **Step 1: ファイルを作成する**

以下の内容で `guide/02-wsl2-dev-env.md` を作成する:

````markdown
# WSL2 + Ubuntu 開発環境の考え方

## 1. WSL2 とは何か

WSL (Windows Subsystem for Linux) は Windows 上で Linux を動かす仕組みです。

### WSL1 と WSL2 の違い

| | WSL1 | WSL2 |
|---|---|---|
| 仕組み | Windows カーネルが Linux システムコールを変換 | **Hyper-V 上で本物の Linux カーネルを動かす** |
| 本物のLinuxカーネル | なし | あり |
| Docker が動くか | 動かない | 動く |
| ファイルシステム性能 | Windows 側は速い | Linux 側が速い |

**Docker を使うには WSL2 が必須**（Docker は Linux カーネルの機能を使うため）。
手順書で「VERSION が 2 であること」を確認するのはこのためです。

### 構成の全体像

```
物理PC (Windows 11)
└─ Hyper-V
    └─ WSL2 仮想マシン (Ubuntu)
        ├─ systemd (サービス管理)
        ├─ Docker Engine
        │   └─ 開発コンテナ (Oracle Linux 9)
        │       ├─ Node.js 22
        │       ├─ JDK 21
        │       └─ Vue + Vuetify + Capacitor
        └─ VSCode Server (Remote - WSL 経由で接続)
```

---

## 2. セットアップの考え方

### Step 1: WSL2 + Ubuntu のインストール

PowerShell(管理者)で `wsl --install` を実行するだけで WSL2+Ubuntu が入ります。
バージョン確認で `VERSION=2` になっていることを確認します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 1

### Step 2: systemd の有効化（なぜ必要か）

systemd は Linux のサービス管理システムです。`docker start` などのコマンドが
自動で起動するには systemd が必要です。

WSL2 はデフォルトでは systemd を使わないため、`/etc/wsl.conf` で明示的に有効にします。

```ini
[boot]
systemd=true
```

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 2

### Step 3: Docker Engine のインストール

**Docker Desktop** (GUI アプリ) ではなく **Docker Engine** (CUI のみ) を Ubuntu 内に直接インストールします。

| | Docker Desktop | Docker Engine |
|---|---|---|
| インストール先 | Windows | Ubuntu(WSL2内) |
| ライセンス | 商用利用に有料プランあり | 無料(Apache 2.0) |
| GUI | あり | なし |
| 本番環境との差異 | あり | **ほぼなし** |

開発環境と本番環境(Linux サーバー)の差異を減らすため、Docker Engine を推奨します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 4〜

### Step 4: プロキシ設定の考え方

社内ネットワークでは PAC スクリプト方式のプロキシが一般的ですが、
**Linux の CLI ツール(apt, curl, Docker)は PAC スクリプトを解釈できません**。

```
Windows      → PAC スクリプトで自動設定 ✓
WSL2/Docker  → PAC スクリプト解釈不可 → 固定URLに変換して設定 ✓
```

PAC ファイルを手動でダウンロードし、`return "PROXY ホスト:ポート"` の記述から
プロキシ URL を取り出して設定します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 3〜

---

## 3. ファイルシステムの注意点

WSL2 には2つのファイルシステムが共存します。

```
/mnt/c/Users/miyaw/  ← Windows 側のファイル (遅い)
/home/miyaw/         ← Linux 側のファイル  (速い)
```

**原則: コードは Linux 側に置く**

`/mnt/c/` 経由でアクセスすると、ファイル I/O が大幅に遅くなります。
`git clone` や `npm install` は WSL2 内の `/home/` 以下で行います。

---

## 4. VSCode でのリモート開発

VSCode の **Remote - WSL** 拡張を使うと、Windows 上の VSCode から WSL2 内のコードを直接編集できます。

```
VSCode (Windows) ─── Remote - WSL ───→ VSCode Server (Ubuntu)
                                              ↓
                                        /home/miyaw/project/
```

**開発ループの全体像:**

```
1. VSCode で Remote - WSL を接続
2. ターミナル(Ubuntu)で docker compose up -d
3. docker compose exec app npm run dev
4. Android エミュレータでアプリ確認
```

> 詳細: [container/4. vscode-extension-layer-management.md](../container/4.%20vscode-extension-layer-management.md)

---

## 次のステップ

→ [03-android-emulator.md](03-android-emulator.md) — Android エミュレータの起動と Capacitor 接続
````

- [ ] **Step 2: コミット**

```bash
git add guide/02-wsl2-dev-env.md
git commit -m "docs: add guide/02-wsl2-dev-env.md — WSL2+Ubuntu開発環境の考え方"
```

---

### Task 4: `guide/03-android-emulator.md` を作成する

**Files:**
- Create: `guide/03-android-emulator.md`

- [ ] **Step 1: ファイルを作成する**

以下の内容で `guide/03-android-emulator.md` を作成する:

````markdown
# Android エミュレータと Capacitor アプリの起動

## 1. なぜ WSL2/コンテナの中でエミュレータが動かないのか

Android エミュレータは **KVM (Kernel-based Virtual Machine)** というハードウェア仮想化機能を使います。

```
物理PC (x86_64 CPU の仮想化支援機能)
└─ KVM → Android エミュレータが高速動作
```

しかし今回の構成では:

```
物理PC
└─ Hyper-V (仮想化レイヤー1)
    └─ WSL2 (仮想化レイヤー2)
        └─ Docker コンテナ (仮想化レイヤー3)
            └─ ここで KVM を使おうとする → 入れ子が深すぎて動かない
```

Hyper-V の中では KVM が利用できないため、エミュレータは **必ず Windows 側で動かします**。

---

## 2. 正しい構成

```
Windows側 (ホスト)
├─ Android エミュレータ (AVD)  ← アプリを表示
└─ Android Studio または SDK CLI

WSL2側
└─ Docker コンテナ
    └─ npm run dev  ← 開発サーバー (例: http://localhost:3000)

↕ 接続方法は後述 (adb reverse または server.url)
```

---

## 3-A. Android Studio を使う方法

### Android Studio のセットアップ

1. [developer.android.com/studio](https://developer.android.com/studio) からインストール
2. 初回起動時の SDK Setup Wizard を完了させる
3. SDK の場所を確認: `C:\Users\<user>\AppData\Local\Android\Sdk`
4. 環境変数を設定:
   ```powershell
   $env:ANDROID_HOME = "C:\Users\<user>\AppData\Local\Android\Sdk"
   $env:PATH += ";$env:ANDROID_HOME\platform-tools"
   ```

### AVD の作成

1. Android Studio → **Device Manager** → **Create Virtual Device**
2. **Hardware:** Pixel 7 など (画面サイズ・解像度の基準)
3. **System Image:** `x86_64` を選ぶ
   - `x86_64` を選ぶ理由: Intel/AMD の PC では x86_64 が KVM を使えて高速
   - `arm64` は M1 Mac 向け。Windows では極端に遅くなる
4. **API Level:** 開発対象の最小 API 以上。例: API 33 (Android 13)
5. 作成後、Device Manager の ▶ ボタンで起動

### Capacitor の接続設定

エミュレータと開発サーバーをつなぐ方法は2通りあります。

#### 方法A: `adb reverse` を使う（推奨）

```powershell
# エミュレータ起動後に実行
adb reverse tcp:3000 tcp:3000
```

これにより、エミュレータ内の `http://localhost:3000` が
Windows/WSL2 の開発サーバーに転送されます。

`capacitor.config.ts` は変更不要で、ビルドした APK をそのまま使えます。

#### 方法B: `server.url` を WSL2 の IP に向ける

```typescript
// capacitor.config.ts
const config: CapacitorConfig = {
  server: {
    url: 'http://192.168.x.x:3000',  // WSL2 の IP アドレス
    cleartext: true,
  },
};
```

WSL2 の IP は `ip addr show eth0` で確認できますが、再起動のたびに変わるため
`adb reverse` の方が管理しやすいです。

### 動作確認フロー

```
1. WSL2: docker compose up -d
2. WSL2: docker compose exec app npm run dev
3. Windows: Android Studio でエミュレータ起動
4. Windows: adb reverse tcp:3000 tcp:3000
5. エミュレータのブラウザ or アプリで http://localhost:3000 を開く
```

> Capacitor でネイティブアプリとして動かす場合は、
> `npx cap sync android` → `npx cap open android` → Studio から実行。
> 詳細: [container/3. vue-vuetify-capacitor-dev-environment.md](../container/3.%20vue-vuetify-capacitor-dev-environment.md)

---

## 3-B. CLI のみで使う方法（Android Studio なし）

Android Studio は GUI フロントエンドに過ぎません。
本体は SDK に含まれる CLI ツール群です。CI 環境や Studio を使いたくない場合はこちら。

### SDK のインストール（cmdline-tools）

1. [developer.android.com/studio#command-tools](https://developer.android.com/studio#command-tools) から
   `commandlinetools-win-*.zip` をダウンロード
2. 解凍して配置:
   ```
   C:\Android\sdk\cmdline-tools\latest\
   ├─ bin\
   │   ├─ sdkmanager.bat
   │   └─ avdmanager.bat
   └─ lib\
   ```
3. 環境変数を設定 (PowerShell / システム環境変数):
   ```powershell
   $env:ANDROID_HOME = "C:\Android\sdk"
   $env:PATH += ";$env:ANDROID_HOME\cmdline-tools\latest\bin"
   $env:PATH += ";$env:ANDROID_HOME\platform-tools"
   $env:PATH += ";$env:ANDROID_HOME\emulator"
   ```

### sdkmanager で必要コンポーネントを取得

```powershell
# ライセンスに同意
sdkmanager --licenses

# 必要コンポーネントをインストール
sdkmanager "platform-tools"
sdkmanager "emulator"
sdkmanager "platforms;android-33"          # API 33 のプラットフォーム
sdkmanager "system-images;android-33;google_apis;x86_64"  # エミュレータ用イメージ
sdkmanager "build-tools;33.0.2"            # Gradle ビルドに必要
```

プロキシ環境では以下のオプションを追加:

```powershell
sdkmanager --proxy=http --proxy_host=<host> --proxy_port=<port> "platform-tools"
```

### AVD を CLI で作成する

```powershell
avdmanager create avd `
  --name "Pixel7_API33" `
  --package "system-images;android-33;google_apis;x86_64" `
  --device "pixel_7"
```

確認:

```powershell
avdmanager list avd
# => Name: Pixel7_API33
```

### エミュレータを CLI で起動する

```powershell
emulator -avd Pixel7_API33
```

バックグラウンドで起動する場合:

```powershell
Start-Process emulator -ArgumentList "-avd Pixel7_API33" -WindowStyle Hidden
# 起動完了を待つ
adb wait-for-device
```

---

## 4. Gradle キャッシュの考え方

Capacitor の Android プロジェクトは Gradle でビルドします。
初回ビルド時は依存ライブラリを大量にダウンロードするため時間がかかります。

### キャッシュの種類

| キャッシュの場所 | 内容 | 共有範囲 |
|---|---|---|
| `~/.gradle/caches/` | 依存 JAR・ビルドキャッシュ | ユーザー全体で共有 |
| `android/.gradle/` | プロジェクト固有のキャッシュ | このプロジェクトのみ |
| `android/build/` | ビルド成果物 | このプロジェクトのみ |

`~/.gradle/caches/` が実質的な「ダウンロードキャッシュ」。
一度ダウンロードしたライブラリは次回から再利用されます。

### プロキシ環境でのキャッシュ準備

`android/gradle.properties` にプロキシを設定してからビルドします:

```properties
systemProp.http.proxyHost=proxy.example.com
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=proxy.example.com
systemProp.https.proxyPort=8080
systemProp.http.nonProxyHosts=localhost|127.0.0.1|*.example.com
```

### ビルドからインストールまでの流れ

```powershell
# WSL2側: アプリをビルド (capacitor sync 後)
# ※ Windows側の Gradle で行う場合:

cd C:\dev\ionic-vuetify3\android

# デバッグ APK をビルド
.\gradlew.bat assembleDebug

# 生成された APK をエミュレータにインストール
adb install app\build\outputs\apk\debug\app-debug.apk
```

初回は依存解決で10〜20分かかります。2回目以降はキャッシュが効くため数分で完了します。

---

## 5. Android Studio vs CLI — 使い分けの基準

| シーン | 推奨 |
|---|---|
| レイアウト確認・デバッグ | **Android Studio** (ビジュアルツールが充実) |
| CI/CD パイプライン | **CLI** (GUI 不要) |
| 初回セットアップ | **Android Studio** (SDK 管理が楽) |
| チームの誰かが Studio を持っていない | **CLI** (無料・軽量) |
| Gradle ビルドエラーの調査 | **どちらでも** (ログは同じ) |

---

## まとめ: 起動チェックリスト

```
□ WSL2 で docker compose up -d
□ docker compose exec app npm run dev (開発サーバー起動確認)
□ Windows でエミュレータ起動 (Studio または emulator -avd <name>)
□ adb devices でエミュレータが認識されていることを確認
□ adb reverse tcp:3000 tcp:3000
□ エミュレータで http://localhost:3000 にアクセス → アプリが表示される
```
````

- [ ] **Step 2: コミット**

```bash
git add guide/03-android-emulator.md
git commit -m "docs: add guide/03-android-emulator.md — エミュレータ起動とCapacitor接続"
```

---

## 自己レビュー結果

- **スペックカバレッジ:** 全セクション（Docker基礎・WSL2・Studio利用・CLI利用・Gradleキャッシュ・Capacitor接続）を網羅
- **プレースホルダー:** なし。すべてのコマンドと内容が具体的
- **整合性:** `adb reverse tcp:3000` のポート番号は全ファイルで3000に統一
- **相互参照:** 全ファイルで `container/` への参照リンクが設定されている
