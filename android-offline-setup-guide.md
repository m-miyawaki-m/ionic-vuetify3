# Android 開発環境 オフライン持ち込み手順書

**対象プロジェクト**: Vue 3 + Vuetify 4 + Capacitor 7  
**対象デバイス**: Android 13 (Xnavis 業務ハンドヘルド)、WebView 120-130  
**作業環境**: Windows 11

---

## 前提条件・バージョン一覧

| ツール | バージョン |
|---|---|
| JDK | 21 (Eclipse Temurin / Amazon Corretto) |
| Android Studio | Ladybug (2024.2.1) |
| AGP (Android Gradle Plugin) | 8.7.2 |
| Capacitor | 7.6.6 |
| Node.js | 22 LTS |
| VSCode | 最新版 |

---

## フェーズ 1: オンライン機での準備（キャッシュ育成）

### Step 1: ソフトウェアをインストール

以下を順番にインストールする。

1. **JDK 21** (Eclipse Temurin または Amazon Corretto)
2. **Android Studio Ladybug** (2024.2.1)
3. **Node.js 22 LTS**
4. **VSCode**

### Step 2: 環境変数を設定（Windows システム環境変数）

| 変数名 | 値 |
|---|---|
| `JAVA_HOME` | `C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot` |
| `ANDROID_HOME` | `C:\Users\<username>\AppData\Local\Android\Sdk` |

PATH に以下を追加:

```
%JAVA_HOME%\bin
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\cmdline-tools\latest\bin
%ANDROID_HOME%\emulator
```

設定後、PowerShell を再起動して確認:

```powershell
java -version    # openjdk 21.x.x と表示されること
adb --version    # Android Debug Bridge と表示されること
```

### Step 3: Android Studio で SDK をインストール

Android Studio を起動 → **SDK Manager** で以下をインストール:

| コンポーネント | 用途 |
|---|---|
| Android SDK Platform 33 | 実機ターゲット（Android 13） |
| Android SDK Platform 35 | compileSdk |
| Android SDK Build-Tools 35.x.x | ビルドツール |
| Android SDK Platform-Tools | adb 等 |
| Android SDK Command-line Tools | avdmanager 等 |
| Android Emulator | エミュレータ本体 |
| System Image: API 33 / Google APIs / x86_64 | AVD 用イメージ |

### Step 4: AVD（エミュレータ）を作成

Android Studio → **AVD Manager** で以下を作成:

| 項目 | 設定値 |
|---|---|
| Device | Pixel 6（任意） |
| System Image | API 33 / Android 13 / x86_64 / Google APIs |
| AVD 名 | `Pixel6_API33` |
| RAM | 2048 MB 以上 |

> **重要**: AVD 名は必ず `Pixel6_API33` にすること。`.vscode/tasks.json` の `emulator:start` タスクにこの名前がハードコードされているため、名前が違うとタスクからエミュレータを起動できない。

作成後、エミュレータを**一度起動して正常動作を確認**する。

### Step 5: プロジェクトの Gradle キャッシュを育成

プロジェクトディレクトリで以下を実行し、全依存関係をダウンロードさせる:

```powershell
npm run build
npx cap sync android
npx cap open android   # Android Studio が開く
```

Android Studio 側で **Gradle sync が完走するまで待つ**（初回は数分かかる）。  
これで `%USERPROFILE%\.gradle` にビルドに必要な全キャッシュが蓄積される。

---

## フェーズ 2: 持ち込みパッケージの作成

USB メモリまたは外付け HDD に以下の構成でコピーする（合計 **~8-10 GB**）。

```
offline-package/
├── installers/                    (~1.3 GB)
│   ├── jdk-21_windows-x64_bin.msi
│   ├── android-studio-2024.2.1.xx-windows.exe
│   ├── node-v22.x.x-x64.msi
│   └── VSCodeSetup-x64.exe
├── android-sdk/                   (~3 GB)
│   └── %ANDROID_HOME% をそのままコピー
├── gradle-cache/                  (~1-2 GB)
│   └── %USERPROFILE%\.gradle をそのままコピー
├── avd/                           (~200 MB)
│   └── %USERPROFILE%\.android\avd\ をコピー
├── vscode-extensions/
│   └── Vue.volar-x.x.x.vsix      (marketplace から手動ダウンロード)
└── project/                       (~500 MB)
    └── プロジェクト一式（node_modules ごとコピー）
```

> **注意**: Gradle sync が完走していない状態でコピーすると、オフライン機でビルドに失敗する。必ず Step 5 を完走させてからコピーすること。

---

## フェーズ 3: オフライン機への展開

### Step 1: インストーラを順番に実行

```
JDK 21 → Node.js 22 → Android Studio Ladybug → VSCode
```

### Step 2: SDK・キャッシュを復元

