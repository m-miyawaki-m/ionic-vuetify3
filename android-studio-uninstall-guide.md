# Android Studio 完全アンインストール手順書（Windows）

**目的**: オフライン環境へのクリーンセットアップ前に、既存の Android Studio 関連ファイルを完全に削除する  
**対象 OS**: Windows 10 / Windows 11

> この手順を完了してから `android-offline-setup-guide.md` の手順を実施すること。

---

## アンインストール順序

```
① Android Studio 本体をアンインストール
② Android SDK フォルダを削除
③ Android Studio 設定・キャッシュを削除
④ AVD（エミュレータ）データを削除
⑤ Gradle キャッシュを削除
⑥ 環境変数を削除
⑦ JDK をアンインストール（別途インストールしていた場合）
⑧ クリーン確認
```

---

## Step 1: Android Studio 本体をアンインストール

**Windows の設定から削除:**

1. `Win + I` → **アプリ** → **インストールされているアプリ**
2. `Android Studio` を検索
3. **アンインストール** をクリック → ウィザードに従う

または PowerShell から確認:

```powershell
Get-Package | Where-Object { $_.Name -like "*Android Studio*" }
```

---

## Step 2: Android SDK フォルダを削除

```powershell
# SDK の場所を確認
Write-Host $env:LOCALAPPDATA\Android\Sdk

# 削除（数GB あるため時間がかかる）
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Android" -ErrorAction SilentlyContinue

# 確認（何も表示されなければ OK）
Test-Path "$env:LOCALAPPDATA\Android"
```

---

## Step 3: Android Studio 設定・キャッシュを削除

```powershell
# AppData\Roaming 配下（設定ファイル）
Remove-Item -Recurse -Force "$env:APPDATA\Google" -ErrorAction SilentlyContinue

# AppData\Local 配下（キャッシュ）
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\Google" -ErrorAction SilentlyContinue

# JetBrains 関連（Android Studio は JetBrains IDE）
Remove-Item -Recurse -Force "$env:APPDATA\JetBrains\AndroidStudio*" -ErrorAction SilentlyContinue
Remove-Item -Recurse -Force "$env:LOCALAPPDATA\JetBrains\AndroidStudio*" -ErrorAction SilentlyContinue

# 確認
Test-Path "$env:APPDATA\Google"
Test-Path "$env:LOCALAPPDATA\Google"
```

---

## Step 4: AVD（エミュレータ）データを削除

```powershell
# AVD と Android 関連設定
Remove-Item -Recurse -Force "$env:USERPROFILE\.android" -ErrorAction SilentlyContinue

# 確認
Test-Path "$env:USERPROFILE\.android"
```

---

## Step 5: Gradle キャッシュを削除

```powershell
# Gradle ホームディレクトリ（ビルドキャッシュ・Wrapper 一式）
Remove-Item -Recurse -Force "$env:USERPROFILE\.gradle" -ErrorAction SilentlyContinue

# 確認
Test-Path "$env:USERPROFILE\.gradle"
```

> Gradle キャッシュは大きい（1〜3GB）。削除後はオフライン環境では再取得できないため、
> **オフライン用 gradle-cache を持ち込む手順書の通りに復元**すること。

---

## Step 6: 環境変数を削除

### GUI での削除

1. `Win + R` → `sysdm.cpl` → **詳細設定** タブ → **環境変数**
2. **システム環境変数** または **ユーザー環境変数** から以下を削除・修正:

| 変数 | 対処 |
|---|---|
| `ANDROID_HOME` | 変数ごと削除 |
| `ANDROID_SDK_ROOT` | 変数ごと削除（存在する場合） |
| `JAVA_HOME` | 変数ごと削除（JDK も削除する場合） |
| `PATH` | 以下のエントリを削除 |

PATH から削除するエントリ:
```
%ANDROID_HOME%\platform-tools
%ANDROID_HOME%\cmdline-tools\latest\bin
%ANDROID_HOME%\emulator
%JAVA_HOME%\bin
C:\Program Files\Android\...  （直接パスで入っている場合）
```

### PowerShell での確認

```powershell
# 削除後の確認（何も表示されなければ OK）
[System.Environment]::GetEnvironmentVariable("ANDROID_HOME", "Machine")
[System.Environment]::GetEnvironmentVariable("ANDROID_HOME", "User")
[System.Environment]::GetEnvironmentVariable("JAVA_HOME", "Machine")
[System.Environment]::GetEnvironmentVariable("JAVA_HOME", "User")
```

---

## Step 7: JDK をアンインストール（必要な場合）

JDK を別途インストールしていた場合:

1. `Win + I` → **アプリ** → **インストールされているアプリ**
2. `Eclipse Temurin` または `Amazon Corretto` または `Java` を検索
3. **アンインストール**

PowerShell で確認:

```powershell
Get-Package | Where-Object { $_.Name -like "*JDK*" -or $_.Name -like "*Temurin*" -or $_.Name -like "*Corretto*" }
```

---

## Step 8: クリーン確認チェックリスト

以下を全て PowerShell で実行し、**False または エラー** が返ることを確認する:

```powershell
# Android Studio 本体
Test-Path "C:\Program Files\Android\Android Studio"

# Android SDK
Test-Path "$env:LOCALAPPDATA\Android"

# Android Studio 設定
Test-Path "$env:APPDATA\Google"
Test-Path "$env:LOCALAPPDATA\Google"

# AVD データ
Test-Path "$env:USERPROFILE\.android"

# Gradle キャッシュ
Test-Path "$env:USERPROFILE\.gradle"

# 環境変数（何も表示されなければ OK）
[System.Environment]::GetEnvironmentVariable("ANDROID_HOME", "Machine")
[System.Environment]::GetEnvironmentVariable("ANDROID_HOME", "User")

# コマンド（'コマンドが見つからない' エラーになれば OK）
Get-Command adb -ErrorAction SilentlyContinue
Get-Command java -ErrorAction SilentlyContinue
```

全て削除されていることを確認したら、`android-offline-setup-guide.md` の手順に進む。

---

## 参考: 削除対象フォルダ一覧

| フォルダ | 内容 | 目安サイズ |
|---|---|---|
| `%LOCALAPPDATA%\Android\Sdk` | Android SDK 本体 | ~3-5 GB |
| `%APPDATA%\Google\AndroidStudio*` | 設定ファイル | ~数 MB |
| `%LOCALAPPDATA%\Google\AndroidStudio*` | キャッシュ | ~数百 MB |
| `%USERPROFILE%\.android` | AVD・デバッグキー | ~数 GB（AVD による） |
| `%USERPROFILE%\.gradle` | Gradle キャッシュ | ~1-3 GB |
| `C:\Program Files\Android\Android Studio` | AS 本体 | ~2 GB |
