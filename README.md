# lcn-cli

LC-Cloud を使うためのコマンドラインクライアントです。

Authentik でログインし、VM へ SSH するための経路を提供します。
**SSH の秘密鍵も証明書も持ちません。** 持つのは OIDC のトークンだけです。

```console
$ lcn-cli login
ブラウザを開きます... 完了しました（taro-yamada）

$ lcn-cli ls
NAME     PROJECT  STATUS
web-01   web      ACTIVE
web-02   web      SHUTOFF

$ lcn-cli ssh web-01
[rocky@web-01 ~]$
```

`~/.ssh/config` に設定すれば、素の `ssh` / `scp` / VS Code Remote がそのまま通ります。

```text
Host *.lc-cloud
  ProxyCommand  lcn-cli proxy %h
  HostKeyAlias  lc-cloud-proxy
  User          rocky
```

## ドキュメント

| # | ドキュメント | 内容 |
| --- | --- | --- |
| 01 | [概要](docs/01-overview.md) | 目的・設計原則・Go を選んだ理由 |
| 02 | [コマンド仕様](docs/02-commands.md) | サブコマンドの一覧と挙動 |
| 03 | [ログイン](docs/03-login.md) | loopback OIDC・PKCE・トークンの保存 |
| 04 | [SSH 連携](docs/04-ssh-integration.md) | `ProxyCommand`・`ssh_config`・ホスト鍵 |

## 関連リポジトリ

| リポジトリ | 関係 |
| --- | --- |
| [lcn-access-proxy](https://github.com/lc-tut/lcn-access-proxy) | 接続先のサーバー。設計の本体はこちら |
| [microservices-terraform](https://github.com/lc-tut/microservices-terraform) | Authentik の OIDC プロバイダ設定を管理する |

## 状態

設計のみ。実装は未着手です。
