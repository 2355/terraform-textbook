# 6. resource と data source、依存関係

> **この章でわかること**
> resource と data source の違い。Terraform が依存関係をどう見抜き、どの順で作るのか。`count` / `for_each` / `dynamic` / `lifecycle` の使いどころと、「置き換え」がなぜ怖いのか。

Terraform のコードの大半は `resource` ブロックです。そして事故の多くは「resource が思ったのと違うやり方で変更された」ことから起きます。この章では、resource がどう識別され、どう並べられ、どう変更されるのかを見ていきます。

---

## 6.1 resource のアドレスと属性

### アドレス = 型 + 名前

```hcl
resource "aws_s3_bucket" "this" {
  bucket = "my-app-assets"
}
```

このリソースは、Terraform の中で **`aws_s3_bucket.this`** という名前で識別されます。これを**リソースアドレス**と呼びます。plan の出力にも、state にも、CLI の引数にも、このアドレスが出てきます。

注意したいのは、`this` は**Terraform の中だけの名前**だということです。AWS 上のバケット名は `bucket` 引数の `"my-app-assets"` です。

```
aws_s3_bucket.this   ← Terraform 内の識別子（コードと state をつなぐ）
"my-app-assets"      ← AWS 上の実際の名前
```

React の `key` が「前回の描画結果との対応付け」のための名前であるのと似ています。だから**アドレスを変えると、Terraform からは別のリソースに見えます**。`this` を `assets` に名前変更しただけで、plan は「古いものを削除、新しいものを作成」になります。これを避ける `moved` ブロックは第 12 章で扱います。

### 引数と属性

**引数**は自分で書く入力、**属性**はリソースが持つ値で、引数に加えて作成後に AWS が決めるものを含みます。

S3 バケットの ARN（AWS 上のリソースの一意な識別子。第 2 章）は、作るまで決まりません。こういう値を **computed な属性**と呼び、plan では次のように表示されます。

```
  + resource "aws_s3_bucket" "this" {
      + arn    = (known after apply)
      + bucket = "my-app-assets"
      + id     = (known after apply)
      ...
```

**`(known after apply)`** は「apply するまでわからない」という意味で、まだ resolve していない Promise に近いものです。

この値は**参照している側にも伝播します**。`aws_s3_bucket.this.arn` を使う IAM ポリシーも、plan 時点では中身が `(known after apply)` になります。

---

## 6.2 data source — 既存のものを読む

resource が「作って管理するもの」なら、**data source は「すでにあるものを読むだけ」**です。

```hcl
# アカウントで使えるアベイラビリティゾーン（AZ）の一覧を取得する
data "aws_availability_zones" "available" {
  state = "available"
}

# 参照するときは data. を付ける
# data.aws_availability_zones.available.names
```

使える AZ はアカウントやリージョンで違うので、ハードコードせずに問い合わせます。

よく使う data source には次のようなものがあります。

- `aws_caller_identity` — 今使っている AWS アカウントの ID
- `aws_ami` — 条件に合う最新の OS イメージ（AMI）の ID
- `aws_vpc` — 別の仕組みで作られた既存の VPC

Web 開発でいえば、resource は `POST` / `PUT` / `DELETE`、data source は `GET` です。**data source は何も作らず、何も壊しません。**

### 使いどころと注意

data source の主な用途は、**Terraform の管理外にあるもの**や、**別の state で管理しているもの**の値を読むことです（state の分割は第 11 章）。

ただし data source は plan のたびに読み直され、読み先が変わると参照しているリソースも変更対象になります。`aws_ami` で「最新の AMI」を取ってくると、新しい AMI が公開されたタイミングで、意図せずサーバの作り直しが plan に出ることがあります。

---

## 6.3 依存関係とグラフ

### 参照が依存になる

HCL には実行順序がない、と第 5 章で書きました。では Terraform は何を先に作るのでしょうか。

答えは**参照**です。

```hcl
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

resource "aws_subnet" "a" {
  vpc_id     = aws_vpc.main.id   # ← この参照が「VPC が先」という依存になる
  cidr_block = "10.0.1.0/24"
}
```

`aws_subnet.a` が `aws_vpc.main.id` を参照しているので、Terraform は「VPC を先に作り、その ID が決まってからサブネットを作る」と判断します。これを**暗黙の依存**と呼びます。削除はその逆順です。

### DAG と並列実行

Terraform は全リソースの依存関係から**有向非巡回グラフ（DAG）**を作ります。npm が `package.json` の依存ツリーを解決するのと似た発想です。

