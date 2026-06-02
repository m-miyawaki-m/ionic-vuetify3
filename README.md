# Ionic vs Vuetify 3 + Capacitor 比較ドキュメント

**対象環境：業務用 Android ハンドヘルドアプリ（フォーム・一覧・データ入力中心）**

---

## 1. エグゼクティブサマリー

| 評価軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 優位 |
|--------|:-----------------:|:---------------------:|:----:|
| Android Material Design 準拠 | △ | ○ | Vuetify |
| 業務コンポーネント充実度 | △ | ◎ | Vuetify |
| ページ遷移・ナビゲーション | ◎ | △ | Ionic |
| セーフエリア・キーボード自動対応 | ◎ | △ | Ionic |
| 低スペック端末パフォーマンス | ○ | △ | Ionic |
| Storybook / デザイントークン連携 | △ | ○ | Vuetify |
| 長期保守性 | ○ | ○ | 同等 |
| iOS 対応（将来拡張） | ○ | △ | Ionic |

> **結論：Android 単独の業務アプリであれば Vuetify 3 + Capacitor への移行を推奨。**  
> 業務コンポーネントの充実度・Material Design 親和性・Storybook 連携で優位。  
> ネイティブ遷移の自前実装と低スペック端末のチューニングが主な移行コスト。

---

## 2. 判断軸比較表

### 2-1. 機能・品質

| 判断軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 補足 |
|--------|------------------|----------------------|------|
| Android Material Design | △ MD2 ベースの自動適応 | ○ MD3 対応済み | Android ネイティブ感は Vuetify が上 |
| iOS / Android スタイル自動切替 | ○ 自動 | ✗ Material 一択 | Android 単独なら問題なし |
| 業務テーブル（ソート・フィルタ・ページング） | △ 標準なし、自前実装要 | ◎ v-data-table 標準搭載 | 業務アプリで最大の差異 |
| フォーム部品の充実度 | ○ 基本は揃う | ◎ バリデーション統合・豊富 | Vuetify が有利 |
| ページ遷移アニメーション | ○ 組み込み（slide/fade） | △ vue-router + CSS で自前実装 | 移行コスト 中 |
| スタックナビゲーション / 戻る操作 | ○ 組み込み | △ vue-router + 自前実装 | 移行コスト 中 |
| Pull-to-Refresh | ○ IonRefresher | △ 自前実装 | 移行コスト 小〜中 |
| 仮想スクロール（大量データ） | △ 廃止予定だった | ○ v-virtual-scroll 安定 | Vuetify が有利 |
| 無限スクロール | ○ IonInfiniteScroll | ○ v-infinite-scroll（v3.3+） | 同等 |
| ドラッグ＆ドロップ並べ替え | ○ IonReorderGroup | △ vuedraggable 等で代替 | 移行コスト 小 |

### 2-2. モバイル対応

| 判断軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 補足 |
|--------|------------------|----------------------|------|
| セーフエリア（ノッチ・インセット）自動対応 | ○ 自動 | △ CSS `env()` + 手動設定 | 対応工数 小（0.5日） |
| キーボード押し上げ（スクロール連動） | ○ ion-content が自動対応 | △ @capacitor/keyboard + 手動 | 対応工数 小〜中（1〜2日） |
| ハードウェアバックボタン（Android） | ○ ionBackButton イベント | △ @capacitor/app で代替 | 対応工数 小（1日） |
| ステータスバー制御 | ○ @capacitor/status-bar 連携容易 | ○ @capacitor/status-bar 連携容易 | 同等 |
| ハプティクス（振動フィードバック） | ○ @capacitor/haptics 連携容易 | ○ @capacitor/haptics 連携容易 | 同等 |
| カメラ・GPS 等ネイティブ API | ○ Capacitor プラグイン | ○ Capacitor プラグイン | 同等（Capacitor 共通） |

### 2-3. 開発・保守

