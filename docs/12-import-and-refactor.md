# 12. 既存リソースの取り込みとリファクタリング

> **この章でわかること**
> 手で作った既存リソースを Terraform の管理下に入れる方法（`import` ブロック）。リソースを壊さずにコードを組み替える方法（`moved` / `removed` ブロック）。drift への向き合い方と、大きな変更を安全に進める手順。

ここまでは「Terraform でゼロから作る」前提で話してきました。現実はそう綺麗ではありません。

コンソールで誰かが作った S3 バケット、昔のスクリプトで立てた RDS、とりあえず手で設定した DNS。Terraform を導入する現場には、たいてい**すでに動いているインフラ**があります。

そして Terraform のコードも、書いた瞬間から古くなります。名前を直したい、module にまとめたい、と思ったときに、**リソースを消さずにコードだけを組み替える**道具が必要です。

---

## 12.1 なぜ「取り込み」が必要なのか

第 8 章のとおり、Terraform は **state に記録されたものだけを「自分の管理物」と認識します**。

コンソールで作ったバケット `my-logs` があるとします。これと同じ内容の `resource` ブロックを書いて plan すると、どうなるでしょうか。

```
  # aws_s3_bucket.logs will be created
```

Terraform は「state にないから、新しく作る」と判断します。apply すると、同名のバケットはすでにあるのでエラーになります。名前の衝突が起きないリソースなら、**そっくりな 2 つ目が作られてしまいます**。

必要なのは「この resource ブロックは、AWS 上のこの既存リソースのことだ」と **state に紐付けを書き込む**ことです。これを **import** と呼びます。

実体はもうある。台帳に載せるだけです。

---

## 12.2 `import` ブロック

Terraform 1.5 から、import を**コードとして宣言**できるようになりました。

```hcl
import {
  to = aws_s3_bucket.logs
  id = "my-logs"   # AWS 側の識別子。S3 ならバケット名
}

resource "aws_s3_bucket" "logs" {
  bucket = "my-logs"
}
```

`to` は取り込み先の resource アドレス、`id` は AWS 側でそのリソースを特定する値です。**何を `id` に渡すかはリソースの種類ごとに違う**ので、Registry の各リソースのドキュメント末尾にある「Import」の節を確認してください。バケット名のこともあれば、`vpc-0abc...` のような ID のこともあります。

この状態で plan すると、次のように表示されます。

```
aws_s3_bucket.logs: Preparing import... [id=my-logs]
  # aws_s3_bucket.logs will be imported
```

**import も plan に出てくる**のがポイントです。通常の変更と同じく、PR でレビューしてからマージ・apply できます。

### import と同時に差分も出る

plan では「取り込む」だけでなく、**取り込んだ後にコードとの差分があれば、それも一緒に表示されます**。

たとえば実物のバケットにはタグが付いているのに、コードに書いていなければ「タグを消す」変更が plan に載ります。**import の plan は「取り込み + 0 件の変更」になるまでコードを直す**のが基本です。差分を残したまま apply すると、取り込んだ瞬間に本番の設定を書き換えることになります。

### コードを自動生成する: `-generate-config-out`

既存リソースの設定を全部手で書き起こすのは大変です。`resource` ブロックを書かずに `import` ブロックだけを置き、次のように実行すると、Terraform が実物の設定から HCL を生成してくれます。

```sh
terraform plan -generate-config-out=generated.tf
```

ただし注意点があります。

- 公式ドキュメント上は**実験的機能**の扱いです。生成結果の形式は今後変わる可能性があります
- 生成されるのは**実物の値をすべて書き出したコード**です。デフォルト値と同じ引数まで並び、参照（`aws_vpc.main.id` など）ではなくハードコードされた ID になります
- 互いに排他な引数が両方出力されるなど、そのままでは通らないこともあります

生成コードは**下書き**です。不要な引数を削り、ID を参照に置き換え、ファイルや module に整理してから使ってください。公式ドキュメントも、生成された HCL を出発点に手で整えていくことを勧めています。

### 複数をまとめて取り込む

Terraform 1.7 からは `import` ブロックに `for_each` が使えます。

```hcl
locals {
  buckets = {
    logs   = "my-logs"
    assets = "my-assets"
  }
}

import {
  for_each = local.buckets
  to       = aws_s3_bucket.this[each.key]
  id       = each.value
}
```

さらに Terraform 1.14 では、`.tfquery.hcl` ファイルに `list` ブロックを書き、`terraform query` コマンドで**既存リソースを検索して、`import` ブロックと `resource` ブロックをまとめて生成する**ワークフローが入りました。provider 側の対応が必要な新しい機能なので、使えるリソースの範囲は provider のドキュメントで確認してください。

### import ブロックはその後どうするか

