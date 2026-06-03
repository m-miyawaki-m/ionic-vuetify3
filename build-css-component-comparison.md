# ビルドツール・CSS・コンポーネント自作・APKビルド 比較ドキュメント

**対象環境：Vue 3 + Capacitor（Ionic または Vuetify 3）業務用 Android ハンドヘルドアプリ**

---

## 1. エグゼクティブサマリー

| 評価軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 優位 |
|--------|:-----------------:|:---------------------:|:----:|
| ビルドツール構成のシンプルさ | △ Ionic CLI が隠蔽 | ○ Vite 直接制御 | Vuetify |
| バンドルサイズ最適化 | ○ | ○ Tree-shaking 必須だが対応済み | 同等 |
| CSS テーマカスタマイズの深さ | ○ CSS 変数ベース | ◎ Sass 変数 + CSS 変数の二層構造 | Vuetify |
| コンポーネント自作のしやすさ | △ Web Components の制約あり | ◎ 通常の Vue SFC | Vuetify |
| 既存コンポーネントの拡張しやすさ | △ | ○ | Vuetify |
| APK ビルドの流れ | ○ Ionic CLI で統合 | ○ Vite + Capacitor CLI | 同等 |
| CI/CD パイプライン構築 | ○ | ○ | 同等 |

> **結論：ビルド制御・CSS カスタマイズ・コンポーネント自作の観点では Vuetify 3 が優位。**  
> APK ビルドは Capacitor を共有するため実質的な差はない。  
> Ionic CLI は便利だが、その分ビルド設定の透過性が低い。

---

## 2. ビルドツール比較

### 2-1. ビルドツールチェーン全体像

```
【Ionic + Capacitor】
ionic build（内部で Vite 実行）
  └─ @ionic/vite-plugin
       └─ Vite ビルド
            └─ cap sync → Android Studio / Gradle → APK

【Vuetify 3 + Capacitor】
vite build
  └─ vite-plugin-vuetify（Tree-shaking）
       └─ Vite ビルド
            └─ cap sync → Android Studio / Gradle → APK
```

### 2-2. ビルド設定比較

| 評価軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 備考 |
|--------|------------------|----------------------|------|
| ビルドコマンド | `ionic build` + `ionic cap sync` | `vite build` + `cap sync` | Ionic CLI が Capacitor をラップ |
| Vite 設定の透過性 | △ Ionic CLI が一部を隠蔽 | ◎ vite.config.ts を直接制御 | Vuetify は設定が見通しやすい |
| Tree-shaking（未使用コード除去） | ○ 自動 | ○ vite-plugin-vuetify で対応（要設定） | どちらも要確認 |
| コード分割（Route-based splitting） | ○ | ○ vue-router の lazy import | 同等 |
| 環境変数（.env） | ○ `VITE_*` 形式 | ○ `VITE_*` 形式 | 同等 |
| HMR（Hot Module Replacement） | ○ | ○ | 同等 |
| ビルド成果物サイズ（目安） | 中（3〜6MB） | 中〜大（要 Tree-shaking）| vite-plugin-vuetify 必須 |
| ビルド時間 | 速い | 速い | 同等 |

### 2-3. vite.config.ts 設定例

**Ionic + Vue**
```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import path from 'path'

export default defineConfig({
  plugins: [vue()],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
})
```

**Vuetify 3（Tree-shaking 設定が追加で必要）**
```ts
// vite.config.ts
import { defineConfig } from 'vite'
import vue from '@vitejs/plugin-vue'
import vuetify, { transformAssetUrls } from 'vite-plugin-vuetify'
import path from 'path'

export default defineConfig({
  plugins: [
    vue({ template: { transformAssetUrls } }),
    vuetify({ autoImport: true }),  // ← Tree-shaking に必須
  ],
  resolve: { alias: { '@': path.resolve(__dirname, './src') } },
})
```

### 2-4. バンドルサイズ最適化

| 対策 | Ionic | Vuetify 3 | 備考 |
|------|:-----:|:---------:|------|
| コンポーネントの自動 Tree-shaking | ○ デフォルト | ○ vite-plugin-vuetify 必須 | Vuetify は設定必須 |
| アイコンの個別 import | ○ Ionicons で自動 | ○ MDI を個別 import 可 | どちらも対応可 |
| Route-based コード分割 | ○ | ○ | 同等 |
| 本番ビルドのサイズ確認（rollup-plugin-visualizer） | ○ | ○ | 同等 |

---

## 3. CSS・スタイリング比較

### 3-1. スタイリング方式の全体像

| 評価軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 優位 |
|--------|------------------|----------------------|:----:|
| テーマ変数の構造 | CSS カスタムプロパティ（`--ion-*`） | Sass 変数 + CSS カスタムプロパティ（`--v-*`） | Vuetify |
| カスタマイズの深さ | ○ 主要色・フォント | ◎ コンポーネント単位まで変数で制御可 | Vuetify |
| Sass / SCSS サポート | ○ | ◎ Vuetify 内部が Sass ベース | Vuetify |
| CSS スコープ（SFC `<style scoped>`） | ○ | ○ | 同等 |
| グローバル CSS の管理 | ○ | ○ | 同等 |
| Tailwind CSS との共存 | △ 一部スタイル競合の可能性 | △ 一部スタイル競合の可能性 | 同等（要調整） |
| ダークモード対応 | ○ CSS 変数切り替え | ◎ Vuetify テーマ機構で一元管理 | Vuetify |
| RTL（右→左）対応 | ○ | ○ | 同等 |
| アニメーション・トランジション | ○ Ionic 組み込み | △ Vue `<Transition>` + 手動実装 | Ionic |

