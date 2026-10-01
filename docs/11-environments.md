# 11. 環境の分け方と state の分割

> **この章でわかること**
> dev / stg / prd をどこで分けるか（AWS アカウント、Terraform の構成、state）。分けた state 同士で値を受け渡す方法と、よく使われるリポジトリ構成。

前章で Web アプリのインフラを 1 セット組みました。実務では、これが 1 セットで終わることはまずありません。開発用、検証用、本番用と、**同じ形のインフラが複数必要**になります。

この章は「同じコードから安全に複数の環境を作るには」の話です。答えは 1 つではありません。

---

## 11.1 なぜ環境を分けるのか

Web 開発でも、ローカル、ステージング、本番と環境を分けますね。理由はインフラでも同じです。

- **本番を壊さずに試したい**。変更はまず dev で apply してみる
- **本番と同じ形で検証したい**。stg は「本番と同じ構成、ただし小さい」が理想
- **権限を分けたい**。prd を触れるのは限られた人と CI だけ

大事なのは、環境間の差分は**「形」ではなく「大きさと設定値」であるべき**ことです。

```
dev : ECS タスク 1 個、DB は小さいインスタンス、マルチ AZ なし
prd : ECS タスク 4 個、DB は大きいインスタンス、マルチ AZ あり
      ↑ 違うのは数字だけ。リソースの種類と繋がり方は同じ
```

React で言えば、同じコンポーネントに違う props を渡す感覚です。環境固有のリソースや設定が増えるほど、「dev で動いたのに prd で壊れた」が起きやすくなります。

**同じコードを、違う値で何度も使う**。この章の手法はすべて、これをどう実現するかのバリエーションです。

---

## 11.2 まず AWS アカウントを分ける

Terraform の話に入る前に、もっと外側の分け方があります。**AWS アカウントそのものを環境ごとに分ける**方法です。

### マルチアカウント

AWS アカウントは、課金と権限の一番大きな境界です。1 つのアカウントの中に dev と prd を同居させると、

- IAM の設定ミスひとつで、dev 用の権限が prd に届く
- 「dev を全部消す」つもりの操作が prd を巻き込む
- 環境ごとの費用が、タグを丁寧に付けないとわからない

といった問題が起きます。アカウントを分ければ、これらは**構造的に起きなくなります**。dev アカウントの認証情報では、prd アカウントのリソースにそもそも触れません。

### AWS Organizations

複数のアカウントを束ねて管理するのが **AWS Organizations** です。

```
管理アカウント（請求をまとめる。ここには普段リソースを置かない）
 ├─ dev アカウント
 ├─ stg アカウント
 ├─ prd アカウント
 └─ 共有アカウント（ログ集約、CI 用ロールなど）
```

請求の一本化、アカウントをまたいだ制限（SCP: サービスコントロールポリシー）、IAM Identity Center による一括ログインが使えます。

小さく始めるなら 1 アカウントでも構いませんが、**本番だけは別アカウントにする**ところから始めるのが現実的です。

### Terraform から見ると

Terraform にとってアカウントの違いは「どの認証情報で API を呼ぶか」でしかありません。provider の設定で引き受けるロールを切り替えるだけです。

```hcl
provider "aws" {
  region = "ap-northeast-1"
  # 環境ごとに引き受けるロールを変えることで、apply 先のアカウントを切り替える
  assume_role {
    role_arn = var.deploy_role_arn
  }
}
```

アカウントを分けても、**Terraform のコードをどう分けるか**という問題は残ります。ここからがその話です。

---

## 11.3 手法 1: workspaces

Terraform には **workspace**（CLI workspace）という機能があります。**1 つのコードに対して、複数の state を持たせる**仕組みです。

```sh
terraform workspace new dev
terraform workspace new prd
terraform workspace select dev
terraform apply   # dev 用の state に対して apply される
```

コードの中では `terraform.workspace` で現在の workspace 名を参照できます。

```hcl
locals {
  # workspace 名で値を引き分けると、ファイルを増やさずに環境差を表現できる
  task_count = {
    dev = 1
    prd = 4
  }[terraform.workspace]
}
```

S3 backend の場合、default 以外の workspace の state は `env:/<workspace 名>/<key>` というパスに保存されます。**同じバケットの中に、環境ごとの state が並ぶ**イメージです。

### 利点

- **コードが完全に 1 つ**で、ファイルが増えない
- `workspace new` だけで環境を作れる。PR ごとの一時環境に向く

### 落とし穴

ただし、HashiCorp 自身が公式ドキュメントで**「別々の認証情報やアクセス制御が必要なデプロイには向かない」**と明言しています。理由は構造にあります。

- **全 workspace が同じ backend を共有する**。dev の state を読める人は prd の state も読める。前述のアカウント分割と相性が悪い
- **今どの workspace にいるかが、コードにもコマンドにも現れない**。`terraform apply` という同じコマンドが、直前の `select` 次第で dev にも prd にも効く。Git でブランチを確認せずに `push --force` するのに近い怖さです
- **環境差が `terraform.workspace` の分岐として散らばる**

