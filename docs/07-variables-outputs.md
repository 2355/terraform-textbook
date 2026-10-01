# 7. variables と outputs

> **この章でわかること**
> variable / output / locals を「関数の引数・戻り値・ローカル変数」として捉える見方。値の渡し方と優先順位。秘密情報を扱うときに `sensitive` だけでは足りない理由と、新しい仕組み（ephemeral 値、write-only 引数）。

ここまでの章では、バケット名もリージョンもコードに直接書いてきました。動かすだけならそれで十分です。

しかし dev と prd で名前を変えたい、同じ構成を別のチームでも使いたい、となった瞬間にハードコードは破綻します。この章では、Terraform のコードを「**値を受け取って、値を返す**」ものにする仕組みを見ていきます。

---

## 7.1 module を関数として見る

Terraform では**ディレクトリ 1 つが module** です。これまで `apply` していたディレクトリも「root module」という module です（詳しくは第 9 章）。

この module を **1 つの関数**だと思うと、この章の登場人物がすっきり整理できます。

```
TypeScript の関数            Terraform の module
─────────────────           ─────────────────────
引数 (parameters)      ↔    variable ブロック
ローカル変数 (const)   ↔    locals ブロック
戻り値 (return)        ↔    output ブロック
関数本体               ↔    resource / data ブロック
```

HCL で書くとこうなります。

```hcl
variable "env" {
  type = string
}

variable "project" {
  type = string
}

locals {
  # 命名規則を 1 か所に集めておけば、変えるときに迷わない
  bucket_name = "${var.project}-${var.env}-assets"
}

resource "aws_s3_bucket" "this" {
  bucket = local.bucket_name
}

output "bucket_arn" {
  value = aws_s3_bucket.this.arn
}
```

参照の書き方は `var.<名前>`、`local.<名前>` です。宣言が `locals`、参照が `local` と単複で違うのは第 5 章で触れたとおりです。

### 似ているが違う点

- **呼び出し側が明示的に呼ぶわけではない**。root module の場合、「呼び出し」は `terraform plan` / `apply` の実行そのものです。引数はコマンドラインやファイルから渡します（7.3 節）
- **引数は名前付きだけ**。位置引数はありません。TS で言えば常に `fn({ env, project })` の形です
- **locals は再代入できない**。すべて `const` です。`let` に相当するものはありません

---

## 7.2 variable ブロックを読み解く

`variable` ブロックには、関数の型シグネチャと JSDoc とバリデーションをまとめて書けます。

```hcl
variable "env" {
  description = "デプロイ先の環境名"
  type        = string
  default     = "dev"

  validation {
    condition     = contains(["dev", "stg", "prd"], var.env)
    error_message = "env は dev / stg / prd のいずれかにしてください。"
  }
}
```

### type — 型注釈

`string`、`number`、`bool`、`list(string)`、`map(number)`、`object({...})` などが書けます（型そのものは第 5 章で扱いました）。

`type` は省略できますが、**省略すると何でも受け付けてしまいます**。TS で引数を `any` にするのと同じです。必ず書いてください。

`object` 型では `optional()` を使うと、一部の属性を省略可能にできます。TS の `?:` に近いものです。

```hcl
variable "bucket" {
  type = object({
    name       = string
    versioning = optional(bool, true) # 省略時は true
  })
}
```

### default — デフォルト値

`default` があれば省略可能な引数、なければ**必須の引数**です。TS の `env = "dev"` というデフォルト引数と同じ感覚です。

必須の variable に値を渡さずに `plan` すると、Terraform は対話的に値を聞いてきます。

### description — ドキュメント

JSDoc のようなものです。module の利用者にとっての説明書になり、terraform-docs（第 16 章）はここから README を生成します。

### validation — 実行時チェック

`condition` が `false` になると、`plan` の時点でエラーにして止めます。Zod のスキーマで入力を検証するのに近い発想です。

型だけでは表現できない制約を書けます。**apply してから AWS 側で怒られるより、plan の時点で止めるほうがずっと安全**です。

Terraform 1.9 から、`condition` の中で**他の variable や data source も参照できる**ようになりました。それ以前は検証対象の variable 自身しか参照できなかったので、古い記事を読むときは注意してください。

### nullable — null を許すか

`nullable = false` にすると、呼び出し側が `null` を渡したときにデフォルト値が使われるようになります。デフォルトは `true`（null を許す）です。

