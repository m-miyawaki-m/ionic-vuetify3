# Android 開発環境 更新・持ち込み手順 & Android Studio 設定詳細

**前提**: `android-offline-setup-guide.md` の初回セットアップが完了済みであること

---

## Part 1: SDK / Gradle 更新をオフラインに持ち込む

### 更新の種類と影響範囲

| 更新内容 | 影響するキャッシュ | 持ち込み対象 |
|---|---|---|
| SDK Platform追加 (例: API 36) | `ANDROID_HOME\platforms\` | `android-sdk/platforms/android-36` |
| Build-Tools更新 | `ANDROID_HOME\build-tools\` | `android-sdk/build-tools/36.x.x` |
| AGP (Android Gradle Plugin) 更新 | `%USERPROFILE%\.gradle\caches\` | `gradle-cache/caches/` 差分 |
| アプリの依存ライブラリ追加・更新 | `%USERPROFILE%\.gradle\caches\` | `gradle-cache/caches/` 差分 |
| Gradle ラッパー本体更新 | `%USERPROFILE%\.gradle\wrapper\` | `gradle-cache/wrapper/` 差分 |

---

### ケース A: Android SDK バージョンを追加する

#### オンライン機での作業

```powershell
# 1. Android Studio → Tools → SDK Manager を開く
#    または SDK Manager の CLI を使う場合:
$sdkmanager = "$env:ANDROID_HOME\cmdline-tools\latest\bin\sdkmanager.bat"

# インストール済みの一覧確認
& $sdkmanager --list_installed

# 新しい Platform をインストール（例: API 36）
& $sdkmanager "platforms;android-36"
& $sdkmanager "build-tools;36.0.0"
```

#### 差分の特定と持ち込み

```powershell
# ANDROID_HOME の該当ディレクトリだけコピーすればOK
# フォルダ構造例:
# ANDROID_HOME\
#   platforms\
#     android-33\   ← 既存
#     android-35\   ← 既存
#     android-36\   ← 新規追加分 ← これだけ持ち込む
#   build-tools\
#     35.0.0\       ← 既存
#     36.0.0\       ← 新規追加分

# 新規フォルダのみUSBメモリにコピー
xcopy /E /I "$env:ANDROID_HOME\platforms\android-36" "D:\sdk-update\platforms\android-36"
xcopy /E /I "$env:ANDROID_HOME\build-tools\36.0.0"  "D:\sdk-update\build-tools\36.0.0"
```

#### オフライン機での復元

```powershell
xcopy /E /I "D:\sdk-update\platforms\android-36" "$env:ANDROID_HOME\platforms\android-36"
xcopy /E /I "D:\sdk-update\build-tools\36.0.0"  "$env:ANDROID_HOME\build-tools\36.0.0"

# Android Studio の SDK Manager を開いて認識されているか確認
# （インストール済みリストに表示されれば成功）
```

---

### ケース B: AGP または Gradle Wrapper バージョンを更新する

#### オンライン機での作業

```
# 1. プロジェクトの android/build.gradle を編集
plugins {
    id 'com.android.application' version '8.9.0' apply false  // バージョン変更
}

# 2. gradle/wrapper/gradle-wrapper.properties を編集
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip  // バージョン変更

# 3. Android Studio で Gradle sync を実行（または）
```

```powershell
# コマンドラインで sync
cd C:\dev\my-app\android
.\gradlew dependencies --no-daemon
# ↑ これで .gradle\caches 配下に新バージョンのファイルが展開される
```

#### .gradle キャッシュの差分持ち込み

```powershell
# 方法1: robocopy で差分同期（推奨）
# /MIR = ミラーリング（新規・更新ファイルのみ転送）
robocopy "$env:USERPROFILE\.gradle" "D:\sdk-update\gradle-cache" /MIR /XD tmp /R:1 /W:1

# 方法2: 更新日付フィルタで絞り込み
# 今日以降に変更されたファイルだけ確認する
Get-ChildItem "$env:USERPROFILE\.gradle\caches" -Recurse |
    Where-Object { $_.LastWriteTime -gt (Get-Date).AddDays(-1) } |
    Select-Object FullName, LastWriteTime | Out-File "D:\sdk-update\changed-files.txt"
