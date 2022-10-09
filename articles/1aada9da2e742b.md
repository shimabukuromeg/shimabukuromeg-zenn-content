---
title: "Terraformを使ってインフラと触れ合いたいぞ"
emoji: "🌊"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["terraform"]
published: true
publication_name: "monicle"
---

# はじめに

Terraformで書かれたコードの読み方が全体的によくわからなかなかったので、調べたことを整理してまとめた記事です。（特にモジュールや全体的なコードの読み方）
[ファイルとディレクトリ構成](https://www.terraform.io/language/files)、[モジュール](https://www.terraform.io/language/modules)、[変数](https://www.terraform.io/language/values/variables) あたりのドキュメントをちゃんと読むと、モジュールの仕組みの理解が深まったので、整理してまとめました。
アプリケーションエンジニアが Terraform のコード読んでインフラ構成を理解できるようになるぞ！！！が目標です。

:::details 環境構築などの事前準備関連メモ

手元の環境（Mac）でterraformを使う準備

```bash
$ brew install tfenv
$ tfenv --version
$ tfenv list-remote
$ tfenv install 1.3.0
$ tfenv use 1.3.0
```


この記事ではGCPを使います。GCPのアカウト、プロジェクトの作成、Google Cloud SDKが必要なので用意する。

https://cloud.google.com/sdk/docs/downloads-interactive


:::


# 流れ

まずはterraformでインフラを作ってみるときの全体的な流れの確認

- アーキティクチャを設計する。
- 設計できたらTerraform化。最も依存されるコンポーネントから作る。
- ルートモジュールを作って、`.tf` 拡張子のファイルを作って、プロバイダーやリソースなどのConfigurationを定義して `terraform init`する。[terraform init](https://www.terraform.io/cli/commands/init) はTerraform 構成ファイルを含む作業ディレクトリを初期化してくれて、必要なプラグインや子モジュールのインストールをしてくれる。

- terraformのコードを書き終えたら、`terraform plan` を実行する。[terraform plan](https://www.terraform.io/cli/commands/plan) は実行計画が確認できる。

- 実行計画の結果、問題なかったら `terraform apply` 実行する。[terraform apply](https://www.terraform.io/cli/commands/apply) は実際にプラットフォームにリソースを作成することができる。

- リソースを消す場合は `terraform destroy` を実行する。[terraform destroy](https://www.terraform.io/cli/commands/destroy) は作ったもの全部消してくれる。

- `terraform init`、`terraform plan`、`terraform apply`、`terraform destroy` は、Configurationの設定とリソースのステートを考慮しながら、新規作成、変更、削除してくれて、基本的な操作としては、Configuration書く、コマンド叩く、一連の操作の繰り返し。
- 一連の操作のなかで意図しないリソースの変更や削除を防ぐ設定（[ライフサイクル](https://www.terraform.io/language/meta-arguments/lifecycle)） などもできる


# サンプルコード

仮想マシンインスタンスを作るだけのサンプルです。主にmodule、variablesの使い方を理解できるよう試してみました。

https://github.com/shimabukuromeg/sample-terraform


以下、Terraformのこのルールがわかったら、コード理解しやすくなったよ。と個人的に感じた部分です。上記サンプルコードをベースにまとめました。

# terraformのファイル、ディレクトリ構成

> Terraform モジュールは、ディレクトリ内の最上位の構成ファイルのみで構成されます。ネストされたディレクトリは完全に別のモジュールとして扱われ、構成に自動的に含まれません。

- terraformのコマンド（`$ terraform apply` など）を叩いたときに読まれるファイルは、基本的にディレクトリの最上位（terraformのコマンド叩いた階層。ルート モジュール）にあるファイルのみで、ネストされた下の階層のモジュールは自動的に読み込まれたりはしない。最上位のファイルで別階層のモジュールを呼び出してたら読み込まれる。
- モジュールには、ローカル ディレクトリ（同じファイル管理してる別の階層のモジュール）や外部公開されてるモジュールがあり、それらを呼び出すことができる。
- terraformはモジュール内の全てのファイルを評価する。ブロックごとに、ファイル分割するのは、メンテしやすくするためで、モジュールの動作には影響しない。

ドキュメントだよ👇

https://www.terraform.io/language/files

# terraform の モジュール呼び出し

- 呼び出し側のサンプルコード（ルートモジュール）

```hcl
# ルートモジュール
module "sample_instance" {
  # 呼び出すモジュールのパスを指定する
  source = "./module"

  # ルートモジュール側で子モジュールで使う変数の値を指定
  # 子モジュール側の変数の定義でデフォルト値が指定されてたら省略可能だけど、デフォルト値が指定されていなくて、ここでも指定されてなかったらエラーが出る
  project       = var.project
  service_name  = "test-service"
  environment   = "development"
}
```

- 呼び出されるモジュール（子モジュール）側のvariablesの定義例
- モジュールを別の階層から呼び出した際に、ここで定義してる変数に具体的な値を指定することができる（上記、呼び出し側のサンプルコードの例で書いた感じ）

```hcl
# 子モジュールの変数の定義
variable "project" {
  description = "A name of a GCP project"
  type        = string
  default     = null
}

variable "zone" {
  description = "A zone used in a compute instance"
  type        = string
  default     = "asia-northeast1-c"
  validation {
    condition     = contains(["asia-northeast1-a", "asia-northeast1-b", "asia-northeast1-c"], var.zone)
    error_message = "The zone must be in asia-northeast1 region."
  }
}

variable "service_name" {
  description = "A name of a service"
  type        = string
}

variable "environment" {
  description = "A name of an environment"
  type        = string
  default     = "development"

  validation {
    condition     = contains(["development", "staging", "production"], var.environment)
    error_message = "The environment must be development, staging, or production."
  }
}

variable "machine_type" {
  description = "value of machine_type"
  type        = string
  default     = "f1-micro"
}
```

ドキュメントだよ👇

https://www.terraform.io/language/modules

# 図でまとめてみた

図で整理整頓

![](https://storage.googleapis.com/zenn-user-upload/7632ce71687a-20221009.png)


# 参考になりそうな情報

### 標準モジュール構造

モジュールの構造について。ドキュメントの Standard Module Structure のページが参考になりそう。

```
$ tree complete-module/
.
├── README.md
├── main.tf
├── variables.tf
├── outputs.tf
├── ...
├── modules/
│   ├── nestedA/
│   │   ├── README.md
│   │   ├── variables.tf
│   │   ├── main.tf
│   │   ├── outputs.tf
│   ├── nestedB/
│   ├── .../
├── examples/
│   ├── exampleA/
│   │   ├── main.tf
│   ├── exampleB/
│   ├── .../
```

https://www.terraform.io/language/modules/develop/structure

### Terraform を使用するためのベストプラクティス

GCPで Terraform を使用するためのベストプラクティスのガイドが提供されてた。参考になりそう

https://cloud.google.com/docs/terraform/best-practices-for-terraform?hl=ja


# 終わりに

- モジュールやディレクトリ、ファイルの構成のルールがわかると、terraformのコードが読みやすくなったので勉強になった
- Terraformでコード読み書きするにあたって、ライフサイクル、ステート管理、標準で提供してる関数、ループや条件式など、ほかにもいろいろあるのでその辺りもドキュメントちゃんと読んでいこ
- Terraformで書かれたコードからインフラ構成が追えそうな気がしてきたのでGCPの勉強するぞ
- ドキュメントわかりやすかったので、良かった

https://www.terraform.io/language

- 触り始めとして、この本がとても参考になった。実践編も読みたい

https://techbookfest.org/product/6331235183886336

