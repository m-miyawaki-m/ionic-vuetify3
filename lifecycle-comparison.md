# ライフサイクル制御 比較ドキュメント

**対象環境：Vue 3 + Capacitor（Ionic または Vuetify 3）業務用 Android ハンドヘルドアプリ**

---

## 1. エグゼクティブサマリー

| ライフサイクル種別 | Ionic + Capacitor | Vuetify 3 + Capacitor | 差分の大きさ |
|------------------|:-----------------:|:---------------------:|:-----------:|
| Vue 標準フック（mount/unmount/update） | ○ 共通 | ○ 共通 | **なし** |
| ナビゲーション遷移時のフック | ◎ ionView* 専用フックあり | △ vue-router ガード + onActivated で代替 | **大きい** |
| ページのDOM保持挙動 | ◎ IonRouterOutlet が自動管理 | △ KeepAlive を手動設定 | **大きい** |
| アプリ前面/背面切り替え | ○ @capacitor/app | ○ @capacitor/app | **なし** |
| ハードウェアバックボタン | ○ ionBackButton イベント | △ @capacitor/app で代替実装 | **中程度** |
| ディープリンク | ○ @capacitor/app + Ionic ルーター連携 | △ @capacitor/app + vue-router 手動連携 | **中程度** |

> **最大の差はナビゲーションライフサイクル。**
> Ionic の `ionViewWillEnter` は「画面に戻ってきたとき」にも確実に発火するが、
> Vuetify（vue-router）ではデフォルトでコンポーネントが再マウントされないため、
> `<KeepAlive>` + `onActivated` または `watch(route)` で明示的に対応が必要。

---

## 2. Vue 標準ライフサイクルフック（共通）

Ionic・Vuetify どちらも Vue 3 の標準フックは同じ動作をする。

```
コンポーネント作成
   │
   ├─ onBeforeMount   ← DOM 生成前。初期データ取得の開始タイミング
   ├─ onMounted       ← DOM 生成後。DOM 操作・外部ライブラリ初期化
   │
   │  ［データ変更ループ］
   ├─ onBeforeUpdate  ← 再描画前
   ├─ onUpdated       ← 再描画後
   │
   ├─ onBeforeUnmount ← 破棄前。タイマー・サブスクリプション解除
   └─ onUnmounted     ← 破棄後
```

```ts
// どちらのフレームワークでも共通
import { onMounted, onUnmounted, onBeforeUnmount } from 'vue'

onMounted(() => {
  // API 取得・DOM 操作・タイマー開始
})

onBeforeUnmount(() => {
  // タイマー解除・イベントリスナー削除
})
```

---

## 3. ナビゲーションライフサイクル（最重要な差分）

### 3-1. Ionic 専用フック：ionView*

Ionic の `IonRouterOutlet` はページを DOM に保持したまま「表示/非表示」を切り替える。
そのため Vue の `onMounted` は初回のみ発火し、**画面に戻るたびに発火する専用フックが必要**。

```
画面A → 画面B に遷移
  画面A: ionViewWillLeave → ionViewDidLeave
  画面B: ionViewWillEnter → ionViewDidEnter（onMounted も発火）

画面B → 画面A に戻る（バックナビゲーション）
  画面B: ionViewWillLeave → ionViewDidLeave（onUnmounted は発火しない）
  画面A: ionViewWillEnter → ionViewDidEnter（onMounted は発火しない ← 重要）
```

| フック | 発火タイミング | 主な用途 |
|--------|--------------|---------|
| `ionViewWillEnter` | 画面がアクティブになる直前（毎回） | データ再取得・状態リセット |
| `ionViewDidEnter` | 画面がアクティブになった後（毎回） | アニメーション後処理・フォーカス設定 |
| `ionViewWillLeave` | 画面を離れる直前（毎回） | 入力保存・確認ダイアログ |
| `ionViewDidLeave` | 画面を離れた後（毎回） | リソース解放・タイマー停止 |

```ts
// Ionic でのデータ再取得パターン
import { onMounted } from 'vue'
import { onIonViewWillEnter } from '@ionic/vue'

onMounted(() => {
  // 初回のみ：初期化処理
  initializeComponent()
})

onIonViewWillEnter(() => {
  // 画面に戻るたびに：データ更新
  fetchLatestData()
})
```

### 3-2. Vuetify（vue-router）での対応

Vuetify には `ionView*` に相当するフックがない。
目的に応じて3つのアプローチを組み合わせる。

#### アプローチ A：`<KeepAlive>` + `onActivated`（ionView* に最も近い）

```ts
// App.vue または RouterView のラッパー
```
```vue
<template>
  <RouterView v-slot="{ Component }">
    <KeepAlive>
      <component :is="Component" />
    </KeepAlive>
  </RouterView>
</template>
```