workspace は「同じものの使い捨てコピーを手軽に作る」道具だと考えてください。**dev / prd の恒久的な分離には、次のディレクトリ分割のほうがよく使われます**。

なお HCP Terraform にも「workspace」という言葉がありますが、こちらは別物でより独立性の高い単位です。第 14 章で扱います。

---

## 11.4 手法 2: ディレクトリ分割

もう 1 つの方法は、**環境ごとにディレクトリ（= root module）を分ける**ことです。共通部分は module にして、各環境から呼び出します。

```
infra/
├── modules/
│   └── app/            # 環境に依存しない「形」の定義
│       ├── main.tf
│       └── variables.tf
└── envs/
    ├── dev/
    │   ├── main.tf     # module "app" を dev 用の値で呼ぶ
    │   └── backend.tf  # dev 用の state の置き場所
    └── prd/
        ├── main.tf
        └── backend.tf
```

```hcl
# envs/prd/main.tf
module "app" {
  source = "../../modules/app"

  # 環境差はここに集約する。module 側に環境名の分岐を書かずに済む
  task_count  = 4
  db_instance = "db.r6g.large"
  multi_az    = true
}
```

React で言えば、`modules/app` がコンポーネント、`envs/dev` と `envs/prd` がそれを違う props で使うページです。

### 利点

- **どの環境を触っているかがパスでわかる**。`cd envs/prd` してから apply するので、取り違えが起きにくい
- **backend を環境ごとに完全に分けられる**。prd の state は prd アカウントのバケットに置き、dev の人には読ませない、ができる
- **環境ごとに違う provider バージョンや module バージョンを使える**。新しい module バージョンを dev で先に試してから prd に上げる、という段階的な展開ができます

### 代償

- **ファイルが重複する**。`backend.tf` や provider 設定は各環境にほぼ同じものを書く
- **環境の数 × 分割数**だけディレクトリが増え、「全環境に同じ変更を入れる」のが面倒になる

### Terragrunt などのラッパー

**Terragrunt** は Gruntwork 社の OSS で、Terraform（と OpenTofu）を包んで動くツールです。backend や provider の設定を親ディレクトリに 1 回だけ書いて継承させる、複数ディレクトリを依存順にまとめて plan / apply する、といったことができます。2026 年には 1.0 がリリースされ、複数の構成をまとめて定義する Stacks という仕組みも入っています。

便利ですが、**Terraform の上にもう 1 層の設定言語と概念が乗る**のが代償です。チーム全員が両方を理解する必要があり、トラブル時の切り分けも 1 段深くなります。

まずは素の Terraform のディレクトリ分割で始め、重複が本当に辛くなってから検討する、で十分です。

---

## 11.5 state をどの単位で分けるか

環境で分けたら終わり、ではありません。**1 つの環境の中でも、state を複数に分ける**のが普通です。

前章の構成を 1 つの state に入れると、VPC も DB も ECS も 1 つの `apply` で動きます。これには問題があります。

### blast radius（爆発半径）

**1 回の操作で壊れうる範囲**のことです。

ECS のタスク数を変えるだけの PR でも、同じ state に VPC が入っていれば、plan には VPC も載ります。誤った変更が混ざれば、ネットワークごと巻き込まれます。

**頻繁に変えるものと、めったに変えないものは、別の state に分ける**。これが基本方針です。

### plan の速度

plan は state 内の全リソースについて AWS に現状を問い合わせます（第 8 章の refresh）。リソースが数百個になると、plan に数分かかります。

### チームの責任境界

ネットワークはインフラチーム、アプリのリソースはアプリチーム、のように**持ち主が違うなら state も分けます**。state はロックされるので、1 つにまとめると互いの作業がブロックし合います。

### よくある分け方: 層

```
network   : VPC、サブネット、NAT     … ほぼ変えない
data      : RDS                      … 消えたら困る。prevent_destroy 推奨
app       : ECS、ALB、IAM ロール      … よく変える
edge      : CloudFront、静的サイト用 S3、Route 53 … たまに変える

依存の向き: app → data → network、edge → app（Route 53 が ALB を指す）
          （左が右を参照する。参照される側は参照する側を知らない）
```

**変更頻度と壊れたときの痛さ**で層を切ると、うまくいくことが多いです。

### 分けすぎにも代償がある

逆に細かく分けすぎると、

- 1 つの機能追加で複数の state を順番に apply する必要が出る
- state 間の値の受け渡し（次節）が増え、依存関係を人間が管理することになる

**state をまたいだ依存は Terraform には見えません**。「network を先に apply してから app」という順序は、人間か CI が守る必要があります。

最初は 2〜4 個程度の層から始め、痛みが出たところで分けるのが無難です。

