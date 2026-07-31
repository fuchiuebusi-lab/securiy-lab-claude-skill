---
name: security-lab
description: >
  ローカルのセキュリティ学習環境（OWASP Juice Shop, WebGoat, DVWA, OWASP crAPI）をDockerで
  構築・開始・停止・リセット・状態確認するときに使用する。
  「Juice Shopを起動して」「WebGoatを構築して」「DVWAを止めて」「crAPIをリセットして」
  「セキュリティ学習環境を立ち上げて」「全部止めて」「今何が動いてる？」のような依頼で使う。
  引数は `<環境名> <アクション>` の形式（例: `juice-shop up`, `dvwa reset`, `all stop`, `all check`）。
  環境名: juice-shop / webgoat / dvwa / crapi（すべて可: all）。
  アクション: setup（初期化）/ build（構築）/ up（開始）/ stop（停止）/ reset（リセット）/ check（状態確認）。
---

# security-lab

このSkill単体で完結する、ローカルのセキュリティ学習環境（OWASP Juice Shop, WebGoat, DVWA, OWASP crAPI）を
Dockerで構築・操作するスキル。docker-compose定義をSkillの中に静的なファイルとして同梱するのではなく、
`setup`アクションでその場生成（または各プロジェクト公式リポジトリから取得）する方式にしているため、
Skill本体は `SKILL.md` 1枚だけで完結する。どのプロジェクトのどのマシンから使っても、
外部ファイルへの依存なしにそのまま動作する。Docker（と`docker compose`）とインターネット接続（イメージ・compose定義の取得用）
さえあれば、他に前提条件はない。

## Skillディレクトリ（`$SKILL_DIR`）の特定

このSkillが起動される際、システムメッセージに次のように表示される。

```
Base directory for this skill: <パス>
```

この `<パス>` を `$SKILL_DIR` として扱うこと。以降のすべてのコマンドはこの `$SKILL_DIR` を起点にする
（ユーザーや環境によってパスが変わるため、固定の絶対パスをハードコードしない）。
`setup`で生成するcompose定義一式は `$SKILL_DIR/compose/<環境名>/` に書き出す。

## 環境一覧

| 環境名 | ディレクトリ（`$DIR`） | 主なURL | 備考 |
|---|---|---|---|
| `juice-shop` | `$SKILL_DIR/compose/juice-shop` | http://localhost:3000 | 単一コンテナ |
| `webgoat` | `$SKILL_DIR/compose/webgoat` | http://localhost:8080/WebGoat, WebWolf: http://localhost:9090/WebWolf | 単一コンテナ、起動に数十秒かかる |
| `dvwa` | `$SKILL_DIR/compose/dvwa` | http://localhost:4280 | DVWA本体 + MariaDBの2コンテナ |
| `crapi` | `$SKILL_DIR/compose/crapi` | http://localhost:8888, MailHog: http://localhost:8025 | マイクロサービス構成（10コンテナ）。`--compatibility`オプションが必須 |

## アクションの意味

| アクション | 内容 |
|---|---|
| `setup`（初期化） | `$DIR`にdocker-compose定義一式を生成する（起動はしない）。既にファイルがあれば上書きする |
| `build`（構築） | `setup`を実行してから、最新イメージをpullして起動する。初回セットアップや最新化に使う |
| `up`（開始） | 既存の状態のまま起動（停止中のコンテナも再開）。`$DIR`にファイルが無ければ先に`setup`する |
| `stop`（停止） | コンテナを止める。データ（DBの中身など）は保持される |
| `reset`（リセット） | コンテナとボリュームを完全に削除してから再作成。DBやユーザーデータも初期化される |
| `check`（状態確認） | 起動・停止操作は行わず、現在「稼働中」か「停止中」かだけを表示する |

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

### setup（初期化）

`$DIR/docker-compose.yml` が存在するかに関わらず、常に最新の内容で書き出す。

#### juice-shop