```ts
// 各ページコンポーネント
import { onMounted, onActivated, onDeactivated } from 'vue'

onMounted(() => {
  // 初回のみ
  initializeComponent()
})

onActivated(() => {
  // ページに戻るたびに ← ionViewWillEnter に相当
  fetchLatestData()
})

onDeactivated(() => {
  // ページを離れるたびに ← ionViewDidLeave に相当
  clearTimers()
})
```

**注意：`<KeepAlive>` は全ページのコンポーネントを DOM に保持するため、メモリ使用量が増加する。**
`include` / `exclude` / `max` で保持対象を絞ること。

```vue
<!-- 最大3ページまでキャッシュ、特定ページは除外 -->
<KeepAlive :max="3" exclude="LoginPage,ErrorPage">
  <component :is="Component" />
</KeepAlive>
```

#### アプローチ B：`watch(route)` でルート変化を検知

```ts
import { watch } from 'vue'
import { useRoute } from 'vue-router'

const route = useRoute()

watch(
  () => route.fullPath,
  (newPath, oldPath) => {
    if (newPath === '/target-page') {
      fetchLatestData()
    }
  }
)
```

#### アプローチ C：vue-router ナビゲーションガード

```ts
import { onBeforeRouteLeave, onBeforeRouteUpdate } from 'vue-router'

// この画面を離れる直前（ionViewWillLeave に相当）
onBeforeRouteLeave((to, from, next) => {
  if (hasUnsavedChanges.value) {
    const confirmed = window.confirm('変更が保存されていません。離れますか？')
    confirmed ? next() : next(false)
  } else {
    next()
  }
})

// 同じルートでパラメータが変わったとき（例：/item/1 → /item/2）
onBeforeRouteUpdate((to, from) => {
  fetchItem(to.params.id)
})
```

### 3-3. ionView* と vue-router の対応表

| Ionic フック | vue-router / Vue 対応手段 | KeepAlive 必要 | 備考 |
|-------------|--------------------------|:--------------:|------|
| `ionViewWillEnter` | `onActivated` | ○ | 最も近い代替。KeepAlive 必須 |
| `ionViewWillEnter` | `watch(route.path)` | △ | KeepAlive 不要だが全ページ再マウント |
| `ionViewDidEnter` | `onActivated` + nextTick | ○ | DOM 確定後に処理したい場合は nextTick を追加 |
| `ionViewWillLeave` | `onBeforeRouteLeave` | — | ナビゲーションガードで代替 |
| `ionViewDidLeave` | `onDeactivated` | ○ | KeepAlive 使用時 |
| `ionViewDidLeave` | `onUnmounted` | — | KeepAlive 未使用時（再マウントされる） |

### 3-4. ページの DOM 保持挙動の違い

```
【Ionic + IonRouterOutlet】
画面A  ─────────────────────────  常に DOM に存在
画面B       ──────────                 常に DOM に存在
         遷移  戻る

【Vuetify + vue-router（KeepAlive なし）】
画面A  ────      ────              遷移時に unmount → 戻ると再 mount
画面B      ──────                  存在する間だけ mount

【Vuetify + vue-router（KeepAlive あり）】
画面A  ─────────────────────────  Ionic と同等：常に DOM に存在
画面B       ──────                 常に DOM に存在
```

---

## 4. アプリ前面/背面ライフサイクル（Capacitor 共通）

Ionic・Vuetify どちらも `@capacitor/app` を使う。実装は完全に同一。

```ts
// どちらのフレームワークでも共通
import { App } from '@capacitor/app'
import { onMounted, onUnmounted } from 'vue'

onMounted(() => {
  App.addListener('appStateChange', ({ isActive }) => {
    if (isActive) {
      // フォアグラウンドに戻った
      refreshDataIfStale()
    } else {
      // バックグラウンドに移行
      saveTemporaryState()
    }
  })
})

onUnmounted(() => {
  App.removeAllListeners()
})
```

### 4-1. 主要アプリイベント

| イベント | 発火タイミング | 主な用途 |
|---------|--------------|---------|
| `appStateChange` | フォア/バックグラウンド切り替え | データ更新・状態保存 |
| `appUrlOpen` | ディープリンクで起動 | URL パラメータ解析・ページ遷移 |
| `backButton` | Androidバックボタン | カスタム戻る処理 |
| `resume`（Cordova 互換） | 非推奨 → appStateChange を使用 | — |

### 4-2. ディープリンク処理

**Ionic（自動的にルーターと連携）**
```ts
// Ionic は App URL を自動的にルーターに渡す設定が可能
import { App } from '@capacitor/app'
import { useIonRouter } from '@ionic/vue'

const ionRouter = useIonRouter()

App.addListener('appUrlOpen', (data) => {
  const slug = data.url.split('.app').pop()
  if (slug) ionRouter.push(slug)
})
```