| 判断軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 補足 |
|--------|------------------|----------------------|------|
| 初期セットアップ難易度 | ○ CLI で即時 | ○ Vite + 手動設定 | Vuetify はやや手順多め |
| TypeScript サポート | ○ | ○ | 同等 |
| Storybook 統合 | △ 一部制限・設定が複雑 | ○ 標準的な Vue コンポーネント構成 | Vuetify が有利 |
| デザイントークン連携 | △ CSS 変数で部分対応 | ○ Vuetify テーマ変数と直結可能 | Vuetify が有利 |
| バンドルサイズ | 中（〜5MB 程度） | 中〜大（Tree-shaking 必須） | vite-plugin-vuetify で最適化 |
| 低スペック端末での描画パフォーマンス | ○ モバイル向け最適化済み | △ 重い画面は要チューニング | v-virtual-scroll 等で対応 |
| 学習コスト（Ionic 経験者） | ○ 既存知識活用 | △ 新規学習（Vue 3 知識は共通） | Vuetify ドキュメントは豊富 |
| コミュニティ規模 | ○ 安定 | ○ 大規模（Web 向け） | 同等 |
| 長期サポート見通し | ○ Ionic 8 安定 | ○ Vuetify 3 安定 | 同等 |

### 2-4. 移行コスト概算

| 作業項目 | 工数目安 | 備考 |
|---------|---------|------|
| プロジェクト初期構成（Vite + Vuetify 3 + Capacitor） | 1〜2日 | |
| コンポーネント置き換え（1対1対応の画面） | 3〜5日 | 下記コンポーネント対応表参照 |
| ページ遷移・ナビゲーション自前実装 | 3〜7日 | vue-router + カスタムトランジション |
| セーフエリア・キーボード・バックボタン対応 | 2〜3日 | |
| デザイントークン → Vuetify テーマ変数マッピング | 3〜5日 | Storybook 統合含む |
| 既存画面移行・動作確認 | 規模次第 | 1画面あたり 0.5〜1日 |
| **合計（新規構成ベース、既存画面除く）** | **約 2〜4 週間** | |

---

## 3. コンポーネント対応表

**凡例：** ○ 同等の代替あり　△ 組み合わせ・設定で対応　✗ 標準なし（自前実装要）

### ナビゲーション

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonTabs / IonTabBar | v-tabs / v-bottom-navigation | ○ | API は異なるが同等機能 |
| IonMenu | v-navigation-drawer | ○ | 同等 |
| IonSplitPane | v-navigation-drawer + レイアウト | △ | 自前でペイン構成が必要 |
| IonHeader / IonToolbar | v-app-bar | ○ | 同等 |
| IonFooter | v-footer | ○ | 同等 |
| IonBackButton | v-btn + `$router.back()` | △ | 遷移アニメーションは別途実装 |
| IonBreadcrumb | v-breadcrumbs | ○ | 同等 |

### レイアウト

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonPage | v-main（またはレイアウト div） | ○ | 同等 |
| IonContent | v-container / div（スクロール領域） | △ | 慣性スクロールは CSS `overflow-y: scroll` で対応 |
| IonGrid / IonRow / IonCol | v-container / v-row / v-col | ○ | ほぼ同等 |

### フォーム

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonInput | v-text-field | ○ | Vuetify の方がバリデーション統合が充実 |
| IonTextarea | v-textarea | ○ | 同等 |
| IonSelect | v-select | ○ | 同等 |
| IonDatetime | v-date-picker / v-time-picker | ○ | Vuetify の方が豊富 |
| IonCheckbox | v-checkbox | ○ | 同等 |
| IonRadio / IonRadioGroup | v-radio / v-radio-group | ○ | 同等 |
| IonToggle | v-switch | ○ | 同等 |
| IonRange | v-slider | ○ | 同等 |
| IonSearchbar | v-text-field（prepend-inner-icon） | △ | 組み合わせで対応 |
| IonSegment / IonSegmentButton | v-btn-group / v-tabs | △ | 組み合わせで対応 |