```

#### 重要: .gradle 内のキャッシュ構造

```
%USERPROFILE%\.gradle\
  caches\
    modules-2\
      files-2.1\          ← ライブラリのJAR/POM（主要キャッシュ）
        com.android.tools.build\
          gradle\8.9.0\   ← AGPのJAR
        org.jetbrains.kotlin\
        ...
      metadata-*/         ← メタデータキャッシュ
    transforms-*/         ← Transform後のファイル（再生成可能だが重い）
  wrapper\
    dists\
      gradle-8.13-bin\    ← Gradleラッパー本体
```

> **注意**: `transforms-*/` は再生成可能なので、容量が大きければ除外してもビルドは通る（初回ビルドが遅くなるだけ）。

---

### ケース C: アプリの依存ライブラリを追加・更新する

#### 典型的な手順（オンライン機）

```powershell
# 例: Retrofit 追加
# android/app/build.gradle に追記後

cd C:\dev\my-app\android
.\gradlew app:dependencies --configuration releaseRuntimeClasspath --no-daemon
# または単に sync だけでも取得できる
.\gradlew app:assembleDebug --no-daemon  # ビルドまで通しておく
```

```powershell
# 追加されたJARの確認
Get-ChildItem "$env:USERPROFILE\.gradle\caches\modules-2\files-2.1" -Recurse |
    Where-Object { $_.LastWriteTime -gt (Get-Date).AddHours(-2) } |
    Select-Object Name
```

#### 持ち込み

```powershell
# .gradle\caches 全体を再コピー（確実）
robocopy "$env:USERPROFILE\.gradle\caches" "D:\sdk-update\gradle-cache\caches" /MIR /R:1 /W:1
```

---

### ケース D: Gradle ラッパーバージョンだけ更新する

```powershell
# 新バージョンのラッパーzipがここに展開される
# %USERPROFILE%\.gradle\wrapper\dists\gradle-8.13-bin\

# これだけ持ち込めばOK
xcopy /E /I "$env:USERPROFILE\.gradle\wrapper" "D:\sdk-update\gradle-cache\wrapper"
```

---

### 差分持ち込みのベストプラクティス

```
【初回】  → ANDROID_HOME 全体 + .gradle 全体を持ち込む（8-10GB）
【更新時】 → 変更したキャッシュフォルダだけ差分で持ち込む（数百MB）

更新内容       持ち込むフォルダ                             目安容量
SDK Platform   ANDROID_HOME\platforms\android-XX\            ~600MB
Build-Tools    ANDROID_HOME\build-tools\XX.X.X\              ~400MB
AGP更新        .gradle\caches\modules-2\files-2.1\com.android.tools.build\  ~200MB
ライブラリ追加 .gradle\caches\modules-2\files-2.1\<groupId>\ 数MB〜数十MB
Gradle本体     .gradle\wrapper\dists\gradle-X.X-bin\         ~150MB
```

---

## Part 2: Android Studio 設定の詳細

### 設定画面の開き方

```
File → Settings  (Ctrl+Alt+S)
または
Android Studio ロゴ画面 → Customize → All settings...
```

---

### 2-1. Gradle 設定 (最重要)

**Settings → Build, Execution, Deployment → Build Tools → Gradle**

| 設定項目 | 推奨値 | 説明 |
|---|---|---|
| Gradle user home | `C:\Users\<user>\.gradle` | キャッシュの保存場所。変更するなら全環境で統一する |
| Use Gradle from | `'wrapper' task in Gradle build file` | プロジェクトの gradle-wrapper を使う（推奨） |
| Gradle JDK | `jbr-21 (JetBrains Runtime...)` または `JAVA_HOME` | JDK 21 が選ばれていること |
| **Offline work** | **チェックON（オフライン環境）** | ネット接続を試みなくなる |

> **Gradle JDK の選び方**:  
> `JAVA_HOME` を選ぶとシステム環境変数のJDKを使う。  
> `Project SDK` は Android Studio 付属JDK（JetBrains Runtime）。  
> どちらでも動くが、`JAVA_HOME`（= JDK 21）に統一しておくと CLI との一貫性が保てる。

---

### 2-2. メモリ設定 (大規模プロジェクトで重要)

#### IDE 本体のヒープサイズ

**Help → Edit Custom VM Options** （`studio.vmoptions` を直接編集）

```
-Xms512m        # 初期ヒープ
-Xmx4096m       # 最大ヒープ（デフォルト2GBを4GBに拡張）
-XX:ReservedCodeCacheSize=512m
```

適用は Android Studio 再起動後。

#### Gradle デーモンのヒープサイズ

`android/gradle.properties` に記述:

```properties
# Gradle デーモン（ビルド処理プロセス）のメモリ
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m -XX:+HeapDumpOnOutOfMemoryError -Dfile.encoding=UTF-8
```

> **目安**: 物理メモリ 16GB → Xmx4g、8GB → Xmx2g

---

### 2-3. gradle.properties の全設定項目

```properties
# ========================
# android/gradle.properties  (プロジェクトスコープ)
# ========================

