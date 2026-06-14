# WSL2 + Ubuntu 開発環境の考え方

## 1. WSL2 とは何か

WSL (Windows Subsystem for Linux) は Windows 上で Linux を動かす仕組みです。

### WSL1 と WSL2 の違い

| | WSL1 | WSL2 |
|---|---|---|
| 仕組み | Windows カーネルが Linux システムコールを変換 | **Hyper-V 上で本物の Linux カーネルを動かす** |
| 本物のLinuxカーネル | なし | あり |
| Docker が動くか | 動かない | 動く |
| ファイルシステム性能 | Windows 側は速い | Linux 側が速い |

**Docker を使うには WSL2 が必須**（Docker は Linux カーネルの機能を使うため）。
手順書で「VERSION が 2 であること」を確認するのはこのためです。

### 構成の全体像

```
物理PC (Windows 11)
└─ Hyper-V
    └─ WSL2 仮想マシン (Ubuntu)
        ├─ systemd (サービス管理)
        ├─ Docker Engine
        │   └─ 開発コンテナ (Oracle Linux 9)
        │       ├─ Node.js 22
        │       ├─ JDK 21
        │       └─ Vue + Vuetify + Capacitor
        └─ VSCode Server (Remote - WSL 経由で接続)
```

---

## 2. セットアップの考え方

### Step 1: WSL2 + Ubuntu のインストール

PowerShell(管理者)で `wsl --install` を実行するだけで WSL2+Ubuntu が入ります。
バージョン確認で `VERSION=2` になっていることを確認します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 1

### Step 2: systemd の有効化（なぜ必要か）

systemd は Linux のサービス管理システムです。`docker start` などのコマンドが
自動で起動するには systemd が必要です。

WSL2 はデフォルトでは systemd を使わないため、`/etc/wsl.conf` で明示的に有効にします。

```ini
[boot]
systemd=true
```

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 2

### Step 3: Docker Engine のインストール

**Docker Desktop** (GUI アプリ) ではなく **Docker Engine** (CUI のみ) を Ubuntu 内に直接インストールします。

| | Docker Desktop | Docker Engine |
|---|---|---|
| インストール先 | Windows | Ubuntu(WSL2内) |
| ライセンス | 商用利用に有料プランあり | 無料(Apache 2.0) |
| GUI | あり | なし |
| 本番環境との差異 | あり | **ほぼなし** |

開発環境と本番環境(Linux サーバー)の差異を減らすため、Docker Engine を推奨します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 4〜

### Step 4: プロキシ設定の考え方

社内ネットワークでは PAC スクリプト方式のプロキシが一般的ですが、
**Linux の CLI ツール(apt, curl, Docker)は PAC スクリプトを解釈できません**。

```
Windows      → PAC スクリプトで自動設定 ✓
WSL2/Docker  → PAC スクリプト解釈不可 → 固定URLに変換して設定 ✓
```

PAC ファイルを手動でダウンロードし、`return "PROXY ホスト:ポート"` の記述から
プロキシ URL を取り出して設定します。

> 詳細: [container/1. wsl2-ubuntu-docker-proxy-setup.md](../container/1.%20wsl2-ubuntu-docker-proxy-setup.md) Step 3〜

---

## 3. ファイルシステムの注意点

WSL2 には2つのファイルシステムが共存します。

```
/mnt/c/Users/miyaw/  ← Windows 側のファイル (遅い)
/home/miyaw/         ← Linux 側のファイル  (速い)
```

**原則: コードは Linux 側に置く**

`/mnt/c/` 経由でアクセスすると、ファイル I/O が大幅に遅くなります。
`git clone` や `npm install` は WSL2 内の `/home/` 以下で行います。

---

## 4. VSCode でのリモート開発

VSCode の **Remote - WSL** 拡張を使うと、Windows 上の VSCode から WSL2 内のコードを直接編集できます。

```
VSCode (Windows) ─── Remote - WSL ───→ VSCode Server (Ubuntu)
                                              ↓
                                        /home/miyaw/project/
```

**開発ループの全体像:**

```
1. VSCode で Remote - WSL を接続
2. ターミナル(Ubuntu)で docker compose up -d
3. docker compose exec app npm run dev
4. Android エミュレータでアプリ確認
```

> 詳細: [container/4. vscode-extension-layer-management.md](../container/4.%20vscode-extension-layer-management.md)

---

## 次のステップ

→ [03-android-emulator.md](03-android-emulator.md) — Android エミュレータの起動と Capacitor 接続
