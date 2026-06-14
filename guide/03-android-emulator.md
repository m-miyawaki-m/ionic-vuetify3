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
sdkmanager "platforms;android-33"                          # API 33 のプラットフォーム
sdkmanager "system-images;android-33;google_apis;x86_64"  # エミュレータ用イメージ
sdkmanager "build-tools;33.0.2"                            # Gradle ビルドに必要
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
# Capacitor の Android プロジェクトを同期（WSL2 側で実行）
npx cap sync android

# Windows 側で Gradle ビルド
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
