# 開発環境セットアップ ガイド — ロードマップ

## このガイド群について

`container/` にある詳細な手順書を読む前に、全体の構造と目的を理解するためのガイド群です。

| # | ファイル | 学べること |
|---|---|---|
| 1 | [01-docker-basics.md](01-docker-basics.md) | Docker とは何か・コンテナと仮想マシンの違い |
| 2 | [02-wsl2-dev-env.md](02-wsl2-dev-env.md) | WSL2 の仕組みと開発環境の全体像 |
| 3 | [03-android-emulator.md](03-android-emulator.md) | Android エミュレータの起動方法と Capacitor 接続 |

## 最終的に何が動く状態になるか

```
物理PC (Windows)
├─ Hyper-V
│   └─ WSL2 (Ubuntu)
│       └─ Docker Engine
│           └─ 開発コンテナ (Oracle Linux 9)
│               ├─ Node.js / npm
│               └─ Capacitor CLI  ← ここで npm run dev を実行
│
└─ Android エミュレータ (AVD)  ← ここでアプリを表示
    ↑
    localhost でつながる (adb reverse / server.url)
```

## 既存ドキュメントとの関係

- **このガイド群 (`guide/`)** — 概念・考え方・全体像（入門）
- **`container/` の手順書** — 詳細コマンド・設定値のリファレンス

| このガイド | 詳細は |
|---|---|
| 02-wsl2-dev-env.md | [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) |
| 02-wsl2-dev-env.md | [container/5. prerequisite-concepts-guide.md](../container/5.%20prerequisite-concepts-guide.md) |
| 03-android-emulator.md | [container/3. vue-vuetify-capacitor-dev-environment.md](../container/3.%20vue-vuetify-capacitor-dev-environment.md) |