### 3-2. テーマカスタマイズの比較

**Ionic：CSS カスタムプロパティで上書き**
```css
/* variables.css */
:root {
  --ion-color-primary: #3880ff;
  --ion-color-primary-shade: #3171e0;
  --ion-color-primary-tint: #4c8dff;
  --ion-font-family: 'Noto Sans JP', sans-serif;
}
```

**Vuetify 3：テーマオブジェクト + Sass 変数の二層構造**
```ts
// main.ts
import { createVuetify } from 'vuetify'

const vuetify = createVuetify({
  theme: {
    themes: {
      light: {
        colors: {
          primary: '#3880ff',
          secondary: '#5c6bc0',
          error: '#B00020',
        },
      },
      dark: {
        colors: { primary: '#82b1ff' },
      },
    },
  },
})
```

```scss
// settings.scss（コンポーネント単位の細かい調整）
@use 'vuetify/settings' with (
  $button-height: 44px,
  $text-field-border-radius: 8px,
);
```

### 3-3. デザイントークンとの連携

| 連携方法 | Ionic | Vuetify 3 | 備考 |
|---------|:-----:|:---------:|------|
| CSS カスタムプロパティへのマッピング | ○ `--ion-*` に直接代入 | ○ `--v-*` または Vuetify theme 経由 | どちらも対応可 |
| Storybook でのテーマ反映 | △ 設定が複雑 | ○ `createVuetify()` を decorator に渡す | Vuetify が容易 |
| デザイントークン → Sass 変数 | ✗ | ○ `settings.scss` で直接上書き可 | Vuetify が有利 |

---

## 4. コンポーネント自作のしやすさ比較

### 4-1. コンポーネント実装方式の違い

| 評価軸 | Ionic + Capacitor | Vuetify 3 + Capacitor | 優位 |
|--------|------------------|----------------------|:----:|
| コンポーネントの実装形式 | Web Components（Custom Elements） | 通常の Vue SFC（.vue ファイル） | Vuetify |
| カスタムコンポーネントの作成 | ○ Vue SFC で作れる | ◎ Vue SFC で作れる（標準的） | Vuetify |
| 既存コンポーネントのラップ・拡張 | △ Web Components は extends が難しい | ○ Vue の extends / composition で容易 | Vuetify |
| Slot（コンテンツ差し込み）の柔軟性 | ○ | ◎ Vuetify は slot が豊富 | Vuetify |
| Props / Emits の型安全性 | ○ | ◎ defineProps / defineEmits + TS | Vuetify |
| Composables（ロジック分離） | ○ | ◎ Vue 3 Composition API との親和性 | Vuetify |
| Storybook でのコンポーネント開発 | △ Web Components の扱いに注意 | ◎ 標準的な Vue コンポーネントとして開発可 | Vuetify |
| テスト（Vitest での単体テスト） | △ jsdom で Web Components が動かない場合あり | ◎ Vue Test Utils で標準的にテスト可 | Vuetify |

### 4-2. カスタムコンポーネント実装例

**Ionic での業務用入力コンポーネント自作**
```vue
<!-- Ionic は Vue SFC で作れるが、ion-* との組み合わせに注意 -->
<template>
  <div class="custom-input">
    <ion-label>{{ label }}</ion-label>
    <ion-input :value="modelValue" @ion-input="$emit('update:modelValue', $event.detail.value)" />
    <span v-if="error" class="error-msg">{{ error }}</span>
  </div>
</template>

<script setup lang="ts">
defineProps<{ label: string; modelValue: string; error?: string }>()
defineEmits<{ 'update:modelValue': [value: string] }>()
</script>
```

**Vuetify 3 での業務用入力コンポーネント自作**
```vue
<!-- 通常の Vue SFC。v-text-field を拡張するだけ -->
<template>
  <v-text-field
    v-bind="$attrs"
    :label="label"
    :model-value="modelValue"
    :rules="rules"
    variant="outlined"
    density="compact"
    @update:model-value="$emit('update:modelValue', $event)"
  >
    <template v-for="(_, name) in $slots" #[name]="slotData">
      <slot :name="name" v-bind="slotData ?? {}" />
    </template>
  </v-text-field>
</template>

<script setup lang="ts">
defineProps<{ label: string; modelValue: string; rules?: unknown[] }>()
defineEmits<{ 'update:modelValue': [value: string] }>()
</script>
```

### 4-3. Vuetify コンポーネント拡張パターン

