# 9. module — インフラのコンポーネント化

> **この章でわかること**
> module とは何か、どう呼び出し、どこから取ってくるか。良い module と悪い module の分かれ目と、バージョン管理・配布の考え方。

ここまでの章では、1 つのディレクトリに `.tf` ファイルを並べてリソースを書いてきました。小さいうちはそれで十分です。

しかし同じような VPC やバケットを何度も書くようになると、React で同じ JSX をコピペし始めたときと同じ匂いがしてきます。そこで登場するのが **module** です。

---

## 9.1 module とは何か

Terraform の module の定義はとても単純です。

**`.tf` ファイルが入ったディレクトリ 1 つが、module 1 つです。**

特別な宣言は要りません。`package.json` に相当するマニフェストもありません。ディレクトリがあり、中に `.tf` があれば、それはもう module です。

つまり、あなたはここまでずっと module を書いていました。`terraform plan` を実行したディレクトリそのものが **root module**（ルートモジュール）です。

その root module から `module` ブロックで呼び出される別のディレクトリを **child module**（子モジュール）と呼びます。

```
envs/prd/            ← root module（ここで plan / apply する）
  main.tf
    └─ module "network" ──→ modules/network/   ← child module
    └─ module "app"     ──→ modules/app/       ← child module
```

### React コンポーネントとの対比

第 7 章で「module は関数のようなもの」と説明しました。React で言えばコンポーネントです。

| React | Terraform module |
| --- | --- |
| props | `variable` |
| 戻り値として外に渡す値 | `output` |
| コンポーネント内のローカル変数 | `locals` |
| `<Button />` を書く | `module "x" { ... }` を書く |
| `App` がルートコンポーネント | 作業ディレクトリが root module |

### 似ているが違う点

React コンポーネントは、レンダリングのたびに呼ばれて捨てられる関数です。

module はそうではありません。**module の中のリソースは、AWS 上に実体として生き続けます**。そして state には `module.network.aws_vpc.this` のように、**module 名を含んだアドレス**で記録されます。

これが後で効いてきます。module の名前を変えたり、既存のリソースを module に移したりすると、Terraform からは「古いアドレスのものを消して、新しいアドレスで作る」ように見えます。React でコンポーネントの位置を変えると state がリセットされるのと似た現象ですが、こちらは**本物のサーバや DB が消えます**。対処法（`moved` ブロック）は第 12 章で扱います。

---

## 9.2 module ブロックで呼び出す

child module はこう呼び出します。

```hcl
module "network" {
  source = "./modules/network"

  # module 側の variable に値を渡す（React の props に相当）
  name       = "myapp-prd"
  cidr_block = "10.0.0.0/16"
}

resource "aws_instance" "web" {
  # module の output は module.<名前>.<output名> で参照する
  subnet_id = module.network.private_subnet_ids[0]
  # ...
}
```

**`source` は必須**で、module のコードの取得元を指定します（次節）。

**`source` 以外の引数は、module 側の `variable` にそのまま対応します**。`default` のない variable を渡し忘れると、`validate` や `plan` の時点でエラーになります。TypeScript で必須 props を渡し忘れたときと同じです。

**外から触れるのは `output` だけ**です。module 内部の `aws_subnet.private` を呼び出し側から直接参照することはできません。公開したい値は module 側で `output` として明示します。これは関数のカプセル化そのものです。

### module を足したら init し直す

`module` ブロックを追加したり `source` を変えたりしたら、`terraform init` を再実行する必要があります。init が module のコードを取得して `.terraform/` の下に置くからです。`npm install` し直すのと同じ感覚です。

### module にも count と for_each が使える

第 6 章の `count` / `for_each` は module にも使えます（Terraform 0.13 から）。同じ構成を複数並べたいときに便利です。`count` のインデックスずれ問題もそのまま起きるので、基本は `for_each` を選んでください。

---

## 9.3 source の種類とバージョン固定

`source` には大きく 3 種類の書き方があります。

### ローカルパス

```hcl
source = "./modules/network"
```

`./` か `../` で始まるとローカルパスとして扱われます。同じリポジトリの中で使い回すならこれが最も単純です。ダウンロードは発生せず、ファイルを直接参照します。

### Terraform Registry

```hcl
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 6.0"
}
```

`<名前空間>/<名前>/<provider>` の形式で、Terraform Registry に公開された module を取ってきます。npm の `npm install some-package` に一番近い使い方です。

**`version` 引数が使えるのは Registry の module だけ**です。制約の書き方は provider の `required_providers` と同じで、`~> 6.0` なら「6.x の最新」を意味します。

