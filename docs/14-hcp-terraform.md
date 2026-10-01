# 14. HCP Terraform とマネージドサービス

> **この章でわかること**
> 自前の CI/CD で足りなくなるものと、HCP Terraform が肩代わりするもの・その代償。Atlantis や他社サービスを含めた選び方。

前章では GitHub Actions で plan / apply の流れを組みました。小さなチームならそれで十分回ります。

しかし規模が大きくなると、CI のワークフローファイルに書ききれない要求が次々に出てきます。そこを製品として引き受けるのが、この章で扱うマネージドサービスです。

---

## 14.1 自前 CI で足りなくなるもの

前章の仕組みを 1 年運用すると、たいてい次のような要望が出てきます。

- **state 管理**: S3 バケットとロックの設定、暗号化、バージョニングを、state の数だけ面倒を見るのがつらい
- **権限**: 「チーム A は dev だけ apply できる。prd は SRE の承認が要る」を細かく設定したい
- **承認フロー**: plan を見たうえで、特定の人がボタンを押したら apply したい
- **監査**: 誰がいつ、どの plan を承認し、何が変わったかを一元的に追いたい
- **秘密管理**: 環境ごとの変数や認証情報を、リポジトリの外で安全に管理したい
- **ドリフト検知**: 誰かがコンソールで手作業したら、PR を待たずに気づきたい
- **ポリシー**: 第 13 章の OPA チェックを、全リポジトリに漏れなく強制したい

1 つ 1 つは GitHub Actions と AWS の機能を組み合わせれば作れます。ただし作るのも保守するのも自分たちです。

これは Web 開発で「自前の Jenkins を運用するか、GitHub Actions を使うか」と同じ判断です。**作れるかどうかではなく、作り続けたいかどうか**が問題になります。

---

## 14.2 HCP Terraform の基本

**HCP Terraform** は HashiCorp が提供する SaaS です。2024 年 4 月に **Terraform Cloud** から名前が変わりました。機能は同じで、HashiCorp のクラウド基盤（HCP: HashiCorp Cloud Platform）の一製品という位置づけになっただけです。古い記事で「Terraform Cloud」と書かれていたら、同じものだと読み替えてください。

### organization / project / workspace

HCP Terraform のリソースは 3 階層です。

```
organization（会社・組織単位。課金と SSO はここ）
  └─ project（チームやプロダクト単位のフォルダ）
       ├─ workspace: network-prd
       ├─ workspace: app-prd
       └─ workspace: app-dev
```

中心にあるのは **workspace** です。1 つの workspace は **1 つの state と、それを操作するための設定一式**（変数、権限、実行履歴、紐づくリポジトリ）を持ちます。

GitHub で言えば、organization → リポジトリのまとまり → リポジトリ、のような関係です。権限は project か workspace の単位で、チームに付与します。

注意したいのは、第 11 章の **CLI の workspace（`terraform workspace`）とは別物**だという点です。同じ名前で紛らわしいですが、HCP Terraform の workspace は「state ごとの作業場所」であり、第 11 章で扱ったディレクトリ分割の 1 ディレクトリに近い単位です。

---

## 14.3 run の流れと連携方法

HCP Terraform では、plan / apply を **HashiCorp 側のサーバで実行**します（remote 実行）。手元や CI ランナーで動かすのではありません。1 回の plan / apply を **run** と呼びます。

```
トリガー（PR、マージ、CLI、API）
   ↓
plan
   ↓
（ポリシーチェック / run task）
   ↓
承認（手動 or 自動）
   ↓
apply
```

### 3 つのワークフロー

**VCS 連携（VCS-driven）**
GitHub などのリポジトリを workspace に紐づけます。PR を作ると **speculative plan**（適用しない、見るだけの plan）が走り、結果が PR のステータスに出ます。マージすると本番の run が始まります。前章で手作りしたものが、設定だけで手に入るイメージです。

**CLI 連携（CLI-driven）**
コードに `cloud` ブロックを書くと、手元の `terraform plan` が HCP Terraform 上で実行されます。

```hcl
terraform {
  # state の保存先と実行場所を HCP Terraform にする宣言。
  # 手元で plan しても、実行と state は向こう側に置かれる
  cloud {
    organization = "my-company"
    workspaces {
      name = "app-dev"
    }
  }
}
```

コマンドは手元から打ちますが、実行と state は HCP Terraform 側です。手元に強い権限は要りません。

**API 連携（API-driven）**
API から run を起こします。既存の CI から HCP Terraform を呼ぶ構成に使います。

### 変数と認証情報

