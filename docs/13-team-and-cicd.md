# 13. チーム開発と CI/CD

> **この章でわかること**
> 手元 apply をやめ、PR で plan・マージで apply する流れの組み方とその落とし穴。静的解析・ポリシー・テストで人間のレビューを支える方法。

ここまでは、自分のマシンから `terraform apply` を叩く前提で話してきました。この章では、アプリ開発で当たり前の「PR → レビュー → CI → マージ → デプロイ」を Terraform に持ち込みます。

---

## 13.1 なぜ手元で apply してはいけないのか

アプリで言えば、各自が手元でビルドしたものを本番サーバに直接アップロードしている状態です。手元 apply には同じ問題があります。

**誰が何を適用したか残らない**
apply の結果は実行した人のターミナルにしか出ません。「昨日セキュリティグループを変えたのは誰か」を後から追えません。

**手元の状態に依存する**
コミットしていない変更、古いブランチ、違う Terraform のバージョン、違う provider のバージョン。apply されるのは「リポジトリのコード」ではなく **「その人の手元にあるコード」** です。

**強い権限を人に配ることになる**
本番を変更できる AWS 権限を、全員のノート PC に置く必要があります。PC が侵害されれば、本番も侵害されます。

**レビューが形骸化する**
PR で承認されたコードと、実際に apply されたコードが同じだと保証できません。

つまり問題は「apply という行為」ではなく、**apply が人と手元環境に紐づいていること**です。だから apply を人の手から取り上げて、**決まった場所・決まった権限・決まった手順でしか実行されない**ようにします。それが CI/CD です。

---

## 13.2 PR で plan、マージで apply

もっとも一般的な流れはこうです。

```
1. ブランチで .tf を変更し、PR を作る
          ↓
2. CI が terraform plan を実行し、結果を PR にコメントする
          ↓
3. レビュアーは「コードの差分」と「plan の結果」を両方読む
          ↓
4. 承認されたらマージ
          ↓
5. main ブランチで CI が terraform apply を実行する
```

ポイントは 3 です。アプリの PR では「コードの差分」を読めば挙動の変化がおおよそわかります。Terraform では **コードの差分と、現実に起きる変化が一致しない**ことがあります。

たとえば引数を 1 つ変えただけなのに、plan を見ると `-/+`（作り直し）になっている。コードの diff だけ見ていたら気づけません。**plan はレビュー対象の一部**です。

このように「Git リポジトリを唯一の正とし、マージを契機に現実を寄せる」運用を **GitOps** と呼びます。React で言えば、state（Git）を変えれば UI（インフラ）が追従する、という関係を運用レベルで実現する考え方です。

### 代償

- **緊急時に遅い**。障害対応中でも PR → レビュー → マージを通す必要があります。そのため「緊急時は手元 apply を許す。ただし事後に必ずコードへ反映する」といった例外ルールを決めておくチームが多いです
- **CI の権限が強くなる**。本番を変更できる権限が CI に集まります。次節の OIDC で、その権限をできるだけ安全に渡します

---

## 13.3 GitHub Actions で組む

実装イメージです。細かい書き方より、**各部品の役割**に注目してください。

```yaml
name: terraform
on:
  pull_request:
  push:
    branches: [main]

permissions:
  id-token: write      # OIDC トークンを発行してもらうために必要
  contents: read

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v7
      # 簡略化のため 1 ジョブ・1 ロール。実際は後述のとおり plan と apply で分ける
      - uses: hashicorp/setup-terraform@v4
        with:
          terraform_version: "1.16.4" # 全員・CI で同じバージョンに固定する
      - uses: aws-actions/configure-aws-credentials@v6
        with:
          role-to-assume: arn:aws:iam::123456789012:role/terraform-ci
          aws-region: ap-northeast-1
      - run: terraform init
      - run: terraform plan
        if: github.event_name == 'pull_request'
      - run: terraform apply -auto-approve
        if: github.ref == 'refs/heads/main' && github.event_name == 'push'
```

`hashicorp/setup-terraform` は Terraform CLI を入れる公式 Action で、`actions/setup-node` と同じ役割です。

この例は root module が 1 つの前提です。第 11 章のようにディレクトリを分けた場合は、`pull_request` の `paths` フィルタで対象を絞るか、変更のあったディレクトリを検出して `matrix` に渡し、ディレクトリごとに plan します。変更のないディレクトリまで毎回 plan すると、遅いうえに読むべき出力が埋もれるからです。

### OIDC で長期キーを置かない

注目してほしいのは `configure-aws-credentials` の部分です。**AWS のアクセスキーをどこにも書いていません。**

IAM ユーザのアクセスキーを Secrets に登録する方法は、「期限のない合鍵」を外部に預けることになります。今は **OIDC（OpenID Connect）** を使います。

