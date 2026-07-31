# security-lab

[Claude Code](https://claude.com/claude-code) 用のSkill。DockerでOWASP Juice Shop / WebGoat / DVWA / OWASP crAPI の
脆弱性学習環境をローカルに構築・起動・停止・リセット・状態確認できる。

docker-compose定義をあらかじめファイルとして同梱するのではなく、`setup`アクションでその場生成（または各プロジェクトの
公式リポジトリから直接取得）するため、Skill本体は `SKILL.md` 1枚だけで完結する。他のプロジェクトのファイルには
一切依存しない。必要なのは Docker（と `docker compose`）、そしてイメージ・compose定義取得用のインターネット接続だけ。

## インストール

`security-lab/` ディレクトリを、Claude Codeの設定ディレクトリにコピーする。

- 全プロジェクトで使いたい場合: `~/.claude/skills/security-lab/`
- 特定のプロジェクトだけで使いたい場合: `<プロジェクト>/.claude/skills/security-lab/`

```bash
git clone https://github.com/fuchiuebusi-lab/securiy-lab-claude-skill.git
cp -r securiy-lab-claude-skill/security-lab ~/.claude/skills/security-lab
```

Claude Codeを再起動（または新しいセッションを開始）すると、Skillとして認識される。

## 使い方

Claude Codeに自然文で依頼するか、`/security-lab <環境名> <アクション>` の形式で呼び出す。

```
/security-lab juice-shop build   # compose定義を生成してJuice Shopを構築・起動
/security-lab dvwa reset         # DVWAをリセット
/security-lab all check          # 全環境の稼働状況を確認
/security-lab check              # 環境名省略時は all 扱い
```

| 環境名 | URL |
|---|---|
| `juice-shop` | http://localhost:3000 |
| `webgoat` | http://localhost:8080/WebGoat（WebWolf: http://localhost:9090/WebWolf） |
| `dvwa` | http://localhost:4280（初回は`/setup.php`でDB初期化が必要。`admin`/`password`） |
| `crapi` | http://localhost:8888（MailHog: http://localhost:8025） |

アクション一覧: `setup`（初期化・compose生成） / `build`（構築） / `up`（開始） / `stop`（停止） / `reset`（リセット） / `check`（状態確認）

詳細な動作仕様は [`security-lab/SKILL.md`](./security-lab/SKILL.md) を参照。

## 注意事項

これらはすべて意図的に脆弱性を仕込んだセキュリティ学習用アプリケーション。
各環境は`127.0.0.1`にのみバインドされる設定になっているが、インターネットに公開されたネットワークや
クラウド上のインスタンスでは絶対に動かさないこと。教育・自己学習目的以外での使用は想定していない。

## ライセンス・出典

各アプリケーション本体（Juice Shop / WebGoat / DVWA / crAPI）は、それぞれの公式プロジェクトのライセンスに従う。
本リポジトリが提供するのはClaude Code向けのSkill定義（docker-compose設定と操作手順）のみ。
