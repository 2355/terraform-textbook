# 10. 実例で見る: Web アプリのインフラを組む

> **この章でわかること**
> 典型的な Web アプリを AWS に載せるとき、どんなリソースが必要で、それをどう module に分け、どの順で作るか。Terraform で管理するもの・しないものの線引き。

ここまでで部品は揃いました。この章では、それを組み立てた「完成図」を眺めます。

写経用のコードは載せません。**全体の地図を頭に入れること**が目的です。実際に書くときは、この地図と第 9 章の公開 module を手がかりにしてください。

---

## 10.1 題材: どんなアプリを載せるか

よくある構成を想定します。

- **API サーバ**: Node.js（例えば Express や NestJS）をコンテナにしたもの
- **データベース**: PostgreSQL
- **フロントエンド**: React の SPA。ビルドすると静的ファイル（HTML / JS / CSS）になる
- **ドメイン**: `example.com`。フロントは `app.example.com`、API は `api.example.com`

ローカルでは `docker compose up` で API と DB が立ち上がり、`npm run dev` でフロントが動く。そんなアプリを本番に出すイメージです。

### 構成図

```
                     ユーザ（ブラウザ）
                    │                 │
        app.example.com           api.example.com
                    │                 │
              [Route 53] ── DNS で振り分け
                    │                 │
             [CloudFront]           [ALB]  ← パブリックサブネット
                    │                 │
               [S3 バケット]   [ECS on Fargate タスク] ← プライベートサブネット
             （SPA の静的ファイル）     │
             ※ S3 は VPC の外      [RDS PostgreSQL] ← プライベートサブネット

  横断: ACM（証明書） / IAM ロール / Secrets Manager / CloudWatch Logs
```

`docker compose` の `api` が ECS に、`db` が RDS に、dev サーバが配っていた静的ファイルが S3 + CloudFront に置き換わる、と捉えると見通しが良くなります。

---

## 10.2 必要なリソースを洗い出す

図の箱を、Terraform のリソースに落としていきます。層ごとに見ると整理しやすくなります。

### ネットワーク層

- **VPC**: AWS 内に作る自分専用のプライベートネットワーク
- **サブネット**: VPC をさらに区切った範囲。インターネットから直接届く**パブリックサブネット**と、届かない**プライベートサブネット**を、AZ（データセンター群）ごとに用意する
- **インターネットゲートウェイ / NAT ゲートウェイ**: パブリックサブネットの出入口と、プライベートサブネットから外へ出るための片道の出口
- **ルートテーブル**: 「この宛先にはどこを通るか」の経路表
- **セキュリティグループ（SG）**: リソースごとのファイアウォール。「ALB からの 3000 番だけ受け付ける」のように書く

これだけで十数リソースになります。第 9 章で触れた `terraform-aws-modules/vpc/aws` を使う代表的な場面です。

### エッジ層（ユーザに近い側）

- **Route 53**: DNS。ドメイン名を ALB や CloudFront に向ける
- **ACM**: TLS 証明書を発行し、自動更新してくれるサービス
- **ALB**: HTTP のロードバランサ。複数のコンテナにリクエストを振り分け、TLS を終端する
- **CloudFront**: CDN。世界中のエッジから静的ファイルを配る
- **S3**: 静的ファイルの置き場所。OAC（Origin Access Control）という設定で、CloudFront 経由でのみ読めるようにする

### アプリケーション層

- **ECR**: コンテナイメージのレジストリ。Docker Hub の AWS 版
- **ECS クラスタ / タスク定義 / サービス**: Fargate でコンテナを動かす一式。タスク定義は「どのイメージを、どれだけの CPU・メモリで、どんな環境変数で動かすか」を書いた設計図で、`docker-compose.yml` の 1 サービス分に近いものです。サービスは「それを常に N 個動かし続け、ALB につなぐ」役

### データ層

- **RDS**: マネージドな PostgreSQL。バックアップやパッチ適用を AWS がやってくれる
- **DB サブネットグループ**: RDS をどのプライベートサブネットに置くかの指定

### 権限・秘密・観測

- **IAM ロール**: ECS には 2 種類あります。**タスク実行ロール**は ECS がイメージを pull したりログを送ったりするための権限、**タスクロール**はアプリのコード自身が S3 などを触るための権限です。混同しやすいので注意してください
- **Secrets Manager**: DB パスワードなどの秘密情報の保管庫
- **CloudWatch Logs**: コンテナの標準出力の行き先。`console.log` はここに届く

数えると、目安で **40〜60 リソース**になります。コンソールの手作業で正確に再現するのは難しい量です。

---

## 10.3 module とファイルにどう分けるか

40 リソースを 1 つの `main.tf` に書くこともできますが、すぐに読めなくなります。分け方の一例です。

