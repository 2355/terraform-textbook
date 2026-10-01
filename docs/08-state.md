# 8. state — Terraform の記憶

> **この章でわかること**
> Terraform がなぜ state を必要とするのか。drift とは何か。state をチームで安全に共有する方法（S3 backend とロック）と、state の扱いを誤ったときの怖さ。

ここまで何度も「state」という言葉が出てきました。`apply` したディレクトリに勝手にできている `terraform.tfstate` というファイルのことです。

Terraform の本当に怖い事故は、ほとんどが state 絡みです。逆に言えば、**state を理解すると Terraform の挙動の大半が説明できます**。

---

## 8.1 なぜ state が必要なのか

素朴な疑問から始めましょう。コードにはあるべき姿が書いてあり、AWS には現実がある。**両者を直接比べればいいのでは？**

それができない理由がいくつかあります。

### 理由 1: どれがどれかわからない

コードに次のように書いたとします。

```hcl
resource "aws_s3_bucket" "logs" {
  bucket_prefix = "myapp-logs-"
}
```

`bucket_prefix` を使うと、実際のバケット名は `myapp-logs-20261001xxxx` のように末尾が自動生成されます。次に `plan` したとき、AWS 上にある何十ものバケットのうち **どれが `aws_s3_bucket.logs` なのか**、コードだけでは判断できません。

state はこの**対応表**です。「`aws_s3_bucket.logs` は、AWS 上の ID `myapp-logs-20261001xxxx` のことだ」と記録しています。公式ドキュメントも、state の第一の目的を「リモートのオブジェクトと、設定内のリソースとの対応付け（binding）を保存すること」としています。

### 理由 2: コードから消えたものを覚えておく

コードから `resource` ブロックを削除すると、Terraform はそのリソースを destroy します。でも、**コードにもう書かれていないもの**をどうやって知るのでしょうか。

答えは state です。「前回はあったのに、今回のコードにはない」から削除すべきだとわかります。削除の順序（依存先より先に依存元を消す）も、state に記録された依存関係から決めます。

### 理由 3: 性能

数百のリソースを毎回 AWS API で一から探し回るのは遅すぎます。state があれば、ID で狙い撃ちして問い合わせできます。

### React の仮想 DOM と比べる

React は、前回の render 結果（仮想 DOM）を覚えておき、今回の render 結果と比べて、差分だけを実 DOM に反映します。

Terraform も似ています。

```
React                          Terraform
──────                         ─────────
コンポーネント (JSX)       ↔   .tf ファイル（あるべき姿）
前回の仮想 DOM             ↔   state（前回 apply した結果）
実 DOM                     ↔   AWS 上の実リソース
reconciliation             ↔   plan / apply
```

ただし、決定的な違いがあります。

React の仮想 DOM はメモリ上にあり、ページをリロードすれば消えて作り直されます。実 DOM を変えられるのも、基本的には React だけです。

Terraform の state は**ファイルとして永続化され、何年も使い続けられます**。そして AWS 上のリソースは、Terraform 以外の誰か（コンソールで操作する人、別のツール）も変更できます。この 2 点が、state 特有の問題を生みます。

---

## 8.2 state の中身を覗く

`terraform.tfstate` は JSON ファイルです。大幅に省略すると、こんな形をしています。

```json
{
  "version": 4,
  "terraform_version": "1.16.4",
  "serial": 12,
  "lineage": "3f1c...",
  "resources": [
    {
      "mode": "managed",
      "type": "aws_s3_bucket",
      "name": "logs",
      "provider": "provider[\"registry.terraform.io/hashicorp/aws\"]",
      "instances": [
        {
          "attributes": {
            "id": "myapp-logs-20261001xxxx",
            "arn": "arn:aws:s3:::myapp-logs-20261001xxxx",
            "region": "ap-northeast-1"
          }
        }
      ]
    }
  ]
}
```

注目してほしい点がいくつかあります。

- **`attributes` にリソースの全属性が入っている**。コードに書いていない computed な値（ARN など）も含みます。output の値も別の欄（`outputs`）に保存されています
- **`serial`** は更新のたびに増える番号です。古い state で新しい state を上書きしないための目印です
- **`lineage`** は別の state との取り違えを防ぐ ID です

### 秘密も平文で入っている

前章で見たとおり、`sensitive = true` を付けた値も**ここには平文で入ります**。RDS のパスワードを普通の `password` 引数で渡していれば、それもここにあります。

つまり **state ファイルは、それ自体が秘密情報**です。公式ドキュメントも、state をバージョン管理システムや保護されていない場所に置くと、データ消失や秘密の漏洩につながると警告しています。

### 手で編集しない

