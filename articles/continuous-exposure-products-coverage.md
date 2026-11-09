---
title: "CNAPP や Security Hub は、本番のセキュリティをどこまで継続的に見てくれるのか"
emoji: "🕸️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","Security","CNAPP","SecurityHub","Compliance"]
published: false
---

:::message
この記事は、Claude と一緒に公開されている資料や各社のドキュメントを調べ、整理した内容をまとめたものです。各記述の根拠は、記事中のリンク先で確認できます。製品は実際には試していません。
:::

## はじめに

【ハジメが書く：この記事を書こうと思ったきっかけ。前の記事（年 1 回のアセスメントを AI エージェントで継続的な検査にする話）を書いたあと、「Wiz のような製品なら、構成のトポロジーを作って CSPM 以上のことができるらしい」と聞いて、製品はどこまでやってくれるのかが気になった、など】

https://zenn.dev/anfangd/articles/continuous-security-assessment-ai-agent

## TL;DR

- 「年 1 回の点検を継続的なサイクルにする」流れは、枠組み（Gartner の CTEM：継続的な脅威エクスポージャー管理）、製品（CNAPP、AWS Security Hub の exposure findings）、監査（FedRAMP）のどれでも進んでいます
- CNAPP の強みは、設定・権限・ネットワーク・脆弱性を 1 つのグラフに載せて、**組み合わせを攻撃経路として**見せることです。2026 年には、修正まで担うエージェントも出てきました
- CNAPP や exposure findings が見るのは**設定の組み合わせ**までです。設計書や体制表といった**文書**と、**運用の実態**は対象外です
- 設定の中でも、対象のリソースや見方によっては拾えない穴があります。気づいた穴をルールに落とすのは、製品を入れても人とエージェントの仕事として残ります

## 年 1 回の点検を、継続的なサイクルにする動きはあちこちで進んでいる

### CTEM は、点検を「範囲を決めて、直すまで」のサイクルにする

Gartner は、**継続的な脅威エクスポージャー管理（CTEM）** という考え方を打ち出しています。デジタルと物理の資産について、届くか・露出しているか・悪用できるかを継続的に評価する、体系的なアプローチです。範囲をインフラの部品ではなく、攻撃の経路や事業のプロジェクトに合わせて決めるのが特徴です。

CTEM は、範囲の決定（scoping）・発見（discovery）・優先順位付け（prioritization）・検証（validation）・対応の実行（mobilization）の 5 段階を繰り返すサイクルとして説明されます。製品ではなくプログラムの枠組みです。関連する製品の分類として、Gartner は露出を継続的に見つけて優先順位を付ける **Exposure Assessment Platforms（EAP）** を評価しています。

