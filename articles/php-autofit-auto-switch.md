---
title: "composer.json に合わせて PHP バージョンを自動切り替えする"
emoji: "🐘"
type: "tech"
topics: ["PHP", "Homebrew", "zsh", "composer", "macOS"]
published: true
---

Docker は使わない(使ってるけどリードタイムがしんどい故に一部分だけローカルのPHPを使うみたいなユースケースもある)し、mise の `.mise.toml` をリポジトリにコミットするのもちょっと・・
みたいなことがよくあるので**php-autofit** という zsh スクリプトを作りました。

## php-autofit を作った

https://github.com/PokoTechDev/php-autofit

**composer.json の `require.php` を読んで、Homebrew の PHP バージョンを自動で切り替える** zsh スクリプトです。

### インストール

```sh
curl -fsSL https://raw.githubusercontent.com/PokoTechDev/php-autofit/main/install.sh | sh
source ~/.zshrc
```

### できること

- `cd` でプロジェクトに入ると自動切り替え
- IDE からターミナルを開いても自動切り替え
- `git checkout` でブランチを変えても自動切り替え
- 必要な PHP バージョンが未インストールなら `brew install` まで自動実行

## 何をやっているのか

### 1. precmd フックで毎プロンプト前にチェック

zsh には `precmd` という、プロンプトが表示される直前に実行されるフックがあります。php-autofit はこれを使って、ディレクトリやブランチの変化を検知しています。

```zsh
autoload -Uz add-zsh-hook
add-zsh-hook precmd _auto_phpuse
```

`chpwd`（ディレクトリ変更時フック）ではなく `precmd` を使っているのは、IDE のターミナルを新しく開いた場合など、`cd` を伴わないケースにも対応したいためです。

### 2. 無駄な発火を防ぐ

毎回 `composer.json` を読むのは無駄なので、前回の状態をキャッシュしています。

```zsh
_phpuse_last_git_root=""
_phpuse_last_branch=""
_phpuse_last_dir=""
```

**git リポジトリ内**では、git root とブランチ名の組み合わせが変わったときだけ発火します。サブディレクトリ間の移動（`cd src/` → `cd tests/`）では何も起きません。

**git リポジトリ外**では、単純にディレクトリの変化で発火します。

```zsh
# git リポジトリ内: root + branch の変化で発火
if [ "$git_root" != "$_phpuse_last_git_root" ] || \
   [ "$current_branch" != "$_phpuse_last_branch" ]; then
  need_check=1
fi
```

### 3. composer.json からバージョンを読む

`jq` があれば使い、なければ `grep` でフォールバックします。

```zsh
# jq がある場合
jq -r '.require.php // empty' "$file" | grep -oE '[0-9]+\.[0-9]+' | head -1

# grep フォールバック
grep '"php"[[:space:]]*:' "$file" | head -1 | grep -oE '[0-9]+\.[0-9]+' | head -1
```

`"^8.1"` でも `">=8.1"` でも `"~8.1.0"` でも、メジャー.マイナー（`8.1`）の部分を抽出します。

### 4. Homebrew で切り替え

Homebrew では `php`（最新版）と `php@8.1` のように、バージョン付きの formula が別々にインストールされています。php-autofit は現在リンクされている PHP を `brew unlink` してから、対象バージョンを `brew link` します。

```zsh
# 既存の PHP を全部 unlink
echo "$installed" | while IFS= read -r _pkg; do
  [ -n "$_pkg" ] && brew unlink "$_pkg" >/dev/null 2>&1
done

# 対象バージョンを link
brew link "$pkg" --force --overwrite
```

未インストールのバージョンが指定されていた場合は、`brew install` を自動で実行します。

### 5. detached HEAD にも対応

`git checkout <commit-sha>` で detached HEAD になった場合も、コミット SHA の変化として検知します。

```zsh
current_branch=$(git symbolic-ref --short HEAD 2>/dev/null \
  || git rev-parse --short HEAD 2>/dev/null)
```

`git symbolic-ref` が失敗する（= detached HEAD）場合は、`git rev-parse --short HEAD` で短縮 SHA を取得します。

## 使い方

### 自動切り替え（デフォルト動作）

```sh
# PHP 8.3 のプロジェクトに移動
$ cd ~/projects/laravel-app
→ PHP 8.3 に切り替えます (composer.json)
✓ PHP 8.3.15

# PHP 8.1 のプロジェクトに移動
$ cd ~/projects/legacy-app
→ PHP 8.1 に切り替えます (composer.json)
✓ PHP 8.1.31

# ブランチ切り替え（composer.json の PHP バージョンが違う場合）
$ git checkout feature/php84-upgrade
→ PHP 8.4 に切り替えます (composer.json)
✓ PHP 8.4.5
```

### 手動切り替え

```sh
$ phpuse 8.1
✓ PHP 8.1.31

$ phpuse-composer
→ PHP 8.3 に切り替えます (composer.json)
✓ PHP 8.3.15
```

## 動作要件

- macOS
- Homebrew
- zsh
- `jq`（オプション。なければ `grep` で代替）

## まとめ

php-autofit は「Homebrew で PHP を管理しているなら、あとは composer.json に書いてあるバージョンに勝手に合わせてくれ」というシンプルな発想のツールです。

phpenv や mise のような大きな仕組みは不要で、zsh スクリプト1つで動きます。

https://github.com/PokoTechDev/php-autofit