JSON なので手で書き換えられそうですが、**やってはいけません**。整合性が壊れると、存在するリソースを作り直そうとしたりします。変更には 8.6 節のコマンドを使います。

---

## 8.3 refresh と drift — 現実がずれるとき

### drift とは

誰かが AWS コンソールで、Terraform 管理下のセキュリティグループに手作業でルールを追加したとします。

すると、**コードと state と現実の 3 つがずれた状態**になります。これを **drift**（ドリフト、漂流）と呼びます。

```
コード:  ルール A
state:   ルール A        ← 前回 apply した時点の記憶
現実:    ルール A + B    ← 誰かがコンソールで B を足した
```

### refresh — 記憶を現実に合わせる

`terraform plan` は、差分を計算する前に **refresh** という処理を行います。state に記録された各リソースについて AWS に現状を問い合わせ、記憶を現実に合わせ直す処理です。

その結果、plan はこう判断します。

1. refresh で「現実にはルール B もある」と知る
2. コードにはルール A しかない
3. **ルール B を削除する plan を出す**

Terraform はコードを正とするので、**手作業の変更は次の apply で巻き戻されます**。これは IaC として正しい挙動ですが、緊急対応でコンソールから入れた設定が、翌日の何気ない apply で消える、という事故はここから起きます。

手作業で変更したなら、**同じ内容をコードにも反映する**のが原則です。

### refresh だけ行う

コードとの比較はせず、state を現実に合わせるだけの操作もあります。

```sh
# 現実と state の差分だけを確認する（何も変更しない）
terraform plan -refresh-only

# 確認したうえで state を現実に合わせる
terraform apply -refresh-only
```

昔の `terraform refresh` コマンドは、確認なしで state を書き換えるため現在は非推奨で、公式ドキュメントも `-refresh-only` を案内しています。drift への具体的な対処方針は第 12 章で扱います。

---

## 8.4 ローカル state の問題

何も設定しなければ、state はカレントディレクトリの `terraform.tfstate` に保存されます。これを**ローカル state** と呼びます。

一人で試すぶんには問題ありません。しかしチームで使うと、すぐに破綻します。

### 問題 1: 共有できない

A さんの PC にある state を、B さんは持っていません。B さんが同じコードで `apply` すると、Terraform は「何もない」と判断して**すべてを新規作成しようとします**。

Git にコミットすれば、と思うかもしれませんが、それは秘密を Git に入れることです。pull し忘れれば古い state で上書きもします。

### 問題 2: 同時に実行できてしまう

A さんと B さんが同時に `apply` すると、2 人が別々に AWS を変更し、別々に state を書き込みます。後から書いたほうが勝ち、もう片方の変更は state から消えます。

### 問題 3: なくなる

PC が壊れたら state も消えます。state を失うと、AWS 上のリソースはそのまま残っているのに、Terraform はそれらとの対応を忘れてしまいます。取り戻すには、すべてを 1 つずつ import し直すことになります（第 12 章）。

### 解決策: remote backend

これらを解決するのが **remote backend** です。state の保存先を共有ストレージに移し、**ロック**で同時実行を防ぎます。

選択肢はいくつかあります。

- **S3 backend** — AWS の S3 に保存する。AWS を使うなら最も一般的
- **HCP Terraform** — HashiCorp のマネージドサービス。state 管理以外の機能も持つ（第 14 章）
- その他、GCS（Google Cloud）、Azure Blob Storage など

---

## 8.5 S3 backend を設定する

S3 backend の設定は、`terraform` ブロックの中に `backend` ブロックとして書きます。

```hcl
terraform {
  backend "s3" {
    bucket       = "myapp-tfstate"
    key          = "prd/network/terraform.tfstate" # バケット内のパス
    region       = "ap-northeast-1"
    encrypt      = true
    # S3 だけでロックする（DynamoDB は不要）
    use_lockfile = true
  }
}
```

書き換えたら `terraform init` を実行します。既存のローカル state があれば、Terraform が新しい backend へ移行するか聞いてきます。公式ドキュメントは、移行の前に `terraform.tfstate` を手でバックアップしておくことを強く勧めています。

### ロック: DynamoDB から S3 ネイティブへ

**ロック**とは、誰かが state を更新している間、他の人の更新を待たせる仕組みです。

S3 backend のロックは、最近大きく変わりました。

- **従来**: DynamoDB（AWS の NoSQL データベース）のテーブルをロック用に別途用意し、`dynamodb_table` で指定していた
- **現在**: `use_lockfile = true` で、**S3 だけでロックできる**。state と同じ場所にロック用の `.tflock` ファイルを置く方式

S3 ネイティブのロックは Terraform 1.10 で導入され、1.11 で正式版（GA）になりました。同時に **DynamoDB を使うロックは非推奨**になり、公式ドキュメントには「将来のマイナーバージョンで削除される」と書かれています。

