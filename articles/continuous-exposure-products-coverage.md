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

- 「年 1 回の点検を継続的なサイクルにする」流れは、枠組み（Gartner の CTEM）、製品（CNAPP、AWS Security Hub の exposure findings）、監査（FedRAMP）のどれでも進んでいます
- CNAPP の強みは、設定・権限・ネットワーク・脆弱性を 1 つのグラフに載せて、**組み合わせを攻撃経路として**見せることです。2026 年には、修正まで担うエージェントも出てきました
- 製品が見るのは**設定の組み合わせ**までです。設計書や体制表といった**文書**と、**運用の実態**は対象外です
- 設定の中でも、対象のリソースや見方によっては拾えない穴があります。気づいた穴をルールに落とすのは、製品を入れても人とエージェントの仕事として残ります

## 年 1 回の点検を、継続的なサイクルにする動きはあちこちで進んでいる

### CTEM は、点検を「範囲を決めて、直すまで」のサイクルにする

Gartner は、**継続的な脅威エクスポージャー管理（CTEM）** という考え方を打ち出しています。デジタルと物理の資産について、届くか・露出しているか・悪用できるかを継続的に評価する、体系的なアプローチです。範囲をインフラの部品ではなく、攻撃の経路や事業のプロジェクトに合わせて決めるのが特徴です。

CTEM は、範囲の決定（scoping）・発見（discovery）・優先順位付け（prioritization）・検証（validation）・対応の実行（mobilization）の 5 段階を繰り返すサイクルとして説明されます。製品ではなくプログラムの枠組みで、それを支える製品の分類として、Gartner は **Exposure Assessment Platforms（EAP）** を置いています。

