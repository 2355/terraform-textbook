# 4. 最初の一歩: init から destroy まで

> **この章でわかること**
> Terraform のインストールと AWS 認証の正しいやり方。S3 バケットを 1 つ作って消すまでの流れと、各コマンドの出力の読み方。Git に何を入れて何を入れないか。

前章で全体像を眺めました。この章では実際に手を動かす流れを追います。題材は **S3 バケットを 1 つ作って消す**だけです。

小さな題材ですが、Terraform の基本サイクルはこれで全部出てきます。

---

## 4.1 インストールとバージョン管理

### Homebrew で入れる

macOS なら、HashiCorp 公式の tap（Homebrew の追加リポジトリ）から入れます。

```sh
brew tap hashicorp/tap
brew install hashicorp/tap/terraform
terraform -version
```

`brew install terraform` ではない点に注意してください。前章のライセンス変更を受けて、Terraform は Homebrew 本体（homebrew/core）から外れています。homebrew/core はオープンソースのソフトウェアしか扱わないためです。

### バージョン管理ツールを使う

実務では、プロジェクトごとに Terraform のバージョンを固定したくなります。Node.js で nvm や `.nvmrc` を使うのと同じ理由です。

| Node.js | Terraform |
| --- | --- |
| nvm + `.nvmrc` | **tfenv** + `.terraform-version` |
| mise（`mise.toml`） | **mise**（`mise.toml`） |

tfenv は Terraform 専用、mise は Node.js も Terraform もまとめて扱える汎用ツールです。すでに mise を使っているなら、`mise use terraform@<バージョン>` で十分です。

チーム全員と CI が**同じバージョンの Terraform を使う**ことが大切です。人によってバージョンが違うと、新しい機能を使ったコードが他の人の環境で動かない、plan の結果が人によって違う、といったことが起きるからです。

---

## 4.2 AWS の認証

Terraform は AWS の API を叩くので、当然認証が必要です。ここは**最初に正しいやり方を覚えてください**。間違ったやり方が一番事故につながる部分です。

### アクセスキーを直書きしない

ネットの古い記事には、こんなコードがよく載っています。

```hcl
# 悪い例: 絶対に真似しない
provider "aws" {
  access_key = "AKIA..."
  secret_key = "..."
}
```

これは `.env` の中身をソースコードに直書きして GitHub に push するのと同じです。アクセスキーは**有効期限のない合鍵**なので、漏れた瞬間に誰でもあなたの AWS アカウントでサーバを立てられます。公開リポジトリに漏れたキーを狙って自動で悪用する人たちもいます。

### IAM Identity Center で一時的な認証情報を使う

現在の推奨は、**IAM Identity Center**（旧 AWS SSO）経由でログインし、**期限付きの一時的な認証情報**を使う方法です。会社の Google アカウントで各種 SaaS にログインする SSO と同じ発想です。

```sh
# 初回だけ: 対話形式でプロファイル（接続設定の名前）を作る
aws configure sso

# 作業のたびに: ブラウザが開いてログインする
aws sso login --profile my-dev

# Terraform にどのプロファイルを使うか伝える
export AWS_PROFILE=my-dev
```

AWS provider は、AWS CLI と同じ設定ファイル（`~/.aws/config`）と環境変数を読みます。つまり **CLI でログインできていれば、Terraform も動きます**。コードに認証情報を書く必要はありません。

`AWS_PROFILE` を環境変数で渡すのは、`provider` ブロックに個人のプロファイル名を書くと、チームメンバーや CI と設定を共有しにくくなるからです。

個人で AWS アカウントを作ったばかりなら、IAM Identity Center の有効化から始める必要があります。手順は AWS 公式ドキュメントに従ってください。CI からの認証（OIDC）は第 13 章で扱います。

---

## 4.3 最小構成を書く

空のディレクトリを作り、`main.tf` を置きます。