```
          aws_vpc.main
          /          \
 aws_subnet.a    aws_subnet.b      ← 互いに依存しないので並列に作られる
          \          /
       aws_lb.web (両方のサブネットを参照)
```

依存のないもの同士は `Promise.all` のように**並列に処理されます**。同時実行数はデフォルトで最大 10 で、`-parallelism` オプションで変えられます。

循環参照はエラーになります。

### terraform graph

依存グラフは `terraform graph` で出力できます。出力は Graphviz の DOT 形式なので、可視化には Graphviz を使います。

```sh
terraform graph | dot -Tsvg > graph.svg
```

小さい構成で一度見ておくと感覚が掴めます。

### depends_on は最後の手段

参照では表現できない依存もあります。

```hcl
resource "aws_lambda_function" "api" {
  role = aws_iam_role.api.arn   # role への依存は参照で表現されている
  # ...

  # ロールへのポリシー付与は別リソースで参照関係がないため、
  # 権限が揃う前に関数が作られないよう明示的に待たせる
  depends_on = [aws_iam_role_policy_attachment.api_logs]
}
```

Lambda 関数（コードを実行する AWS のサービス）はロールを参照していますが、**ロールにポリシーを付けるリソース**は参照していません。そのままだと両者が並列に作られます。

こういう「コードには表れない依存」を伝えるのが `depends_on` です。ただし公式も**最後の手段**と位置づけています。`depends_on` を使うと、Terraform は依存先の何が変わるか判断できず、`(known after apply)` が増え、**必要以上に置き換えを含む保守的な plan** になりうるからです。**まず参照で表現できないか考える**のが原則です。

---

## 6.4 count と for_each

同じ種類のリソースを複数作るには、`count` か `for_each` を使います。どちらも resource ブロックに書く**メタ引数**（provider ではなく Terraform 自体が解釈する引数）です。

### count — 数で増やす

```hcl
resource "aws_s3_bucket" "data" {
  count  = length(var.names)
  bucket = "my-app-${var.names[count.index]}"
}
```

アドレスは `aws_s3_bucket.data[0]`、`aws_s3_bucket.data[1]`… と**数値のインデックス**になります。

`count = var.enabled ? 1 : 0` で「条件付きで作る」使い方もよく見ます。

### count のインデックスずれ問題

React でリストを描画するとき、`key` に配列のインデックスを使うと、要素の削除で状態が別の要素にずれる問題がありました。**count でも同じことが起きます。**

```
names = ["logs", "assets", "backup"]
  [0] logs   [1] assets   [2] backup

assets を消すと → ["logs", "backup"]
  [0] logs   [1] backup   ← [1] の中身が assets → backup に変わった
                          ← [2] は消えた
```

Terraform はアドレスで対応を取るので、plan は「`[1]` の名前を assets から backup に変更、`[2]` を削除」になります。バケット名の変更は置き換え（6.7）なので、**無関係な backup まで作り直しの対象になります**。

### for_each — キーで増やす

これを避けるのが `for_each` です。

```hcl
resource "aws_s3_bucket" "data" {
  for_each = toset(var.names)   # list はそのまま渡せないので set にする
  bucket   = "my-app-${each.key}"
}
```

アドレスは `aws_s3_bucket.data["logs"]` のように**文字列のキー**になります。assets を消しても logs と backup のアドレスは変わりません。React で安定した ID を `key` に使うのと同じです。

map を渡せば `each.key` と `each.value` を両方使えます。

### 使い分け

- **基本は `for_each`**。要素に意味のある名前がつけられるなら、こちら
- `count` は「0 か 1 か」の条件付き作成や、完全に同一のものを N 個作るときに限る

注意点として、**`for_each` のキーは plan の時点でわかっていなければなりません**。`(known after apply)` な値（たとえば別リソースの ID）をキーにはできません。キーは自分で決めた名前にするのが定石です。

---

## 6.5 dynamic ブロック — ネストしたブロックを繰り返す