### データ表示

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonList / IonItem | v-list / v-list-item | ○ | 同等 |
| IonCard | v-card | ○ | 同等 |
| IonBadge | v-badge | ○ | 同等 |
| IonChip | v-chip | ○ | 同等 |
| IonAvatar | v-avatar | ○ | 同等 |
| IonThumbnail | v-img | ○ | 同等 |
| IonLabel | v-label | ○ | 同等 |
| IonNote | Typography クラス（text-caption 等） | △ | CSS で対応 |
| IonIcon | MDI アイコン（Vuetify 標準） | ○ | アイコン名の変換が必要 |
| —（なし） | **v-data-table** | ◎ | **Ionic にない。業務アプリでの最大メリット** |
| —（なし） | **v-data-table-virtual** | ◎ | 大量データ向け仮想スクロール対応テーブル |

### フィードバック・オーバーレイ

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonModal | v-dialog | ○ | 同等 |
| IonAlert | v-dialog（カスタム） | △ | 簡単な確認ダイアログは自前 |
| IonToast | v-snackbar | ○ | 同等 |
| IonActionSheet | v-bottom-sheet | ○ | 同等 |
| IonLoading | v-overlay + v-progress-circular | △ | 組み合わせで対応 |
| IonSpinner | v-progress-circular | ○ | 同等 |
| IonProgressBar | v-progress-linear | ○ | 同等 |
| IonPopover | v-menu | ○ | 同等 |

### スクロール・インタラクション

| Ionic | Vuetify 3 | 対応 | 備考 |
|-------|-----------|:----:|------|
| IonVirtualScroll（廃止済み） | v-virtual-scroll | ○ | Vuetify の方が安定 |
| IonInfiniteScroll | v-infinite-scroll | ○ | Vuetify 3.3+ 対応 |
| IonRefresher（Pull-to-Refresh） | **なし** | ✗ | @vueuse/core + カスタム実装 |
| IonReorderGroup（ドラッグ並べ替え） | **なし** | ✗ | vuedraggable 等ライブラリで代替 |
| IonSlides（廃止済み） | v-carousel / Swiper.js | △ | Swiper.js 直接利用を推奨 |

---

## 4. 要自前実装リスト

Ionic が標準提供していた機能のうち、Vuetify 3 では自前実装が必要なもの。

| 機能 | Ionic での対応 | Vuetify での対応方法 | 工数目安 |
|------|--------------|-------------------|:-------:|
| ページ遷移アニメーション（スライド・フェード） | 組み込み | vue-router + CSS `<transition>` | 中（3〜5日） |
| ハードウェアバックボタン（Android） | `ionBackButton` イベント | `@capacitor/app` の `backButton` で代替 | 小（1日） |
| Pull-to-Refresh | IonRefresher | `@vueuse/core` + カスタムコンポーネント | 小〜中（2〜3日） |
| ドラッグ＆ドロップ並べ替え | IonReorderGroup | `vuedraggable` | 小（1日） |
| セーフエリア自動対応 | 自動 | CSS `env(safe-area-inset-*)` + `@capacitor/status-bar` | 小（0.5日） |
| キーボード押し上げ（スクロール連動） | ion-content 自動対応 | `@capacitor/keyboard` + スクロール手動制御 | 小〜中（1〜2日） |
| プラットフォーム検出 | `isPlatform()` | `Capacitor.getPlatform()` で代替 | 小（既存 API あり） |
| iOS らしいスタイル適応 | 自動（iOS モード） | 不要（Android 単独のため対応不要） | — |

---

## 5. 参考：推奨スタック構成

```
Vue 3 + Vite
├── vuetify@^3.x          # UI コンポーネント
├── @mdi/font             # アイコン
├── vue-router@^4.x       # ルーティング
├── pinia                 # 状態管理
├── @capacitor/core       # ネイティブブリッジ
├── @capacitor/android    # Android ビルド
├── @capacitor/app        # バックボタン等
├── @capacitor/keyboard   # キーボード制御
├── @capacitor/status-bar # ステータスバー
├── @vueuse/core          # Pull-to-Refresh 等のユーティリティ
└── vite-plugin-vuetify   # Tree-shaking（バンドルサイズ最適化 必須）
```

---

*比較対象バージョン：Ionic 7/8、Vuetify 3.x（MD3 対応版）、Capacitor 6.x*