### Git リポジトリ

```hcl
source = "git::https://github.com/example-org/terraform-modules.git//network?ref=v1.2.0"
```

社内の Git リポジトリから取ってくる書き方です。

- `//network` はリポジトリ内のサブディレクトリ
- `?ref=v1.2.0` はタグ・ブランチ・コミットハッシュ

Git ソースでは `version` 引数は使えません。**`ref` でタグかコミットを指定することがバージョン固定**になります。`ref` を省略すると既定ブランチの最新を取ってくるので、知らないうちに中身が変わります。

### 落とし穴: module はロックファイルで固定されない

npm に慣れていると、ここで勘違いしやすいです。

第 4 章で `.terraform.lock.hcl` を `package-lock.json` に例えました。しかし公式ドキュメントにあるとおり、**このロックファイルが記録するのは provider だけ**です。module のバージョンは記録されません。

取得済みの module は `init -upgrade` しない限りそのままですが、CI のような**まっさらな環境で init すると、`version` 制約を満たす最新**が選ばれます。`~> 6.0` と書いていれば、昨日は 6.1、今日は 6.2 が入る、ということが起き得ます。

対策は単純です。

- 本番で使う module は `version = "6.2.0"` のように**完全一致で固定**する
- 上げるときは PR で `version` を書き換え、plan の差分を見てからマージする

---

## 9.4 Terraform Registry と公開 module