`count` / `for_each` はリソースそのものを増やします。では、**リソースの中のネストしたブロック**を繰り返したいときは？

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  dynamic "rule" {
    for_each = var.expire_rules   # 例: { tmp = 7, archive = 365 }
    content {
      id     = rule.key
      status = "Enabled"
      filter {
        prefix = "${rule.key}/"
      }
      expiration {
        days = rule.value
      }
    }
  }
}
```

`dynamic "rule"` は「`rule` ブロックを `for_each` の要素の数だけ生成する」という意味です。JSX で `{items.map(i => <Rule key={i.id} {...i} />)}` と書くのに近いです。

便利ですが、**読みにくくなりやすい**のが難点です。ブロックが数個なら、そのまま並べて書いたほうがレビューしやすいことも多いです。

---

## 6.6 lifecycle — 変更のされ方を制御する

`lifecycle` ブロックは、リソースの作成・更新・削除の振る舞いを変えるメタ引数です。よく使う 4 つを挙げます。

```hcl
resource "aws_db_instance" "main" {
  # ...
  lifecycle {
    prevent_destroy = true            # 削除を伴う plan をエラーにする
    ignore_changes  = [tags["Owner"]] # この属性の差分は無視する
  }
}
```

**`create_before_destroy`**
置き換えが必要なとき、デフォルトは「削除してから作成」です。これを「**新しいものを作ってから古いものを消す**」に変えます。証明書やロードバランサの設定など、一瞬でもなくなると困るものに使います。ただし、名前が一意でなければならないリソースでは、同じ名前で 2 つ同時に存在できずに失敗します。

**`prevent_destroy`**
そのリソースを削除する plan を**エラーにします**。本番 DB の安全装置です。ただし**コードからブロックごと消せば効かなくなる**ので、万能ではありません。

**`ignore_changes`**
指定した属性の差分を無視します。Terraform の外（オートスケーリング等）が書き換える値に使います。使いすぎると「コードが真実」という前提が崩れます。

**`replace_triggered_by`**
指定したリソースや属性が変わったら、このリソースを**置き換え**ます。Terraform 1.2 で追加されました。「設定が変わったらサーバを作り直す」といった連動に使います。

このほか事前・事後条件を書く `precondition` / `postcondition` などもあります。

---

## 6.7 置き換え（ForceNew）の怖さ

### 「変更」できない引数がある

リソースの引数を変えたとき、Terraform がとる動きは 2 種類あります。

- **更新（in-place update）** — 既存のものをそのまま変更。plan では `~`
- **置き換え（replace）** — 削除して作り直す。plan では `-/+`（`create_before_destroy` なら `+/-`）

どちらになるかは**クラウド側の API の都合**で決まります。S3 バケット名は API に「名前を変える」操作がないので、変えると作り直しです。provider の内部では、こうした引数は **ForceNew** と呼ばれます。

plan では次のように表示されます。

```
  # aws_s3_bucket.this must be replaced
-/+ resource "aws_s3_bucket" "this" {
      ~ bucket = "my-app-assets" -> "my-app-assets-v2" # forces replacement
```

### なぜ怖いのか

**置き換え = 一度消える**ということです。

- S3 バケットなら、中身があると削除に失敗して apply が止まる（`force_destroy` を付けていれば中身ごと消える）
- DB なら、データが消える
- サーバなら、IP アドレスが変わり、その間サービスが止まる

しかも、置き換えは**連鎖します**。あるリソースが作り直されて ID が変わると、それを参照しているリソースも更新や置き換えの対象になります。plan の 1 行目だけ見て「タグを変えただけ」と思っていたら、下の方で DB が `-/+` になっていた、というのは実際に起きる事故です。

### 身を守る習慣

- **plan の `-/+` と `must be replaced` を必ず探す**。`# forces replacement` の付いた行が原因です
- 消えて困るリソースには `prevent_destroy` を付ける
- 名前変更やリファクタリングは `moved` ブロックで行う（第 12 章）

逆に、わざと作り直したいときは `terraform apply -replace=aws_instance.web` のように `-replace` オプションを使います。

---

## まとめ

- リソースは**アドレス**（`aws_s3_bucket.this`）で識別される。アドレスを変えると別物扱いになる
- `(known after apply)` は**作るまで決まらない値**で、参照している側にも伝播する
- **data source は読むだけ**。管理外のものや別 state のものの値を取る
- Terraform は**参照から依存グラフ（DAG）を作り**、独立したものは並列に処理する。`depends_on` は最後の手段
- 複数作るなら**基本は `for_each`**。`count` はインデックスずれで無関係なリソースを作り直すことがある
- `lifecycle` で置き換え順序や削除防止を制御できる
- **置き換え（`-/+`）は一度消えること**。plan で必ず探す

次章では、構成に外から値を渡す variables と、結果を取り出す outputs を扱います。

---

[← 前: HCL を TypeScript 使いの目で読む](05-hcl.md) | [目次](../README.md) | [次: variables と outputs →](07-variables-outputs.md)
