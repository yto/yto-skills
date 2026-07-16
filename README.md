# yto-skills

Claude Code 用のスキル集。ハンズオンの参加者にプラグイン・マーケットプレイスとして配布するためのリポジトリ。

各スキルは `skills/<name>/SKILL.md` に自己完結で入っている。`marketplace.json` がそれらをプラグインとして束ねる。

## 収録スキル

| スキル | 内容 |
| --- | --- |
| [uuid-key-auth](skills/uuid-key-auth/) | UUIDシークレットキー方式のアカウント管理（パスワードレス・メール不要のログインキー認証）のベストプラクティス集 |

## 入れ方

### Claude デスクトップアプリ

1. **設定** を開く
2. 左メニューの **プラグイン**
3. 右上の **追加** → **マーケットプレイスを追加**
4. **リポジトリから追加** を選ぶ
5. `yto/yto-skills` を入力して同期
6. 追加されたマーケットプレイスから、使いたいプラグイン（例: `uuid-key-auth`）を有効化する

> 「スキル」（設定の別項目）とは別レイヤー。プラグインとして配るこのリポは **プラグイン** から追加する。

### ターミナル（Claude Code CLI）

```
/plugin marketplace add yto/yto-skills
/plugin install uuid-key-auth@yto-skills
```

更新は `/plugin marketplace update yto-skills`。

## スキルを追加する

1. `skills/<新しい名前>/SKILL.md` を作る（雛形は [`template/SKILL.md`](template/SKILL.md)）
2. `marketplace.json` の `plugins` に、そのスキルを含むプラグインを1つ追加（または既存プラグインの `skills` 配列に `./skills/<新しい名前>` を足す）

スキルにはスクリプトや参照ドキュメントを同梱してよい（`skills/<名前>/scripts/` など、相対パスで参照する）。

## ライセンス

MIT License（[LICENSE](LICENSE)）。