workspace には変数を登録でき、**sensitive** にすると画面からも API からも読めなくなります。これは workspace 変数の設定で、HCL の `sensitive`（第 7 章）とは別物です。値がリソースに渡れば state には入ります。複数の workspace で共通の値は **変数セット**（variable set）にまとめて、project や organization 単位で配れます。

AWS の認証には **dynamic provider credentials** という仕組みがあります。前章の GitHub Actions と同じく OIDC で、run ごとに一時的な認証情報を受け取ります。ここでも長期キーを置く必要はありません。

---

## 14.4 ガバナンスのための機能

チームが大きくなるほど効いてくる機能です。なお、**どの機能がどのプランで使えるかは改定されることがある**ので、導入時は公式の比較表を確認してください。

### Sentinel / OPA

第 13 章のポリシー as コードを、run の途中に組み込めます。HashiCorp 製の **Sentinel** と、汎用の **OPA** の両方を選べます。

ポリシーごとに強さ（enforcement level）を設定できるのが特徴です。Sentinel なら「警告だけ出す（advisory）」「止めるが権限者は上書きできる（soft mandatory）」「止める（hard mandatory）」の 3 段階、OPA なら advisory と mandatory の 2 段階です。ルールを段階的に厳しくしていく運用がしやすくなります。

### run task

plan の結果を外部サービスに渡して検査させる仕組みです。セキュリティスキャナやコスト管理ツールなど、サードパーティ製品と連携できます。CI のワークフローに 1 ステップ足す感覚です。

### private registry

社内向けの module（第 9 章）を配布する場所です。Terraform Registry の社内版で、npm でいえば **GitHub Packages のようなプライベートレジストリ**にあたります。バージョン付きで公開でき、`source` と `version` で参照します。

### no-code provisioning

private registry の module を、**コードを書かずに** UI から変数を入れるだけで使えるようにする機能です。「アプリチームが自分で検証用の DB を作る」といったセルフサービスを、プラットフォームチームが用意した module の範囲で実現できます。

### ドリフト検知

**health assessments** という機能で、定期的に裏で plan を取り、コードと現実がずれていないかを確かめます。手作業による drift（第 8 章）に、次の PR を待たずに気づけます。Standard / Premium プランの機能です。

### 監査

誰がどの run を起こし、誰が承認したかが記録されます。上位プランでは監査ログを外部に出力できます。

---

## 14.5 Stacks

**Stacks** は、workspace に代わる新しい管理単位で、複数の部品（component）と複数の環境（deployment）をまとめて扱う仕組みです。2025 年秋の HashiConf で **GA（一般提供）**になりました。第 11 章で触れた Terragrunt の Stacks とは別物です。

### 何を解決するのか

第 11 章で、state をネットワーク層・アプリ層のように分割し、環境ごとに用意するという話をしました。正しい設計ですが、数が増えると管理がつらくなります。

```
network-dev   network-stg   network-prd
app-dev       app-stg       app-prd
       ↑ app は network の出力を使う。適用順も守る必要がある
```

6 つの workspace の依存関係と適用順序を、人間が覚えておく必要があります。

Stacks では、この構成を 2 種類のファイルで宣言します。

- **`.tfcomponent.hcl`**: どの module を部品（component）として使い、部品同士がどうつながるか
- **`.tfdeploy.hcl`**: その構成を、どの環境（deployment）に、どの値で展開するか

React で例えるなら、component の組み合わせを一度定義して、props を変えて dev / stg / prd の 3 か所にマウントするイメージです。依存関係の順序は HCP Terraform が解決します。

### 代償

- **HCP Terraform（と対応する Terraform Enterprise）でしか使えません**。素の Terraform CLI と S3 backend だけでは動かないので、プラットフォームへの依存が強まります
- 料金はリソース数ベースのプランが前提で、**旧来のプランでは使えません**
- 比較的新しい仕組みなので、ノウハウや事例はまだ多くありません

これから HCP Terraform を始めるなら候補になりますが、入門段階では「workspace を分けて、依存は手で管理する」を理解してからで十分です。

---

## 14.6 料金と Terraform Enterprise

### 料金の考え方

現在の HCP Terraform は、**管理しているリソースの数**で課金されます。

- 課金対象は state 内の **managed なリソース**（`resource` ブロックで作ったもの）。data source は数えません
- **時間単位**で計算されます。リソースを作ってから壊すまでの時間に対して課金されるイメージです
- **Free** プランは管理リソース 500 個まで。ユーザ数の制限はありません
- 有料プランは **Essentials / Standard / Premium** の 3 段階で、上位ほど同時実行数や使える機能が増えます