新しく作るなら `use_lockfile = true` を使ってください。古い記事には DynamoDB を使う書き方が多く残っています。

### 暗号化とバージョニング

state は秘密を含むので、置き場所のバケットは厳重に扱います。

- **`encrypt = true`** — state とロックファイルを S3 のサーバ側で暗号化する。KMS（AWS の鍵管理サービス）の鍵を指定することもできる
- **バケットのバージョニングを有効にする** — S3 は上書きされた古いバージョンを残せる。公式ドキュメントも、誤削除や人為的ミスから復旧するために「強く推奨」としている
- **パブリックアクセスをブロックし、IAM で読める人を絞る** — state を読める人は、中の秘密も読める

バージョニングは保険です。state を壊しても、1 つ前のバージョンに戻せます。

### backend の中では変数が使えない

`backend` ブロックには **variable や locals を書けません**。backend は他のすべてより先に初期化されるので、その時点ではまだ変数を評価できないからです。

環境ごとにバケットや `key` を変えたい場合は、値を空けておき、`init` 時に渡します。

```sh
terraform init -backend-config="key=dev/network/terraform.tfstate"
```

これを **partial configuration** と呼びます。

### 鶏と卵の問題

ここで気づいた人もいるでしょう。**state を置く S3 バケットは、誰が作るのか？**

Terraform で作るには state が要る。state を置くにはバケットが要る。典型的な鶏と卵の問題です。

よくある解決策は次のどちらかです。

- **バケットだけは手作業（または AWS CLI）で作る**。数年に一度しか触らないものなので割り切る
- **ブートストラップ用の小さな Terraform を用意する**。ローカル state でバケットを作り、その後そのバケットへ state を移す

どちらにしても**このバケットは特別扱い**です。Terraform で作るなら `prevent_destroy`（第 6 章）を付けておきます。

---

## 8.6 state を操作するコマンド

state は手で編集しない代わりに、`terraform state` サブコマンドで扱います。

```sh
# 管理しているリソースのアドレス一覧
terraform state list

# 特定のリソースの属性を表示
terraform state show aws_s3_bucket.logs

# state を JSON で標準出力に取り出す（remote backend でも使える）
terraform state pull > backup.tfstate

# Terraform の管理から外す（AWS 上のリソースは消さない）
terraform state rm aws_s3_bucket.logs
```

`list` と `show` は読むだけなので、いつでも安全に使えます。

`state rm` は注意が必要です。**AWS 上のリソースは残るのに、Terraform はその存在を忘れます**。コードに `resource` ブロックが残ったまま plan すると、Terraform は「ない」と判断して同じものをもう 1 つ作ろうとします。

ほかにも `state mv`（アドレスの付け替え）や `state replace-provider` などがあります。リファクタリングに関わるものは、より安全な `moved` / `removed` ブロックとあわせて第 12 章で扱います。

### ロックが残ったとき

apply の途中で CI が強制終了するなどすると、ロックが外れないまま残ることがあります。その場合は `terraform force-unlock <LOCK_ID>` で解除できます。

ただし、**本当に誰も実行していないことを確かめてから**にしてください。

### state を壊すということ

最後に、state の怖さをまとめておきます。

- **state を失う** → AWS 上のリソースは残るが、Terraform は何も知らない状態になる。全部 import し直し
- **古い state で上書きする** → その後に作ったリソースを Terraform が忘れる。次の apply で重複作成の恐れ
- **state と現実がずれたまま apply** → 想定外の削除や置き換えが起きる

`node_modules` は消しても `npm install` し直せば済みますが、state はそうはいきません。

だからこそ、remote backend、ロック、バージョニングは「あれば便利」ではなく**必須の装備**です。

---

## まとめ

- state は **コードと AWS 上の実リソースの対応表**。消えたリソースの検出や性能のためにも必要
- React の仮想 DOM に似ているが、**永続化され、Terraform 以外からも現実が変わる**点が違う
- plan の前には refresh が走り、手作業の変更（drift）は**次の apply で巻き戻される**
- state には**秘密が平文で入る**。Git にコミットしない、手で編集しない
- チームでは remote backend が必須。S3 backend なら **`use_lockfile = true` で S3 だけでロック**でき、DynamoDB ロックは非推奨
- state 用バケットは暗号化・バージョニング・アクセス制限を施し、特別扱いする

次章では、ここまで「関数」として眺めてきた module を、本格的に部品化して再利用する方法を見ていきます。

---

[← 前: variables と outputs](07-variables-outputs.md) | [目次](../README.md) | [次: module — インフラのコンポーネント化 →](09-modules.md)