apply が終われば、リソースは state に載っています。`import` ブロックはその後も残しておいて構いません。すでに取り込み済みなら何も起きないからです。いつ何を取り込んだかの記録として残すチームもあれば、整理のため消すチームもあります。

---

## 12.3 旧来の `terraform import` コマンド

`import` ブロックが入る前は、CLI コマンドで取り込んでいました。今も使えます。

```sh
terraform import aws_s3_bucket.logs my-logs
```

やっていることは同じですが、性質が大きく違います。

| | `terraform import` コマンド | `import` ブロック |
| --- | --- | --- |
| 実行のされ方 | **その場で state を直接書き換える** | plan → apply の流れに乗る |
| レビュー | できない（手元で叩いて終わり） | PR で plan を見せられる |
| 複数リソース | 1 回 1 個 | `for_each` でまとめて |
| 設定の自動生成 | なし | `-generate-config-out` |

コマンド版は、`git commit` を飛ばして本番サーバのファイルを直接書き換えるようなものです。結果が plan に現れないまま remote state が変わるので、**チームで運用しているなら `import` ブロックを使ってください**。

コマンド版の出番は、手元だけで試すときや、古い Terraform を使っている環境くらいです。

---

## 12.4 `moved` ブロック — 名前を変えても壊さない

Terraform を書いていると、リソースの名前を変えたくなります。

```hcl
# 変更前
resource "aws_s3_bucket" "b" { ... }

# 変更後: もっと分かりやすい名前に
resource "aws_s3_bucket" "logs" { ... }
```

コード上はただのリネームです。しかし Terraform から見ると、**`aws_s3_bucket.b` が消えて、`aws_s3_bucket.logs` が新しく現れた**ことになります。plan はこうなります。

```
  # aws_s3_bucket.b will be destroyed
  # aws_s3_bucket.logs will be created
```

**中身のデータごとバケットを消して作り直す**計画です。TypeScript なら IDE のリネームで済む話が、インフラでは実物の削除になります。React で `key` を変えるとコンポーネントが作り直されて state が消えるのと、同じ構造です。

これを防ぐのが `moved` ブロックです（Terraform 1.1 から）。

```hcl
moved {
  from = aws_s3_bucket.b
  to   = aws_s3_bucket.logs
}
```

これで plan は「state 上のアドレスを付け替えるだけ」になり、実物には何も起きません。

### よく使う場面

```hcl
# count / for_each 化: 単体のリソースを、マップの 1 要素として引き継ぐ
moved {
  from = aws_instance.web
  to   = aws_instance.web["primary"]
}

# module 化: ルートに直接書いていたリソースを module の中へ移す
moved {
  from = aws_s3_bucket.logs
  to   = module.logging.aws_s3_bucket.this
}
```

第 6 章の「count のインデックスずれ」を `for_each` に直したいときも、`moved` で既存の要素を新しいキーに対応付けられます。

なお、`moved` で変えられるのは基本的に**アドレス（名前や置き場所）**です。リソースの種類（`aws_xxx` の部分）をまたぐ移動は、provider が対応している場合に限られます。

### moved ブロックは消してよいか

自分のリポジトリだけで使っている構成なら、全環境で apply が終わった後に消しても構いません。

ただし**他人に配布している module では残してください**。公式ドキュメントも、消すのは破壊的変更になると注意しています。古いバージョンから一気に上げてくる利用者がいると、`moved` がないために削除と再作成が走ってしまうからです。npm パッケージで、古い export 名を互換のために残しておくのと同じ配慮です。

---

## 12.5 `removed` ブロックと `state` コマンド

### 管理から外すが、消さない

逆に「Terraform の管理から外したいが、実物は残したい」こともあります。別チームに引き渡す、別の state に移す、Terraform で管理するのをやめる、などです。

単にコードから `resource` ブロックを消すと、Terraform は**実物を destroy する**計画を立てます。そこで `removed` ブロック（Terraform 1.7 から）を使います。

```hcl
removed {
  from = aws_s3_bucket.legacy

  lifecycle {
    # false にすると state から外すだけで、AWS 上のバケットは残る
    destroy = false
  }
}
```

`resource` ブロックを消し、代わりにこれを置きます。plan には「state から外す（削除はしない）」と表示されます。`from` には module 全体を指定することもできます。

### `terraform state mv` / `state rm` との比較

`moved` や `removed` が入る前は、同じことを CLI でやっていました。

```sh
terraform state mv aws_s3_bucket.b aws_s3_bucket.logs   # moved 相当
terraform state rm aws_s3_bucket.legacy                 # removed 相当
```

違いは、12.3 の import コマンドとまったく同じです。

- **CLI はその場で state を書き換える**。plan に出ず、レビューできない
- 環境が複数あると、**各環境で同じコマンドを手で叩く**必要がある。1 つ忘れると、その環境だけ次の plan で destroy が出る
- ブロック版は**コードと一緒にマージされ、どの環境でも次の apply で自動的に適用される**

