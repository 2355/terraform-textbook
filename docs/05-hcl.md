# 5. HCL を TypeScript 使いの目で読む

> **この章でわかること**
> HCL の文法（ブロック、型、式、`for`、関数）を TypeScript と比べながら読めるようになる。HCL が「プログラミング言語ではない」ことの意味と、その利点・不便さ。

第 4 章で S3 バケットを 1 つ作りました。そのとき書いた `.tf` ファイルの言語が **HCL**（HashiCorp Configuration Language）です。

全部覚える必要はありません。**既存のコードを読んで意味がわかる**ことを目標に、TS との違いを中心に見ていきます。

---

## 5.1 HCL はプログラミング言語ではない

一番大事なこと。**HCL は設定言語です。**

TypeScript は「手順」を書く言語ですが、HCL は「あるべき姿」を書く言語です。書かれた内容は**値の宣言の集まり**で、実行順序という概念がありません。

感覚としては、`tsconfig.json` や `package.json` に**式と参照を足したもの**が近いです。JSON では「他のキーの値を参照する」「文字列を組み立てる」ができませんが、HCL ではできます。逆に、HCL では次のことが**できません**。

- 変数の再代入（すべてイミュータブル）
- `while` や `for` 文による手続き的なループ
- 自分で関数を定義すること（5.6 で詳しく）
- 例外処理、副作用のある処理

### なぜ制限するのか

意図的な制約です。インフラのコードは PR で「本番の何が変わるか」をレビューされるものです。コードが自由すぎると、読む側が実行を頭の中でシミュレーションしなければなりません。

HCL は表現力を絞ることで、**ファイルを読めば最終形がおおむね見える**状態を保とうとしています。この「自由度 vs 読みやすさ」のトレードオフは第 15 章（CDK や Pulumi との比較）でもう一度扱います。

---

## 5.2 ブロック・引数・式

HCL の文法要素は、ほぼ 3 つだけです。

```hcl
resource "aws_s3_bucket" "logs" {   # ← ブロック（種類 + ラベル 2 つ）
  bucket = "my-app-logs-2026"       # ← 引数（名前 = 式）

  tags = {                          # ← 引数の値が map の式
    Env = "dev"
  }
}
```

### ブロック

`種類 "ラベル" "ラベル" { ... }` という形です。`resource` ブロックはラベルを 2 つ取り（リソースの型と名前）、`provider` ブロックは 1 つ、`locals` や `terraform` ブロックは 0 個です。ラベルの数はブロックの種類ごとに決まっています。

ブロックの中にブロックを入れることもできます。

```hcl
resource "aws_s3_bucket_lifecycle_configuration" "logs" {
  bucket = aws_s3_bucket.logs.id

  rule {                 # ← ネストしたブロック。= は付かない
    id     = "expire-old"
    status = "Enabled"
    # ...（実際は対象や期限のブロックも要る）
  }
}
```

**`tags = { ... }` と `rule { ... }` の違い**に注意してください。前者は「map という値を引数に代入している」、後者は「ネストしたブロック」です。どちらを使うかは provider のスキーマで決まっているので、迷ったら Registry のドキュメントを見ます。

### 引数と式

`名前 = 式` が引数です。式はリテラル、参照、関数呼び出し、演算などです。TS との違いをいくつか挙げます。

- 引数の後にセミコロンやカンマは不要。list の要素はカンマ区切り、map の要素は改行でも区切れる
- コメントは `#`。`//` や `/* */` も書けるが、公式スタイルガイドは `#` に統一することを勧めている
- `=` の位置揃えは `terraform fmt` が自動でやってくれる（Prettier 相当）

---

## 5.3 型

HCL には型があります。TS ほど細かくはありませんが、考え方は似ています。

### プリミティブ型

| HCL | TS での近いもの |
|---|---|
| `string` | `string` |
| `number` | `number`（整数と小数の区別なし） |
| `bool` | `boolean` |

`null` もあります。引数に `null` を渡すと「その引数を書かなかった」のと同じ扱いになり、provider のデフォルト値が使われます。

### コレクション型（要素の型がすべて同じ）

| HCL | TS での近いもの |
|---|---|
| `list(string)` | `string[]` |
| `map(string)` | `Record<string, string>` |
| `set(string)` | `Set<string>`（順序なし・重複なし） |

### 構造型（要素ごとに型が違ってよい）

| HCL | TS での近いもの |
|---|---|
| `object({ name = string, port = number })` | `{ name: string; port: number }` |
| `tuple([string, number])` | `[string, number]` |

### `any`

TS の `any` とは意味が違います。HCL の `any` は**型チェックの放棄ではなく、渡された値から具体的な型を推論させる**ためのプレースホルダです。たとえば `list(any)` に文字列のリストを渡せば `list(string)` として扱われます。