[Terraform Registry](https://registry.terraform.io/) には provider だけでなく、大量の module が公開されています。

AWS 向けで特に有名なのが **terraform-aws-modules** です。VPC、RDS、ECS、ALB、IAM など、よく使う構成がひととおり揃ったコミュニティ製 module 群で、ダウンロード数も桁違いです。

例えば VPC を自分で書くと、VPC 本体、サブネット（AZ ごとにパブリック / プライベート）、インターネットゲートウェイ、NAT ゲートウェイ、ルートテーブルとその関連付け……と十数個のリソースが必要です。`terraform-aws-modules/vpc/aws` を使えば、それが数十行の呼び出しで済みます。

多くの人が踏んだ落とし穴がすでに潰されていること、Registry 上で README と入出力の一覧が読めることも利点です。

### 代償

- **変数が非常に多い**。あらゆる用途に応えるため、数十〜百以上の variable を持つ module も珍しくありません。どれを設定すべきか調べるコストがかかります
- **メジャーバージョンアップで破壊的変更がある**。リソースのアドレスが変わり、plan で大量の「置き換え」が出ることがあります。CHANGELOG とアップグレードガイドは必読です
- **中身がブラックボックスになりがち**。plan に出てくるリソースが何者かわからないまま apply する、という状態は危険です

### 選ぶときに見る点

メンテナンスが続いているか、provider のバージョン要件が合うかに加えて、**ソースを一度は読んでください**。module は `.tf` ファイルの集まりなので、読めば何を作るか必ずわかります。npm パッケージと違って読める量であることが多く、しかも**本番のインフラと権限に直結**するからです。

---

## 9.5 module 設計の勘所

自分で module を書く段になると、「どこまで module にするか」で迷います。公式ドキュメントの指針を中心に、勘所をまとめます。

### 薄すぎるラッパーを作らない

最もよくある失敗です。

```hcl
# アンチパターン: aws_s3_bucket を包んだだけの module
module "my_bucket" {
  source = "./modules/s3-bucket"
  bucket = "example"
}
```

これは React で `<div>` を包んだだけの `<MyDiv>` を作るのに似ています。読む人にとっては参照先が 1 つ増えるだけで、何も得をしません。

公式ドキュメントも、**単一のリソースを薄く包むだけの module は推奨しない**と明言しています。判断基準として挙げられているのは、「**中のメインのリソースと違う名前が思いつかないなら、新しい抽象を作れていない兆候**」というものです。

良い module は「**新しい概念**」を表します。`s3-bucket` ではなく `static-site`（バケット + CloudFront + 証明書 + DNS レコード）のように、複数のリソースを束ねて 1 つの意味を持たせたときに価値が出ます。

### 抽象化しすぎない

逆方向の失敗もあります。「あらゆる用途に使える汎用 module」を自作し始めると、variable が増え続け、`count = var.enable_x ? 1 : 0` だらけになります。

props が 50 個ある「何でもできるコンポーネント」と同じ末路です。**共通化は 2〜3 回同じものを書いてから**で十分です。

### 階層は浅く保つ

module の中から module を呼び、その中からさらに module を……と深くすると、値の受け渡しを追うだけで疲弊します。公式ドキュメントも、**child module は 1 階層に留めることを強く推奨**しています。

root module が各 child module を呼び、その間を output と variable でつなぐ。これが基本形です。

### 依存は外から渡す

module の中で「VPC がなければ作る」のような条件分岐を書くのは避けます。代わりに、**必要なもの（VPC ID やサブネット ID）を variable で受け取る**ようにします。

公式ドキュメントでは「dependency inversion（依存性逆転）」と呼ばれています。React で言えば、子コンポーネントが自分で fetch するのではなく、props で受け取るほうがテストしやすく使い回しやすいのと同じ理屈です。

### 入出力のインタフェースを丁寧に

module の variable と output は、**関数のシグネチャであり、公開 API** です。

- variable には `type` と `description` を書く。`validation` で不正な値を早めに弾く（第 7 章）
- 呼び出し側が使いそうな値（ID、ARN、エンドポイント）は output で出しておく
- 名前を後から変えるのは破壊的変更になる。最初に慎重に決める

### provider は module の中で定義しない

再利用される module の中に `provider "aws" { region = ... }` を書いてはいけません。公式ドキュメントも、他から呼ばれる module は `provider` ブロックを含んではならないとしています。

理由は 2 つあります。

- **provider を内部に持つ module には `count` / `for_each` / `depends_on` が使えない**。実際にエラーになります
- module を呼び出し側から削除すると、リソースを消すための provider 設定まで一緒に消えてしまい、destroy できなくなる

module 側は `required_providers` で「どの provider のどのバージョンが必要か」だけを宣言し、実際の設定（リージョンや認証）は root module が持ちます。別リージョンなど複数の provider 設定が必要なときは、`module` ブロックの `providers` 引数で明示的に渡します。

React の `<ThemeProvider>` をアプリのルートで 1 回だけ置くのと同じ発想です。

---

## 9.6 module のバージョニングと配布

module を複数のリポジトリやチームで使い回すようになると、配布の問題が出てきます。

### 選択肢は 3 つ

**(A) 同じリポジトリ内でローカルパス参照**

最も単純です。module の変更は使っている全環境に即座に効きます。小さなチームならまずこれで始めてください。

**(B) 別の Git リポジトリに置き、タグで参照**

module 用リポジトリを作り、`v1.2.0` のようなタグを打って `?ref=v1.2.0` で参照します。**環境ごとに別のバージョンを使える**ので、「dev で新バージョンを試してから prd を上げる」ができます。

**(C) Registry に公開する**

公開 Registry に載せるには、GitHub の公開リポジトリで `terraform-<PROVIDER>-<NAME>`（例: `terraform-aws-static-site`）という名前にし、セマンティックバージョンのタグ（`v1.0.4` など）を打つ、といった要件があります。社内向けには HCP Terraform の private registry という選択肢もあります（第 14 章）。

### セマンティックバージョニングの意味

npm と同じく、`MAJOR.MINOR.PATCH` で考えます。Terraform module で特に「破壊的変更」に当たるのは次のようなものです。

- 必須の variable を追加した
- variable や output の名前・型を変えた
- **内部のリソースのアドレスが変わり、既存環境で置き換えが発生する**

最後の点が Terraform 特有です。コードの互換性だけでなく、**既存の state に対して plan したときに何が起きるか**までが互換性の範囲です。内部リファクタリングで避けられないときは `moved` ブロックを module 内に同梱することで、利用者側の置き換えを防げます（第 12 章）。

module を変更したら、**それを使うすべての root module で plan を実行して**差分を確認します。自動テスト（`terraform test`）は第 13 章で扱います。

---

## まとめ

- **ディレクトリ 1 つ = module 1 つ**。plan を実行する場所が root module、`module` ブロックで呼ぶのが child module
- module は React コンポーネントに似ている。variable が props、output が戻り値。ただし**中身は AWS 上に実体として残り、アドレスに module 名が含まれる**
- `source` はローカルパス・Registry・Git。`version` は Registry 専用で、Git は `ref` で固定する
- **ロックファイルは module を固定しない**。本番では version を完全一致で書くのが安全
- 薄いラッパーを作らない、抽象化しすぎない、階層は浅く、依存は外から渡す、**provider は module に書かない**
- module の互換性には「既存 state に対して置き換えが起きないか」まで含まれる

次章では、ここまでの道具を総動員して、典型的な Web アプリのインフラを AWS 上にどう組むかを俯瞰します。

---

[← 前: state — Terraform の記憶](08-state.md) | [目次](../README.md) | [次: 実例で見る: Web アプリのインフラを組む →](10-real-world-example.md)
