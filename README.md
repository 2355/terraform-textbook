# Web エンジニアのための Terraform 入門

Web アプリを書いてきた人が、**Terraform とインフラのコード化（IaC）の「考え方」を掴む**ための教科書です。

## この本の前提

- 読者は Terraform 未経験。クラウド（AWS）もほぼ触ったことがない
- ただし TypeScript / Node.js / React の経験はある
- 「動くコードを写経する」ことより、**なぜそういう設計なのかを理解する**ことを重視

なので、コードは最小限です。スマホでスキマ時間に読めるよう、1 節はだいたい 3〜10 分で読み切れる長さにしてあります。

クラウド未経験でも読めるように、第 2 章で Terraform を読むのに必要な AWS の基礎をまとめています。AWS をすでに知っている人は飛ばして構いません。

## 読み方

上から順に読むことを想定しています。前の章を参照する箇所もありますが、気になる章から読んでも大筋はつかめます。

全部通して **3〜4 時間**くらいのボリュームです。1 日で読み切る必要はありません。「1 日 1〜2 章」くらいのペースがちょうどいいはずです。

各章の冒頭に「この章でわかること」、末尾に「まとめ」があります。時間がないときはそこだけ拾ってください。

## 目次

### 第 1 部 — 前提を揃える

1. [インフラをコードにするということ](docs/01-why-iac.md)
   手作業のインフラ運用は何が辛いのか。宣言的にインフラを書くという発想。
2. [Terraform のための AWS 超入門](docs/02-cloud-basics.md)
   リージョン、IAM、VPC、代表的なサービス。Web アプリを動かすのに何が要るか。
3. [Terraform の全体像](docs/03-terraform-overview.md)
   CLI、provider、Registry、HCL、state。write → init → plan → apply という基本の流れ。得意なこと・苦手なこと。

### 第 2 部 — 基本を掴む

4. [最初の一歩: init から destroy まで](docs/04-first-steps.md)
   インストール、AWS 認証、S3 バケットを 1 つ作って消すまで。
5. [HCL を TypeScript 使いの目で読む](docs/05-hcl.md)
   型、式、`for`、関数。プログラミング言語ではなく設定言語であること。
6. [resource と data source、依存関係](docs/06-resources-and-data.md)
   依存グラフ、`count` と `for_each`、`lifecycle`、置き換えの怖さ。
7. [variables と outputs](docs/07-variables-outputs.md)
   引数と戻り値としての入出力。値の渡し方と秘密情報の扱い。
8. [state — Terraform の記憶](docs/08-state.md)
   なぜ state が要るのか。drift、remote backend、ロック。

### 第 3 部 — 構造化して育てる

9. [module — インフラのコンポーネント化](docs/09-modules.md)
   再利用の単位。Registry、バージョン固定、設計の勘所。
10. [実例で見る: Web アプリのインフラを組む](docs/10-real-world-example.md)
    API + DB + 静的フロントを AWS に載せると、何をどう分けて書くか。
11. [環境の分け方と state の分割](docs/11-environments.md)
    dev / stg / prd、workspaces とディレクトリ分割、blast radius。
12. [既存リソースの取り込みとリファクタリング](docs/12-import-and-refactor.md)
    `import`、`moved`、`removed`。壊さずに構造を変える。

### 第 4 部 — チームで運用する

13. [チーム開発と CI/CD](docs/13-team-and-cicd.md)
    PR で plan、マージで apply。OIDC、静的解析、テスト。
14. [HCP Terraform とマネージドサービス](docs/14-hcp-terraform.md)
    自前 CI では足りないもの。HCP Terraform と代替サービス。
15. [他の IaC ツールとの比較](docs/15-iac-tools-comparison.md)
    CloudFormation、CDK、Pulumi、OpenTofu。TS で書けるツールではだめなのか。
16. [ここから先へ](docs/16-next-steps.md)
    落とし穴チェックリスト、アップグレード、便利ツール、AI との付き合い方、学習の次の一歩。

## 補足

コード例は説明のための最小限のものです。そのままコピーして動くことを保証していません。手を動かすときは公式ドキュメントを併用してください。また、**AWS 上にリソースを作ると料金が発生します**。試したあとは `terraform destroy` で消すのを忘れないでください。

内容は 2026 年 10 月時点の情報に基づいています。Terraform や各サービスは変化が速いので、バージョンや料金などは公式の最新情報を確認してください。

- [Terraform 公式ドキュメント](https://developer.hashicorp.com/terraform/docs)
- [Terraform チュートリアル](https://developer.hashicorp.com/terraform/tutorials)
- [Terraform Registry](https://registry.terraform.io/)
- [AWS provider ドキュメント](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
