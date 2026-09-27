# コマンド仕様

---

## 一覧

| command | 動作 | サーバーへの通信 |
| --- | --- | --- |
| `lcn-cli login` | ブラウザで Authentik にログインしてトークンを保存する | Authentik のみ |
| `lcn-cli logout` | 保存したトークンを削除する | なし |
| `lcn-cli whoami` | 自分の `lcn_id`・ユーザー名・所属・トークンの残り時間を表示する | なし（トークンを復号して表示） |
| `lcn-cli ls` | 入れる VM の一覧を表示する | `GET /api/access/targets` |
| `lcn-cli ssh <target>` | VM に接続する（`ssh` を exec する） | `GET /api/access/tunnel/{target}` |
| `lcn-cli proxy <target>` | `ProxyCommand` 用。stdio と WebSocket を中継する | `GET /api/access/tunnel/{target}` |
| `lcn-cli sessions` | 自分の現在のセッションを表示する | `GET /api/access/sessions` |
| `lcn-cli sessions kill <id>` | 自分のセッションを切る | `DELETE /api/access/sessions/{id}` |

---

## 共通の挙動

### トークンの自動更新

サーバーへ通信する全コマンドは、実行前に access token の期限を確認します。

1. 期限内ならそのまま使う
1. 期限切れなら refresh token で**無言で**更新する
1. refresh token も期限切れなら `lcn-cli login` を促して終了する

`proxy` はとくに重要です。`ssh` から起動されるため、
**標準出力に余計なものを書いてはいけません**（[04](04-ssh-integration.md)）。
更新のログは標準エラーに出します。

### 設定

| 項目 | 既定値 | 上書き |
| --- | --- | --- |
| プロキシの URL | ビルド時に埋め込む | `LCN_PROXY_URL` 環境変数 |
| Authentik の issuer | ビルド時に埋め込む | `LCN_OIDC_ISSUER` 環境変数 |
| client_id | ビルド時に埋め込む | `LCN_OIDC_CLIENT_ID` 環境変数 |
| 設定ディレクトリ | `~/.config/lcn/` | `LCN_CONFIG_DIR` 環境変数 |

既定値をビルド時に埋め込むのは、**利用者に初期設定をさせないため**です。
「ドキュメントを読まずに設定する人が多い」という反省を踏まえて、
インストールしたら `lcn-cli login` だけで動く状態にします。

環境変数での上書きは開発時とローカル検証のために残します。

### 終了コード

| code | 意味 |
| --- | --- |
| 0 | 成功 |
| 1 | 一般的なエラー |
| 2 | 引数が不正 |
| 3 | 未ログイン、またはトークンが期限切れ（`lcn-cli login` が必要） |
| 4 | 認可エラー（その VM に入る権限が無い） |
| 5 | セッションの競合（別の場所から接続中） |

3 と 5 を分けているのは、`ssh` 経由で失敗したときに
利用者が原因を判断できるようにするためです。

---

## lcn-cli ssh と lcn-cli proxy の違い

どちらも同じエンドポイントに繋ぎますが、役割が違います。

**`lcn-cli ssh <target>`** は利用者が直接叩くものです。
内部で `ssh` コマンドを組み立てて exec します。
`~/.ssh/config` を書いていない人でもすぐ使えるようにするための入口です。

**`lcn-cli proxy <target>`** は `ssh` から呼ばれるものです。
利用者が直接叩くことはありません。
stdin / stdout を WebSocket に中継するだけで、他には何もしません。

```text
利用者が直接 →  lcn-cli ssh web-01
                  └ ssh -o ProxyCommand="lcn-cli proxy web-01" rocky@web-01

ssh から     →  lcn-cli proxy web-01
                  └ stdin/stdout ↔ wss://.../api/access/tunnel/web-01
```

---

## 出力の方針

### 人間向けが既定

`ls` や `sessions` は整列した表を出します。
`--json` を付けたときだけ機械可読な JSON を出します。

### 色は自動判定

標準出力が端末でないとき、`NO_COLOR` 環境変数が設定されているときは
色を付けません。

### エラーメッセージは次の行動を書く

原因だけを書いて終わらないようにします。

```text
悪い例: 401 Unauthorized

良い例: ログインの有効期限が切れています。
        lcn-cli login を実行してください。
```

```text
悪い例: 409 Conflict

良い例: 別の場所（203.0.113.45）から接続中のため、ここからは接続できません。
        同時に使える場所は 1 箇所までです。
        心当たりがなければ管理者に連絡してください。
        現在のセッション: lcn-cli sessions
```

最後の例は貸しの検出も兼ねています。
本人が「心当たりがない」と気づけることが、アカウント共有の発見につながります。