```
infra/
  modules/
    network/      # VPC・サブネット・NAT（公開 module を薄く呼ぶだけでもよい）
    database/     # RDS・サブネットグループ・SG
    api-service/  # ECR・ECS・ALB・タスク用 IAM ロール・ロググループ
    static-site/  # S3・CloudFront・OAC
  envs/
    prd/
      main.tf      # 上の module を呼び、output と variable でつなぐ
      dns.tf       # Route 53 のゾーンとレコード、ACM 証明書
      backend.tf   # state の保存先（第 8 章）
      variables.tf
      outputs.tf
```

### 分け方の基準

**「一緒に変わるもの」を同じ module に入れる**のが基本です。

- ALB とターゲットグループ（ALB が転送する先の束）と ECS サービスは、ポート番号やヘルスチェックのパスを共有し、いつも一緒に変わる → `api-service` にまとめる
- VPC は一度作ったらほぼ触らない → 他と分ける
- RDS は**消えたら取り返しがつかない** → 独立させ、`prevent_destroy`（第 6 章）を付ける

React で「状態とそれを使う UI を近くに置く（コロケーション）」のと同じ考え方です。逆に `security-groups/` や `iam/` のように**リソースの種類ごとに分けるのは避けたほう**が無難です。1 つの機能を変えるたびに、あちこちの module を触ることになるからです。

### module 間は output でつなぐ

```hcl
module "network" {
  source = "../../modules/network"
  name   = "myapp-prd"
}

module "database" {
  source = "../../modules/database"

  # network の戻り値を database の引数に渡す。この参照が依存関係にもなる
  vpc_id     = module.network.vpc_id
  subnet_ids = module.network.private_subnet_ids
}
```

第 9 章の「依存は外から渡す」の実践です。`database` module は VPC がどう作られたかを知りません。

ここではすべてを 1 つの root module（＝ 1 つの state）に入れていますが、規模が大きくなるとネットワーク層とアプリ層で state を分けたくなります。その判断基準は第 11 章で扱います。

---

## 10.4 どの順で作るか

### 依存の向き

```
network ──→ database ──┐
   │                   ├──→ api-service ──→ DNS レコード
   └──→ ACM 証明書 ────┘
static-site ←── ACM 証明書（us-east-1）
```

矢印は作る順序で、`A ──→ B` は「A を先に作る」を表します。

Terraform は参照から依存グラフを作り、この順序を自動で守ってくれます（第 6 章）。1 回の `apply` で全部作ることもできます。

### それでも少しずつ apply する

しかし実際には、**層ごとに少しずつ apply する**ことを勧めます。

1. ネットワークだけ書いて apply。VPC とサブネットができたことをコンソールで確認する
2. RDS を足して apply。RDS の作成は数分〜十数分かかります
3. ECR を作り、CI からイメージを 1 つ push しておく
4. ALB と ECS を足して apply。タスクが起動し、ヘルスチェックが通るか確認する
5. 最後に S3 + CloudFront と DNS

理由はデバッグのしやすさです。60 リソースを一度に apply して途中で失敗すると、切り分けが大変です。**小さく作って確かめる**のは、コードを書くときと変わりません。

### 順序で詰まりやすい点

**イメージがないと ECS タスクは起動しない**。ECS サービスを作った時点で ECR にイメージがなければ、タスクは起動に失敗し続けます。手順 3 でイメージを先に用意するのはこのためです。

**ACM 証明書の DNS 検証**。証明書は「このドメインの持ち主である」ことを DNS レコードで証明しないと発行されません。Route 53 で同じドメインを管理していれば、検証用レコードも Terraform で作れます（`aws_acm_certificate_validation` リソースで発行完了を待てます）。

**CloudFront 用の証明書は us-east-1 に作る**。AWS のドキュメントで、CloudFront で ACM 証明書を使うには米国東部（バージニア北部）リージョンで発行する必要があると定められています。東京リージョンで全体を作っていても、ここだけ別リージョンです。Terraform では provider に別名（alias）を付けて対応します。

```hcl
# CloudFront の証明書だけは us-east-1 で発行する必要があるため、別名付きの provider を用意する
provider "aws" {
  alias  = "us_east_1"
  region = "us-east-1"
}

resource "aws_acm_certificate" "cdn" {
  provider          = aws.us_east_1
  domain_name       = "app.example.com"
  validation_method = "DNS"
}
```

ALB 用の証明書は ALB と同じリージョンに作ります。証明書が 2 枚要るのは、初見で戸惑う点です。

---

## 10.5 要所の書き方

全部は載せませんが、「ここは知らないと事故る」という箇所を 2 つだけ抜粋します。

### DB パスワードを Terraform に持たせない

第 7・8 章で見たとおり、`password = var.db_password` と書くと、パスワードは state に平文で残ります。RDS にはこれを避ける仕組みがあります。