```
GitHub Actions ──「私は org/repo の main ブランチのジョブです」という
                  署名付きトークン（ID トークン）を発行
      ↓
AWS（IAM）──── トークンの署名と中身を検証。条件に合えば
                  数十分〜数時間で失効する一時的な認証情報を渡す
      ↓
terraform ─── その一時的な認証情報で AWS API を呼ぶ
```

「Google でログイン」と同じく、GitHub が ID を保証し、AWS がそれを信じてロール（第 2 章）を貸し出します。AWS 側では、IAM ロールの信頼ポリシーに **「どのリポジトリの、どのイベント・ブランチなら引き受けてよいか」** を書きます。

さらに **plan 用と apply 用でロールを分ける**のが定石です。

- **plan 用ロール**: 読み取り中心。PR のイベント（ブランチを push できる人なら誰でも起こせる）から引き受けられる
- **apply 用ロール**: 変更権限あり。main ブランチからのみ引き受けられる

悪意ある PR が CI 上で apply しようとしても、権限がないので失敗します。なお plan 用ロールにも、state の読み書き（ロック取得を含む）とリソースの読み取り権限は必要です。

---

## 13.4 plan と apply の間に起きること

「PR で plan を見て、マージで apply」には落とし穴があります。**レビューした plan と、実際に apply される内容が同じとは限らない**のです。

### 時間差の問題

PR を作ってからマージまで、2 日かかったとします。その間に、

- 別の PR が先にマージされて、state が変わった
- 誰かがコンソールで手作業をして、drift が起きた（第 8 章）
- provider の新しいバージョンが出て、`init` で別のバージョンが入った（ロックファイルを使っていれば防げます。ただし第 9 章で見たとおり、module のバージョンはロックファイルでは固定されません）

すると main で走る apply は、レビュー時とは違う plan を作ってそのまま実行します。

### 対策 1: plan ファイルを保存する

```sh
# PR 時に plan をファイルに保存し、CI の成果物として残す
terraform plan -out=tfplan

# マージ後、その「レビュー済みの plan」をそのまま適用する
terraform apply tfplan
```

保存した plan を渡すと、Terraform は **その plan に書かれた変更だけ**を実行します。plan の後に state が変わっていれば「plan が古い」としてエラーで止まります。勝手に別の変更をすることはありません。

ただし plan ファイルには秘密情報も含まれます。保存先のアクセス権と保存期間に注意してください。

### 対策 2: apply 直前にもう一度 plan を見せる

main で plan を取り直し、人間が承認してから apply する方法です。GitHub Actions なら Environments の承認機能（required reviewers）で実現できます。手間は増えますが、確実です。

### 並行 PR とロック

PR A と PR B が同時に同じ state に plan を出すと、どちらの plan も「相手の変更がない前提」で作られます。state のロック（第 8 章）は **同時実行**は防ぎますが、この **前提のずれ** は防ぎません。

対処は、state を小さく分けて PR が重なりにくくする（第 11 章）か、マージ後に plan を取り直す（対策 2）ことです。

### Atlantis

こうした問題を専用に解くツールが **Atlantis** です。OSS（Apache 2.0）で、2024 年に CNCF の Sandbox プロジェクトになりました。

自前のサーバ（コンテナ）として立てて、GitHub などの Webhook を受けます。PR を作ると自動で plan が走り、`atlantis apply` とコメントすると apply して結果を返します。

特徴的なのは **PR 単位のロック** です。ある PR が plan を出したディレクトリは、その PR がマージかクローズされるまで他の PR から apply できません。前提のずれをツール側で防ぐ発想です。

また Atlantis では「**マージ前に apply し、成功したらマージする**」流れが一般的です。main のコードは必ず適用済み、という状態を保てます。

---

## 13.5 静的解析とポリシー

レビュアーの目だけに頼ると、見落としは必ず起きます。機械で弾けるものは CI で弾きます。ESLint や Prettier を CI に入れるのと同じ発想です。

| ツール | 役割 | アプリ開発での相当物 |
| --- | --- | --- |
| `terraform fmt -check` | フォーマット違反を検出 | `prettier --check` |
| `terraform validate` | 構文・型・参照の整合性 | `tsc --noEmit` |
| TFLint | provider 固有の誤り、命名規則など | ESLint |
| Trivy / Checkov | セキュリティ上の設定ミス | `npm audit`、SAST ツール |
| OPA / Sentinel | 組織独自のルールを強制 | カスタム lint ルール |

### TFLint

`validate` は「Terraform として正しいか」しか見ません。存在しない EC2 インスタンスタイプを書いても通ります。TFLint は AWS 用のプラグインを入れると、こうした **provider 固有の誤り**を apply 前に見つけてくれます。

### Trivy / Checkov

「S3 バケットが公開されている」「暗号化が無効」「セキュリティグループが全世界に開いている」といった、**動くけれど危ない**設定を検出します。

Trivy は Aqua Security のスキャナで、以前よく使われていた **tfsec は Trivy に統合されました**。これから導入するなら Trivy を選びます。Checkov は Prisma Cloud（Palo Alto Networks）系の OSS で、同じ領域をカバーします。

