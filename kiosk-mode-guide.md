# Android端末 キオスクモード 調査レポート

> 調査日: 2026-06-12  
> 対象: 業務用Androidハンドヘルド端末（Xnavisなど）  
> 構成: Vue3 + Vuetify3 + Capacitor/Ionic

---

## 1. キオスクモードの実装方法

### Lock Task Mode（中核技術）

AndroidキオスクモードはAPIレベルで **Lock Task Mode** を中核技術として実現される。

```kotlin
// Device Policy Controller (DPC) 側
dpm.setLockTaskPackages(adminName, arrayOf("com.example.kioskapp"))

// アプリ側
activity.startLockTask()   // 開始
activity.stopLockTask()    // 終了
```

| 設定 | 挙動 |
|------|------|
| Device Owner **あり** | プロンプトなし・ナビゲーション無効・完全ロック |
| Device Owner **なし** | ユーザー承諾プロンプトあり（スクリーンピニング相当） |

### UI要素の制御（API 28+ / Android 9.0+）

```kotlin
dpm.setLockTaskFeatures(adminName,
    DevicePolicyManager.LOCK_TASK_FEATURE_HOME or
    DevicePolicyManager.LOCK_TASK_FEATURE_NOTIFICATIONS
)
```

### セキュリティ強化

```kotlin
arrayOf(
    UserManager.DISALLOW_FACTORY_RESET,
    UserManager.DISALLOW_SAFE_BOOT,
    UserManager.DISALLOW_MOUNT_PHYSICAL_MEDIA
).forEach { dpm.addUserRestriction(adminName, it) }
```

---

## 2. EMM/MDM連携（Android Enterprise）

### Android Management API（推奨アプローチ）

```json
{
  "applications": [{
    "packageName": "com.example.kioskapp",
    "installType": "KIOSK"
  }]
}
```

- 起動時自動ランチ＋フルスクリーン固定が一発で実現
- 旧来の `statusBarDisabled`・`lockTaskAllowed` は**非推奨**
- Google公式日本語サポートページでも「専用デバイスの画面にアプリをロックし、ユーザーがアプリを終了できないようにします」と明記

### 主要EMM/MDMソリューション

| ソリューション | 特徴 |
|--------------|------|
| VMware Workspace ONE | カスタムDPC連携、エンタープライズ向け |
| Microsoft Intune | AMAPI対応、Microsoft Japan日本語サポートあり |
| CLOMO（国内） | 日本国内実績豊富、日本語サポート充実 |
| Google Zero Touch | 工場出荷時プロビジョニング対応 |

---

## 3. Capacitor/Ionic+Vue3 への適用

### 利用可能プラグイン比較

| プラグイン | Capacitor対応 | 方式 | 更新状況 |
|-----------|-------------|------|---------|
| `capacitor-plugin-kiosk` | 3.x | Lock Task API | 2022年以降停止 ⚠️ |
| `capacitor-android-kiosk` | 6/7 対応 | SystemUI非表示＋FGサービス | 2026-06 アクティブ ✅ |

### capacitor-plugin-kiosk の使い方（Vue3）

```bash
npm install capacitor-plugin-kiosk
npx cap sync android
```

```typescript
// composables/useKiosk.ts
import { KioskMode } from 'capacitor-plugin-kiosk'

export function useKiosk() {
  const enter = async () => await KioskMode.enterKioskMode()
  const exit  = async () => await KioskMode.exitKioskMode()
  const check = async () => {
    const { isInKioskMode } = await KioskMode.isInKioskMode()
    return isInKioskMode
  }
  return { enter, exit, check }
}
```

```vue
<!-- コンポーネント例 -->
<script setup lang="ts">
import { useKiosk } from '@/composables/useKiosk'
const { enter, exit } = useKiosk()
</script>
```

### capacitor-android-kiosk（Cap-go製、現時点で推奨）

```bash
npm install capacitor-android-kiosk
npx cap sync android
```

- `WindowInsetsController.hide()` でシステムUI非表示
- `startForegroundService()` でキープアライブ
- `shouldBlockKey()` でハードウェアキーインターセプト
- **Device Owner不要で動作**するが、脱出防止の強度は弱い

---

## 4. 実装時の注意点・制限事項

### Device Ownerセットアップ

完全ロックダウン（ユーザーが脱出不能な状態）には **Device Ownerの事前設定が必須**。

```bash
# Device Owner設定（工場出荷・Googleアカウントなし状態で実行）
adb shell dpm set-device-owner com.example.dpc/.MyDeviceAdminReceiver
```

> ⚠️ 端末にGoogleアカウントが登録済みだと失敗する

### プロビジョニング方法の選択

| 方法 | 向いているケース |
|------|---------------|
| ADB | 開発・テスト用、少数台 |
| QRコード/NFCプロビジョニング | 中規模展開 |
| ゼロタッチ登録 | 大規模・工場出荷時 |
| NFC bump | 既存管理端末から転送 |

### 主な制限事項

- `DISALLOW_SAFE_BOOT` はソフトウェアからの安全モード起動を防止するが、**ハードウェアキー組み合わせによるリカバリーモードは防止できない**場合あり
- `capacitor-plugin-kiosk` は Capacitor 5/6/7 との互換性**未確認**
- Xnavis等OEM端末固有のAPIや追加エスケープベクターが存在する可能性あり（Samsung Knox等と同様）
- AMAPI の `installType: KIOSK` とカスタムDPC方式では、Capacitor WebView内の挙動に差異が生じる可能性あり

---

## 5. 日本語リソース・参考情報

| リソース | 内容 |
|---------|------|
| [Google公式（日本語）](https://support.google.com/work/android/answer/9560920?hl=ja) | 専用デバイス管理の概要 |
| [Android Developers - Lock Task Mode](https://developer.android.com/work/dpc/dedicated-devices/lock-task-mode) | Lock Task Mode 公式ドキュメント |
| [Android Developers - Dedicated Devices](https://developer.android.com/work/dpc/dedicated-devices) | 専用デバイス管理の全体像 |
| [AMAPI 専用デバイスポリシー](https://developers.google.com/android/management/policies/dedicated-devices) | Android Management API リファレンス |
| CLOMO公式サイト | 国内EMM、日本語技術ドキュメント充実 |
| Microsoft Intune（MS Japan） | 国内サポートあり、Android Enterprise対応 |

---

## 6. 未解決の検討事項

1. **Xnavis固有のDevice Owner設定手順** — OEMが提供するEMM連携APIの有無を確認推奨
2. **Device Owner不要の代替構成** — `capacitor-android-kiosk`（DPC不使用）で業務要件を満たせるか評価が必要
3. **AMAPI vs カスタムDPC** — Capacitor/Ionic WebView内のJavaScriptイベント挙動差異の検証
4. **国内MDMベンダー実績** — CLOMO・Intune日本向けでの実績事例確認

---

## 推奨構成（Xnavis業務ハンドヘルド）

**Android Enterprise + EMM（CLOMO or Intune）+ `capacitor-android-kiosk` プラグイン（Capacitor 6/7）**

Device Ownerはゼロタッチ登録またはQRコードプロビジョニングで事前設定。