**Vuetify（vue-router に手動で渡す）**
```ts
import { App } from '@capacitor/app'
import { useRouter } from 'vue-router'

const router = useRouter()

App.addListener('appUrlOpen', (data) => {
  const slug = data.url.split('.app').pop()
  if (slug) router.push(slug)
})
```

---

## 5. ハードウェアバックボタン制御

### 5-1. Ionic

```ts
import { useBackButton } from '@ionic/vue'
import { useIonRouter } from '@ionic/vue'

const ionRouter = useIonRouter()

// priority: 数値が高いほど優先。デフォルトの戻る動作より高くする場合は 10 以上
useBackButton(10, () => {
  if (ionRouter.canGoBack()) {
    ionRouter.back()
  } else {
    // アプリを終了するか確認
    App.exitApp()
  }
})
```

### 5-2. Vuetify（@capacitor/app で手動実装）

```ts
import { App } from '@capacitor/app'
import { useRouter } from 'vue-router'
import { onMounted, onUnmounted } from 'vue'

const router = useRouter()

onMounted(async () => {
  const handler = await App.addListener('backButton', ({ canGoBack }) => {
    if (canGoBack) {
      router.back()
    } else {
      App.exitApp()
    }
  })

  onUnmounted(() => handler.remove())
})
```

---

## 6. ライフサイクル全体の発火順序まとめ

### 6-1. 初回表示

```
【Ionic】                           【Vuetify（KeepAlive なし）】
onBeforeMount                       onBeforeMount
onMounted                           onMounted
ionViewWillEnter
ionViewDidEnter
```

### 6-2. 別画面に遷移

```
【Ionic】                           【Vuetify（KeepAlive なし）】
ionViewWillLeave                    onBeforeRouteLeave（ガード使用時）
ionViewDidLeave                     onBeforeUnmount
                                    onUnmounted

【Vuetify（KeepAlive あり）】
onBeforeRouteLeave（ガード使用時）
onDeactivated
```

### 6-3. 画面に戻る

```
【Ionic】                           【Vuetify（KeepAlive なし）】
ionViewWillEnter  ← 必ず発火        onBeforeMount    ← 再マウント
ionViewDidEnter                     onMounted

【Vuetify（KeepAlive あり）】
onActivated       ← ionViewWillEnter に相当
```

---

## 7. 実装パターン別の推奨対応

| シナリオ | Ionic | Vuetify 3 | 注意点 |
|---------|-------|-----------|-------|
| 画面表示のたびにデータ再取得 | `onIonViewWillEnter` | `onActivated`（KeepAlive 必須） | KeepAlive がないと再マウントで onMounted が代替になる |
| 戻ったときだけデータ更新（前進時は不要） | `onIonViewWillEnter` + 前回パス判定 | `onActivated` + 前回ルート判定 | router.currentRoute で来た方向を判定 |
| 画面離脱時の未保存チェック | `ionViewWillLeave` | `onBeforeRouteLeave` | Vuetify はナビゲーションガードが明確 |
| タイマー・ポーリングの停止 | `ionViewDidLeave` | `onDeactivated`（KeepAlive）または `onUnmounted` | KeepAlive の有無で使い分け |
| フォームの状態保持（戻っても消えない） | IonRouterOutlet が自動保持 | KeepAlive 必須 | Vuetify は明示的に設定が必要 |
| アプリ復帰時のデータ更新 | `App.appStateChange` | `App.appStateChange` | 両方同一実装 |
| ディープリンク処理 | `App.appUrlOpen` + useIonRouter | `App.appUrlOpen` + useRouter | 呼び出すルーターの違いのみ |
| バックボタンのカスタム制御 | `useBackButton(priority, cb)` | `App.addListener('backButton', cb)` | Ionic は優先度制御が組み込み |

---

## 8. Vuetify 移行時の実装指針

| Ionic のコード | Vuetify での置き換え方 |
|--------------|---------------------|
| `onIonViewWillEnter(() => fetch())` | `<KeepAlive>` 追加 + `onActivated(() => fetch())` |
| `onIonViewDidLeave(() => clearTimer())` | `onDeactivated(() => clearTimer())` または `onUnmounted` |
| `onIonViewWillLeave` で離脱確認 | `onBeforeRouteLeave((to, from, next) => { ... })` |
| `useBackButton(10, cb)` | `App.addListener('backButton', cb)` |
| `ionRouter.push('/path')` | `router.push('/path')` |
| `ionRouter.canGoBack()` | `window.history.length > 1`（または独自履歴管理） |

---

*比較対象バージョン：Ionic 7/8（@ionic/vue）、Vue Router 4.x、Capacitor 6.x*