```sh
trivy config .   # ディレクトリ内の IaC をスキャン
checkov -d .
```

指摘は多めに出ます。**採用するルールを選び、例外は理由付きで抑制する**運用にします。

### OPA / Sentinel — ポリシー as コード

「本番の RDS は必ずマルチ AZ」「タグ `owner` を必須にする」といった **組織固有のルール**を、コードで書いて強制する仕組みです。

- **OPA（Open Policy Agent）** は CNCF の汎用ポリシーエンジンで、Rego という言語でルールを書きます。`terraform show -json tfplan`（plan の内容を JSON で出力するコマンド）で plan を JSON にし、Conftest などで検査します
- **Sentinel** は HashiCorp 製で、HCP Terraform / Terraform Enterprise に組み込まれています（第 14 章）

ここで大事なのは、コードではなく **plan（これから起きる変化）を検査できる**ことです。「この PR で何が作られ、何が消えるか」に対してルールを書けます。

---

## 13.6 terraform test と check ブロック

### terraform test

Terraform 1.6 で入った、Terraform 自身のテスト機能です。`*.tftest.hcl` というファイルにテストを書きます。

```hcl
# tests/bucket.tftest.hcl
run "bucket_name_has_prefix" {
  # plan だけで検証できるので、実リソースを作らず速く回せる
  command = plan

  assert {
    condition     = startswith(aws_s3_bucket.this.bucket, "myapp-")
    error_message = "バケット名は myapp- で始める規約"
  }
}
```

`terraform test` を実行すると、`run` ブロックが順に実行されます。Jest の `test()` と `expect()` に近い感覚です。

似ているが違う点があります。`command = apply`（省略時の既定）にすると、**本当にクラウドにリソースを作り、終わったら壊します**。速度もお金もかかります。主な用途は、社内で配布する **module（第 9 章）の入出力の検証**です。

provider をモックする機能もあり、実リソースを作らずにテストすることもできます。

### check ブロック

Terraform 1.5 で入った、**apply 後も現実が期待どおりか確かめる**仕組みです。

```hcl
check "api_health" {
  # 外部から実際に叩いて、デプロイ後も応答するかを見る
  data "http" "health" {
    url = "https://api.example.com/health"
  }

  assert {
    condition     = data.http.health.status_code == 200
    error_message = "ヘルスチェックが 200 を返さない"
  }
}
```

失敗しても **apply は止まらず、警告が出るだけ**です。variable の `validation` のように「間違った入力を拒否する」ものではなく、E2E の監視に近い位置づけです。

---

## 13.7 レビューで見るべきポイント

機械で弾けないものは人間が見ます。plan を読むときの優先順位はこうです。

**1. destroy と replace（`-` と `-/+`）があるか**
最優先です。DB やストレージの作り直しは**データ消失**を意味します。plan の末尾にある `X to add, Y to change, Z to destroy` の Z が 0 でなければ、必ず理由を確かめてください。「名前を変えただけ」なら `moved` ブロック（第 12 章）で避けられます。

**2. IAM とネットワークの変更**
IAM ポリシー、セキュリティグループ、S3 のバケットポリシー。**1 行で権限が全開になる**領域です。`"*"` や `0.0.0.0/0` が増えていないかを見ます。

**3. 意図しない差分**
PR の目的と関係ないリソースに差分が出ていたら、drift か、誰かの未反映の変更です。そのまま apply すると他人の手作業を巻き戻してしまうかもしれません。

**4. 影響範囲（blast radius）**
変更されるリソースの数が、PR の説明から想像できる規模と合っているか。

フォーマットや命名は機械に任せます。人間の注意力は有限です。

---

## まとめ

- 手元 apply の問題は、apply が**人と手元環境に紐づくこと**。CI に集約して、決まった手順・権限でしか実行されないようにする
- **PR で plan、マージで apply**。plan はコード差分と並ぶレビュー対象
- CI から AWS へは **OIDC** で一時的な認証情報を得る。長期キーを置かず、plan 用と apply 用でロールを分ける
- レビューした plan と apply される内容のずれに注意。**plan ファイルの保存**、apply 前の再承認、Atlantis の PR ロックなどで防ぐ
- fmt / validate / TFLint / Trivy / Checkov / OPA で、**機械で弾けるものは機械で弾く**
- `terraform test` は主に module の検証、`check` ブロックは apply 後の継続的な確認
- 人間は **destroy / replace と IAM・ネットワーク**を最優先で見る

次章では、ここで自前で組み立てた仕組みをまとめて提供するマネージドサービス、HCP Terraform を見ていきます。

---

[← 前: 既存リソースの取り込みとリファクタリング](12-import-and-refactor.md) | [目次](../README.md) | [次: HCP Terraform とマネージドサービス →](14-hcp-terraform.md)