ブロックは「マイグレーションファイル」、CLI は「本番 DB に直接 SQL を流す」と考えるとわかりやすいでしょう。ORM のマイグレーションがリポジトリに残り全環境に同じ順で適用されるのと同じく、`moved` / `removed` も全環境に同じ変更を届けます。

`state mv` / `rm` が今も必要になるのは、ブロックでは表現できない操作（別の state ファイルへの移動など）や、壊れた state の修復くらいです。その場合も**事前に state のバックアップを取り**、作業後に plan で差分がないことを確かめてください。

---

## 12.6 drift への対処

第 8 章で見た drift は、誰かがコンソールでセキュリティグループのルールを足した、AWS 側の自動処理で値が変わった、などで起きます。

### まず現状を知る: `-refresh-only`

```sh
terraform plan -refresh-only
```

これは「**コードは見ずに、実物と state のずれだけを表示する**」plan です。通常の plan はずれとコードの差分が混ざって表示されるので、「誰かが手で変えたもの」を切り分けたいときに使います。

ずれを確認したら、方針は 3 つです。

**1. コードの通りに戻す**
手作業の変更が間違いだった場合です。普通に `terraform apply` すれば、コードの状態に戻ります。

**2. 実物の変更をコードに取り込む**
手作業の変更が正しかった場合（障害対応で急いで直した、など）です。コードを実物に合わせて書き換え、plan が差分なしになることを確認します。

**3. state だけを実物に合わせる**
`terraform apply -refresh-only` は、実物の値を state に書き込むだけで、インフラには何もしません。ただし**コードは変わらない**ので、次の通常の plan で「コードの通りに戻す」差分が出ます。結局は 1 か 2 を選ぶことになります。

### 方針として決めておくこと

drift は「直す」より「起こさない」ほうが大事です。

- **Terraform で管理しているものは、コンソールで変えない**とチームで合意する。本番アカウントではコンソールの書き込み権限自体を絞る
- 自動でずれる値（オートスケーリングで変わる台数など）は、第 6 章の `lifecycle { ignore_changes }` で**最初から無視する**
- 定期的に `plan -refresh-only` を流して、ずれを早めに見つける（HCP Terraform のドリフト検知は第 14 章で扱います）

---

## 12.7 大きな変更を安全に進める手順

module 化や state の分割のような大きなリファクタリングは、次の順で進めると事故が減ります。

**1. 変更前に plan が「差分なし」であることを確認する**
既に差分がある状態で始めると、リファクタリング由来の差分と混ざって見分けがつきません。drift があれば先に片付けます。

**2. リファクタリングと機能変更を同じ PR に混ぜない**
「module 化しつつ、ついでにインスタンスサイズも上げる」はやめてください。リファクタリングの PR は **plan が `moved` / `removed` / `import` だけになり、add / change / destroy が 0 件**になるのが理想です。そうなれば、コードは変わったが実物は何も変わらないことが保証されます。

**3. plan の destroy と replace を必ず探す**
`-` や `-/+` が 1 つでもあれば、`moved` の書き漏れを疑います。特にデータを持つリソース（RDS、S3）は、`prevent_destroy` を付けておくと、万一の destroy を apply 前にエラーで止められます。

**4. dev から順に apply する**
同じ変更を dev → stg → prd の順に当て、各環境で plan が想定通りかを見ます。環境ごとに state の中身が微妙に違うことは珍しくありません。

**5. 小さく刻む**
100 個のリソースを一度に動かすより、10 個ずつ 10 回のほうが、plan を人間が読み切れます。**読み切れない plan は、承認してはいけない plan です。**

---

## まとめ

- 既存リソースを管理下に入れるには、state に紐付けを書く **import** が必要。`import` ブロックなら plan に乗り、レビューできる
- `-generate-config-out` で設定を自動生成できるが、**生成コードは下書き**。整えてから使う
- リソースの改名や module 化は **`moved` ブロック**で。書かないと destroy → create になる
- 管理から外すだけなら **`removed` ブロック**で `destroy = false`
- `terraform import` / `state mv` / `state rm` はその場で state を書き換え、レビューできない。**チームではブロック版を使う**
- drift は `plan -refresh-only` で確認し、「コードに戻す」か「コードを合わせる」かを決める
- リファクタリングの PR は **add / change / destroy が 0 件**になることを目指し、機能変更と混ぜない

次章では、これまで手元で叩いてきた plan と apply を、チームで安全に回すための CI/CD の仕組みを見ていきます。

---

[← 前: 環境の分け方と state の分割](11-environments.md) | [目次](../README.md) | [次: チーム開発と CI/CD →](13-team-and-cicd.md)