### 暗黙の型変換に注意

HCL は**かなり緩く型変換します**。`string` が必要な場所に `5` を、`number` が必要な場所に `"5"` を書いても自動で変換されます。`true` と `"true"` も同様です。変換できない値（`"abc"` を `number` に等）はエラーになります。

型を明示的に書く場面は、主に **variable の宣言**です。これは第 7 章で扱います。

---

## 5.4 文字列・参照・条件式

### 文字列テンプレート

TS のテンプレートリテラル `` `${name}-logs` `` に相当するのが、ダブルクォート内の `${}` です。

```hcl
bucket = "${var.project}-${var.env}-logs"
```

バッククォートではなく**普通のダブルクォート**で補間できる点が TS と違います。補間が不要なら、`"${var.name}"` と書かずに `var.name` と直接書きます。古いコードでは前者をよく見かけますが、冗長なだけです。

### heredoc

複数行の文字列はシェルと同じ heredoc で書きます。

```hcl
user_data = <<-EOT
  #!/bin/bash
  echo "hello from ${var.env}"
EOT
```

`<<-` のようにハイフンを付けると、行頭の共通インデントが取り除かれます。

**JSON を heredoc で書くのは避けてください。** IAM ポリシー（AWS の権限定義。第 2 章）でよく見かけますが、カンマ一つで壊れます。後述の `jsonencode` を使うのが定石です。

### 参照式

他の値は**ドット区切りの名前**で参照します。

```hcl
var.env                    # 入力変数（第 7 章）
local.common_tags          # ローカル値（5.6）
aws_s3_bucket.logs.arn     # リソースの属性（第 6 章）
data.aws_vpc.main.id       # data source の属性（第 6 章）
module.network.vpc_id      # module の出力（第 9 章）
```

`import` 文はありません。**同じディレクトリ内のものは、どのファイルに書かれていても名前で参照できます**（5.7）。

### 条件式

三項演算子は TS と同じ形です。

```hcl
# instance_type は EC2 のサーバの大きさ。本番だけ大きくする
instance_type = var.env == "prd" ? "m7g.large" : "t4g.micro"
```

`if` 文はありません。「条件によってリソースを作る / 作らない」は `count` を使います（第 6 章）。

---

## 5.5 for 式と splat 式

### for 式は Array.map / filter

HCL で一番 TS 使いが戸惑うのがここです。`for` は**文ではなく式**で、新しいコレクションを返します。

```hcl
# TS: names.map(n => n.toUpperCase())
upper_names = [for n in var.names : upper(n)]

# TS: names.filter(n => n !== "").map(n => n.toUpperCase())
non_empty = [for n in var.names : upper(n) if n != ""]
```

角括弧 `[ ]` で囲むと list、波括弧 `{ }` で囲むと map を返します。map を作るときは `キー => 値` と書きます。

```hcl
# TS: Object.fromEntries(users.map(u => [u.name, u.role]))
roles = { for u in var.users : u.name => u.role }

# map を回すときはキーと値を両方受け取れる
# TS: Object.entries(tags).map(([k, v]) => `${k}=${v}`)
pairs = [for k, v in var.tags : "${k}=${v}"]
```

### splat 式 `[*]`

「リストの各要素から 1 つの属性だけ取り出す」は頻出なので、専用の短縮記法があります。

```hcl
# この 2 つは同じ意味
ids = [for s in aws_subnet.private : s.id]
ids = aws_subnet.private[*].id
```

TS なら `subnets.map(s => s.id)` です。ただし splat 式は list 向けで、`for_each` で作ったリソース（map になる。第 6 章）には for 式を使います。

---

## 5.6 locals と組み込み関数

### locals はローカル定数

同じ式を何度も書きたくないときは `locals` ブロックで名前を付けます。

```hcl
locals {
  name_prefix = "${var.project}-${var.env}"
  common_tags = {
    Project   = var.project
    ManagedBy = "terraform"
  }
}
```

参照するときは `local.name_prefix` です（ブロック名は `locals`、参照は単数形の `local`。誰もが一度は間違えます）。

TS の `const` に近く、再代入も重複定義もできません。書く順序は関係なく、Terraform が依存関係を見て評価します。

### 組み込み関数

組み込み関数は 100 以上あります。よく見るものだけ挙げます。

| 関数 | 用途 | TS で近いもの |
|---|---|---|
| `merge(a, b)` | map を結合 | `{ ...a, ...b }` |
| `lookup(m, k, default)` | キーがなければデフォルト | `m[k] ?? default` |
| `jsonencode(v)` | 値を JSON 文字列に | `JSON.stringify(v)` |
| `file(path)` | ファイルを文字列で読む | `fs.readFileSync(path, "utf8")` |
| `templatefile(path, vars)` | テンプレートファイルに値を埋める | EJS などのテンプレートエンジン |
| `cidrsubnet(cidr, bits, n)` | IP アドレス範囲を分割 | （なし） |