---

## 11.6 分割した state 間で値を渡す

state を分けると、「app 層が network 層のサブネット ID を知りたい」という需要が必ず出ます。方法は主に 3 つです。

### 1. `terraform_remote_state`

他の state の output を直接読む data source です。

```hcl
data "terraform_remote_state" "network" {
  backend = "s3"
  config = {
    bucket = "example-tfstate-prd"
    key    = "network/terraform.tfstate"
    region = "ap-northeast-1"
  }
}

# （app 層の resource の中で）network 層が output として公開した値だけが読める
subnet_ids = data.terraform_remote_state.network.outputs.private_subnet_ids
```

手軽ですが、公式ドキュメントは**可能なら別の方法を使うよう勧めています**。読めるのは output だけでも、**読む側には state ファイル全体へのアクセス権が必要**になるからです。state には秘密情報が入りうる（第 8 章）ので、権限が広がりすぎます。読む側が backend の場所を知る必要があり、層同士も密結合になります。

### 2. data source で AWS から直接引く

AWS 自体に問い合わせる方法です。

```hcl
data "aws_vpc" "main" {
  # タグで探すので、network 層の state がどこにあるかを知らずに済む
  tags = {
    Name = "main-prd"
  }
}
```

state を読まないので権限の問題がなく、network 層が Terraform 以外で作られていても動きます。代わりに、**タグや名前の規約が「インタフェース」になる**ので、その規約を守り続ける必要があります。

### 3. SSM Parameter Store に書いて読む

**SSM Parameter Store** は AWS の設定値置き場です。キーと値を保存できる、AWS 版の環境変数ストアのようなものです。

```
network 層 : aws_ssm_parameter で "/prd/network/vpc_id" に書き込む
app 層     : data "aws_ssm_parameter" で同じキーを読む
```

**公開したい値だけを明示的に置く**ので、公式ドキュメントが勧める「外部向けの値は別の場所に明示的に公開する」形そのものです。パラメータごとに IAM で読み取り権限を絞れ、アプリや CI からも読めます。

### 選び方

```
手軽さ重視・小規模・同じ人が全層を管理 → terraform_remote_state
既存リソースや Terraform 外のものを参照 → data source
層の持ち主が違う・権限を絞りたい       → SSM Parameter Store
```

TypeScript で言えば、`terraform_remote_state` は他パッケージの内部ファイルを直接 import すること、SSM は公開 API（`exports`）を介することに近いです。

---

## 11.7 モノレポ構成の例

ここまでをまとめた、よくある構成の一例です。

```
infra/
├── modules/                  # 再利用する部品（環境を知らない）
│   ├── network/
│   ├── database/
│   └── api-service/
└── envs/
    ├── dev/
    │   ├── network/          # 1 ディレクトリ = 1 root module = 1 state
    │   ├── data/
    │   └── app/
    ├── stg/
    │   └── ...（dev と同じ形）
    └── prd/
        └── ...（dev と同じ形）
```

- **`modules/` は環境を知らない**。環境名による分岐を書かない
- **`envs/<環境>/<層>/` が apply の単位**。state もこの単位で 1 つ
- 各 root module の `backend` の key は `<層>/terraform.tfstate` のように揃え、バケットは環境（アカウント）ごとに分ける

CI で変更のあったディレクトリだけ plan する話は第 13 章で扱います。

### アプリと同じリポジトリに置くか

アプリのコードと同じリポジトリに `infra/` を置く形と、インフラ専用リポジトリに分ける形があります。

- **同じリポジトリ**: アプリとインフラの変更を 1 つの PR にできる。小規模チーム向き
- **別リポジトリ**: 権限を分けやすい。共有基盤を複数アプリが使う場合に向く

**誰がレビューし、誰が apply するか**で決めてください。

---

## まとめ

- 環境間の違いは**「形」ではなく「値」**にする。同じコードを違う値で使う
- 一番強い分離は **AWS アカウントの分割**。少なくとも本番は別アカウントにする
- **workspaces** はコードが 1 つで済むが、backend を共有し、取り違えやすい。恒久的な環境分離より一時環境向き
- **ディレクトリ分割**は重複が出るが、環境が明示され、backend と権限を完全に分けられる
- 1 つの環境の中でも、**変更頻度と壊れたときの痛さ**で state を層に分ける。分けすぎると層をまたぐ依存を人間が管理することになる
- state 間の値は `terraform_remote_state`、data source、**SSM Parameter Store** で渡す。公開する値を明示する方法ほど結合が緩い

次章では、すでに手で作ってしまったリソースを Terraform に取り込む方法と、コードを安全に組み替えるリファクタリングの道具を見ていきます。

---

[← 前: 実例で見る: Web アプリのインフラを組む](10-real-world-example.md) | [目次](../README.md) | [次: 既存リソースの取り込みとリファクタリング →](12-import-and-refactor.md)
