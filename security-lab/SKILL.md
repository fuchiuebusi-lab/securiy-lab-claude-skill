---
name: security-lab
description: >
  ローカルのセキュリティ学習環境（OWASP Juice Shop, WebGoat, DVWA, OWASP crAPI）をDockerで
  構築・開始・停止・リセット・状態確認するときに使用する。
  「Juice Shopを起動して」「WebGoatを構築して」「DVWAを止めて」「crAPIをリセットして」
  「セキュリティ学習環境を立ち上げて」「全部止めて」「今何が動いてる？」のような依頼で使う。
  引数は `<環境名> <アクション>` の形式（例: `juice-shop up`, `dvwa reset`, `all stop`, `all check`）。
  環境名: juice-shop / webgoat / dvwa / crapi（すべて可: all）。
  アクション: build（構築）/ up（開始）/ stop（停止）/ reset（リセット）/ check（状態確認）。
---

# security-lab

このSkill単体で完結する、ローカルのセキュリティ学習環境（OWASP Juice Shop, WebGoat, DVWA, OWASP crAPI）を
Dockerで構築・操作するスキル。docker-compose定義は本Skillディレクトリ内の `compose/<環境名>/` に同梱されており、
どのプロジェクトのどのマシンから使っても、外部ファイルへの依存なしにそのまま動作する。
Docker（と`docker compose`）さえインストールされていれば、他に前提条件はない。

## Skillディレクトリ（`$SKILL_DIR`）の特定

このSkillが起動される際、システムメッセージに次のように表示される。

```
Base directory for this skill: <パス>
```

この `<パス>` を `$SKILL_DIR` として扱うこと。以降のすべてのコマンドはこの `$SKILL_DIR` を起点にする
（ユーザーや環境によってパスが変わるため、固定の絶対パスをハードコードしない）。

## 環境一覧

| 環境名 | ディレクトリ（`$DIR`） | 主なURL | 備考 |
|---|---|---|---|
| `juice-shop` | `$SKILL_DIR/compose/juice-shop` | http://localhost:3000 | 単一コンテナ |
| `webgoat` | `$SKILL_DIR/compose/webgoat` | http://localhost:8080/WebGoat, WebWolf: http://localhost:9090/WebWolf | 単一コンテナ、起動に数十秒かかる |
| `dvwa` | `$SKILL_DIR/compose/dvwa` | http://localhost:4280 | DVWA本体 + MariaDBの2コンテナ |
| `crapi` | `$SKILL_DIR/compose/crapi` | http://localhost:8888, MailHog: http://localhost:8025 | マイクロサービス構成（10コンテナ）。`--compatibility`オプションが必須 |

## アクションの意味

| アクション | 内容 | 実行するdocker composeコマンド |
|---|---|---|
| `build`（構築） | 最新イメージをpullしてから起動。初回セットアップや最新化に使う | `pull` → `up -d` |
| `up`（開始） | 既存のイメージ・状態のまま起動（停止中のコンテナも再開） | `up -d` |
| `stop`（停止） | コンテナを止める。データ（DBの中身など）は保持される | `stop` |
| `reset`（リセット） | コンテナとボリュームを完全に削除してから再作成。DBやユーザーデータも初期化される | `down -v` → `up -d` |
| `check`（状態確認） | 起動・停止操作は行わず、現在「稼働中」か「停止中」かだけを表示する | `ps --status running -q` |

環境名に `all` を指定した場合は、上記4環境（juice-shop / webgoat / dvwa / crapi）それぞれに対して同じアクションを順番に実行する。
引数が1語だけ（環境名が省略されアクションのみ）の場合も `all` を指定したものとして扱う（例: `check` だけの呼び出しは `all check` と同じ）。

## 実行方法

**重要**: シェルの`cd`で作業ディレクトリを移動すると、その状態が次のコマンドにも引き継がれてしまうことがある。
必ず `--project-directory` と `-f` に絶対パス（`$DIR`）を指定し、シェルの`cd`状態に依存しない形で実行すること。

