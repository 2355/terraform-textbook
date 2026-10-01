# 15. 他の IaC ツールとの比較

> **この章でわかること**
> CloudFormation、AWS CDK、Pulumi、OpenTofu など、Terraform 以外の IaC ツールの立ち位置。「TypeScript で書けるツールのほうが良いのでは？」という疑問への答えと、選び方の目安。

ここまで Terraform だけを見てきました。しかし IaC ツールは Terraform 1 つではありません。

TypeScript を書く人なら、一度はこう思ったはずです。**「HCL を覚えるより、TS で書けるツールを使えばいいのでは？」** この章ではその疑問に正面から答えます。

---

## 15.1 IaC ツールの地図

ツールを比べる前に、比べる軸を揃えておきます。軸は 3 つあります。

**1. 何で書くか** — HCL や YAML のような**設定言語**か、TypeScript や Python のような**汎用言語**か

**2. 何を対象にするか** — AWS 専用か、provider さえあれば何でも扱えるか

**3. 状態をどこに持つか** — 自分で state ファイルを管理するか、サービス側が持ってくれるか

この 3 軸で主要なツールを並べると、こうなります。

```
              書き方        対象         状態の持ち主
CloudFormation YAML/JSON    AWS 専用     AWS（スタック）
AWS CDK        汎用言語     AWS 専用     AWS（CloudFormation 経由）
Terraform      HCL          何でも       自分（state ファイル）
OpenTofu       HCL          何でも       自分（state ファイル）
Pulumi         汎用言語ほか 何でも       Pulumi Cloud か自分
```

もう 1 つ、別の層にいるツールもあります。サーバの**中身**を設定する Ansible と、Kubernetes の上でインフラを管理する Crossplane です。これらは 15.6 で扱います。

---

## 15.2 CloudFormation と AWS CDK — AWS 純正の道

### CloudFormation

AWS 自身が提供する IaC サービスです。YAML か JSON で**テンプレート**を書き、それを AWS に渡すと**スタック**という単位でリソースが作られます。

Terraform との最大の違いは、**state を AWS が持ってくれる**ことです。第 8 章で見た state ファイルの保管場所、ロック、暗号化、鶏と卵の問題。これらを一切考えなくて済みます。

他にも AWS 純正ならではの利点があります。

- **失敗したら自動でロールバック**する。途中で止まって中途半端な状態が残りにくい
- **変更セット**（Change Set）で、適用前に差分を確認できる。Terraform の `plan` に相当します
- 追加のツールを入れずに、AWS アカウントだけで使える

代償もあります。

- **AWS 以外は扱えない**。DNS は Cloudflare、監視は Datadog、という構成だと別のツールが要ります
- YAML は書くほど長くなる。繰り返しや条件分岐の表現力が弱く、大きなテンプレートは読むのがつらくなります
- ロールバックは安心材料ですが、**ロールバック自体が失敗して身動きが取れなくなる**こともあります

### AWS CDK

CloudFormation の YAML を手で書く代わりに、**TypeScript などの汎用言語で書けるようにした**ものが AWS CDK です。TypeScript、JavaScript、Python、Java、C#、Go に対応しています。

仕組みは**コンパイラ**に近いです。

```
TypeScript のコード  --cdk synth-->  CloudFormation テンプレート  --deploy-->  AWS
```

TS を書いて JS にトランスパイルするのと同じように、CDK のコードは CloudFormation テンプレートに変換されます。実際にリソースを作るのは CloudFormation です。

CDK の強みは**コンストラクト**という部品です。数行書くだけで、VPC、ロードバランサ、コンテナ実行環境、IAM ロールなどを、AWS 推奨の設定込みでまとめて作れます。React のコンポーネントライブラリに近い感覚です。

ただし、**数行のコードから数百行のテンプレートと数十個のリソースが生まれる**ことがあります。便利さの裏返しで、「実際に何が作られるのか」がコードから見えにくくなります。

---

## 15.3 Pulumi — 汎用言語で何でも

Pulumi は「Terraform のように何でも扱えて、CDK のように汎用言語で書ける」ツールです。

- TypeScript、JavaScript、Python、Go、.NET、Java などで書ける。YAML にも対応し、2026 年 8 月には **HCL も正式サポート**されました
- AWS 以外にも、GitHub や Cloudflare など幅広い provider がある。**Terraform の provider をブリッジして使う仕組み**も持っています
- `pulumi preview` で差分を見て、`pulumi up` で適用する。流れは Terraform の `plan` / `apply` と同じです

