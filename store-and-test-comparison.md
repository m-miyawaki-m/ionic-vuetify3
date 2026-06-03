# Store・テストフレームワーク比較ドキュメント

**対象環境：Vue 3 + Capacitor（Ionic または Vuetify 3）業務用 Android ハンドヘルドアプリ**

---

## 1. エグゼクティブサマリー

| カテゴリ | 推奨 | 理由 |
|---------|------|------|
| Store | **Pinia** | Vue 3 公式推奨。TypeScript 親和性・DevTools 統合・軽量さで Vuex 4 を上回る |
| ユニット / コンポーネントテスト | **Vitest** | Vite プロジェクトとの統合が最もシームレス。Jest 互換で移行コスト低 |
| E2E テスト | **Playwright**（推奨）または Cypress | Playwright は CI 安定性・マルチブラウザ・モバイル Viewport 対応で優位 |

> **推奨構成：Pinia + Vitest + Playwright**  
> Vuetify 3 + Capacitor プロジェクトでの標準構成として採用を推奨。  
> Ionic から移行する場合も Store・テスト構成は独立しており、移行コストは低い。

---

## 2. Store 比較：Pinia vs Vuex 4

### 2-1. 機能・設計比較

| 評価軸 | Pinia | Vuex 4 | 優位 |
|--------|:-----:|:------:|:----:|
| Vue 3 / Composition API ネイティブ対応 | ○ | △ Options API が主体 | Pinia |
| TypeScript サポート | ◎ 型推論が強力 | △ 明示的な型定義が必要 | Pinia |
| ボイラープレートの少なさ | ◎ mutations 不要 | △ state/mutations/actions/getters が必要 | Pinia |
| モジュール分割 | ○ Store ファイルを分けるだけ | △ modules 定義が必要 | Pinia |
| Vue DevTools 統合 | ○ | ○ | 同等 |
| SSR 対応 | ○ | ○ | 同等 |
| バンドルサイズ | 小（〜1KB gzip） | 中（〜6KB gzip） | Pinia |
| 公式サポート状況 | ◎ Vue 3 公式推奨 | △ メンテナンスモード | Pinia |
| Ionic での採用実績 | ○ | ○ | 同等 |
| Vuetify 3 との親和性 | ◎ | ○ | Pinia |

### 2-2. API 記述比較

**Pinia（Composition API スタイル）**
```ts
// stores/user.ts
import { defineStore } from 'pinia'
import { ref, computed } from 'vue'

export const useUserStore = defineStore('user', () => {
  const name = ref('')
  const isLoggedIn = computed(() => name.value !== '')

  async function login(id: string) {
    name.value = id
  }

  return { name, isLoggedIn, login }
})
```

**Vuex 4（Options スタイル）**
```ts
// store/user.ts
import { createStore } from 'vuex'

export default createStore({
  state: { name: '' },
  getters: {
    isLoggedIn: (state) => state.name !== '',
  },
  mutations: {
    SET_NAME(state, name: string) { state.name = name },
  },
  actions: {
    async login({ commit }, id: string) { commit('SET_NAME', id) },
  },
})
```

### 2-3. Capacitor ネイティブストレージとの連携

| 連携パターン | Pinia | Vuex 4 | 備考 |
|------------|:-----:|:------:|------|
| `@capacitor/preferences`（キー・バリューストア） | ○ pinia-plugin-persistedstate で容易 | △ カスタムプラグイン実装要 | Pinia が有利 |
| 永続化プラグイン（localforage / pinia-plugin-persistedstate） | ○ | △ | Pinia が有利 |
| Capacitor SQLite との組み合わせ | △ 自前実装 | △ 自前実装 | 同等 |

**Pinia 永続化の実装例**
```ts
// main.ts
import { createPinia } from 'pinia'
import piniaPluginPersistedstate from 'pinia-plugin-persistedstate'

const pinia = createPinia()
pinia.use(piniaPluginPersistedstate)
```

```ts
// stores/user.ts
export const useUserStore = defineStore('user', () => { ... }, {
  persist: {
    storage: {
      getItem: (key) => Preferences.get({ key }).then(r => r.value),
      setItem: (key, value) => Preferences.set({ key, value }),
    },
  },
})
```

### 2-4. Ionic vs Vuetify でのセットアップ差分

| 項目 | Ionic + Pinia | Vuetify 3 + Pinia |
|------|--------------|------------------|
| セットアップ | 同一（`createPinia()`） | 同一（`createPinia()`） |
| DevTools 統合 | ○ | ○ |
| Storybook との組み合わせ | △ 要設定 | ○ 標準的な Vue 構成で問題なし |
| 差分 | **なし** | **なし** |

> Store の選択は Ionic/Vuetify どちらを使っても影響なし。Pinia の移行コストはゼロ。

---

## 3. テストフレームワーク比較

### 3-1. テストレイヤーの整理

```
┌─────────────────────────────────────────────────────────┐
│  E2E テスト（Playwright / Cypress）                      │
│  ブラウザ or WebView 上での実際のユーザー操作を検証       │
├─────────────────────────────────────────────────────────┤
│  コンポーネントテスト（Vitest + Vue Test Utils）          │
│  単一コンポーネントの描画・イベント・Props を検証         │
├─────────────────────────────────────────────────────────┤
│  ユニットテスト（Vitest）                                │
│  Store・utils・ビジネスロジックを単体で検証              │
└─────────────────────────────────────────────────────────┘
```

### 3-2. Vitest（ユニット / コンポーネントテスト）

