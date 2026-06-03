# Vuetify 3 vs Vuetify 4 比較ドキュメント

**Vuetify 4（コードネーム Revisionist）は 2026年2月 に安定版リリース。**  
**本ドキュメントは業務用 Android ハンドヘルドアプリ（Capacitor 構成）視点での比較。**

---

## 1. エグゼクティブサマリー

| 評価軸 | Vuetify 3 | Vuetify 4 | 変化の大きさ |
|--------|:---------:|:---------:|:-----------:|
| CSS カスタマイズ・他フレームワーク共存 | △ `!important` が多く上書き困難 | ◎ CSS `@layer` で上書き容易 | **大きい** |
| Material Design 3 準拠度 | △ 一部 MD3 対応 | ◎ タイポグラフィ・エレベーション完全 MD3 | **大きい** |
| グリッドシステム | ○ 安定 | ○ 改善（RTL・flexbox 強化） | **中程度** |
| CSS リセット | △ ブラウザデフォルトを多く上書き | ○ 最小限のリセット | **中程度** |
| コンポーネント API の互換性 | — | ○ ほぼ互換（破壊的変更は限定的） | **小さい** |
| 移行コスト（v3 → v4） | — | △ 段階的移行ツールあり | **要計画** |
| デザイントークン連携 | ○ Sass 変数 + CSS 変数 | ◎ CSS `@layer` で上書き精度が上がる | **有利** |

> **新規プロジェクトは Vuetify 4 を採用推奨。**  
> 既存の Vuetify 3 プロジェクトは破壊的変更が段階的に戻せる設計のため、  
> 互換スニペットを適用しながら余裕を持って移行できる。

---

## 2. 主要変更点の詳細

### 2-1. CSS Layers（最重要変更）

Vuetify 4 はすべてのスタイルを CSS `@layer` ブロックに格納し、`!important` を廃止。

**Vuetify 3 の課題**
```css
/* Vuetify 3 の内部 CSS（例） */
.v-btn {
  text-transform: uppercase !important;  /* 上書きするには !important が必要だった */
}
```

**Vuetify 4 の改善**
```css
/* Vuetify 4 の内部 CSS */
@layer vuetify {
  .v-btn { text-transform: uppercase; }
}

/* プロジェクト側（@layer 外）は自動的に優先される */
.v-btn { text-transform: none; }  /* !important 不要で上書き可能 */
```

**業務アプリへの影響**

| 項目 | Vuetify 3 | Vuetify 4 |
|------|-----------|-----------|
| デザイントークンの上書き | `!important` または Sass 変数が必要 | CSS 変数 / 通常クラスで上書き可 |
| Tailwind / UnoCSS との共存 | △ クラス競合・`!important` 競合 | ○ `@layer` で優先度が自動整理 |
| グローバル CSS のスタイル競合 | △ 発生しやすい | ○ 発生しにくい |
| カスタムコンポーネントのスタイル調整 | △ Sass 変数か `!important` が必要 | ○ 通常 CSS で対応できる範囲が広い |

---

### 2-2. タイポグラフィ（MD3 完全対応）

Vuetify 3 は MD2 ベースのタイポグラフィ。Vuetify 4 は MD3 タイプスケールに移行。

**タイポグラフィクラスの変更**

| MD2（Vuetify 3） | MD3（Vuetify 4） | フォントサイズ目安 |
|-----------------|-----------------|---------------|
| `text-h1` | `text-display-large` | 57px |
| `text-h2` | `text-display-medium` | 45px |
| `text-h3` | `text-display-small` | 36px |
| `text-h4` | `text-headline-large` | 32px |
| `text-h5` | `text-headline-medium` | 28px |
| `text-h6` | `text-headline-small` | 24px |
| `text-subtitle-1` | `text-title-large` | 22px |
| `text-subtitle-2` | `text-title-medium` | 16px |
| `text-body-1` | `text-body-large` | 16px |
| `text-body-2` | `text-body-medium` | 14px |
| `text-caption` | `text-body-small` | 12px |
| `text-overline` | `text-label-small` | 11px |

