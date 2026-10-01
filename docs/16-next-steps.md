# 16. ここから先へ

> **この章でわかること**
> 実務でつまずきやすい落とし穴のチェックリスト。Terraform と provider のアップグレードの進め方。開発体験を上げるツール、AI との付き合い方、次の学び方。

最後の章です。ここまでで Terraform の考え方と道具は一通り揃いました。あとは実際に手を動かすときの装備を整えます。

---

## 16.1 落とし穴チェックリスト

本書に出てきたものも含め、実務で事故になりやすい点をまとめます。apply の前に、ここを引いてください。

### お金

- [ ] 検証で作ったリソースを `destroy` したか。**NAT ゲートウェイ、RDS、ロードバランサ**は置いておくだけで課金が続く
- [ ] AWS Budgets のアラートを設定したか（第 2 章）
- [ ] 料金が大きく動く変更か、事前に見積もったか（16.3 の Infracost）

### 消える・置き換わる

- [ ] plan に `-/+`（置き換え）や `destroy` が含まれていないか。**最後の 1 行の件数だけでなく、中身を読む**
- [ ] DB や state 用バケットなど、消えたら困るものに `prevent_destroy` を付けたか（第 6 章）
- [ ] リソース名の変更や module 化で、`moved` ブロックを書き忘れていないか（第 12 章）
- [ ] `count` の要素を途中から削除して、後ろの要素が全部作り直しになっていないか（第 6 章）

### state とロック

- [ ] state をローカルに置いたままにしていないか（第 8 章）
- [ ] ロックが残って plan できないとき、**本当に誰も実行していないか確認してから** `terraform force-unlock` を使う
- [ ] 1 つの state が大きくなりすぎて、plan に何分もかかっていないか（第 11 章）

### コードの書き方

- [ ] VPC ID やアカウント ID を**ハードコード**していないか。data source や variable で受け取る
- [ ] タグを付けているか。AWS provider の `default_tags` を使えば、全リソースに一括で付けられる
- [ ] 秘密情報をコードや tfvars に直書きしていないか（第 7 章）
- [ ] provider と Terraform 本体のバージョンを固定しているか

タグは軽視されがちですが、**「この請求は誰のどのリソースか」を後から追える唯一の手がかり**になります。最初から付けてください。

```hcl
provider "aws" {
  region = "ap-northeast-1"

  # 個々の resource に書き忘れても、コスト配分と持ち主の追跡ができるようにする
  default_tags {
    tags = {
      Project     = "my-app"
      Environment = "dev"
      ManagedBy   = "terraform"
    }
  }
}
```

---

## 16.2 アップグレードの進め方

Terraform のコードは、書いて終わりではありません。Terraform 本体と provider は頻繁に更新されます。放置すると、ある日まとめて上げることになり、苦しみます。

### バージョン指定の基本

```hcl
terraform {
  # 本体は 1.x の範囲で、チーム全員が同じ前提に立てるようにする
  required_version = "~> 1.11"

  required_providers {
    aws = {
      source  = "hashicorp/aws"
      # メジャーアップは破壊的変更を含むので、意図して上げるまで止めておく
      version = "~> 6.0"
    }
  }
}
```

`~>` は npm の `^` や `~` に近い制約です。`~> 6.0` なら 6.x の範囲で上がり、7.0 には上がりません。

実際に使うバージョンは `.terraform.lock.hcl` に記録され、制約の範囲内で上げるときは `terraform init -upgrade` を使います（第 4 章）。

### provider のアップグレード

マイナー・パッチは比較的安全ですが、**メジャーバージョンアップは破壊的変更を含みます。** 引数の削除、リソースの分割、デフォルト値の変更などです。

進め方はこうです。

1. provider の **アップグレードガイド**（Registry のドキュメントに公開される）を読む
2. ブランチで制約を上げ、`terraform init -upgrade` する
3. **plan の差分がゼロ**（または意図したものだけ）になるまでコードを直す
4. 影響の小さい環境（dev）から順に apply する

ポイントは 3 です。**アップグレードだけの PR では、インフラは何も変わらないのが正解**です。差分が出たら、それは非互換の兆候です。依存パッケージを上げた PR でスナップショットテストの差分を確認するのと同じ感覚です。

### Terraform 本体のアップグレード

Terraform は 1.x の間、後のバージョンへ上げても既存の設定がそのまま動くよう互換性を保つ方針を公式に示しています（v1 compatibility promises）。とはいえ、**一度に何バージョンも飛ばさない**ほうが安全です。各バージョンのリリースノートとアップグレードガイドを確認しながら、少しずつ上げてください。

注意点が 1 つあります。新しい Terraform で apply すると、**state が新しい形式で書き込まれ、古い Terraform では読めなくなる**ことがあります。CI とローカルのバージョンを揃えてから上げましょう。第 4 章の tfenv / mise がここで効きます。

---

## 16.3 開発体験を上げるツール

### エディタ

VS Code なら、HashiCorp 公式の **HashiCorp Terraform** 拡張を入れてください。シンタックスハイライト、補完、定義ジャンプ、フォーマットが使えます。provider のリソースの引数も補完されるので、ドキュメントを開く回数が減ります。

### terraform-docs

module の variables と outputs から、**README の表を自動生成**するツールです。第 9 章の「module のインタフェース」を文書化する手間がなくなります。TypeDoc に近い存在です。

### pre-commit

コミット前にチェックを走らせる仕組みです。Husky + lint-staged の Terraform 版だと思ってください。`terraform fmt`、`validate`、TFLint、terraform-docs などをまとめて実行できる **pre-commit-terraform** というフック集がよく使われます。

CI（第 13 章）と同じチェックを手元でも走らせると、「CI で落ちてから直す」往復が減ります。

### Infracost