| 評価軸 | Vitest | Jest | 備考 |
|--------|:------:|:----:|------|
| Vite プロジェクトとの統合 | ◎ 設定共通 | △ 追加設定が必要 | Vitest が圧倒的に有利 |
| Jest 互換 API | ○ | ◎ | 既存 Jest テストはほぼそのまま移行可能 |
| 実行速度 | ◎ HMR ベース | △ | Vitest が高速 |
| TypeScript サポート | ◎ | ○ | 同等以上 |
| Vuetify 3 コンポーネントテスト | ○ | ○ | どちらも設定は必要 |
| Ionic コンポーネントテスト | △ Web Components のため設定複雑 | △ 同様 | 両方やや複雑 |
| Coverage | ○ v8 / istanbul | ○ | 同等 |

**Vitest + Vuetify 3 セットアップ例**
```ts
// vitest.setup.ts
import { config } from '@vue/test-utils'
import { createVuetify } from 'vuetify'
import * as components from 'vuetify/components'
import * as directives from 'vuetify/directives'

const vuetify = createVuetify({ components, directives })

config.global.plugins = [vuetify]
```

**Vitest + Ionic セットアップ例**
```ts
// vitest.setup.ts
import { config } from '@vue/test-utils'
import { IonicVue } from '@ionic/vue'

config.global.plugins = [IonicVue]
// IonApp ラッパーが必要なコンポーネントは個別対応が必要
```

> **Ionic では Web Components（`<ion-*>`）のレンダリングがテスト環境で制限される場合がある。**  
> Vuetify は通常の Vue コンポーネントのためテスト環境での扱いが簡単。

### 3-3. E2E テスト比較：Playwright vs Cypress

| 評価軸 | Playwright | Cypress | 優位 |
|--------|:----------:|:-------:|:----:|
| マルチブラウザ対応（Chromium / Firefox / Safari） | ○ | △ Firefox のみ追加対応（Safari は限定的） | Playwright |
| モバイル Viewport エミュレーション | ○ デバイス設定が豊富 | △ 手動設定 | Playwright |
| CI 安定性（フレーキーテスト耐性） | ◎ 自動待機が優秀 | ○ | Playwright |
| 実行速度（並列実行） | ◎ | ○ | Playwright |
| セットアップ難易度 | ○ | ◎ 初学者に易しい | Cypress |
| デバッグ・UIモード | ○ Playwright UI Mode | ◎ Time Travel Debugging | Cypress |
| コンポーネントテスト機能 | △ 実験的 | ○ 安定 | Cypress |
| 日本語ドキュメント・事例 | △ 少なめ | ○ 多い | Cypress |
| Capacitor WebView テスト | △ WebView ではなく browser で代替 | △ 同様 | 同等 |
| 無料プラン（ダッシュボード） | ○ | △ 有料プランで機能制限 | Playwright |

### 3-4. Playwright vs Cypress：ユースケース別推奨

| ユースケース | 推奨 | 理由 |
|------------|------|------|
| CI/CD パイプラインでの自動テスト | Playwright | 並列実行・安定性・コスト面で優位 |
| ローカルでのデバッグ・開発体験 | Cypress | Time Travel Debugging が直感的 |
| モバイル Viewport の再現テスト | Playwright | デバイスプリセットが充実 |
| チームの習得コストを抑えたい | Cypress | 学習曲線がなだらか |
| コンポーネントテストも一元化したい | Cypress | Component Testing 機能が安定 |

### 3-5. Ionic vs Vuetify でのテスト設定差分

| 項目 | Ionic + Vue | Vuetify 3 | 差分 |
|------|------------|-----------|------|
| Vitest セットアップ | IonicVue plugin + IonApp ラッパー要 | createVuetify() を global plugin に追加 | どちらも設定は必要 |
| Web Components の扱い | ✗ `<ion-*>` はブラウザ依存。jsdom では動作しない場合あり | ○ 通常の Vue コンポーネント | **Vuetify が有利** |
| コンポーネント単体テストの容易さ | △ | ◎ | Vuetify が有利 |
| E2E での Capacitor native API モック | △ `@capacitor/core` のモックが必要 | △ 同様 | 同等（Capacitor 共通） |
| Playwright / Cypress の設定 | 同一 | 同一 | 差分なし |

---

## 4. 推奨構成まとめ

```
Vue 3 + Vite + Vuetify 3 + Capacitor
├── pinia                           # State Management
│   └── pinia-plugin-persistedstate # @capacitor/preferences と連携
│
├── vitest                          # ユニット / コンポーネントテスト
│   ├── @vue/test-utils
│   ├── jsdom
│   └── @vitest/coverage-v8
│
└── playwright                      # E2E テスト（推奨）
    または
    cypress                         # E2E テスト（デバッグ重視の場合）
```

### テスト方針

| テストレベル | ツール | 対象 |
|------------|-------|------|
| ユニット | Vitest | Store（Pinia）・utils・ビジネスロジック |
| コンポーネント | Vitest + Vue Test Utils | 個別コンポーネントの描画・イベント |
| E2E | Playwright | 画面遷移・フォーム送信・一覧操作の全体フロー |

---

## 5. Ionic からの移行時の注意点

| 項目 | 内容 |
|------|------|
| Store（Pinia） | Ionic・Vuetify どちらでも同じ。移行コスト **ゼロ** |
| Vitest | Ionic の `IonicVue` セットアップを `createVuetify()` に差し替えるだけ |
| Web Components テスト | Ionic の `<ion-*>` コンポーネントテストは jsdom で動かないケースがある。Vuetify 移行でこの問題が解消される |
| E2E | Playwright / Cypress の設定は Ionic・Vuetify 問わず同一。既存テストはそのまま使える |

---

*比較対象バージョン：Pinia 2.x、Vuex 4.x、Vitest 2.x、Playwright 1.x、Cypress 13.x*