**旧クラスへの互換対応（段階的移行用）**
```scss
// Sass config で MD2 タイポグラフィを一時的に復元可能
@use 'vuetify' with (
  $typography-scale: 'md2'  // MD2 スケールを維持する
);
```

---

### 2-3. エレベーション（MD3 6段階システム）

Vuetify 3（MD2）は 0〜24 の 25 段階。Vuetify 4（MD3）は 6 段階に整理。

| Vuetify 3（MD2・25 段階） | Vuetify 4（MD3・6 段階） | 用途 |
|--------------------------|------------------------|------|
| `elevation-0` | `elevation-0` | フラット（背景と同一） |
| `elevation-1` | `elevation-1` | カード、メニュー |
| `elevation-2` ～ `elevation-4` | `elevation-2` | 検索バー、FAB |
| `elevation-6` ～ `elevation-8` | `elevation-3` | ナビゲーションバー |
| `elevation-12` | `elevation-4` | ダイアログ |
| `elevation-16` ～ `elevation-24` | `elevation-5` | モーダル最前面 |

**業務アプリへの影響：** 詳細な `elevation-*` 数値を使っていた場合はクラス名の見直しが必要。  
既存の `elevation-N` クラスは互換 CSS スニペットで一時的に復元可能。

```css
/* MD2 エレベーション互換スニペット（移行猶予期間中に使用） */
@import 'vuetify/styles/elevation-compat.css';
```

---

### 2-4. グリッドシステムの変更

| 変更点 | Vuetify 3 | Vuetify 4 | 備考 |
|--------|-----------|-----------|------|
| `v-row dense` prop | あり | **削除** → `density="compact"` に統一 | 破壊的変更 |
| `v-col offset-*` クラス | あり | **変更** | コードモドで自動移行可 |
| `v-row align` / `justify` | prop として存在 | **一部変更** | アップグレードガイド参照 |
| RTL サポート | △ | ○ 改善 | 業務アプリでの多言語対応に有利 |
| 負のマージン（Gutter） | あり | **廃止** → ギャップベースに変更 | 互換 CSS スニペットで復元可能 |

**Vuetify 3 互換グリッドの一時復元**
```css
/* @layer vuetify-overrides でネガティブマージングリッドを復元 */
@layer vuetify-overrides {
  .v-row { margin: -12px; }
  .v-col { padding: 12px; }
}
```

---

### 2-5. CSS リセットの変更

| 項目 | Vuetify 3 | Vuetify 4 |
|------|-----------|-----------|
| ブラウザデフォルト上書き範囲 | 広い（`h1`〜`h6` 等を積極的に上書き） | 最小限（ブラウザデフォルトを尊重） |
| `h1`〜`h6` のデフォルトスタイル | Vuetify が上書き | ブラウザのデフォルト + MD3 タイポグラフィクラス |
| カスタムリセット追加 | 困難（`!important` 競合） | `@layer vuetify-core.reset` で制御しやすい |

---

### 2-6. ボタンの text-transform

| 項目 | Vuetify 3 | Vuetify 4 |
|------|-----------|-----------|
| `v-btn` のデフォルト文字変換 | `uppercase`（大文字） | **変更なし**（`uppercase` のまま） |
| 変更したい場合 | Sass 変数 or スタイル上書き | Sass 変数 or グローバルデフォルト |

```ts
// v4 でのグローバルデフォルト変更
createVuetify({
  defaults: {
    VBtn: { style: 'text-transform: none' }
  }
})
```

---

## 3. コンポーネント API の互換性

### 3-1. 互換性が維持されるもの

| 項目 | 状況 |
|------|------|
| コンポーネント名（`v-btn`、`v-text-field` 等） | ○ 変更なし |
| `createVuetify()` の基本構造 | ○ 変更なし |
| テーマ設定（`theme.themes.light.colors`） | ○ 変更なし |
| Composition API サポート | ○ 変更なし |
| TypeScript サポート | ○ 変更なし |
| Capacitor / Vite 統合 | ○ 変更なし |

### 3-2. 破壊的変更一覧

