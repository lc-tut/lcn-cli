# SSH 連携

`lcn-cli` は `ssh` を置き換えません。`ProxyCommand` で経路だけを差し替えます。

---

## ssh_config

```text
# ~/.ssh/config
Host *.lc-cloud
  ProxyCommand  lcn-cli proxy %h
  HostKeyAlias  lc-cloud-proxy
  User          rocky
```

これで素の `ssh` / `scp` / `rsync` / `sftp` / VS Code Remote-SSH / git over ssh が
そのまま通ります。

```bash
ssh web-01.lc-cloud
scp ./app.tar.gz web-01.lc-cloud:/tmp/
rsync -av ./src/ web-01.lc-cloud:/srv/app/
code --remote ssh-remote+web-01.lc-cloud /srv/app
```

`lcn-cli ssh <target>` はこの設定を書いていない人向けの入口で、
内部で同等の `ssh` コマンドを組み立てて exec します。

---

## ProxyCommand が何をするか

`ssh` は `ProxyCommand` に指定されたプロセスを起動し、
**その標準入出力を TCP ソケットの代わりに使います**。

`lcn-cli proxy` はその stdin / stdout を WebSocket に中継するだけです。
SSH プロトコルは一切解釈しません。

```text
ssh                          lcn-cli proxy              lcn-access-proxy
 │                                │                          │
 ├─ プロセス起動 ────────────────→│                          │
 │                                ├─ wss:// 接続（Bearer）──→│
 │                                │←─ 101 Switching ─────────┤
 ├─ SSH ハンドシェイク ──stdout──→├──── バイナリフレーム ───→│
 │←──────────────────── stdin ───←├←─── バイナリフレーム ────┤
```

サーバー側は SSH サーバーとして応答し、
内部で対象 VM への別の SSH 接続を張ってチャネルを橋渡しします。
詳細は
[lcn-access-proxy の docs/02-architecture.md](https://github.com/lc-tut/lcn-access-proxy/blob/main/docs/02-architecture.md)
を参照してください。

---

## proxy 実装の注意点

### 標準出力を汚さない

**標準出力は SSH のデータチャネルです。**
ログ・進捗・警告を 1 バイトでも書くと、SSH のハンドシェイクが壊れます。

- ログはすべて標準エラーに出す
- トークンの自動更新のメッセージも標準エラー
- 色付けの ANSI エスケープも標準出力には出さない

### 終了の扱い

| 状況 | 挙動 |
| --- | --- |
| WebSocket が閉じた | stdin / stdout を閉じて終了コード 0 |
| `ssh` が終了した（stdin が EOF） | WebSocket を閉じて終了 |
| 認証エラー | 標準エラーにメッセージを出して終了コード 3 |
| セッション競合 | 標準エラーにメッセージを出して終了コード 5 |

`ssh` は `ProxyCommand` の終了コードを表示しません。
**エラーの理由は必ず標準エラーに書いてください。**
それが利用者に見える唯一の情報になります。

### 双方向の中継を止めない

stdin → WebSocket と WebSocket → stdout を独立に回します。
片方が終わってももう片方を即座に切らず、バッファを流し切ってから閉じます。
`scp` の転送終了時に末尾が欠けるのを防ぐためです。

### keepalive

WebSocket の ping / pong を送ります。
Ingress のアイドルタイムアウトで無操作の SSH セッションが切られるのを防ぎます。

---

## ホスト鍵

**提示されるホスト鍵は VM のものではなく `lcn-access-proxy` のものです。**

そのため `web-01` でも `web-02` でも同じ鍵が返ります。
`HostKeyAlias lc-cloud-proxy` を設定すると、
`known_hosts` のエントリが VM 名ごとに増えず 1 個に収まります。

設定しなくても動きますが、VM 名ごとに同じ鍵のエントリが並びます。

### 警告が出たとき

`WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED!` が出た場合、
プロキシのホスト鍵が変わったことを意味します。

利用者に「消して繋ぎ直す」を習慣づけさせてはいけません。
中間者攻撃に対する唯一の防御を捨てることになります。
**サーバー側はホスト鍵を環境ごとに固定する**必要があります
（[lcn-access-proxy の docs/04-keys-certificates.md](https://github.com/lc-tut/lcn-access-proxy/blob/main/docs/04-keys-certificates.md)）。

---

## できないこと

| 操作 | 可否 | 理由 |
| --- | --- | --- |
| `ssh -A`（agent forwarding） | 不可 | 証明書の extension で禁止。VM 側も `AllowAgentForwarding no` |
| `ssh -X`（X11 forwarding） | 不可 | 証明書の extension で禁止 |
| `ssh -L` / `-R`（port forwarding） | 可 | `permit-port-forwarding` を付与している |
| 自分の SSH 鍵を登録して使う | 不可 | VM 側が `AuthorizedKeysFile none`。CA 署名の証明書のみ受け付ける |
| `root` で直接ログイン | 不可 | VM 側が `PermitRootLogin no`。`sudo` を使う |

`-R` が使えるので、逆トンネルを作ることは技術的には可能です。
これを塞ぐのは VM 側の Security Group の egress 制限であり、
CLI 側の責務ではありません。