- [Gartner Identifies the Top Cybersecurity Trends for 2024](https://www.gartner.com/en/newsroom/press-releases/2024-02-22-gartner-identifies-top-cybersecurity-trends-for-2024)
- [Exposure Assessment Platforms Reviews and Ratings - Gartner](https://www.gartner.com/reviews/market/exposure-assessment-platforms)

### 監査の側も、証拠を機械で読める形で継続的に出す方向に動いている

米国政府のクラウド認証制度 FedRAMP は、2026 年 6 月に公開した Consolidated Rules for 2026 で、認証後も 3 か月ごとに継続的なモニタリングの報告を出すよう求めています。報告や証拠はできるだけ機械で読める形で受け取る方針で、その形式として NIST の OSCAL を指定しています。

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

代表的なのが Wiz で、Security Graph を中心に据えています。ほかに、Palo Alto Networks の Cortex Cloud（旧 Prisma Cloud）、Microsoft Defender for Cloud、CrowdStrike Falcon Cloud Security、エージェントレスのスキャンの先駆けである Orca Security などがあります。

- [What is attack path analysis? - Wiz](https://www.wiz.io/academy/detection-and-response/attack-path-analysis)
- [Orca SideScanning - Orca Security](https://orca.security/platform/agentless-sidescanning-technology/)

### Wiz は、ネットワークの露出を計算し、外からも確かめる

Wiz は、ロードバランサー、ファイアウォール、VPC、サブネットなどのネットワークルールを解析して、実際に外から届くリソースを計算します。さらに dynamic scanner で外から接続し、ポートやプロトコルが本当に開いているかを確かめます。

設定ルールは 2,800 以上が用意されていて、足りないところは Rego や AI で独自のルールを書けます。

- [Wiz enhances dynamic scanner to analyze and validate external exposure](https://www.wiz.io/blog/wiz-enhances-dynamic-scanner-to-analyze-and-validate-external-exposure)
- [Wiz CSPM](https://www.wiz.io/solutions/cspm)

### AWS Security Hub は、AWS のサービスのシグナルを関連付けて「露出」として出す

AWS にも近い機能があります。**Security Hub の exposure findings** は、Security Hub CSPM、Inspector、GuardDuty、Macie などのシグナルを関連付けて、1 つの「露出」として出します。攻撃経路のグラフも見られます。2026 年 7 月には、侵害されたときに届く範囲と、各段階の権限を示す影響分析も加わりました。

exposure findings は 6 時間ごとに作られます。外から実際に届くかは、Network Scanning が AWS の IP アドレスから調べます。

- [Exposure findings in Security Hub - AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings.html)
- [Generating exposure findings - AWS Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/exposure-findings-generate.html)
- [AWS Security Hub adds impact analysis for exposure findings (2026)](https://aws.amazon.com/about-aws/whats-new/2026/07/impact-analysis-aws-security-hub/)
- [Network Scanning in Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-network-scanning.html)

## 見つけるだけでなく、直すところまでエージェントが担い始めている

2026 年に入って、製品の側にもエージェントが入ってきました。

- **Wiz**：3 つの AI エージェントを出しています。Red Agent は悪用できるリスクを探して確かめ、Blue Agent は脅威の調査を早め、Green Agent はリスクを調べて修正を提案・実行します。Claude Code や Cursor 向けのスキルも用意し、コーディングエージェントが Security Graph の検出結果を取り込んで、コードを直せるようにしています
- **AWS**：Security Hub MCP App をプレビューで出しています。Claude Desktop から、exposure findings の攻撃経路や関連する検出結果を調べられます。ツールはすべて読み取り専用です

修正の実行まで任せるかどうかは、組織ごとに決めることになります。Gartner は、AI には大量の分析やトリアージを任せ、重要な判断の権限は人が持つ「human-on-the-loop」の形を勧めています。

- [Wiz at Black Hat 2026: Driving AI Threat Readiness](https://www.wiz.io/blog/wiz-at-black-hat-2026)
- [Next '26: Redefining security for the AI era with Google Cloud and Wiz](https://cloud.google.com/blog/products/identity-security/next26-redefining-security-for-the-ai-era-with-google-cloud-and-wiz)
- [AWS Security Hub MCP App brings exposure findings into your AI-assisted workflow (Preview)](https://aws.amazon.com/about-aws/whats-new/2026/07/aws-security-hub-mcp-app/)
- [When Reactive Security Falls Behind AI-Driven Threats - Gartner](https://www.gartner.com/en/articles/preemptive-cybersecurity-solutions)

## 製品が見るのは設定の組み合わせまでで、文書と運用の実態は残る

ここまでを、前の記事の「証拠の種類」で整理すると次のようになります。

| 証拠の種類 | CNAPP・exposure findings | 残る仕事 |
|---|---|---|
| 設定・IaC | 得意。組み合わせも攻撃経路として出す | 製品の対象外のリソースや、製品が見ていない組み合わせを補う |
| 文書 | 対象外 | 設計書や体制表と、実際の構成とのずれを確かめる |
| 運用の実態 | 対象外 | 記録（チケット、ログ）と、人へのヒアリングで確かめる |

製品を入れても、文書と運用の実態を確かめる仕事はそのまま残ります。前の記事で、AI エージェントにこの部分を担わせることを考えたのはそのためです。

## 設定の中にも、製品から見えにくい穴がある

設定の列でも、すべてが拾えるとは限りません。例として、前の記事で扱った「CloudFront の裏の ALB」を考えます。ALB の SG が CloudFront のマネージドプレフィックスリストだけを許可していて、リスナーにヘッダー条件がない構成です。このプレフィックスリストは世界中の CloudFront を通すので、他人の CloudFront を経由すれば ALB に直接届きます。

https://zenn.dev/anfangd/articles/cloudfront-alb-origin-exposure-audit

この穴は、製品から次の理由で見えにくいと考えられます（どちらも試していません）。

- **対象のリソース**：Security Hub の exposure findings が対象にするリソースの種類（2026 年 11 月時点）には、EC2、Lambda、S3 などはありますが、ALB は入っていません
- **外からの確かめ方**：Wiz の dynamic scanner も Security Hub の Network Scanning も、外から実際に接続して確かめます。ただ、接続元は CloudFront ではないので SG で弾かれ、「届かない」と判定されそうです。設定からの計算でも、プレフィックスリストが「他人の CloudFront も通す」ことまでモデル化しているかは、公開情報からは分かりませんでした

一方で、Wiz なら Rego で独自のルールを書けます。「SG が CloudFront のプレフィックスリストを許可しているのに、リスナーにヘッダー条件がない ALB」を検出するルールは作れるはずです。

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