```hcl
resource "aws_db_instance" "main" {
  engine         = "postgres"
  instance_class = "db.t4g.micro"
  # ...

  # パスワードを RDS 自身に生成させ Secrets Manager で管理させる。Terraform の state には値が入らない
  manage_master_user_password = true

  lifecycle {
    # DB の消失は復旧不能なので、誤った destroy を plan の段階で止める
    prevent_destroy = true
  }
}
```

`manage_master_user_password = true` は第 7 章で触れた、パスワードを RDS と Secrets Manager に任せる設定です。Terraform が知るのはシークレットの ARN だけです。

アプリには、ECS のタスク定義でこのシークレットの ARN（と JSON 内の `password` キー）を指定し、**起動時に環境変数として注入**させます。このときタスク実行ロールに `secretsmanager:GetSecretValue` の権限が必要です。アプリから見れば `process.env.DB_PASSWORD` で読めるだけで、値がコードにも state にも現れません。

### CI と ECS サービスの取り合いを防ぐ

```hcl
resource "aws_ecs_service" "api" {
  name            = "api"
  task_definition = aws_ecs_task_definition.api.arn
  desired_count   = 2
  # ...

  lifecycle {
    # デプロイ（新しいタスク定義への切り替え）とオートスケールは Terraform の外で行うため、差分として扱わない
    ignore_changes = [task_definition, desired_count]
  }
}
```

この `ignore_changes` がない状態で CI が新しいイメージをデプロイすると、次の `terraform apply` が**古いタスク定義に戻してしまいます**。Terraform から見れば「コードに書いてある状態と違う」ので、律儀に元に戻すわけです。

この「誰がどの値の持ち主か」という問題が、次節のテーマです。

---

## 10.6 Terraform で管理するもの・しないもの

Terraform は何でも管理できますが、**何でも管理すべきではありません**。

### 線引きの原則

**変更頻度とオーナーが違うものは分ける**。

インフラの構成は週に数回変わる程度です。一方、アプリのデプロイは 1 日に何回も起きます。これを同じ `terraform apply` に乗せると、デプロイのたびにインフラ全体の plan を読むことになり、レビューが形骸化します。

### 典型的な分担

| 対象 | 誰が管理するか |
| --- | --- |
| VPC、ALB、ECS サービスの「枠」、RDS、IAM | Terraform |
| コンテナイメージのビルドと ECR への push | CI |
| 新しいイメージタグでのデプロイ（タスク定義の更新） | CI（ECS の API や専用のデプロイツールを使う） |
| SPA のビルド成果物の S3 アップロードとキャッシュ削除 | CI |
| DB のスキーマ変更（マイグレーション） | アプリ側のツール（Prisma など） |
| 秘密情報の**値** | Secrets Manager（Terraform は「箱」だけ作る） |
| ドメインの登録そのもの | 手作業（年に一度の更新程度で、コード化の利点が薄い） |

`package.json` で例えるなら、Terraform が管理するのは「アプリが動く土台」で、そこに載る「アプリのバージョン」は別のパイプラインが管理する、という分担です。

### 境界には印を付ける

管理を分けたら、その境界はコードに残します。前節の `ignore_changes` がまさにそれです。**「この値は Terraform の外で変わる」とコードを読んだ人にわかる**ことが大事です。

境界が曖昧なまま運用すると、CI と Terraform が同じ値を互いに上書きし合う、手作業での変更が次の apply で消える、といった事故が起きます。第 8 章の drift と同じ問題が、チームの中で起きるわけです。

### 最初から全部やらない

最後に現実的な助言です。この章の構成を最初から完璧に作ろうとしないでください。

まずはネットワークと RDS と ECS だけを Terraform にし、フロントは既存のホスティングのまま、ということでも構いません。**動くものを小さく作り、管理範囲を少しずつ広げる**ほうが、結果的に早く確実です。既存の手作りリソースを後から取り込む方法は第 12 章で扱います。

---

## まとめ

- 典型的な Web アプリでも、ネットワーク・エッジ・アプリ・データ・権限の各層で **40〜60 リソース**になる
- `docker compose` の各サービスが、ECS・RDS・S3 + CloudFront に置き換わるイメージで捉えると見通しが良い
- module は**一緒に変わるもの**でまとめる。リソースの種類ごとに分けない
- 依存は Terraform が解決してくれるが、**apply は層ごとに小さく刻む**。CloudFront の証明書は us-east-1、など順序の罠を知っておく
- DB パスワードは `manage_master_user_password` で Terraform の外に出す
- **アプリのデプロイは CI、インフラの枠は Terraform**。境界は `ignore_changes` などでコードに明示する

次章では、この構成を dev / stg / prd の複数環境に展開するとき、環境と state をどう分けるかを考えます。

---

[← 前: module — インフラのコンポーネント化](09-modules.md) | [目次](../README.md) | [次: 環境の分け方と state の分割 →](11-environments.md)