state は Pulumi も持ちます。保存先は、デフォルトでは **Pulumi Cloud**（Pulumi 社のマネージドサービス）です。S3 などに自分で置くこともできますが、その場合は保管場所や権限管理を自分で面倒を見ることになります。考え方は第 8 章・第 14 章の Terraform とほぼ同じです。

CDK と違い、Pulumi は CloudFormation のような中間層を挟みません。**Pulumi 自身が provider 経由でクラウドの API を叩きます。** 構造としては「汎用言語で書ける Terraform」と捉えるのが近いです。

---

## 15.4 「TS で書けるほうが良いのでは？」に答える

TS 使いにとって、CDK や Pulumi は魅力的に見えます。型補完が効き、慣れたエディタとテストツールが使え、npm のエコシステムも使えます。

それでも Terraform が広く選ばれ続けているのには理由があります。**HCL の「できなさ」が、そのまま利点になっている**からです。

### 汎用言語の自由度は、インフラでは諸刃の剣

汎用言語では、何でも書けます。

```ts
// 汎用言語なら、環境変数や外部 API の結果でリソース構成を変えられてしまう
const azs = process.env.STAGE === "prd" ? await fetchAzList() : ["a"];
for (const az of azs) {
  new Subnet(this, `subnet-${az}`, { /* ... */ });
}
```

このコードを PR で読んだとき、**実際にいくつのサブネットができるか**は、実行してみないとわかりません。

HCL ではユーザ定義関数が書けず、任意のコードも実行できません（第 5 章）。できることが限られているので、**コードを読めば何が作られるかがほぼわかります。** インフラの変更レビューでは、この「読めばわかる」が何より効きます。

React に例えるなら、JSX が「ただの JavaScript」だからこそ、render の中に副作用を書けてしまう問題に似ています。自由度が高いほど、規律はチームに委ねられます。

### 公平に言えば、汎用言語にも明確な利点がある

一方的に HCL が優れているわけではありません。

- **抽象化の表現力**。クラス、継承、ジェネリクスで、複雑なパターンを部品化しやすい
- **テストの書きやすさ**。普段の Jest / Vitest の感覚でユニットテストが書ける
- **アプリと同じ言語**。アプリ開発者がインフラにも手を出しやすい

逆に言えば、HCL の弱点はこの裏返しです。複雑なロジックを書こうとすると `for` 式と三項演算子が入り組み、TS より読みにくくなることもあります。

### 判断の軸

結局のところ、問いは「**誰が読み、誰がレビューするか**」です。

- インフラ担当者や複数チームがレビューする → 制約のある HCL のほうが安全
- アプリ開発者が自分のサービスのインフラを書き、抽象化を積極的に使いたい → 汎用言語が活きる

そしてどちらを選んでも、**plan / preview を人間が読む**という本質は変わりません。

---

## 15.5 Terraform の親戚 — CDK for Terraform と OpenTofu

### CDK for Terraform は終了した

かつて HashiCorp は **CDK for Terraform**（CDKTF）を提供していました。TypeScript などで書いたコードを Terraform 用の設定に変換し、Terraform で実行するツールです。「CDK の書き心地で Terraform の provider を使える」という触れ込みでした。

しかし CDKTF は **2025 年 12 月 10 日に開発終了（sunset）し、GitHub リポジトリはアーカイブ**されました。HashiCorp は十分な利用が広がらなかったことを理由に挙げています。

今後の更新や互換性対応はありません。公式は移行先として次を案内しています。

- `cdktf synth --hcl` で HCL を生成し、通常の Terraform に移る
- すでに AWS CDK と密に組み合わせているなら AWS CDK に寄せる

古い記事で CDKTF を勧めているものがあっても、**新規採用はしない**でください。

### OpenTofu

第 3 章で触れたとおり、2023 年の Terraform のライセンス変更をきっかけに生まれたフォークが **OpenTofu** です。Linux Foundation 傘下で開発され、2025 年 4 月には CNCF の Sandbox プロジェクトにもなっています。ライセンスはフォーク前と同じ MPL 2.0 です。

コマンドは `terraform` の代わりに `tofu` を使います。HCL、provider、state の形式は共通の土台を持っており、**多くの Terraform コードはそのまま動きます。**

一方で、フォーク後は両者が別々に機能を足しています。OpenTofu 独自の機能には次のようなものがあります。

- **state の暗号化**。state ファイルそのものをクライアント側で暗号化できる（第 8 章の「state に秘密が入る問題」への一つの答え）
- `provider` ブロックでの `for_each`（複数リージョン・複数アカウントの設定が書きやすい）
- 特定リソースを操作対象から外す `-exclude` オプション

