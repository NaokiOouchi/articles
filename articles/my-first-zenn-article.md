---
title: "複数リポジトリで copilot-instructions.md を一元管理する"
emoji: "🔄"
type: "tech"
topics: ["GitHub", "GitHubActions", "Copilot", "運用"]
published: true
---

## はじめに

GitHub Copilot を使い始めてから `copilot-instructions.md` に**コーディング規約やセキュリティルール**を書くようになりました。ただ、フロントエンド・バックエンド・管理画面と複数リポジトリに分かれている構成だと、最初からちょっと面倒でした。

リポジトリが増えるたびに同じファイルを手でコピーして回るのは地味に手間ですし、内容を変えたいときに全部のリポジトリに反映するのも忘れそうで嫌だなと。

というわけで、**共有リポジトリに push するだけで各リポジトリへ自動でPRを作成する仕組み**を作ることにしました。

使うのは [cross-repo-sync-action](https://github.com/NaokiOouchi/cross-repo-sync-action) です。

---

## 解決の流れ

考え方はシンプルで、**共通ルールを1つのリポジトリに集約し、push をトリガーに各リポジトリへPRが飛ぶ**ようにします。

```
[共有リポ: org/copilot-shared]
   └── sync-config.yml              ← ルートに置く（デフォルトパス）
   └── .github/copilot-instructions.md  ← ここだけ編集する
   └── .github/workflows/sync.yml

        ↓ main への push で自動実行

[org/frontend]  ← PR が自動作成される
[org/api]       ← PR が自動作成される
[org/admin]     ← PR が自動作成される
```

各リポジトリのメンバーはPRをレビュー・マージするだけです。
共通ルールの「正」は常に1箇所に集約されるので、更新漏れが起きません。

---

## セットアップ

### 1. 共有リポジトリを用意する

`org/copilot-shared` という名前（任意）で空のリポジトリを作成し、以下の構成を用意します。

```
copilot-shared/
├── sync-config.yml               ← ルートに置く（デフォルトパス）
├── .github/
│   ├── copilot-instructions.md
│   └── workflows/
│       └── sync.yml
```

`copilot-instructions.md` には全リポジトリで共通にしたいルールを書いておきます。

```markdown
# Copilot Instructions

## 全般
- コードの変更は最小限にとどめ、関係ない箇所には触れないこと
- 既存の命名規則・フォーマットに必ず合わせること
- コメントは書かない。必要なら既存コメントを踏襲する

## PHP / Laravel
- Eloquent は使わない。クエリビルダか生SQL を使うこと
- バリデーションは FormRequest に書く
- N+1 に気をつけること。with() を忘れずに

## セキュリティ
- 機密情報（APIキー等）をコードにハードコードしない
- ユーザー入力は必ずバリデーションを通してからビジネスロジックに渡す
```

### 2. sync-config.yml を作成する

リポルートに `sync-config.yml`（デフォルトのパス）を作成して、「どのファイルを」「どのリポジトリに」同期するかを定義します。

```yaml
sync:
  - src: .github/copilot-instructions.md
    dest: .github/copilot-instructions.md
    repos:
      - org/frontend
      - org/api
      - org/admin
```

`src` は共有リポ内のパス、`dest` は各対象リポジトリ内の配置先パスです。`repos` は同期先を `sync` エントリごとに指定します。

### 3. ワークフローを設定する

`.github/workflows/sync.yml` を作成します。

```yaml
name: Sync copilot-instructions

on:
  push:
    branches:
      - main
    paths:
      - '.github/copilot-instructions.md'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Sync to repositories
        uses: NaokiOouchi/cross-repo-sync-action@v1
        with:
          token: ${{ secrets.SYNC_PAT }}
          # sync-config.yml をルート以外に置く場合は config-path を指定
          # config-path: .github/sync-config.yml
```

`paths` フィルターを入れておくと無駄なPR作成を避けられます。`sync-config.yml` 自体の更新（同期先リポジトリの追加など）でもワークフローを動かしたい場合は、両方指定しておきましょう。

```yaml
    paths:
      - '.github/copilot-instructions.md'
      - 'sync-config.yml'
```

### 4. PAT を設定する

対象リポジトリへのPR作成には、各リポジトリへのアクセス権を持つPATが必要です。
Classic PAT の `repo` スコープは権限が広すぎるため、**Fine-grained PAT** の使用を推奨します。

1. **Settings > Developer settings > Personal access tokens > Fine-grained tokens** でPATを発行する
   - 対象リポジトリを選択
   - 必要な権限: **Contents: Read and write** / **Pull requests: Read and write**
2. 共有リポジトリの **Settings > Secrets and variables > Actions** に `SYNC_PAT` という名前で登録する

---

## 実際に動かす

設定が完了したら、`copilot-instructions.md` を編集して push します。

```bash
git add .github/copilot-instructions.md
git commit -m "fix: XSS対策ルールを追加"
git push origin main
```

Actionsタブを開くと `Sync copilot-instructions` が起動しているのが確認できます。

![](/images/sync-workflow.png)

数十秒後、対象リポジトリそれぞれにPRが自動作成されます。

![](/images/sync-pr-created.png)

PRには差分がそのまま表示されるので、通常のレビューと同じ感覚でマージするだけです。

---

## 応用

### lint設定・Cursorルールも同期する

`sync-config.yml` にファイルを追加するだけで他のファイルも同期対象にできます。

```yaml
sync:
  - src: .github/copilot-instructions.md
    dest: .github/copilot-instructions.md
    repos:
      - org/frontend
      - org/api
      - org/admin
  - src: .cursor/rules/common.mdc
    dest: .cursor/rules/common.mdc
    repos:
      - org/frontend
      - org/api
  - src: .eslintrc.shared.json
    dest: .eslintrc.shared.json
    repos:
      - org/frontend
```

Cursor を使っているリポジトリにも同じルールを配れます。

### ディレクトリごと同期する

ファイル単位だけでなく、ディレクトリ単位での同期もできます。

```yaml
sync:
  - src: .github/instructions/
    dest: .github/instructions/
    repos:
      - org/frontend
      - org/api
```

ロール別に instructions ファイルを分けて管理している場合に便利です。

### delete オプションで不要ファイルを自動削除する

`delete: true` は**ディレクトリ同期専用**のオプションで、ソース側に存在しないファイルをターゲットから削除してくれます。
共有リポのディレクトリ構成を常に各リポジトリへ反映したい場合に使います。

```yaml
sync:
  - src: configs/
    dest: .github/configs/
    delete: true
    repos:
      - org/frontend
      - org/api
```

これで `configs/` から削除したファイルが各リポジトリに残り続けるという状況を防げます。
単一ファイルへの `delete: true` は意味を持たないので注意してください。

### GitHub App 認証を使う

PATは発行した個人に紐づくため、その人がチームを離れると失効します。
本番運用では GitHub App のインストールトークンを使うほうが安全です。

```yaml
- name: Sync to repositories
  uses: NaokiOouchi/cross-repo-sync-action@v1
  with:
    token: ${{ steps.generate_token.outputs.token }}
```

GitHub App を Organization に紐づけておけば個人依存を排除できます。

---

## 終わりに

`copilot-instructions.md` を一元管理することで、どのリポジトリでも同じルールでCopilotを使えるようになります。チームで「このルール、全リポに入っているか」を確認する手間もなくなるのでおすすめです。
複数リポジトリで開発しているチームにはぜひ試してみてください。
