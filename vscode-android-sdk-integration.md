# VSCode × Android SDK 連携 詳細ガイド

**対象プロジェクト**: Vue 3 + Vuetify 4 + Capacitor 7  
**作成日**: 2026-06-08

---

## 概要: なぜ VSCode から Android 開発ができるのか

VSCode は Android 専用 IDE ではないが、`tasks.json` でシェルコマンドを実行できる。  
**Capacitor CLI がブリッジ役**を担い、以下の連携を実現する。

```
VSCode tasks.json
    │
    ├─ npm / npx cap  → Vite ビルド・Capacitor sync
    ├─ gradlew.bat    → Gradle ビルド（AGP 8.7.2）
    └─ adb            → APK インストール・デバイス管理
```

Android Studio は不要（Logcat・署名・AVD 管理など高度な操作時のみ使用）。

---

## 必要な環境変数

tasks.json は `${env:変数名}` で Windows 環境変数を参照する。  
以下が未設定だと特定タスクが失敗する。

| 環境変数 | 参照箇所 | 未設定時の影響 |
|---|---|---|
| `ANDROID_HOME` | `emulator:start` タスク | エミュレータが起動しない |
| `JAVA_HOME` | Gradle Wrapper（タスク外） | `android:build:debug` が失敗 |
| `PATH` ← `%ANDROID_HOME%\platform-tools` | `android:install:debug` の `adb` | `adb` コマンドが見つからない |
| `PATH` ← `%ANDROID_HOME%\emulator` | `emulator:start` の代替実行 | エミュレータ起動不可 |

### 設定確認コマンド

```powershell
# VSCode のターミナルから確認（タスクと同じ環境）
echo $env:ANDROID_HOME   # C:\Users\<name>\AppData\Local\Android\Sdk
echo $env:JAVA_HOME      # C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot
adb --version            # Android Debug Bridge と表示される
```

---

## tasks.json 各タスクのデータフロー

### `dev:browser` — Mode 1: ブラウザ確認

```
npm run dev
  └─ Vite dev server 起動（http://localhost:5173）
       └─ ブラウザで開く（ネイティブ機能なし）
```

ネイティブ API（カメラ・ファイル等）は動作しない。UI・ロジックの確認用。

---

### `build:web` → `cap:sync` — Web ビルド & Android 反映

```
npm run build
  └─ Vite プロダクションビルド → dist/ に出力

npx cap sync android
  ├─ dist/ を android/app/src/main/assets/public/ にコピー
  ├─ capacitor.config.ts の設定を android/app/src/main/res/ に反映
  ├─ capacitor.settings.gradle を再生成（プラグインのサブプロジェクトを更新）
  └─ android/app/build.gradle の依存関係を更新
```

`cap:sync` は `build:web` に `dependsOn` しているため、単独実行でも自動的にビルドから始まる。

---

### `emulator:start` — AVD 起動

```
${env:ANDROID_HOME}/emulator/emulator -avd Pixel6_API33
  └─ AVD 名は tasks.json にハードコード（Pixel6_API33 固定）
       └─ isBackground: true のためターミナルはブロックされない
```

**注意**: AVD 名が `Pixel6_API33` でない場合、タスクが失敗する。  
AVD Manager で名前を確認・統一すること。

---

### `android:livereload` — Mode 2: HMR 付きデプロイ

最も複雑なタスク。内部で以下を自動実行する。

```
npx cap run android --livereload --external
  │
  ├─[1] Vite dev server を起動（ポート 5173）
  │
  ├─[2] PC の LAN IP を自動検出
  │      └─ capacitor.config にサーバー URL として埋め込む
  │         例: { server: { url: "http://192.168.1.10:5173" } }
  │
  ├─[3] Gradle でデバッグ APK をビルド（gradlew.bat assembleDebug）
  │
  ├─[4] adb install -r で APK をデバイスにインストール
  │
  └─[5] APK 起動 → WebView が http://192.168.1.10:5173 に接続
              └─ Vite の HMR が有効（ファイル変更で自動リロード）
```

**実機での注意**: PC と Android 端末が同一 LAN 上にある必要がある。  
Windows Defender でポート 5173 の受信を許可すること。

```powershell
# ファイアウォール許可（管理者 PowerShell）
New-NetFirewallRule -DisplayName "Vite Dev Server" -Direction Inbound -Protocol TCP -LocalPort 5173 -Action Allow
```

---

### `android:build:debug` — Mode 3: APK ビルド

```
.\\gradlew.bat assembleDebug  （cwd: android/）
  │
  ├─ gradle/wrapper/gradle-wrapper.properties の Gradle バージョンを使用
  ├─ settings.gradle → pluginManagement + dependencyResolutionManagement で一元解決
  ├─ AGP 8.7.2 でビルド（~/.gradle/caches/ からキャッシュ使用）
  └─ 出力: android/app/build/outputs/apk/debug/app-debug.apk
```