**plan からコストの増減を見積もる**ツールです。CLI は無料で使え、GitHub Actions などに組み込むと、PR に「この変更で月額がどれくらい増えるか」をコメントしてくれます。

インフラの変更は、**コードレビューで見落とすと請求書で気づく**ことになります。コストを PR の段階で見えるようにする価値は大きいです。

---

## 16.4 AI とインフラコードの付き合い方

Claude Code のような AI コーディングエージェントは、Terraform のコードを書くのが得意です。リソースの引数を調べ、module の雛形を作り、エラーメッセージから原因を探す。手作業よりずっと速くなります。

ただし、インフラのコードには**アプリのコードと違う怖さ**があります。

- アプリのバグは、デプロイし直せば直る。インフラの `destroy` は、**データごと消えて戻らない**ことがある
- AI は古い書き方や、存在しない引数を自信満々に書くことがある。provider は頻繁に変わるからです

だから原則は 1 つです。**AI に書かせてよいが、plan は必ず人間が読む。**

具体的には次のように線を引くとよいでしょう。

- **任せてよい**: コードの生成、リファクタリング、`fmt` / `validate` / `plan` の実行、エラーの調査
- **人間が判断する**: `apply`、`destroy`、`state` を書き換える操作、IAM 権限の変更

AI エージェントに渡す認証情報も、**plan に必要な読み取り権限だけ**にしておくのが安全です。第 2 章の最小権限の考え方は、人間だけでなく AI にも当てはまります。

HashiCorp は、AI が Registry の最新ドキュメントや HCP Terraform の情報を参照できるようにする **Terraform MCP server** も公開しています。古い知識で書かれる問題を減らす助けになります。

---

## 16.5 学ぶ順番の提案

この本を読み終えたあと、手を動かすなら次の順番を勧めます。

**1. AWS アカウントを用意し、Budgets アラートを設定する**
AWS には新規アカウント向けの無料プランやクレジット、一部サービスの無料枠があります。内容は時期によって変わるので、公式ページで確認してください。まず「使いすぎたら気づける」状態を作ります。

**2. S3 バケットを 1 つ作って消す**
第 4 章の実践です。init → plan → apply → destroy を一周します。plan の記号が読めるか確認してください。

**3. state を S3 backend に移す**
ローカル state から remote state に移す作業を体験します。第 8 章が腹落ちします。

**4. 手で作ったリソースを import する**
コンソールで 1 つリソースを作り、`import` ブロックで取り込みます。第 12 章の実践です。

**5. 小さな Web 構成を組む**
第 10 章の構成を縮めたもの（静的サイト + CloudFront など）を、module に分けて作ります。

**6. GitHub Actions で plan を PR に出す**
OIDC で認証し、PR ごとに plan を見られるようにします。第 13 章の実践です。

**公式チュートリアル**も充実しています。HashiCorp Developer の Terraform チュートリアルは、AWS 向けの入門コースから順に進められます。

### 資格

腕試しをしたいなら、HashiCorp の公式資格 **HashiCorp Certified: Terraform Associate** があります。2026 年 10 月時点の最新版は **004** で、Terraform 1.12 を前提にした選択式の試験です。上位には、実技を含む **Terraform Authoring and Operations Advanced** もあります。

試験のバージョンや名称は改訂されます。受験前に公式ページを確認してください。

---

## 16.6 参考リンク

**公式**

- [Terraform ドキュメント](https://developer.hashicorp.com/terraform/docs)
- [Terraform チュートリアル](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Registry](https://registry.terraform.io/) — provider と module のドキュメント
- [AWS provider ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs) — 引数を調べるときに最も開くページ
- [HashiCorp 認定資格](https://developer.hashicorp.com/certifications/infrastructure-automation)

**ツール**

- [TFLint](https://github.com/terraform-linters/tflint)
- [terraform-docs](https://terraform-docs.io/)
- [pre-commit-terraform](https://github.com/antonbabenko/pre-commit-terraform)
- [Infracost](https://www.infracost.io/)
- [Terraform MCP server](https://developer.hashicorp.com/terraform/mcp-server)

**周辺**

- [OpenTofu](https://opentofu.org/)
- [AWS 無料利用枠](https://aws.amazon.com/free/)

---

## まとめ

- apply の前に**落とし穴チェックリスト**を引く。特に置き換え・削除・消し忘れ課金
- **タグは最初から付ける**。`default_tags` で一括設定できる
- アップグレードは**少しずつ、plan の差分ゼロを確認しながら**進める
- エディタ拡張、terraform-docs、pre-commit、Infracost で**レビュー前に問題を潰す**
- AI には書かせてよいが、**plan は人間が読み、apply と destroy は人間が判断する**
- 学ぶなら、**Budgets を設定してから**小さく作って消すところから

この本では「なぜそうなっているのか」を中心に扱いました。構文は調べれば出てきますが、**設計の理由は調べても出てこない**からです。

最後に、Terraform を書く上で繰り返し効いてくる原則を 5 つだけ再掲します。

**1. インフラはコードの関数である。**
あるべき姿をコードに書き、差分を埋めるのは Terraform に任せます。

**2. plan を読むまで apply しない。**
plan は Terraform が見せてくれる唯一の「未来」です。読み飛ばした差分は、本番で現実になります。

**3. コンソールで直さない。**
手作業の変更は、次の apply で消えるか、drift として残り続けます。

**4. state は爆発半径で分ける。**
1 つの失敗が巻き込む範囲を、state の単位で小さく保ちます。

**5. 迷ったら小さく始める。**
module も環境分割も、増やすのは簡単で、減らすのは難しい。

お疲れさまでした。

---

[← 前: 他の IaC ツールとの比較](15-iac-tools-comparison.md) | [目次](../README.md)