```powershell
xcopy /E /I offline-package\android-sdk  "$env:LOCALAPPDATA\Android\Sdk"
xcopy /E /I offline-package\gradle-cache "$env:USERPROFILE\.gradle"
xcopy /E /I offline-package\avd          "$env:USERPROFILE\.android\avd"
```

### Step 3: 環境変数を設定

オンライン機と同じ内容を設定する（`<username>` はオフライン機のユーザー名に合わせる）:

| 変数名 | 値 |
|---|---|
| `JAVA_HOME` | `C:\Program Files\Eclipse Adoptium\jdk-21.x.x-hotspot` |
| `ANDROID_HOME` | `C:\Users\<オフライン機ユーザー名>\AppData\Local\Android\Sdk` |

PATH にも同様に追加する。

### Step 4: Gradle をオフラインモードに固定

```powershell
Add-Content "$env:USERPROFILE\.gradle\gradle.properties" "org.gradle.offline=true"
```

Android Studio も: **Settings → Build → Gradle → Offline work にチェック**

### Step 5: プロジェクトを展開

```powershell
xcopy /E /I offline-package\project C:\dev\my-app
```

### Step 6: local.properties のパスを書き換え

`C:\dev\my-app\android\local.properties` をテキストエディタで開き、`sdk.dir` をオフライン機のパスに修正:

```properties
sdk.dir=C:\\Users\\<オフライン機ユーザー名>\\AppData\\Local\\Android\\Sdk
```

### Step 7: VSCode 拡張をインストール

```powershell
code --install-extension offline-package\vscode-extensions\Vue.volar-x.x.x.vsix
```

---

## 動作確認チェックリスト

展開完了後、以下を順番に確認する:

```
□ java -version → openjdk 21.x.x と表示される
□ adb --version → Android Debug Bridge と表示される
□ JAVA_HOME / ANDROID_HOME / PATH が正しく設定されている
□ android/local.properties の sdk.dir がオフライン機のパスになっている
□ %USERPROFILE%\.gradle\gradle.properties に org.gradle.offline=true がある
□ Android Studio → Gradle → Offline work にチェックが入っている
□ AVD 名が Pixel6_API33 で作成されている（Step 4 参照）
□ VSCode: Ctrl+Shift+P → Tasks: Run Task → emulator:start でエミュレータが起動する
□ VSCode: Tasks: Run Task → android:build:debug で APK がビルドできる
□ VSCode: Tasks: Run Task → android:livereload でエミュレータに APK がデプロイされる
□ livereload で WebView が Vite dev server に接続し、画面が表示される（Vite 8 動作確認）
□ 実機（Xnavis）で livereload を使う場合: Windows Defender でポート 5173 の受信を許可する
```

> **Vite 8 の livereload 動作は要確認。** ビルド（`android:build:debug`）は問題なし。`android:livereload` が失敗する場合はトラブルシューティングを参照。

---

## VSCode タスク一覧

`Ctrl+Shift+P` → **Tasks: Run Task** から実行する。

| タスク名 | 用途 |
|---|---|
| `dev:browser` | Vite dev server をブラウザで起動（ネイティブ機能なし） |
| `emulator:start` | AVD エミュレータを起動 |
| `android:livereload` | エミュレータ/実機に APK をデプロイ + HMR |
| `android:build:debug` | デバッグ APK をビルド |
| `android:install:debug` | 接続デバイスに APK をインストール |
| `android:open` | Android Studio でリリースビルド・署名・Logcat |

---

## トラブルシューティング

### Gradle ビルドが失敗する

```
原因: JAVA_HOME が未設定 or JDK 21 でない
確認: java -version → openjdk 21.x.x であること
確認: echo $env:JAVA_HOME → jdk-21 のパスが表示されること
```

```
原因: Gradle キャッシュが不完全（オンライン機で sync 完走前にコピーした）
対処: オンライン環境で再度 Gradle sync を完走させ、gradle-cache を再コピーする
```

### エミュレータが起動しない

```
原因: ANDROID_HOME が未設定 or パスが違う
確認: echo $env:ANDROID_HOME → Sdk フォルダのパスが表示されること
確認: $env:ANDROID_HOME\emulator\emulator.exe が存在すること
```

### adb が見つからない

```
確認: echo $env:PATH に %ANDROID_HOME%\platform-tools が含まれること
対処: 環境変数 PATH を再確認・再設定する
```

### android:livereload が失敗する（WebView が繋がらない）

```
原因: Vite 8 と Capacitor 7 CLI の相性問題の可能性
確認: npx cap run android --livereload --external を直接実行してエラー内容を確認する
```

Vite 8 で失敗する場合は Vite 6 へのダウングレードで解決できる可能性がある。
オンライン環境で以下を実行してから node_modules を再パッケージ化する:

```powershell
cd C:\dev\vue-vuetify3-orval-material
npm install vite@6 @vitejs/plugin-vue@5
npm run build   # ビルドが通ることを確認
```

その後 offline-package の project/ を再コピーする。