# --- パフォーマンス ---
org.gradle.jvmargs=-Xmx2g -XX:MaxMetaspaceSize=512m
org.gradle.parallel=true          # モジュールを並列ビルド
org.gradle.caching=true           # ビルドキャッシュを有効化（2回目以降が速い）
org.gradle.daemon=true            # デーモンを使う（デフォルトtrue）
org.gradle.configureondemand=true # 必要なモジュールだけ設定

# --- オフライン ---
# org.gradle.offline=true         # オフライン機ではコメントを外す

# --- Android 固有 ---
android.useAndroidX=true
android.enableJetifier=false      # AndroidX 移行済みなら false でOK
android.nonTransitiveRClass=true  # R クラスの最適化（AGP 7.0+）
android.defaults.buildfeatures.buildconfig=false  # BuildConfig 不要なら false

# --- Kotlin ---
kotlin.code.style=official
kotlin.incremental=true           # インクリメンタルコンパイル
```

```properties
# ========================
# %USERPROFILE%\.gradle\gradle.properties  (グローバルスコープ)
# ========================

# オフライン環境では必須
org.gradle.offline=true

# グローバルにキャッシュを有効化
org.gradle.caching=true
```

---

### 2-4. local.properties の管理

`android/local.properties` は **git管理対象外** にすること（`.gitignore` に含まれているはず）。

```properties
# local.properties（プロジェクトルートの android/ 配下）

# SDK のパス（オフライン機のユーザー名に合わせて変更）
sdk.dir=C\:\\Users\\<ユーザー名>\\AppData\\Local\\Android\\Sdk

# NDK を使う場合（通常 Capacitor では不要）
# ndk.dir=C\:\\Users\\<ユーザー名>\\AppData\\Local\\Android\\Sdk\\ndk\\26.x.x
```

> **バックスラッシュは `\\` でエスケープ**。Windowsパスのみこの書き方が必要。

---

### 2-5. SDK Manager の詳細

**Tools → SDK Manager** または **Settings → Appearance & Behavior → System Settings → Android SDK**

#### タブ別の説明

| タブ | 内容 | 操作のコツ |
|---|---|---|
| **SDK Platforms** | API バージョン別 Platform | `Show Package Details` にチェックすると細かい内訳が出る |
| **SDK Tools** | adb / Emulator / Build-Tools 等 | `Show Package Details` で旧バージョンも選択可 |
| **SDK Update Sites** | ダウンロード元URL | オフライン環境では全て無効化してもよい |

#### 必須コンポーネントの一覧（このプロジェクト向け）

```
SDK Platforms:
  ✅ Android 13 (API 33)    ← 実機ターゲット
  ✅ Android 15 (API 35)    ← compileSdk

SDK Tools:
  ✅ Android SDK Build-Tools 35.x.x
  ✅ Android SDK Platform-Tools  (adb)
  ✅ Android SDK Command-line Tools (latest)
  ✅ Android Emulator
  ✅ Google Play services  (任意)