`dependsOn: cap:sync` のため、`build:web` → `cap:sync` → `assembleDebug` の順で実行される。

---

### `android:install:debug` — APK インストール

```
adb install -r android/app/build/outputs/apk/debug/app-debug.apk
  │
  ├─ -r: 既存インストールを上書き（再インストール）
  ├─ PATH に %ANDROID_HOME%\platform-tools が必要
  └─ デバイスが adb で認識されていること（adb devices で確認）
```

```powershell
# デバイス認識確認
adb devices
# List of devices attached
# emulator-5554  device   ← エミュレータ
# XXXXXXXXXXXXXX device   ← 実機
```

---

### `android:open` — Android Studio を開く

```
npx cap open android
  └─ capacitor.config.ts の androidStudioPath or レジストリから AS を検索して起動
       └─ android/ ディレクトリを Android Studio で開く
```

用途: Logcat 確認・リリースビルド・APK 署名・詳細な AVD 管理。

---

## 拡張機能（extensions.json）

### 現在の設定

```json
{
  "recommendations": [
    "vue.volar",
    "vuetifyjs.vuetify-vscode"
  ]
}
```

### 推奨追加拡張

| 拡張機能 ID | 用途 |
|---|---|
| `vue.volar` | Vue 3 の型チェック・補完（必須） |
| `vuetifyjs.vuetify-vscode` | Vuetify コンポーネント補完（必須） |
| `vscjava.vscode-gradle` | Gradle ファイルのシンタックス・タスクビュー |
| `redhat.vscode-xml` | `AndroidManifest.xml` 等の XML 補完 |

> Android Studio のような全機能 IDE は不要。タスク駆動の開発フローには上記で十分。

---

## settings.json 推奨設定

`.vscode/settings.json` を作成し以下を追加する。

```json
{
  "terminal.integrated.env.windows": {
    "ANDROID_HOME": "${env:ANDROID_HOME}",
    "JAVA_HOME": "${env:JAVA_HOME}"
  },
  "files.exclude": {
    "android/build": true,
    "android/app/build": true,
    "android/.gradle": true
  },
  "search.exclude": {
    "android/build": true,
    "android/app/build": true,
    "android/.gradle": true,
    "node_modules": true
  }
}
```

- `terminal.integrated.env.windows`: VSCode 統合ターミナルに環境変数を明示的に渡す（システム環境変数が引き継がれない場合の保険）
- `files.exclude` / `search.exclude`: Gradle ビルド成果物を VSCode の検索・ファイルツリーから除外

---

## オフライン持ち込み時のチェックポイント

### 持ち込みが必要な VSCode 関連ファイル

```
project/
└─ .vscode/
    ├─ tasks.json       ← タスク定義
    ├─ extensions.json  ← 推奨拡張リスト
    └─ settings.json    ← 上記の設定（作成している場合）
```

### 拡張機能のオフラインインストール

```powershell
# オンライン機: .vsix をダウンロード
# marketplace から手動ダウンロード or code --list-extensions で確認後ダウンロード

# オフライン機: .vsix からインストール
code --install-extension vue.volar-x.x.x.vsix
code --install-extension vuetifyjs.vuetify-vscode-x.x.x.vsix
code --install-extension vscjava.vscode-gradle-x.x.x.vsix
```

### 環境変数が tasks.json に渡らない場合

PowerShell プロファイル (`$PROFILE`) に追記して VSCode 起動前に設定されるようにする。

```powershell
$env:ANDROID_HOME = "C:\Users\<username>\AppData\Local\Android\Sdk"
$env:JAVA_HOME    = "C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot"
$env:PATH        += ";$env:ANDROID_HOME\platform-tools;$env:ANDROID_HOME\emulator"
```

---

## トラブルシューティング

### `emulator:start` が `${env:ANDROID_HOME}` 未解決でエラー

```
原因: ANDROID_HOME が Windows 環境変数に設定されていない
確認: VSCode ターミナルで echo $env:ANDROID_HOME
対処: システム環境変数に ANDROID_HOME を追加後、VSCode を再起動
```

### `adb: command not found`

```
原因: PATH に %ANDROID_HOME%\platform-tools が含まれていない
確認: adb --version をターミナルで実行
対処: PATH に追加後、VSCode を再起動
```

### `android:livereload` で WebView がサーバーに繋がらない

```
原因1: PC と端末が別ネットワーク
確認: PC の IP と端末の接続先 IP が一致しているか確認
原因2: Windows Defender でポート 5173 がブロックされている
対処: 受信規則にポート 5173 を追加（上記コマンド参照）
```

### Gradle ビルドが `JAVA_HOME` エラーで失敗

```
原因: JAVA_HOME が未設定 or JDK 21 でない
確認: echo $env:JAVA_HOME → jdk-21 のパスが表示されること
     java -version → openjdk 21.x.x と表示されること
重要: @capacitor/android 7.6.6 は JavaVersion.VERSION_21 を要求する
```