module を作る側なら `nullable = false` にしておくと、内部で null チェックを書かずに済みます。

### 新しめの引数: const と deprecated

Terraform 1.15（2026 年 4 月）で、`init` 時点で値が確定することを宣言する `const = true`（第 9 章で扱う module の読み込み元 `source` などに変数を使えるようになる）と、TS の `@deprecated` に相当する `deprecated` が加わりました。入門段階では「書ける」とだけ覚えておけば十分です。

---

## 7.3 値の渡し方と優先順位

root module の variable に値を渡す方法は何通りもあります。Node.js で設定値を `process.env` や `.env` ファイルや CLI 引数から受け取るのと同じで、**どれが勝つか**を知っておかないと混乱します。

### 渡し方の一覧

```sh
# 1. コマンドライン引数
terraform plan -var="env=prd"

# 2. 変数ファイルを明示的に指定
terraform plan -var-file="prd.tfvars"

# 3. 環境変数（TF_VAR_ + 変数名）
export TF_VAR_env=prd
terraform plan
```

加えて、以下のファイルは**置いておくだけで自動的に読み込まれます**。

- `terraform.tfvars` / `terraform.tfvars.json`
- `*.auto.tfvars` / `*.auto.tfvars.json`

`.tfvars` ファイルの中身は、単純な `名前 = 値` の羅列です。

```hcl
# prd.tfvars
env     = "prd"
project = "myapp"
```

### 優先順位

複数の場所から同じ variable に値が来た場合、**後から読まれたものが勝ちます**。公式ドキュメントの順序を、強い順に並べるとこうなります。

```
強い ┌ -var / -var-file（コマンドラインで書いた順に評価）
     │ *.auto.tfvars（ファイル名のアルファベット順）
     │ terraform.tfvars.json
     │ terraform.tfvars
     │ 環境変数 TF_VAR_xxx
弱い └ variable ブロックの default
```

`TF_VAR_` が意外に弱いことに注意してください。`.env` より環境変数を優先する Node.js の慣習（dotenv はデフォルトで既存の環境変数を上書きしない）とは逆の感覚です。「環境変数で上書きしたつもりが、`terraform.tfvars` の値が使われていた」というのはよくある事故です。

### 実務での使い分け

よく見るのは次のような分担です。

- **環境ごとに違う値** → `dev.tfvars` / `prd.tfvars` を作り、`-var-file` で切り替える
- **秘密情報や CI 固有の値** → `TF_VAR_` 環境変数で渡す（ファイルに書かずに済む）
- **全環境で共通のデフォルト** → `variable` の `default`

環境の分け方そのものは第 11 章で詳しく扱います。

---

## 7.4 output — module の戻り値

`output` ブロックは、module の外に値を公開します。

```hcl
output "bucket_name" {
  description = "アプリが参照するバケット名"
  value       = aws_s3_bucket.this.bucket
}
```

output には 2 つの使い道があります。

### 1. 人間や他のツールに値を見せる

`apply` が終わると、output の値が画面に表示されます。あとから見たいときは `terraform output` を使います。

```sh
terraform output                    # すべて表示
terraform output bucket_name        # 1 つだけ
terraform output -raw bucket_name   # 引用符なしの生文字列（シェルで使う用）
terraform output -json              # JSON（jq などで加工する用）
```

Terraform で作ったリソースの値を CI のスクリプトに渡す、といった使い方をします。

### 2. 親 module に値を返す

child module の output は、呼び出し側から `module.<名前>.<output名>` で参照できます。**module の外から見えるのは output だけ**です。child module 内の resource に外から直接触ることはできません。TS のモジュールで `export` したものしか import できないのと同じで、これがカプセル化になっています。

### output の値はどこから来るのか

`terraform output` は AWS に問い合わせているわけではありません。**state に保存された値を読んでいるだけ**です。state については次章で詳しく扱います。

---

## 7.5 sensitive の正体

DB のパスワードなど、画面に出したくない値には `sensitive = true` を付けます。

```hcl
variable "db_password" {
  type      = string
  sensitive = true
}
```

こうすると `plan` や `apply` の出力で値が `(sensitive value)` と伏せられます。sensitive な値を参照する output にも `sensitive = true` を付ける必要があり、付け忘れると Terraform がエラーで教えてくれます。

### 隠れるのは「表示」だけ

ここが**この章で最も大事な落とし穴**です。