```hcl
terraform {
  # Terraform 本体のバージョン範囲（1.11 以上 2.0 未満）。古い CLI で誤って実行されるのを防ぐ
  required_version = "~> 1.11"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      # 6.x の範囲内でのみ更新を許す。メジャーアップデートは破壊的変更を含むため
      version = "~> 6.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1" # 東京リージョン
}

resource "aws_s3_bucket" "this" {
  # S3 のバケット名は全世界で一意でなければならない
  bucket = "yourname-tf-textbook-20261001"
}
```

3 種類のブロックがあります。

**`terraform` ブロック** — このプロジェクト自体の設定です。`required_providers` は `package.json` の `dependencies` に相当します。`~> 6.0` は「6.0 以上 7.0 未満」で、npm の `^6.0.0` とほぼ同じ意味です。

**`provider` ブロック** — provider への設定を渡します。ここではリージョンだけです。

**`resource` ブロック** — 「こういうリソースが存在すべき」という宣言です。`"aws_s3_bucket"` がリソースの種類、`"this"` がコード内での名前です。同じ種類のリソースがそのディレクトリ内に 1 つしかないときは、`this` や `main` のような名前にする慣習があります。

`aws_s3_bucket.this` が、このリソースを指す**アドレス**になります（第 6 章）。

---

## 4.4 terraform init

```sh
terraform init
```

`init` は `npm install` に相当します。`required_providers` を読み、Registry から AWS provider をダウンロードします。

終わると 2 つのものができます。

```
.
├── main.tf
├── .terraform/            ← ダウンロードした provider 本体（node_modules 相当）
└── .terraform.lock.hcl    ← 選ばれたバージョンとハッシュ（package-lock.json 相当）
```

### .terraform.lock.hcl

`~> 6.0` のような範囲指定から、実際にどのバージョンを選んだかを記録します。次回以降の `init` は、範囲内に新しいバージョンがあっても**ロックファイルのバージョンを使い続けます**。更新したいときは `terraform init -upgrade` です。

`package-lock.json` と同じく、**Git にコミットします**。

似ていない点もあります。

- **記録するのは provider だけ**です。module（第 9 章）のバージョンは記録されません
- ファイルにはダウンロードした provider の**ハッシュ**が入ります。provider は OS ごとに別のバイナリなので、取得方法によっては Mac で作ったロックファイルが Linux の CI で検証に失敗することがあります。その場合は `terraform providers lock` で複数のプラットフォーム分のハッシュを記録できます

---

## 4.5 terraform plan を読む

```sh
terraform plan
```

出力は次のような形です（一部省略）。

```
Terraform will perform the following actions:

  # aws_s3_bucket.this will be created
  + resource "aws_s3_bucket" "this" {
      + arn           = (known after apply)
      + bucket        = "yourname-tf-textbook-20261001"
      + force_destroy = false
      + id            = (known after apply)
      ...
    }

Plan: 1 to add, 0 to change, 0 to destroy.
```

### 記号の意味

行頭の記号が、リソースに何が起きるかを表します。

| 記号 | 意味 | 危険度 |
| --- | --- | --- |
| `+` | 新規作成 | 低 |
| `~` | その場で変更（作り直さない） | 中 |
| `-` | 削除 | 高 |
| `-/+` | **削除してから作り直す**（置き換え） | 高 |
| `+/-` | 新しいものを作ってから古いものを削除する（置き換え） | 高 |
| `<=` | data source の読み取り（第 6 章） | 低 |

一番注意すべきは `-/+` と `+/-`、つまり**置き換え**です。見た目は「設定を 1 か所変えただけ」でも、リソースによっては作り直しになります。データベースなら中身が消えます。**plan では最終行の数字と、`-` を含む記号を必ず確認する**癖をつけてください。

### (known after apply)

`arn` や `id` は AWS が実際に作ってみるまで決まらない値です（詳しくは第 6 章）。

### plan は何も変更しない

`plan` は計算して表示するだけで、AWS には何も作りません。`git diff` と同じく、**何度実行しても安全**です。迷ったらまず plan です。

