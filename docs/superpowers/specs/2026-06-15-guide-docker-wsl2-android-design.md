# 設計ドキュメント: Docker/WSL2/Android 入門ガイド群

Date: 2026-06-15

## 目的

「そもそもDockerとは何か」から始まり、WSL2+Ubuntu開発環境の構築、
Androidエミュレータの起動までをエンドツーエンドで説明するガイド群を作成する。

対象読者: 自分(宮脇)の整理用だが、将来チームメンバーへ渡せる品質を目指す。

## ディレクトリ構成

```
guide/
├── 00-overview.md          # ロードマップ・全体像
├── 01-docker-basics.md     # Docker とは何か
├── 02-wsl2-dev-env.md      # WSL2 + Ubuntu 開発環境構築
└── 03-android-emulator.md  # Android エミュレータ〜Capacitor 実行
```

既存の `container/` との住み分け:
- `guide/` = 概念理解・考え方・全体像（入門）
- `container/` = 詳細なセットアップ手順・コマンドリファレンス

## 各ファイルの設計

### 00-overview.md

- この4本を読む順番と「何が分かるか」の一覧表
- 最終的な構成ゴールイメージ（テキスト図）
- 既存 `container/` との関係説明

### 01-docker-basics.md

- 仮想マシン vs コンテナの違い（物理PC → Hyper-V → WSL2 → Docker の階層図）
- Docker の3要素: イメージ / コンテナ / レジストリ
- なぜ開発環境に Docker を使うのか（再現性・環境汚染防止）
- コンテナの「使い捨て設計」の考え方

### 02-wsl2-dev-env.md

- WSL2 とは何か（Windows上でLinuxカーネルが動く仕組み、WSL1との違い）
- 構成の全体像: Windows → Hyper-V → WSL2(Ubuntu) → Docker Engine
- セットアップの流れ（概念説明メイン、コマンドは補足）
  1. WSL2 + Ubuntu インストール
  2. systemd 有効化（なぜ必要か）
  3. Docker Engine インストール（Docker Desktop との違い）
  4. プロキシ設定の考え方（PACスクリプトが使えない理由）
- ファイルシステムの注意点（/mnt/c/ とWSL2側のパス混在）
- VSCode Remote - WSL での接続方法（開発ループの全体像）
- 詳細コマンドは `container/1.` に委譲

### 03-android-emulator.md

**Section A: エミュレータが動く場所を理解する**
- なぜWSL2/コンテナの中でエミュレータが動かないのか（KVMの制約）
- 正しい構成:
  ```
  Windows側: Android Studio/SDK + AVD (エミュレータ)
  WSL2側:    Docker + 開発サーバー (npm run dev)
  ↕ localhost でつながる
  ```

**Section B: Android Studio を使う方法**
- Android Studio のセットアップ（Java/SDK パスの確認）
- AVD 作成手順（API レベル選定、x86_64 を選ぶ理由）
- Capacitor の接続設定
  - `capacitor.config.ts` の `server.url` をWSL2のIPに向ける方法
  - `adb reverse` でポートフォワードする方法
- 動作確認フロー: `npm run dev` → Studio起動 → AVD起動 → アプリ表示確認

**Section C: CLIのみで使う方法（Android Studio なし）**
- Android Studio なしで何ができるか（Studioは GUI フロントエンド、本体はSDKツール群）
- SDK のインストール（cmdline-tools / sdkmanager）
  - `ANDROID_HOME` / `ANDROID_SDK_ROOT` の設定
  - Windows側 vs WSL2側の配置判断基準
- AVD をCLIで作成する（`avdmanager create avd`）
- エミュレータをCLIで起動する（`emulator -avd <name>`）
- Gradle キャッシュの考え方
  - `~/.gradle/caches/` に溜まるもの（依存JAR等）
  - `android/.gradle/` との違い
  - CI/プロキシ環境でキャッシュを事前準備する方法
- ビルド〜インストールの流れ（`./gradlew assembleDebug` → `adb install`）
- Android Studio との使い分け基準

## 既存ドキュメントとの参照関係

| 本ガイド | 参照先（詳細） |
|---|---|
| 02-wsl2-dev-env.md | container/1. wsl2-ubuntu-docker-proxy-setup.md |
| 02-wsl2-dev-env.md | container/5. prerequisite-concepts-guide.md |
| 03-android-emulator.md | container/3. vue-vuetify-capacitor-dev-environment.md |