以前はユーザ数ベースの古い無料プランもありましたが、2026 年 3 月末で終了し、現在の Free プランへ移行しました。

ここでの注意点は、**課金の単位がリソース数である**ことです。たとえば IAM ポリシーのアタッチメントやセキュリティグループのルールのような細かいリソースも、1 つずつ数えられます。アプリの規模に比べて請求が大きく感じられることがあります。導入前に、自分たちの state に managed リソースがいくつあるかを `terraform state list` で（`data.` で始まる行を除いて）数えておくとよいでしょう。

具体的な金額は変わることがあるので、公式の料金ページを確認してください。

### Terraform Enterprise

**Terraform Enterprise** は、HCP Terraform とほぼ同じものを**自社環境にセルフホスト**する製品です。

- データを外部の SaaS に置けない（金融、公共など）
- 社内ネットワークの中からしか触れないインフラを扱う

といった事情がある組織向けです。代わりに、そのサーバ自体の運用（アップグレード、バックアップ、可用性）を自分たちで負います。

---

## 14.7 代替サービスと選び方

Terraform の実行基盤は HashiCorp 製以外にも選択肢があります。

| 選択肢 | 形態 | 特徴 |
| --- | --- | --- |
| 自前 CI（GitHub Actions 等） | 自分で組む | 最も安く柔軟。機能はすべて自作 |
| Atlantis | OSS、セルフホスト | PR コメントで plan / apply。PR 単位のロック |
| HCP Terraform | SaaS | 開発元の製品。Stacks、Sentinel、private registry |
| Terraform Enterprise | セルフホスト | HCP Terraform の自社運用版 |
| Spacelift / env0 / Scalr | SaaS（一部セルフホストあり） | 他社製の実行基盤。OpenTofu や他の IaC ツールにも対応 |

Spacelift、env0、Scalr は、いずれも HCP Terraform と同じ領域（remote 実行、state 管理、ポリシー、承認フロー、ドリフト検知）をカバーする製品です。**OpenTofu（第 15 章）や Terragrunt、Pulumi など複数のツールを扱える**ことを強みにしているものが多く、課金の単位もサービスごとに異なります。

### 選び方の目安

**数人のチームで、state も数個**
→ 自前 CI で十分です。第 13 章の構成から始めてください。

**PR 中心の運用を手軽に強化したい。サーバを 1 台運用するのは苦にならない**
→ Atlantis。OSS なのでライセンス費用はかかりません。

**複数チームで権限・承認・監査が必要。運用は任せたい**
→ HCP Terraform か、他社 SaaS。

**OpenTofu への移行も視野に入れている**
→ OpenTofu をサポートする他社 SaaS が候補になります。HCP Terraform は Terraform 専用です。

**データを外に出せない**
→ Terraform Enterprise か、セルフホスト版のある他社製品。

最初から完璧な基盤を選ぶ必要はありません。どのサービスも中で動いているのは同じ Terraform（または互換の OpenTofu）なので、**コードは基本的にそのまま持ち運べます**。乗り換えで主に手間になるのは、state の移行と、プラットフォーム固有の機能（Stacks、Sentinel ポリシーなど）への依存です。固有機能に深く依存するほど、乗り換えのコストは上がります。

---

## まとめ

- 自前 CI は**作れるが、作り続ける必要がある**。権限・承認・監査・ドリフト検知が必要になったらマネージドサービスを検討する
- HCP Terraform（旧 Terraform Cloud）は **organization / project / workspace** の 3 階層。workspace は CLI の workspace とは別物
- plan / apply は **remote 実行**。VCS / CLI / API の 3 通りで起動でき、AWS 認証は OIDC で長期キー不要
- Sentinel / OPA、run task、private registry、no-code、health assessments で**ガバナンスを製品として得られる**
- **Stacks** は GA になった、複数の component と環境をまとめて扱う新しい単位。ただしプラットフォーム依存が強まる
- 料金は**管理リソース数ベース**。Free は 500 リソースまで。金額とプランごとの機能は公式で確認する
- 代替に Atlantis（OSS）、Spacelift / env0 / Scalr がある。**固有機能への依存度が乗り換えコストを決める**

次章では視野を広げて、CloudFormation、AWS CDK、Pulumi、OpenTofu など、Terraform 以外の IaC ツールと比べてみます。

---

[← 前: チーム開発と CI/CD](13-team-and-cicd.md) | [目次](../README.md) | [次: 他の IaC ツールとの比較 →](15-iac-tools-comparison.md)