| 変更内容 | 種別 | 自動移行ツール | 互換スニペット |
|---------|------|:------------:|:------------:|
| タイポグラフィクラス名（MD3 移行） | クラス名変更 | ○ codemods | ○ Sass config |
| エレベーションクラス（MD3 6段階） | クラス名変更 | ○ codemods | ○ CSS import |
| グリッド `v-row dense` → `density` | prop 変更 | ○ codemods | ○ CSS snippet |
| `v-col offset-*` クラス変更 | クラス名変更 | ○ codemods | ○ CSS snippet |
| CSS リセット範囲の縮小 | スタイル変化 | — | ○ reset snippet |
| `@layer` によるスタイル優先度変更 | スタイル変化 | — | — |

---

## 4. 移行ツールと移行戦略

### 4-1. 利用可能なツール

| ツール | 用途 |
|--------|------|
| `vuetify-codemods` | タイポグラフィ・グリッド・エレベーションクラス名の自動置換 |
| Vuetify MCP サーバー | AI 支援による破壊的変更スキャン・修正提案 |
| 互換スニペット集 | 各変更を一時的に旧動作に戻す CSS / Sass スニペット |

**Vuetify MCP サーバーのセットアップ（Claude Code 環境）**
```bash
claude mcp add --transport http vuetify-mcp https://mcp.vuetifyjs.com/mcp
```

```text
# プロジェクトスキャンのプロンプト例
Using the vuetify-mcp server, scan this project for Vuetify 3 to 4 breaking changes.
List each issue found with the file, line number, and recommended fix.
```

### 4-2. 推奨移行戦略（既存 v3 プロジェクト）

```
Step 1: パッケージを v4 に更新
  npm install vuetify@^4

Step 2: 互換スニペットをすべて適用（旧動作を一時復元）
  - Grid 互換 CSS
  - Typography: Sass config で MD2 スケール復元
  - Elevation: 互換 CSS import
  - CSS Reset: full reset snippet

Step 3: 動作確認（旧動作のまま v4 で動くことを確認）

Step 4: 段階的に各エリアを MD3 に移行
  - タイポグラフィクラスを codemods で一括置換
  - エレベーションクラスを codemods で一括置換
  - グリッド prop を手動確認・修正

Step 5: 互換スニペットを順次削除
```

### 4-3. 業務アプリへの移行コスト概算

| 作業 | 工数目安 | ツール対応 |
|------|---------|-----------|
| パッケージ更新 + 互換スニペット適用 | 0.5日 | — |
| 動作確認（回帰テスト） | 1〜2日 | — |
| タイポグラフィクラスの一括置換 | 0.5日 | codemods で自動化 |
| エレベーション・グリッドの修正 | 0.5〜1日 | codemods で一部自動化 |
| カスタムスタイルの `!important` 除去 | 1〜2日 | 手動 |
| デザイントークン連携の見直し | 1〜2日 | CSS `@layer` 活用 |
| **合計** | **4〜8日** | |

---

## 5. 新規プロジェクトでの採用判断

| ケース | 推奨 | 理由 |
|--------|------|------|
| 新規プロジェクト（2026年以降） | **Vuetify 4** | MD3 完全対応・CSS layers・長期サポート対象 |
| 既存 v3 プロジェクト（安定稼働中） | **v3 継続 → 段階移行** | 互換スニペットで余裕を持って移行可能 |
| デザイントークン連携を重視 | **Vuetify 4** | CSS `@layer` で上書き精度が大幅向上 |
| Android Material You 準拠を重視 | **Vuetify 4** | MD3 エレベーション・タイポグラフィが Android 標準に近い |

---

## 6. Vuetify0（ヘッドレスプリミティブ）について

2026年4月、Vuetify チームは **Vuetify0（v0）** を公開アルファとして公開。

| 項目 | 内容 |
|------|------|
| 概要 | スタイルなしのヘッドレス Vue プリミティブライブラリ |
| コンポーネント数 | 46 コンポーネント + 63 composables（2026年4月時点） |
| 用途 | 独自デザインシステムの基盤として使用 |
| Vuetify との関係 | 別パッケージ。Vuetify の内部実装を段階的に移行中 |
| 業務アプリへの影響 | 現時点はアルファ。Vuetify 本体の使用感には影響なし |

---

*バージョン情報：Vuetify 3.x（最終安定版）、Vuetify 4.0（2026年2月安定版リリース）*