環境ごとのベースディレクトリ（`$DIR`）:

```
juice-shop : $SKILL_DIR/compose/juice-shop
webgoat    : $SKILL_DIR/compose/webgoat
dvwa       : $SKILL_DIR/compose/dvwa
crapi      : $SKILL_DIR/compose/crapi
```

### build（構築）

```bash
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" pull
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" [--compatibility] up -d
```

### up（開始）

```bash
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" [--compatibility] up -d
```

### stop（停止）

```bash
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" stop
```

### reset（リセット）

```bash
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" down -v
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" [--compatibility] up -d
```

`--compatibility` は **crapiのみ** 必須（`deploy.resources`のCPU/メモリ制限を有効にするため）。他の環境では付けない。

**dvwaに対して`build`または`reset`を実行した場合は、コマンド実行後の応答で必ず毎回、以下の案内をユーザーに提示すること（省略しない）**:

> DVWAはDBが未初期化の状態です。ブラウザで http://localhost:4280/setup.php を開き、「Create / Reset Database」ボタンを押してください。完了後、`admin` / `password` でログインできます。

### check（状態確認）

起動・停止操作は一切行わず、コンテナが1つでも動いているかどうかだけを判定して表示する。

```bash
RUNNING=$(docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" ps --status running -q | wc -l)
if [ "$RUNNING" -gt 0 ]; then
  echo "<環境名>: 稼働中"
else
  echo "<環境名>: 停止中"
fi
```

`all check` の場合は4環境それぞれに対して上記を実行し、一覧で表示する（例: `juice-shop: 停止中` `webgoat: 稼働中` ...）。

## 環境ごとの注意点

- **dvwa**: `build`/`reset`後は毎回ブラウザで `http://localhost:4280/setup.php` にアクセスし「Create / Reset Database」ボタンを押してDBを初期化する必要がある（CSRFトークンが必要なためcurlでの自動化はしない）。ログインは `admin` / `password`。この案内は上記「reset（リセット）」節にある通り、実行結果として毎回ユーザーに提示すること。
- **crapi**: サービス数が多いため `pull` に時間がかかる。全サービスがhealthyになるまで1〜2分程度かかることがある。起動後は `docker compose ... ps` で全サービスが `healthy` になっているか確認するとよい。`compose/crapi/keys/jwks.json` はcrapi-identityサービスが必要とするJWT鍵で、OWASP公式リポジトリのデモ用鍵をそのまま同梱している（本番用途ではない学習環境専用）。
- **webgoat**: 起動直後は `docker ps` 上で `unhealthy` と表示されることがあるが、Javaアプリの初期化中なだけで問題ないことが多い。少し待って再確認する。

## 実行後の確認

各環境ともHTTPアクセスで起動確認する。

```bash
curl -s -o /dev/null -w "HTTP Status: %{http_code}\n" <上記表のURL>
```

## 安全上の注意

これらはすべて意図的に脆弱性を仕込んだ学習用アプリケーション。
各`docker-compose.yml`は`127.0.0.1`（ローカルホスト）にのみポートをバインドする設定になっている。
インターネットに公開されたネットワークやクラウド上のインスタンスで動かさないこと。

## 参考（各環境の公式情報）

- Juice Shop: https://github.com/juice-shop/juice-shop / https://hub.docker.com/r/bkimminich/juice-shop
- WebGoat: https://github.com/WebGoat/WebGoat / https://hub.docker.com/r/webgoat/webgoat
- DVWA: https://github.com/digininja/DVWA / https://github.com/digininja/DVWA/pkgs/container/dvwa
- OWASP crAPI: https://github.com/OWASP/crAPI / https://github.com/OWASP/crAPI/blob/main/docs/setup.md

## 対象外

- bWAPPは扱わない（メンテナンスが止まっているため）。
