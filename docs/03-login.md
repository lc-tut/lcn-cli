# ログイン

`lcn-cli login` は OAuth 2.0 の Authorization Code Flow に
PKCE と loopback redirect を組み合わせた方式です。
`gcloud auth login` や `gh auth login` と同じ形です。

---

## 流れ

1. **CLI が PKCE のパラメータを作る**
   ランダムな `code_verifier` を生成し、その SHA-256 を `code_challenge` とします。
1. **CLI が loopback サーバを立てる**
   `127.0.0.1` の空きポートで HTTP サーバを起動します（例: `:52341`）。
   CSRF 対策の `state` も生成します。
1. **ブラウザを開く**
   Authentik の認可エンドポイントへ飛ばします。
1. **Authentik が認証する**
   既存の `lc-cloud-authentication` フロー
   （identification → password → MFA → login）がそのまま走ります。
   MFA は必須です。
1. **loopback に認可コードが返る**
   CLI は `state` を照合してから次に進みます。
1. **トークンに交換する**
   `code` と `code_verifier` をトークンエンドポイントへ POST します。
   Authentik が `code_challenge` と突き合わせて検証します。
1. **保存して終了**
   トークンを `~/.config/lcn/` に書き、ブラウザに完了画面を表示して
   loopback サーバを閉じます。

```text
lcn-cli                     ブラウザ              Authentik
   │                           │                     │
   ├─ 127.0.0.1:52341 で待機   │                     │
   ├─ ブラウザを開く ─────────→│                     │
   │                           ├─ /authorize ───────→│
   │                           │                     ├─ ログイン + MFA
   │                           │←─ 302 127.0.0.1 ────┤
   │←─ code, state ────────────┤                     │
   ├─ state を照合             │                     │
   ├─ POST /token（code + code_verifier）───────────→│
   │←─ access / refresh / id token ───────────────────┤
   └─ ~/.config/lcn/ に保存
```

---

## 認可リクエスト

```text
https://<authentik>/application/o/authorize/
  ?response_type=code
  &client_id=<client_id>
  &redirect_uri=http://127.0.0.1:52341/callback
  &scope=openid profile lc_cloud
  &code_challenge=<S256(code_verifier)>
  &code_challenge_method=S256
  &state=<random>
```

`scope` に `lc_cloud` が必要です。
`microservices-terraform` の `terraform/platform/idp/provider_middleware.tf` が
このスコープで `lcn_id` と `groups` のクレームを載せています。
これが無いと認可の判断材料が揃いません。

### client_secret は使わない

Authentik 側の provider は `client_type = "public"` で設定済みです。
CLI に秘密を埋め込むことはできないため、PKCE で代替します。

---

## loopback のポートは 127.0.0.1 に固定する

**`localhost` ではなく `127.0.0.1` を使います。**

`localhost` は環境によって `::1`（IPv6）に解決されます。
その場合 Authentik に登録された redirect URI と一致せず失敗します。
`[::1]` も許可するには redirect URI を 1 本増やす必要があり、
順序の問題（後述）を悪化させます。

ポートは毎回変わるため、Authentik 側は正規表現で登録します。

```hcl
{
  matching_mode = "regex"
  url           = "http://127\\.0\\.0\\.1:[0-9]{1,5}/callback"
}
```

Authentik の照合は `re.fullmatch` なので両端がアンカーされており、
部分一致による open redirect は起きません。
詳細と注意点は
[lcn-access-proxy の docs/07-decisions.md](https://github.com/lc-tut/lcn-access-proxy/blob/main/docs/07-decisions.md)
を参照してください。

---

## トークンの保存

| 項目 | 値 |
| --- | --- |
| 場所 | `~/.config/lcn/token.json`（`LCN_CONFIG_DIR` で変更可） |
| パーミッション | ファイル `0600`・ディレクトリ `0700` |
| 中身 | access token・refresh token・有効期限・issuer・client_id |

OS のキーチェーン（macOS Keychain・Windows Credential Manager・
Linux の Secret Service）が使えるなら、そちらを優先します。
ただし**キーチェーンが無い環境でもファイルで動くこと**を必須とします。
Linux のヘッドレス環境や WSL で詰まらせないためです。

### 寿命

`microservices-terraform` 側の設定は次の通りです。

| トークン | 有効期間 |
| --- | --- |
| access token | 1 時間 |
| refresh token | 7 日 |

したがって**最長 7 日ごとに再ログインが必要**です。
その間は access token が自動更新されます。

グループ変更が反映されるまで最大 1 時間かかる点にも注意してください
（access token の寿命が上限になります）。

---

## ログインしていないときの挙動

サーバーへ通信するコマンドは、未ログインなら終了コード 3 で終わります。

```console
$ lcn-cli ssh web-01
ログインしていません。lcn-cli login を実行してください。
```

**自動でブラウザを開いてログインを始めることはしません。**
`ProxyCommand` 経由で呼ばれた場合に、
`ssh` の裏で勝手にブラウザが開くと利用者が状況を理解できないためです。

---

## logout

`~/.config/lcn/` のトークンを削除します。

Authentik 側のセッションは終了させません。
ブラウザのセッションまで切りたい場合は
Authentik の `/outpost.goauthentik.io/sign_out` を使う必要があり、
それは CLI の責務外とします。