---

## 4.6 terraform apply と destroy

### apply

```sh
terraform apply
```

`apply` は**もう一度 plan を計算して表示し、確認を求めます**。`yes` と正確に入力したときだけ実行されます。`y` では進みません。

成功すると、AWS コンソールの S3 画面にバケットが現れます。

同時に、ディレクトリに `terraform.tfstate` というファイルができます。前章で触れた state です。いまは手元にありますが、チームで使うときはここに置いてはいけません。理由と対策は第 8 章で扱います。

### 何も変えずにもう一度

ここで試しに、もう一度 `terraform plan` を実行してみてください。

```
No changes. Your infrastructure matches the configuration.
```

コードと現実が一致しているので、何もすることがない。これが**宣言的**ということです。CLI で作成コマンドを 2 回実行すれば、2 回目は「すでに存在する」というエラーになるのが普通です。Terraform は何度実行しても同じ結果に収束します。

### destroy

```sh
terraform destroy
```

この設定で管理している**すべてのリソース**を削除します。apply と同じく、削除対象を `-` で表示して `yes` を求めます。

ここで 1 つ落とし穴があります。**中身が入っている S3 バケットは削除できません**。AWS 側の制約で、destroy がエラーで止まります。`force_destroy = true` を指定すれば中身ごと消せますが、本番のバケットにこれを付けるのは危険です。

学習中に作ったリソースは、**終わったら必ず destroy してください**。消し忘れは、そのまま請求額になります。

---

## 4.7 fmt、validate、.gitignore

### terraform fmt

```sh
terraform fmt
```

HCL を公式のスタイルに整形します。Prettier に相当しますが、**設定項目がありません**。インデントや `=` の位置揃えは 1 通りに決まっています。チームで好みを議論する余地がないのは利点です。

### terraform validate

```sh
terraform validate
```

構文や型、必須の引数が抜けていないかを検査します。`tsc --noEmit` のような位置づけです。AWS には問い合わせないので、認証なしでも動きます（ただし provider の情報を使うので、`init` は必要です）。

逆に言えば、**validate を通っても apply が成功するとは限りません**。バケット名の重複などは、AWS に送って初めてわかります。

CI での使い方は第 13 章で扱います。

### .gitignore に入れるもの

```gitignore
# provider のバイナリ。init で再生成できる（node_modules と同じ）
.terraform/

# state。秘密情報を含むことがあり、Git で共有するものではない
*.tfstate
*.tfstate.*

# plan -out で保存した計画ファイル（名前は任意）。秘密情報が平文で入りうる
tfplan
*.tfplan

crash.log
```

`.terraform.lock.hcl` は**入れない**（＝コミットする）ことに注意してください。

変数ファイル `*.tfvars`（第 7 章）は、秘密情報を入れるならコミットせず、環境ごとの設定値だけならコミットする、とプロジェクトの方針で分かれます。

---

## まとめ

- Terraform は **hashicorp/tap** から入れる。tfenv や mise で**チーム全員のバージョンを揃える**
- AWS の認証は **IAM Identity Center の一時的な認証情報**を使い、アクセスキーをコードに書かない
- `required_providers` は `dependencies`、`.terraform/` は `node_modules`、`.terraform.lock.hcl` は `package-lock.json` に相当する
- plan では **`-` と置き換え（`-/+`、`+/-`）を見落とさない**。plan 自体は何度実行しても安全
- apply は `yes` を入力して初めて実行される。同じ設定で再度 plan すれば「No changes」になる
- 中身のある S3 バケットは destroy で消せない。学習用のリソースは**必ず後片付けする**
- `.terraform/` と `*.tfstate` は Git に入れない。ロックファイルはコミットする

次章では、ここで書いた HCL という言語そのものを、TypeScript と比べながら読み解きます。

---

[← 前: Terraform の全体像](03-terraform-overview.md) | [目次](../README.md) | [次: HCL を TypeScript 使いの目で読む →](05-hcl.md)
