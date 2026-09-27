# 概要

`lcn-cli` は LC-Cloud の利用者が手元で使うコマンドラインクライアントです。

---

## 目的

やることは 2 つだけです。

1. Authentik でログインして OIDC トークンを手元に持つ
1. そのトークンで `lcn-access-proxy` へのトンネルを張る

VM への接続そのもの、認可の判断、証明書の署名は
すべて [lcn-access-proxy](https://github.com/lc-tut/lcn-access-proxy) が行います。
このリポジトリは**その入口を提供するだけ**です。

---

## 設計原則

### 1. 薄く保つ

複雑さはサーバー側に置きます。
とくに `lcn-cli proxy` は **SSH プロトコルを一切解釈しません**。
stdin / stdout と WebSocket を繋ぐだけです。

理由は運用です。**CLI を配り直さずにサーバーだけ直せる**状態を維持したいためです。
利用者全員に再インストールを依頼するのは、サークル運営では現実的なコストになります。

SSH のサーバー実装とチャネルの橋渡しは `lcn-access-proxy` 側にあります
（同リポジトリの `docs/02-architecture.md`）。

### 2. 秘密を持たない

| 持つもの | 持たないもの |
| --- | --- |
| OIDC の access / refresh トークン | SSH 秘密鍵 |
| | SSH 証明書 |
| | OpenStack の Application Credential |
| | サーバーのパスワード |

トークンは `~/.config/lcn/` に `0600` で保存します。
**これが端末に残る唯一の秘密です。**

refresh トークンは理屈の上では他人に渡せますが、
渡した相手はその人の全権限を得るため、
「VM だけを貸す」手段にはなりません。これは意図した設計です。

### 3. 素の ssh を置き換えない

`ssh` / `scp` / `rsync` / VS Code Remote を自作コマンドで置き換えません。
`ProxyCommand` で経路だけ差し替え、既存のツールチェーンをそのまま使えるようにします。

利用者が新しいコマンド体系を覚える必要がなく、
既存のスクリプトや `~/.ssh/config` の資産も活きます。

---

## Go を選んだ理由

Python も検討しましたが、Go にします。

### 決め手は ProxyCommand の起動回数

`lcn-cli proxy` は **SSH 接続のたびに起動します**。

```text
ssh web-01.lc-cloud       → 1 回
scp file web-01:/tmp/     → 1 回
rsync -av ./ web-01:/     → 1 回
VS Code Remote-SSH        → 接続確立で複数回
git push（over ssh）      → 毎回
```

ここに起動コストが乗ると体感がはっきり悪くなります。
Go なら数ミリ秒で、利用者に気づかれません。

### サイズも Go のほうが小さい

| | 配布サイズの目安 | 起動時間の目安 |
| --- | --- | --- |
| Go（静的バイナリ、`-ldflags="-s -w"`） | 10 MB 前後 | 数 ms |
| Python + PyInstaller（onefile） | 30〜50 MB | 0.3〜1.5 秒（一時展開が入る） |
| Python を `pip install` | 小さい | Python 本体が必要 |

Python を小さく配るには「利用者の環境に Python がある」前提が必要です。
それを捨てて単体配布にすると Go より大きくなります。

### ランタイム依存ゼロ

これが運用上いちばん効きます。Python 配布にすると必ずこうなります。

- `python` が見つからない
- Python のバージョンが古くて動かない
- `pip` が権限エラーで失敗する
- Windows の Microsoft Store 版 Python の挙動の違い

**このコストは作者が 1 回払うのではなく、利用者全員が毎回払います。**
バイナリを置くだけで済むなら、サポートの手間がまるごと消えます。

### 言語が揃う

`lcn-access-proxy`・`lcn-infra-api`・`lcn-billing-api` が Go です。
`microservices-terraform` の
`documents/terraform/14-middleware-architecture.md` が定めた方針でもあります。

CLI も Go にしておくと、**コントリビュートするメンバーが覚える言語が 1 つで済みます**。

### Python が適していたケース

`login` と数本の REST 呼び出しだけで、`ProxyCommand` として使われないなら
Python でも問題なかったはずです。
技術的に Python が有利な部分はこの CLI にはありません
（OIDC も WebSocket も両方ライブラリが揃っています）。
差が出るのは起動時間と配布だけです。

`proxy` だけ Go、他は Python というハイブリッドは、
リリースが 2 系統になるためどちらより悪くなります。

---

## 配布

| 方法 | 対象 |
| --- | --- |
| GitHub Releases | 全プラットフォーム。`darwin` / `linux` / `windows` × `amd64` / `arm64` |
| Homebrew tap | macOS / Linux |
| scoop | Windows |
| `curl \| sh` のインストーラ | 上記が使えない環境 |

サイズを詰めるための方針です。

- `-ldflags="-s -w"` でデバッグ情報を落とす
- WebSocket は軽量な実装を選ぶ
- サブコマンドが少ないうちは CLI フレームワークを入れず標準の `flag` で足りる
- 大きなアセットを埋め込まない