| パターン | 方法 | 難易度 |
|---------|------|:------:|
| Props を追加してラップ | `v-bind="$attrs"` でスルー + 追加 Props | 易 |
| Slot を透過させる | `v-for` + `#[name]` でスロット転送 | 中 |
| グローバルデフォルト変更 | `createVuetify({ defaults: { VTextField: { variant: 'outlined' } } })` | 易 |
| テーマ・色をコンポーネント単位で設定 | `createVuetify({ defaults: { VBtn: { color: 'primary' } } })` | 易 |
| 完全カスタムコンポーネント | 通常の Vue SFC として実装 | 標準的 |

---

## 5. APK ビルド・デプロイ比較

### 5-1. ビルドフロー全体

```
【Ionic + Capacitor】
1. ionic build          ← Web アセットのビルド（dist/）
2. ionic cap sync       ← dist/ を Android プロジェクトにコピー + プラグイン同期
3. ionic cap open android  ← Android Studio を開く
4. Android Studio でビルド → APK / AAB

【Vuetify 3 + Capacitor】
1. vite build           ← Web アセットのビルド（dist/）
2. cap sync             ← dist/ を Android プロジェクトにコピー + プラグイン同期
3. cap open android     ← Android Studio を開く
4. Android Studio でビルド → APK / AAB
```

> Capacitor を共有するため、ステップ 2 以降は**完全に同一**。

### 5-2. コマンド比較

| 操作 | Ionic + Capacitor | Vuetify 3 + Capacitor |
|------|------------------|----------------------|
| Web ビルド | `ionic build` | `vite build` |
| Capacitor 同期 | `ionic cap sync` または `cap sync` | `cap sync` |
| Android Studio 起動 | `ionic cap open android` または `cap open android` | `cap open android` |
| ライブリロード（開発時） | `ionic cap run android -l --external` | `vite --host` + `cap run android` |
| APK 直接ビルド（CLI） | `cap build android` | `cap build android` |

### 5-3. Android プロジェクト構成

| 項目 | Ionic + Capacitor | Vuetify 3 + Capacitor | 備考 |
|------|------------------|----------------------|------|
| android/ ディレクトリ構成 | 同一 | 同一 | Capacitor が生成するため差なし |
| capacitor.config.ts の設定 | 同一 | 同一 | `webDir: 'dist'` を指定 |
| Gradle バージョン管理 | Capacitor が管理 | Capacitor が管理 | 同等 |
| AndroidManifest.xml 変更 | 手動または @capacitor/* プラグイン | 手動または @capacitor/* プラグイン | 同等 |
| APK 署名（release ビルド） | Android Studio / Gradle | Android Studio / Gradle | 同等 |

### 5-4. CI/CD パイプライン構成例（GitHub Actions）

```yaml
# .github/workflows/android-build.yml
name: Android APK Build

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'

      - name: Install dependencies
        run: npm ci

      # Ionic の場合: npm run build（package.json に ionic build を設定）
      # Vuetify の場合: そのまま vite build
      - name: Build web assets
        run: npm run build

      - name: Sync Capacitor
        run: npx cap sync android

      - uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Build APK
        run: |
          cd android
          ./gradlew assembleDebug

      - uses: actions/upload-artifact@v4
        with:
          name: app-debug.apk
          path: android/app/build/outputs/apk/debug/app-debug.apk
```

> **Ionic と Vuetify で CI/CD の差分は `npm run build` コマンド名のみ。**  
> Capacitor 以降のステップは完全に共通。

### 5-5. ライブリロード（実機での開発）

| 操作 | Ionic + Capacitor | Vuetify 3 + Capacitor |
|------|------------------|----------------------|
| 開発サーバー起動 | `ionic serve` | `vite --host` |
| 実機ライブリロード | `ionic cap run android -l --external` | `vite --host` → `cap run android` |
| capacitor.config.ts に開発サーバーURL設定 | Ionic CLI が自動設定 | 手動設定が必要 |

**Vuetify での手動設定例**
```ts
// capacitor.config.ts（開発時のみ）
const config: CapacitorConfig = {
  appId: 'com.example.app',
  appName: 'MyApp',
  webDir: 'dist',
  server: {
    url: 'http://192.168.x.x:5173',  // 開発PCのIPアドレス
    cleartext: true,
  },
}
```

---

## 6. まとめ：観点別推奨

| 観点 | 推奨 | 理由 |
|------|------|------|
| ビルド設定の透過性・制御しやすさ | **Vuetify 3** | vite.config.ts を直接制御。Ionic CLI のラップがない分わかりやすい |
| CSS テーマカスタマイズの深さ | **Vuetify 3** | Sass 変数 + CSS 変数の二層構造。デザイントークン連携に有利 |
| コンポーネント自作・拡張のしやすさ | **Vuetify 3** | 通常の Vue SFC のため、テスト・Storybook・型安全性がすべてシームレス |
| APK ビルド・CI/CD | **同等** | Capacitor を共有するため実質的な差なし |
| ライブリロード（開発時の設定容易さ） | **Ionic** | Ionic CLI が開発サーバー URL を自動設定してくれる |

---

*比較対象バージョン：Ionic 7/8、Vuetify 3.x、Capacitor 6.x、Vite 5.x*