`sensitive` は**表示を隠すだけ**で、値そのものは守っていません。

- **state ファイルには平文で保存されます**。公式ドキュメントにも「sensitive な output の値は state に保存される」と明記されています
- `terraform output -json` や `-raw` を使うと、sensitive な output も平文で表示されます
- plan ファイル（`plan -out` で保存したもの）にも値が含まれます

React で言えば、入力欄を `type="password"` にしても、コンポーネントの state には平文で入っているのと同じです。

つまり、`sensitive = true` を付けたから安心、とはなりません。**state を誰が読めるか**のほうがはるかに重要です。

---

## 7.6 秘密情報をどう扱うか

では秘密情報はどう扱えばよいのでしょうか。考え方は 3 段階あります。

### 段階 1: そもそも Terraform に秘密を通さない

最も安全なのは、**Terraform が秘密の値を一度も触らない**設計です。

AWS には秘密情報を保管する専用サービスがあります。

- **Secrets Manager** — パスワードや API キーを暗号化して保管し、自動ローテーションもできるサービス
- **SSM Parameter Store** — 設定値を階層的に保管するサービス。暗号化した値（SecureString）も置ける

たとえば RDS（AWS のマネージド DB）なら、`aws_db_instance` に `manage_master_user_password = true` を指定すると、**AWS 側がパスワードを生成して Secrets Manager に保管**します。Terraform はパスワードの中身を知らないまま DB を作れます。アプリは実行時に Secrets Manager から読みます。

### 段階 2: state に残さない — ephemeral 値と write-only 引数

どうしても Terraform が秘密を扱う場合のために、最近のバージョンで **「state にも plan ファイルにも残さない値」** の仕組みが入りました。

**ephemeral 値（Terraform 1.10〜）**

`variable` や `output` に `ephemeral = true` を付けると、その値は state にも plan ファイルにも保存されません。ただし、ephemeral な値を使える場所は限られています（provider の設定、後述の write-only 引数など）。また、root module の output には付けられません。

あわせて `ephemeral` ブロック（ephemeral resource）も導入されました。`resource` や `data` と並ぶ第 3 のブロックで、実行のたびに値を取得し、state には残しません。

```hcl
# Secrets Manager から読むが、state には残さない
ephemeral "aws_secretsmanager_secret_version" "db" {
  secret_id = "myapp/db-password"
}
```

**write-only 引数（Terraform 1.11〜）**

provider 側が「書き込み専用」と定義した引数です。値は apply 時に AWS へ送られますが、state には保存されません。AWS provider では `aws_db_instance` の `password_wo` などがこれにあたります。

```hcl
resource "aws_db_instance" "main" {
  # ...
  password_wo         = ephemeral.aws_secretsmanager_secret_version.db.secret_string
  # state に値が残らないので、Terraform は変更を検知できない。
  # パスワードを変えたいときはこの番号を上げて更新を知らせる
  password_wo_version = 1
}
```

代償もあります。state に値がない以上、Terraform は「今の値と違うか」を比べられません。だから `password_wo_version` のような**手動のバージョン番号**で変更を伝える必要があります。また、write-only 引数があるかどうかは provider とリソースごとに違うので、Registry のドキュメントで確認してください。

### 段階 3: state そのものを守る

write-only 引数がないリソースでは、秘密は state に入ってしまいます。どの段階を選んでも **state を安全な場所に置く**ことが前提で、これは次章のテーマです。

なお、秘密を書いた `.tfvars` や `default`、そして state ファイルを Git にコミットしてはいけないのは、`.env` と同じ理屈です。

---

## まとめ

- module は**関数**として見ると整理しやすい。variable = 引数、output = 戻り値、locals = ローカル変数
- variable には `type` を必ず書く。`validation` で plan の時点で不正な値を止められる
- 値の渡し方は複数あり、**`-var` が最強、`TF_VAR_` 環境変数は意外に弱い**
- output は外の世界と親 module への出口。値は state から読まれる
- **`sensitive` は表示を隠すだけで、state には平文で残る**
- 秘密はできるだけ Terraform に通さない。通すなら ephemeral 値と write-only 引数で state に残さない

次章では、ここまで何度も登場した「state」とは何なのか、Terraform の記憶の仕組みを掘り下げます。

---

[← 前: resource と data source、依存関係](06-resources-and-data.md) | [目次](../README.md) | [次: state — Terraform の記憶 →](08-state.md)