逆に、Terraform 側の新機能が OpenTofu にない（あるいは別の形で実装されている）こともあります。**「完全互換」ではなく「同じ祖先を持つ別製品」**と考えてください。npm と、npm 互換をうたう別のパッケージマネージャの関係に近いものです。

選ぶ際の主な観点は 2 つです。

- **ライセンス**。Terraform の BSL が自社の使い方に影響するかどうか（多くの利用者は影響しませんが、Terraform を組み込んだ製品を提供する場合などは要確認）
- **周辺サービス**。HCP Terraform（第 14 章）を使うなら Terraform が自然です

---

## 15.6 隣の領域 — Ansible と Crossplane

### Ansible: サーバの「中身」を整える

第 3 章で「サーバ内部の設定は Terraform の苦手分野」と書きました。そこを担うのが Ansible のような**構成管理ツール**です。

```
Terraform : EC2 インスタンスを「作る」
Ansible   : そのインスタンスに SSH して、パッケージを入れ、設定ファイルを置く
```

Ansible は YAML の **playbook** にタスクを上から順に書きます。考え方は宣言的に寄せていますが、**実行順に意味がある**点でシェルスクリプトに近い面もあります。また Terraform のような state は持たず、毎回サーバの現状を見て必要な変更を加えます。

コンテナや Lambda が主流になり、「サーバの中身を育てる」場面自体が減りました。Web アプリの新規構築では、Ansible の出番は少なくなっています。

### Crossplane: Kubernetes をインフラの司令塔にする

Kubernetes を中心に据えている組織では、**Crossplane** という選択肢もあります。AWS のリソースを Kubernetes のリソース（YAML マニフェスト）として定義し、Kubernetes のコントローラが**常に**現実をあるべき姿に寄せ続けます。

Terraform は「`apply` したときだけ」差分を埋めます。Crossplane は**常時 reconcile** し続けます。第 1 章の reconciliation の考え方を、そのまま常駐させたものです。

ただし Kubernetes クラスタの運用が前提になるため、Kubernetes を使っていないチームが入れる理由はほぼありません。

---

## 15.7 選び方の目安

最後に、状況別の目安を表にまとめます。

| 状況 | 第一候補 | 理由 |
| --- | --- | --- |
| AWS 以外のサービスも IaC で管理したい | Terraform / OpenTofu | provider の幅が最も広い |
| AWS だけ、state 管理をしたくない | CloudFormation / AWS CDK | state を AWS が持つ |
| AWS だけ、アプリ開発者が TS で書きたい | AWS CDK | コンストラクトで少ない記述量 |
| マルチクラウドで、汎用言語で抽象化したい | Pulumi | 汎用言語 + 幅広い provider |
| ライセンスを重視し、Terraform の資産も活かしたい | OpenTofu | MPL 2.0、HCL と互換の土台 |
| サーバ内部の設定を管理したい | Ansible | 構成管理が本業 |
| Kubernetes 中心の組織 | Crossplane | 常時 reconcile |

迷ったときの判断基準は 3 つです。

**1. チームがすでに使っているものを優先する。** ツールの優劣より、知見とコードの蓄積のほうが価値があります。

**2. 求人やドキュメントの量を見る。** Terraform は情報量が多く、困ったときに調べやすいツールです。

**3. 1 つの組織で混ぜすぎない。** 「ネットワークは Terraform、アプリは CDK」のような分担はありえますが、境界が増えるほど、値の受け渡しと責任の所在が難しくなります。

---

## まとめ

- IaC ツールは「**何で書くか**」「**何を対象にするか**」「**状態をどこに持つか**」の 3 軸で比べる
- CloudFormation / AWS CDK は **state を AWS が持つ**のが最大の利点。代わりに AWS 専用
- Pulumi は**汎用言語で書ける Terraform** に近い。state は Pulumi Cloud か自前で持つ
- HCL の**表現力の低さは、レビューのしやすさ**と表裏一体。汎用言語の自由度は諸刃の剣
- **CDK for Terraform は 2025 年 12 月に開発終了**。新規採用しない
- OpenTofu は Terraform と**同じ祖先を持つ別製品**。多くのコードは動くが、機能差はある

次章では、実際に Terraform を使い始めるときの落とし穴と、ここから先の学び方をまとめます。

---

[← 前: HCP Terraform とマネージドサービス](14-hcp-terraform.md) | [目次](../README.md) | [次: ここから先へ →](16-next-steps.md)