```

---

### 2-6. AVD Manager の詳細

**Tools → Device Manager**

#### AVD の設定項目

| 項目 | 設定値 | 備考 |
|---|---|---|
| Device | Pixel 6 | 画面サイズ・解像度の定義 |
| System Image | API 33 / x86_64 / Google APIs | x86_64 = エミュレータ動作が速い |
| AVD Name | `Pixel6_API33` | tasks.json にハードコード済み。変更禁止 |
| RAM | 2048 MB | 4096 MB あると快適 |
| VM Heap | 512 MB | デフォルトで可 |
| Internal Storage | 2048 MB | アプリが大きければ増やす |
| Graphics | Automatic | 環境依存。落ちるなら Software に変更 |

#### エミュレータが遅い場合

**Settings → Appearance & Behavior → System Settings → Android SDK → SDK Tools**
→ `Intel HAXM` または `Android Emulator Hypervisor Driver for AMD Processors` をインストール

> Hyper-V 有効環境では HAXM が使えない。その場合はWindows Hyper-V ベースの高速化が自動で使われる（Windows 11 では通常OK）。

---

### 2-7. Logcat の使い方

**View → Tool Windows → Logcat** (または下部の `Logcat` タブ)

#### フィルタリング

```
# パッケージ名でフィルタ（アプリのログだけ表示）
package:com.example.myapp

# タグでフィルタ
tag:Capacitor

# ログレベル
level:warn  # warn 以上だけ表示

# 複合フィルタ
package:com.example.myapp level:debug tag:WebView
```

#### よく使うフィルタ（Capacitor 開発向け）

```
tag:Capacitor             # Capacitorのログ
tag:chromium              # WebView のコンソールログ
tag:CapacitorBridge       # JS-Native ブリッジ
tag:ActivityManager       # Activity ライフサイクル
```

---

### 2-8. Run/Debug Configuration

**Run → Edit Configurations**

Capacitor 開発では基本的に VSCode タスク経由で実行するため、ここは参照程度でOK。  
ただし以下は確認しておく:

| 項目 | 確認内容 |
|---|---|
| Module | `app` が選ばれているか |
| Deploy | `Default APK` でOK |
| Launch Activity | `Default Activity` でOK |

---

### 2-9. Proxy 設定（社内環境で有線LAN経由の場合）

**Settings → Appearance & Behavior → System Settings → HTTP Proxy**

```
オフライン環境: No proxy を選択

社内Proxy経由でネット接続する場合:
  Manual proxy configuration を選択
  Host name: proxy.example.com
  Port:      8080
  No proxy for: localhost,127.0.0.1
```

Gradle にも同様の設定が必要:

```properties
# gradle.properties に追記
systemProp.http.proxyHost=proxy.example.com
systemProp.http.proxyPort=8080
systemProp.https.proxyHost=proxy.example.com
systemProp.https.proxyPort=8080
systemProp.http.nonProxyHosts=localhost|127.0.0.1
```

---

### 2-10. Build Variants

**View → Tool Windows → Build Variants**

| Variant | 用途 |
|---|---|
| `debug` | 開発・デバッグ。署名不要。`android:build:debug` タスクで生成 |
| `release` | リリース用。署名が必要。`android:open` から Android Studio でビルド |

VSCode タスクは全て `debug` variant を使っている。リリースビルドは Android Studio の **Build → Generate Signed Bundle/APK** から行う。

---

## Part 3: よくある設定ミスと確認手順

### チェックリスト（更新後・初回セットアップ後に確認）

```
□ Settings → Gradle → Gradle JDK が JDK 21 を指しているか
□ Settings → Gradle → Offline work がオフライン環境でONになっているか
□ android/local.properties の sdk.dir がこの機のパスになっているか
□ android/gradle.properties の org.gradle.offline=true （グローバル設定と合わせて）
□ JAVA_HOME 環境変数が JDK 21 のパスを指しているか（PowerShell で $env:JAVA_HOME で確認）
□ ANDROID_HOME 環境変数が Sdk フォルダを指しているか
□ AVD 名が Pixel6_API33 であるか（Device Manager で確認）
```

### Gradle sync エラー時のデバッグ手順

```powershell
# 1. エラーログを詳細表示
cd C:\dev\my-app\android
.\gradlew app:assembleDebug --info --stacktrace 2>&1 | Tee-Object -FilePath build-log.txt

# 2. オフラインモードを一時的に外してみる（ネット接続が可能な場合）
.\gradlew app:assembleDebug --no-offline

# 3. Gradle キャッシュを一部クリアして再実行
.\gradlew clean
.\gradlew app:assembleDebug
```

### Android Studio のキャッシュクリア

```
File → Invalidate Caches → Invalidate and Restart
```

> これをやると Android Studio 自身のインデックスが再構築される（数分かかる）。  
> **Gradle キャッシュ（.gradle フォルダ）は消えない**ので、オフライン環境でも安全に実行できる。