```bash
mkdir -p "$DIR"
cat > "$DIR/docker-compose.yml" <<'EOF'
services:
  juice-shop:
    image: bkimminich/juice-shop:latest
    container_name: juice-shop
    ports:
      - "127.0.0.1:3000:3000"
    restart: unless-stopped
EOF
```

#### webgoat

```bash
mkdir -p "$DIR"
cat > "$DIR/docker-compose.yml" <<'EOF'
services:
  webgoat:
    image: webgoat/webgoat:latest
    container_name: webgoat
    environment:
      - TZ=Asia/Tokyo
    ports:
      - "127.0.0.1:8080:8080"
      - "127.0.0.1:9090:9090"
    restart: unless-stopped
EOF
```

#### dvwa

公式リポジトリの `compose.yml`（https://raw.githubusercontent.com/digininja/DVWA/master/compose.yml）をベースに、
ローカルにDVWAのソースを持たない前提で `build: .` / `pull_policy: always` を除去し、
ビルド済みイメージ（`ghcr.io/digininja/dvwa:latest`）を直接pullする形にしたもの。

```bash
mkdir -p "$DIR"
cat > "$DIR/docker-compose.yml" <<'EOF'
volumes:
  dvwa:

networks:
  dvwa:

services:
  dvwa:
    image: ghcr.io/digininja/dvwa:latest
    container_name: dvwa
    environment:
      - DB_SERVER=db
    depends_on:
      - db
    networks:
      - dvwa
    ports:
      - "127.0.0.1:4280:80"
    restart: unless-stopped

  db:
    image: docker.io/library/mariadb:10
    container_name: dvwa-db
    environment:
      - MYSQL_ROOT_PASSWORD=dvwa
      - MYSQL_DATABASE=dvwa
      - MYSQL_USER=dvwa
      - MYSQL_PASSWORD=p@ssw0rd
    volumes:
      - dvwa:/var/lib/mysql
    networks:
      - dvwa
    restart: unless-stopped
EOF
```

#### crapi

crAPIは公式リポジトリに完全な`docker-compose.yml`（改変不要）があるため、その場でダウンロードして使う。

```bash
mkdir -p "$DIR/keys"
curl -fsSL -o "$DIR/docker-compose.yml" https://raw.githubusercontent.com/OWASP/crAPI/main/deploy/docker/docker-compose.yml
curl -fsSL -o "$DIR/.env" https://raw.githubusercontent.com/OWASP/crAPI/main/deploy/docker/.env
curl -fsSL -o "$DIR/keys/jwks.json" https://raw.githubusercontent.com/OWASP/crAPI/main/deploy/docker/keys/jwks.json
```

`keys/jwks.json` はcrapi-identityサービスが必要とするJWT鍵。OWASP公式リポジトリのデモ用鍵で、学習環境専用（本番用途ではない）。

### build（構築）

```bash
# 上記のsetup手順を実行した後
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" pull
docker compose --project-directory "$DIR" -f "$DIR/docker-compose.yml" [--compatibility] up -d
```

### up（開始）

```bash
# "$DIR/docker-compose.yml" が存在しなければ、先に該当環境のsetup手順を実行する
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
まだ`setup`/`build`を一度も実行していない環境（`$DIR/docker-compose.yml`が存在しない）は「未構築」と表示する。

## 環境ごとの注意点

- **dvwa**: `build`/`reset`後は毎回ブラウザで `http://localhost:4280/setup.php` にアクセスし「Create / Reset Database」ボタンを押してDBを初期化する必要がある（CSRFトークンが必要なためcurlでの自動化はしない）。ログインは `admin` / `password`。この案内は上記「reset（リセット）」節にある通り、実行結果として毎回ユーザーに提示すること。
- **crapi**: サービス数が多いため `pull` に時間がかかる。全サービスがhealthyになるまで1〜2分程度かかることがある。起動後は `docker compose ... ps` で全サービスが `healthy` になっているか確認するとよい。
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