- [Gartner Identifies the Top Cybersecurity Trends for 2024](https://www.gartner.com/en/newsroom/press-releases/2024-02-22-gartner-identifies-top-cybersecurity-trends-for-2024)
- [How to Manage Cybersecurity Threats, Not Episodes - Gartner](https://www.gartner.com/en/articles/how-to-manage-cybersecurity-threats-not-episodes)
- [Exposure Assessment Platforms Reviews and Ratings - Gartner](https://www.gartner.com/reviews/market/exposure-assessment-platforms)

### 監査の側も、継続的に報告する方向に動いている

米国政府のクラウド認証制度 FedRAMP は、2026 年 6 月に公開した Consolidated Rules for 2026 で、認証後も 3 か月ごとに継続的なモニタリングの報告を出すよう求めています。その根拠になっている米国行政管理予算局（OMB）の通達 M-24-15 は、証拠をできるだけ機械で読める形で受け取ることを求め、形式として NIST の OSCAL を挙げています。

### 検出結果の形式も、共通のスキーマにそろいつつある

検出結果の形式では、**OCSF（Open Cybersecurity Schema Framework）** が広がっています。ベンダーに依存しない、セキュリティのイベントと検出結果のスキーマです。Security Hub の検出結果も OCSF で出力されます。

- [Collaborative Continuous Monitoring - FedRAMP Consolidated Rules for 2026](https://www.fedramp.gov/2026/reference/collaborative-continuous-monitoring/)
- [M-24-15 Section V. Automation and Efficiency - FedRAMP](https://www.fedramp.gov/2026/authority/m-24-15/automation/)
- [Security Hub and the Open Cybersecurity Schema Framework (OCSF)](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-ocsf.html)

## 比べる軸は「何を証拠に見るか」と「組み合わせをどう見るか」

前の記事では、セキュリティ基準の項目を「何を証拠に判断するか」で、**設定・IaC**、**文書**、**運用の実態**の 3 つに分けました。製品を比べるときも、この軸がそのまま使えます。

加えて、設定の中でも次の 2 つで差が出ます。

- **組み合わせの見方**：リソースを 1 つずつ見るのか、リソース同士の関係をたどって攻撃経路として見るのか
- **確かめ方**：設定から「届きうる」と計算するだけか、外から実際に接続して確かめるのか

## CNAPP は、設定の組み合わせをグラフで見る

**CNAPP（Cloud-Native Application Protection Platform）** は、CSPM（設定の評価）、CWPP（ワークロードの保護）、CIEM（権限の管理）などを 1 つの製品にまとめたものです。支えている技術は、主に次の 3 つです。

| 技術 | やっていること | 弱いところ |
|---|---|---|
| エージェントレスのスキャン | クラウドの API でディスクのスナップショットを取り、ワークロードに何も入れずに脆弱性や秘密情報を調べる。止まっている VM も見られる | 定期的なスキャンなので、メモリ上だけの脅威やリアルタイムの検知には向かない |
| セキュリティグラフ | リソース・ID・権限・ネットワーク経路・脆弱性を 1 つのグラフに載せ、重要な資産に届く**攻撃経路**を探す。低リスクの問題が連鎖して高リスクになる組み合わせ（toxic combination）を挙げる | グラフに載るのは、製品が読める設定やログだけ |
| ランタイムのセンサー | eBPF などで稼働中のワークロードの動きを取り、グラフ上の経路が本当に悪用できるかを確かめる | エージェントを入れる手間がかかる |

代表的なのが Wiz で、Security Graph を中心に据えています。ほかに、Palo Alto Networks の Cortex Cloud（旧 Prisma Cloud）、Microsoft Defender for Cloud、CrowdStrike Falcon Cloud Security、SideScanning という方式でエージェントレスのスキャンを打ち出している Orca Security などがあります。

- [What is attack path analysis? - Wiz](https://www.wiz.io/academy/detection-and-response/attack-path-analysis)
- [Orca SideScanning - Orca Security](https://orca.security/platform/agentless-sidescanning-technology/)

### Wiz は、ネットワークの露出を計算し、外からも確かめる

Wiz は、ロードバランサー、ファイアウォール、VPC、サブネットなどのネットワークルールを解析して、実際に外から届くリソースを計算します。さらに dynamic scanner で外から接続し、ポートやプロトコルが本当に開いているかを確かめます。

設定ルールは 2,800 以上が用意されていて、足りないところは Rego や AI で独自のルールを書けます。

- [Wiz enhances dynamic scanner to analyze and validate external exposure](https://www.wiz.io/blog/wiz-enhances-dynamic-scanner-to-analyze-and-validate-external-exposure)
- [Wiz CSPM](https://www.wiz.io/solutions/cspm)

### AWS Security Hub は、AWS のサービスのシグナルを関連付けて「露出」として出す

AWS にも近い機能があります。**Security Hub の exposure findings** は、Security Hub CSPM、Inspector、GuardDuty、Macie などのシグナルを関連付けて、1 つの「露出」として出します。攻撃経路のグラフも見られます。2026 年 7 月には、侵害されたときに届く範囲と、各段階の権限を示す影響分析も加わりました。

exposure findings は、新しいシグナルが入るたびにほぼリアルタイムで更新されます。外から実際に届くかは、オプトインで有効にする外部スキャンの機能 Network Scanning が、AWS の IP アドレスから接続して調べます（約 12 時間ごとに再スキャン）。

- [Exposure findings in Security Hub - AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings.html)
- [Generating exposure findings - AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings-generate.html)
- [AWS Security Hub now generally available with near real-time risk analytics (2025)](https://aws.amazon.com/about-aws/whats-new/2025/12/security-hub-near-real-time-risk-analytics)
- [AWS Security Hub adds impact analysis for exposure findings (2026)](https://aws.amazon.com/about-aws/whats-new/2026/07/impact-analysis-aws-security-hub/)
- [Network Scanning in Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-network-scanning.html)

### Wiz と Security Hub を、比べる軸で並べる

| 軸 | Wiz | Security Hub の exposure findings |
|---|---|---|
| 対象のクラウド | 主要なクラウドをまたいで見る | AWS。Azure はコネクターを使えば一部のリソースが対象 |
| 証拠の種類 | 設定・IaC（コードやワークロードの中身も含む） | 設定に加え、Inspector の脆弱性（パッケージ、コンテナイメージ、Lambda のコード）、GuardDuty の脅威、Macie の機密データ |
| 組み合わせの見方 | Security Graph で攻撃経路を出す | 攻撃経路のグラフと、影響の範囲を出す |
| 外からの確かめ方 | dynamic scanner で外から接続する | Network Scanning（オプトイン）で AWS の IP から接続する |
| 独自のルール | Rego や AI で書ける | 独自の検査は Config のカスタムルールなどで足す（exposure findings の判定に直接足せるかは未確認） |
| 修正 | Green Agent が修正を提案し、実行まで担う（プレビュー） | 修正の手順を示す。MCP App も修正の推奨を返す（実行はしない） |

AWS だけで、まず組み合わせの露出を見たいなら Security Hub、複数のクラウドやコード・ワークロードの中までまとめて見たいなら Wiz のような CNAPP、という分かれ方になります。

- [Supported resource types for exposure findings in Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings-supported-resources.html)

## 見つけるだけでなく、直すところまでエージェントが担い始めている

2026 年に入って、製品の側にもエージェントが入ってきました。

- **Wiz**：3 つの AI エージェントを出しています。Red Agent は悪用できるリスクを探して確かめ、Blue Agent は脅威の調査を早め、Green Agent はリスクを調べて修正を提案・実行します（Green Agent はプレビュー）。Claude Code や Cursor 向けのスキルも用意し、コーディングエージェントが Security Graph の検出結果を取り込んで、コードを直せるようにしています
- **AWS**：Security Hub MCP App をプレビューで出しています。Claude Desktop から、exposure findings の攻撃経路や関連する検出結果を調べられます。ツールはすべて読み取り専用です

修正の実行まで任せるかどうかは、組織ごとに決めることになります。Gartner は、AI には大量の分析やトリアージを任せ、重要な判断の権限は人が持つ「human-on-the-loop」の形を勧めています。

- [Wiz at Black Hat 2026: Driving AI Threat Readiness](https://www.wiz.io/blog/wiz-at-black-hat-2026)
- [Wiz at Google Cloud Next 2026](https://wiz.io/blog/wiz-at-google-cloud-next)
- [AWS Security Hub MCP App brings exposure findings into your AI-assisted workflow (Preview)](https://aws.amazon.com/about-aws/whats-new/2026/07/aws-security-hub-mcp-app/)
- [When Reactive Security Falls Behind AI-Driven Threats - Gartner](https://www.gartner.com/en/articles/preemptive-cybersecurity-solutions)

## 製品が見るのは設定の組み合わせまでで、文書と運用の実態は残る

ここまでを、前の記事の「証拠の種類」で整理すると次のようになります。

| 証拠の種類 | CNAPP・exposure findings | 残る仕事 |
|---|---|---|
| 設定・IaC | 得意。組み合わせも攻撃経路として出す | 製品の対象外のリソースや、製品が見ていない組み合わせを補う |
| 文書 | 対象外 | 設計書や体制表と、実際の構成とのずれを確かめる |
| 運用の実態 | 対象外 | 記録（チケット、ログ）と、人へのヒアリングで確かめる |

CNAPP や exposure findings を入れても、文書と運用の実態を確かめる仕事は残ります。前の記事で、AI エージェントにこの部分を担わせることを考えたのはそのためです。なお、AWS Security Agent のように、設計書を組織のセキュリティ要件と照らしてレビューする製品も出始めています。

- [How AWS Security Agent works](https://docs.aws.amazon.com/securityagent/latest/userguide/how-it-works.html)

## 設定の中にも、製品から見えにくい穴がある

設定の列でも、すべてが拾えるとは限りません。例として、前の記事で扱った「CloudFront の裏の ALB」を考えます。ALB のセキュリティグループ（SG）が CloudFront のマネージドプレフィックスリストだけを許可していて、リスナーにヘッダー条件がない構成です。このプレフィックスリストは世界中の CloudFront を通すので、他人の CloudFront を経由すれば、自分の CloudFront を通らずに ALB に届きます。

https://zenn.dev/anfangd/articles/cloudfront-alb-origin-exposure-audit

この穴は、製品から次の理由で見えにくいと考えられます（どちらも試していません）。

- **対象のリソース**：Security Hub の exposure findings が主な対象にするリソースの種類（2026 年 11 月時点）には、EC2、Lambda、S3 などはありますが、ALB 自体は入っていません。裏の EC2 や ECS の露出として、ALB を通る経路が評価される可能性はあります
- **外からの確かめ方**：Wiz の dynamic scanner も Security Hub の Network Scanning も、外から実際に接続して確かめます。Network Scanning は ALB も対象にしますが、接続元は AWS の IP で、CloudFront のプレフィックスリストには入っていません。SG で弾かれ、「届かない」と判定されそうです。設定からの計算でも、プレフィックスリストが「他人の CloudFront も通す」ことまでモデル化しているかは、公開情報からは分かりませんでした

一方で、「SG が CloudFront のプレフィックスリストを許可しているのに、リスナーにヘッダー条件がない ALB」を検出するルールは、自分で書けます。Wiz なら Rego で、AWS だけなら Config のカスタムルールで書けるはずです。

つまり、製品を入れても、**新しい穴に気づいてルールに落とす**ところは残ります。製品が出す検出結果は、それを考えるための入力の 1 つです。

- [Supported resource types for exposure findings in Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings-supported-resources.html)

## おわりに

【ハジメが書く：製品とどう付き合うか。修正の実行までどこまで任せたいか。Wiz で ALB の穴を本当に拾えないか試してみたい、など次にやりたいこと】

## 参考

- [Gartner Identifies the Top Cybersecurity Trends for 2024（CTEM）](https://www.gartner.com/en/newsroom/press-releases/2024-02-22-gartner-identifies-top-cybersecurity-trends-for-2024)
- [FedRAMP Consolidated Rules for 2026](https://www.fedramp.gov/2026/)
- [Open Cybersecurity Schema Framework (OCSF)](https://schema.ocsf.io/)
- [Wiz Security Graph（attack path analysis）](https://www.wiz.io/academy/detection-and-response/attack-path-analysis)
- [Exposure findings in Security Hub - AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings.html)
