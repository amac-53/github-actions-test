# GitHub Actions について

## GitHub Actions を理解する
- workflowsは，1つ以上のジョブを実行する自動化プロセスで，リポジトリ内のイベント，手動，定義したスケジュールによっても実行可能
- イベントは，workflow を実行するリポジトリ内の特定のアクティビティ（pull request が作成sされたとき，issue が開かれた特など）
- ジョブは，同じランナーで実行される workflow 内の一連のステップで，各ステップは，シェルスクリプトやアクションのいずれかで，同じランナーで実行されるのでデータ共有などが可能（**基本的に並列に実行されるが，依存関係がある場合は待ってくれる（？）**）
- アクションは，GitHub Actions 用のカスタムアプリケーションであり，複雑で頻繁に繰り返されるタスクを実行する（Git リポジトリの pull, ビルド環境に適したツールチェーンの設定 etc.）（**独自のアクションも GitHub Marketplace で見つけることも可能**）
- ランナーは，ワークフローがトリガーされると実行されるサーバーで**１つのジョブを実行可能．**

## アクションの検索とカスタマイズ
- アクションはワークフローの構成要素であり，第三者によって作成されたもの，あるいは独自で作ったものの両方を使用できる
- アクションを定義できる場所は
  - ワークフローファイルと同じリポジトリ
  - すべてのパブリックリポジトリ
  - Docker Hub で公開された Docker コンテナイメージ
- GitHub Marketplaceは GitHub コミュニティで作成されたアクションを一元管理する場所でフィルタ処理なども可能
### ワークフローエディタで Marketplace アクションを参照する
  - 右上のペンのボタンから Marketplace 検索できて，Star数，GitHub がパートナーとして認定したかなどを確認できる

### ワークフローにアクションを追加する
- GitHub Marketplace から使いたいアクションにアクセスし，クリックすることで情報を参照すると同時に，[インストール]のコピーアイコンをクリック
- ワークフローファイルがアクションを使用するのと同じリポジトリで定義されている場合は，`{owner}/{repo}@{ref}`または`./path/to/dir`で参照する
```
|-- hello-world (repository)
|   |__ .github
|       └── workflows
|           └── my-first-workflow.yml
|       └── actions
|           |__ hello-world-action
|               └── action.yml
```
```yaml
jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      # This step checks out a copy of your repository.
      - uses: actions/checkout@v4
      # 以下の感じ
      - uses: ./.github/actions/hello-world-action
```
- Docker コンテナイメージで定義されている場合は，`docker://{image}:{tag}`で参照する必要がある
```yaml
jobs:
  my_first_job:
    steps:
      - name: My first step
        uses: docker://alpine:3.8
```
### カスタムアクションにリリース管理を使用する
- アクションの作者はタグ，ブランチ，SHA 値でリリースを管理できる
- 使用するアクションの自動更新を受け入れるか否かに応じて指定すべき（**特にサードパーティのものは SHA 値を使うべき**）
  - タグの例
  ```yaml
  steps:
    - uses: actions/javascript-action@v1.0.1
  ```
  - （コミットの）SHAの例
    - 不変であるため，タグやブランチよりも信頼性が高くなるが，重要なバグ修正やセキュリティ更新によるアクションの自動更新も受信しないというデメリットもある
　```yaml
  steps:
  - uses: actions/javascript-action@a824008085750b8e136effc585c3cd6082bd575f
  ```
  - ブランチ（大きな更新にやられる可能性がある）
  ```yaml
  steps:
  - uses: actions/javascript-action@main
  ```
### アクションで入力と出力を使用する
- 入力と出力を指定できるアクションでは，ファイルへのパスやラベル名，データなどを指定する
- 入出力は，**リポジトリルートディレクトリの action.yaml(action.yml)**で確認できる
```yaml
name: "Example"
description: "Receives file and generates output"
inputs:
  file-path: # id of input
    description: "Path to test script"
    required: true
    default: "test-file.js"
outputs:
  results-file: # id of output
    description: "Path to results file"
```

## GitHub Actions の重要な機能
### ワークフローで変数を使用する
- ワークフロー実行ごとのデフォルトの環境変数が含まれているが，新たに定義することもできる（以下の例は，POSTGRES_HOST, POSTGRES_PORTを定義しており，node client.js で使用できる）
```yaml
jobs:
  example-job:
      steps:
        - name: Connect to PostgreSQL
          run: node client.js
          env:
            POSTGRES_HOST: postgres
            POSTGRES_PORT: 5432
```
### ワークフローにスクリプトを追加する
- run キーワードでスクリプトとシェルコマンドを実行することで，ランナーを実行できる
```yaml
jobs:
  example-job:
    steps:
      - run: npm install -g bats
```
- シェルスクリプトはリポジトリに保存することで，次のように実行できる
```yaml
jobs:
  example-job:
    steps:
      - name: Run build script
        run: ./.github/scripts/build.sh
        shell: bash
```

### ジョブ間でデータを共有する
- 同じワークフロー内の別 job とファイル共有などができる．ある job 内で作成されたファイルには，**実行内のすべてのアクションとワークフローが書き込みの権限を持つ**
- 以下では，ファイル作成を行った後にアップロードする
```yaml
jobs:
  example-job:
    name: Save output
    steps:
      - shell: bash
        run: |
          expr 1 + 1 > output.log
      - name: Upload output file
        uses: actions/upload-artifact@v3
        with:
          name: output-log-file
          path: output.log
```
- 以下の actions/download-artifact により，**別のワークフロー**の実行から成果物をダウンロードできる
```yaml
jobs:
  example-job:
    steps:
      - name: Download a single artifact
        uses: actions/download-artifact@v3
        with:
          name: output-log-file
```
## 式

## コンテキスト

## 変数

## Using starter workflows

## 使用制限、支払い、管理