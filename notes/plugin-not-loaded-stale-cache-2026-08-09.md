# プラグインを入れたのにスキルが出てこない

- **作成日**: 2026-08-09（2026-08-11 に「原因B」を追記）
- **目的**: このリポのプラグインをインストール・有効化しているのに Claude Code のスキル一覧に出てこない、という症状に遭遇したときのために、原因と切り分け手順を残す。原因は今のところ2パターン見つかっている。どちらもリポジトリ側ではなくローカルの `~/.claude/plugins/` 以下が壊れているケースで、`claude plugin validate` は「正常」と答えるので気づきにくい。

## まず切り分ける

`claude plugin list` の表示で、どちらの原因かが分かれる。

| `claude plugin list` の表示 | 原因 |
| --- | --- |
| `✔ enabled` と出ているのにスキルが使えない | [原因A: ローカルのキャッシュが古い形式のまま](#原因a-ローカルのキャッシュが古い形式のまま2026-08-09) |
| `✘ failed to load` ＋ `Plugin <name> not found in marketplace <marketplace>` | [原因B: マーケットプレイスの clone の作業ツリーが汚れている](#原因b-マーケットプレイスの-clone-の作業ツリーが汚れている2026-08-11) |

同じマーケットプレイスの中でも、あるプラグインは正常・別のプラグインだけ壊れている、という混ざった状態になりうる。1つ動いているからといって他も動いているとは限らない。

## 原因A: ローカルのキャッシュが古い形式のまま（2026-08-09）

### 症状

`yto-skills` の3プラグイン（`uuid-key-auth` / `kukude-webapp-safety` / `in-repo-notes`）をインストール済み・有効化済みなのに、セッションのスキル一覧に1つも出てこない。一方で、同じ書き方をしている別リポのプラグインは正常に出ていた。

CLI から見るかぎり異常が無いのがやっかいだった。以下はすべて「正常」と答える。

- `claude plugin list` → 3つとも `✔ enabled`
- `claude plugin validate <repo>` → `✔ Validation passed`
- `claude plugin details <plugin>@yto-skills` → `Skills (1)` を正しく表示
- `~/.claude/settings.json` の `enabledPlugins` → 3つとも `true`
- インストールマニフェスト（`~/.claude/plugins/.install-manifests/`）の SHA-256 照合 → 全ファイル一致

### 原因

**ローカルのプラグインキャッシュが古い形式のまま残っていた。**

`~/.claude/plugins/cache/yto-skills/<plugin>/<sha>/` の中身を、動いているプラグインと比べると差が出た。

| | `.claude-plugin/` の中身 | スキルのロード |
| --- | --- | --- |
| 古いキャッシュ | `marketplace.json` + `plugin.json` | されない |
| 再インストール後 | `marketplace.json` のみ | される |

Claude Code のバージョンが上がる過程でキャッシュの形式が変わり、古い形式のまま残っていたものが読み込まれなくなっていた、と考えられる。リポジトリ側（`marketplace.json`、`skills/*/SKILL.md`）には問題が無かった。

### 直し方

該当プラグインを入れ直す。**リポジトリ側は触らない。**

```sh
claude plugin uninstall <plugin>@yto-skills
claude plugin marketplace update yto-skills
claude plugin install <plugin>@yto-skills
```

反映は次に起動するセッションから。今開いているセッションには反映されない。

### 切り分けの記録（否定された仮説）

同じ症状で悩んだとき、以下は原因ではないと分かっているので飛ばしてよい。検証は手元に使い捨てのマーケットプレースを作って実際に走らせた。

| 疑ったこと | 結果 |
| --- | --- |
| 1つのリポに複数プラグインを置き、全部 `source: "./"` にしているのが悪い | **無関係**。1リポ2プラグイン・両方 `source: "./"`・`skills` で絞る構成を作ったが、正常に両方ロードされた |
| リポ直下の `template/SKILL.md` の `name` がディレクトリ名と一致していないのが悪い | **無関係**。同じものを検証用リポに置いても影響なし |
| 個人スキル `~/.claude/skills/<name>/` と名前が衝突している | **無関係**。個人スキルを退避してもプラグイン側は出てこなかった |
| キャッシュ内の `plugin.json` の `skills` の書き方が悪い | **無関係**。`./skills` に書き換えても直らない。ファイルの中身ではなくキャッシュ全体の世代の問題 |

## 原因B: マーケットプレイスの clone の作業ツリーが汚れている（2026-08-11）

### 症状

`claude plugin list` に、該当プラグインだけエラーが出る。原因Aと違い、CLI がはっきり異常を報告する。

```
❯ in-repo-notes@yto-skills
  Version: 0de6491dbeb1
  Scope: user
  Status: ✘ failed to load
  Error: Plugin in-repo-notes not found in marketplace yto-skills
```

このときも `~/.claude/settings.json` の `enabledPlugins` と `~/.claude/plugins/installed_plugins.json` には正しくエントリがあり、設定ファイルを見ても異常は見つからない。同じマーケットプレイスの他の2つは `✔ enabled` で正常に動いていた。

### 原因

`~/.claude/plugins/marketplaces/<marketplace>/` は、マーケットプレイスのリポジトリの git clone。この clone の**作業ツリー**に未コミットの変更が残っていて、`marketplace.json` からそのプラグインのエントリが消え、`skills/<name>/SKILL.md` も削除された状態になっていた。

```sh
$ git -C ~/.claude/plugins/marketplaces/yto-skills status -s
 M .claude-plugin/marketplace.json
 D skills/in-repo-notes/SKILL.md
```

`git log` は最新のコミットを指しているので、コミットだけ見ても気づけない。GitHub 側のリポジトリは正常だった。

**なぜこの clone の作業ツリーが汚れたのかは未確認。** 何が書き換えたのかは特定できていない。

### やっかいな点

`claude plugin marketplace update <marketplace>` を実行しても直らない。git の HEAD は最新に進むが、**作業ツリーの未コミットの変更はそのまま残る**ので、`marketplace.json` はプラグインが欠けたままになる。「更新したのに直らない」でここに嵌まった。

`marketplace update` して直らなかったら、HEAD ではなく作業ツリーを疑う。

### 直し方

clone の変更を捨ててから、更新し直す。**リポジトリ側は触らない。**

```sh
# 1. 何が変わっているか先に見る
git -C ~/.claude/plugins/marketplaces/<marketplace> status -s
git -C ~/.claude/plugins/marketplaces/<marketplace> diff

# 2. 変更を捨てる
git -C ~/.claude/plugins/marketplaces/<marketplace> checkout -- .

# 3. 更新し直す
claude plugin marketplace update <marketplace>
claude plugin update <plugin>@<marketplace>
```

`checkout -- .` は変更を捨てる操作なので、必ず `status` と `diff` で中身を確認してから実行する。ここは Claude Code が管理する作業用ディレクトリなので、通常は手で編集したものは置かれていないはず。

反映は次に起動するセッションから。今開いているセッションには反映されない。

## 症状が出ているか確かめる方法

セッションを開いて一覧を目で見るより、非対話で聞くほうが速い。

```sh
claude -p "使えるスキル名を1行1つで全部列挙して。説明不要。"
```

プラグイン由来のスキルは `<plugin>:<skill>` の形式で出る。接頭辞が付いていないものは `~/.claude/skills/` に置いた個人スキル。

## 古いキャッシュの掃除

`~/.claude/plugins/cache/<marketplace>/<plugin>/<sha>/` はバージョンごとに溜まる。ただし**稼働中のセッションが参照している版を消すとそのセッションが壊れる**ので、`.in_use/` に置かれた PID が生きているかを見てから消す。

```sh
# 参照している PID が生きているか確認してから消す
find ~/.claude/plugins/cache -path "*/.in_use/*" -type f -exec basename {} \; | sort -u |
  while read pid; do ps -p "$pid" >/dev/null 2>&1 && echo "$pid ALIVE" || echo "$pid dead"; done
```

`.orphaned_at` というファイルが置かれているディレクトリは Claude Code 自身が孤児と印を付けたものなので消してよい。

## 参照

- 出典は無し。すべて手元（macOS / Claude Code 2.1.226、原因Bは 2.1.227）での実験と、`~/.claude/plugins/` 以下の実物の観察による。挙動は Claude Code のバージョンで変わりうる。