`cidrsubnet` は、VPC（AWS 上の仮想ネットワーク）の IP アドレス範囲をサブネットに切り分けるときに使います（第 2 章）。`cidrsubnet("10.0.0.0/16", 8, 1)` は `"10.0.1.0/24"` を返します。

`jsonencode` は IAM ポリシーを書くときの定番です。

```hcl
policy = jsonencode({
  Version = "2012-10-17"
  Statement = [{
    Effect   = "Allow"
    Action   = ["s3:GetObject"]
    # 参照がそのまま書けるので、ARN を文字列で組み立てずに済む
    Resource = "${aws_s3_bucket.logs.arn}/*"
  }]
})
```

HCL のオブジェクトとして書くので、JSON のカンマや引用符で壊れる心配がありません。

### 関数は自分で定義できない

**HCL ではユーザ定義関数を書けません**（公式ドキュメントに明記されています）。ロジックを再利用したいときは、次のどれかで対処します。

- 式を `locals` に切り出す
- 処理ごと **module** にする（第 9 章。module は「引数と戻り値を持つ関数」に近い）
- **provider 定義関数**を使う

provider 定義関数は Terraform 1.8 で入った仕組みで、provider が独自の関数を提供できます。`provider::<provider名>::<関数名>` という形で呼びます。

```hcl
# AWS provider が提供する、ARN を分解する関数
account = provider::aws::arn_parse(aws_s3_bucket.logs.arn).account_id
```

自作するには provider を Go で書く必要があるので、「既存の provider の関数を使える」と理解しておけば十分です。

### terraform console で試す

式を試すには `terraform console` を使います。Node.js の REPL と同じ感覚です。

```sh
$ terraform console
> cidrsubnet("10.0.0.0/16", 8, 1)
"10.0.1.0/24"
> merge({ a = 1 }, { b = 2 })
{
  "a" = 1
  "b" = 2
}
```

`var.` や `local.`、state にあるリソースの属性も参照できます。

---

## 5.7 ファイル分割の慣習

### ディレクトリ内の .tf はすべて結合される

TS ではファイルごとにスコープがあり、`import` / `export` でつなぎます。Terraform は違います。**1 つのディレクトリにある `.tf` ファイルは、すべて 1 つの設定として結合されます。** ファイル名にも並び順にも（`override.tf` などの特殊な名前を除き）意味はありません。`main.tf` に書いたリソースを `outputs.tf` から名前で参照できます。

```
my-infra/
├── main.tf        # 中心となる resource
├── variables.tf   # variable ブロック（入力）
├── outputs.tf     # output ブロック（出力）
├── providers.tf   # provider ブロック
├── terraform.tf   # terraform ブロック（required_providers など）
├── backend.tf     # state の保存先（第 8 章）
└── locals.tf      # locals ブロック
```

これは**公式のスタイルガイドが推奨している構成**です。ただし強制ではなく、`terraform.tf` を `versions.tf` と呼ぶプロジェクトもよくあります。大きくなったら `network.tf` のように**関心ごとに分ける**ことも勧められています。

### 境界はディレクトリ

逆に言うと、**名前空間の境界はファイルではなくディレクトリ**です。この「ディレクトリ 1 つ」が Terraform における **module** の単位になります。サブディレクトリの `.tf` は自動では読み込まれず、`module` ブロックで明示的に呼び出す必要があります（第 9 章）。

### 落とし穴

- **名前の衝突はディレクトリ全体で起きる**。別ファイルに同じ `resource "aws_s3_bucket" "logs"` があるとエラーになる
- **「とりあえず残しておく」`old.tf` も読み込まれる**。拡張子が `.tf` のファイルはすべて対象です。消したい設定は Git の履歴に任せましょう

---

## まとめ

- HCL は**設定言語**。実行順序も再代入もなく、「あるべき姿」を値として宣言する
- 文法は**ブロック・引数・式**の 3 つ。`tags = { }`（値）と `rule { }`（ブロック）は別物
- 型は TS に近いが、**暗黙の型変換が緩い**。`any` は「推論させる」という意味
- `for` 式は `map` / `filter`、splat `[*]` は「属性だけ取り出す map」
- **ユーザ定義関数は書けない**。再利用は `locals`、module、provider 定義関数で行う
- 式の確認は `terraform console`。同じディレクトリの `.tf` は**すべて結合**され、境界はディレクトリ

次章では、HCL で書く中心的な要素である resource と data source、そしてそれらの依存関係を見ていきます。

---

[← 前: 最初の一歩: init から destroy まで](04-first-steps.md) | [目次](../README.md) | [次: resource と data source、依存関係 →](06-resources-and-data.md)
