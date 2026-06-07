# Android 10 端末 WebView バージョンアップ 調査結果

**調査日**: 2026-06-07  
**対象**: オフライン業務用 Android 10 ハンドヘルド端末  
**結論**: **WebView のバージョンアップは不可**

---

## 結論

オフライン業務用 Android 10 端末において、WebView を 120 番台へ更新することは**技術的に不可能**と判断する。

---

## 理由

### Android 10 における WebView の仕組み

Android 10 以降、WebView の実体は「Android System WebView」アプリではなく **Chrome** に統合されている。  
すなわち、**Chrome のバージョン = WebView のバージョン** となる。

| Android バージョン | WebView の実体 |
|---|---|
| Android 7〜9 | Android System WebView（独立したシステムアプリ） |
| Android 10 以降 | Chrome が WebView provider を兼任 |

### 業務端末固有の制限

業務用ハンドヘルド端末（Xnavis 等）は一般に以下の構成をとる。

| 項目 | 状態 | 影響 |
|---|---|---|
| Google Play Services (GMS) | **非搭載** | Chrome は GMS に依存するため動作不可 |
| Chrome | **非インストール** または GMS なしで起動不可 | WebView provider が存在しない |
| ネットワーク | オフライン環境 | Play Store による自動更新不可 |

Chrome は Google Mobile Services (GMS) に強く依存しており、GMS 非搭載端末には APK をサイドロードしても正常に動作しない。

### サイドロードを試みた場合の問題

仮に Chrome APK を USB 経由でサイドロードしても、以下の問題が発生する。

1. **GMS 依存エラー** — 起動時に Google Play Services を要求してクラッシュ
2. **署名不一致** — メーカーが焼いたシステムアプリの署名と異なる場合、インストール自体が拒否される
3. **再起動後の巻き戻し** — システムパーティションにアプリが存在する場合、ファクトリーイメージに戻る

---

## 確認コマンド（参考）

```powershell
# 現在の WebView バージョン確認
adb shell dumpsys webviewupdate

# Chrome のインストール確認
adb shell pm list packages | grep chrome

# WebView provider 確認（Android 10）
adb shell cmd webviewupdate get-current-webview-package
```

---

## 代替案

| 案 | 実現性 | 内容 |
|---|---|---|
| Android 13 端末（現行 Xnavis）へ移行 | ◎ | WebView 120 番台搭載済み。根本解決 |
| アプリ側を低 WebView バージョンで動作するよう実装 | ○ | Vuetify 4 は WebView 80 番台以上で大半動作する |
| カスタム ROM 焼き（AOSP WebView） | ✗ | root 権限・技術知識が必要。業務端末では非現実的 |
| MDM 経由での展開 | ✗ | オフライン環境・GMS 非搭載では MDM も機能しない |

---

## 推奨対応

**Android 13 端末（WebView 120 番台搭載）への切り替えを推奨する。**

Android 10 端末での WebView 更新は構造的に不可能であり、アプリ側での対処にも限界がある。  
新規導入分から Android 13 端末を採用し、Android 10 端末は本システムの対象外とすることが現実的な判断となる。

---

## 補足: 対象プロジェクトの動作要件

| 項目 | 要件 | Android 10 端末 | Android 13 端末 |
|---|---|---|---|
| WebView バージョン | 120 番台以上 | **不可** | ◎ (120〜130) |
| Capacitor 7 動作 | WebView 60 以上 | △ (バージョン依存) | ◎ |
| Vue 3 / Vuetify 4 | WebView 80 以上 | △ (バージョン依存) | ◎ |
