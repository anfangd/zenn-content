---
title: "AWSでWebリクエストを受ける8パターンと「オリジンを裸にしない」設定プラクティス"
emoji: "🛡️"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudFront","ALB","WAF","Security"]
published: false
---

:::message
この記事は、Claude と一緒に AWS 公式ドキュメントを調べ、構成図を作りながら整理した内容をまとめたものです。各設定の根拠は、記事中のリンク先の公式ドキュメントで確認できます。
:::

## はじめに

先日、X（旧Twitter）でこんな趣旨の話題が流れてきました。

- CloudFront + WAF → ALB の構成では、ALB の SG に CloudFront のプレフィックスリストを入れて直接アクセスを防ぐ
- ただし ALB のドメインが知られると、**第三者が作った CloudFront 経由で WAF を迂回される**。そのため秘密ヘッダーでの照合が必要
- さらに、**CT Log が知られるようになってから、Host ヘッダー付きの攻撃が増えた**

元の投稿はこちらです。

@[tweet](https://x.com/nasub7373/status/2117957893232451709)

@[tweet](https://x.com/___nix___/status/2117996284783300621)

どちらも重要な指摘です。一方で、2024年11月に **CloudFront VPC オリジン** が登場してから、選べる構成はかなり変わっています。

この記事では、AWS で Web リクエストを受ける代表的な 8 パターンを、**公式ドキュメントの根拠付き**で、構成図と設定プラクティスにまとめます。

## TL;DR

- 公式ドキュメントから読み取れる優先順位は **「VPC オリジン / OAC（マネージド）→ 使えない場合のみ プレフィックスリスト＋秘密ヘッダー」**
- **プレフィックスリストだけでは不十分**です。他人の CloudFront も同じ IP 帯から来るため、本体は秘密ヘッダーの照合です
- **CT Log** にはオリジン用に取った証明書のホスト名が載るので、「ホスト名は知られる前提」で設計します
- 新規構築なら **VPC オリジン＋`CloudFront-VPCOrigins-Service-SG` 限定** が第一候補です。名前を知られても、そもそも到達できません
- **WAF は「中身を検査する層」、VPC オリジン / OAC / SG は「届く経路を絞る層」** です。役割が違うので両方必要で、WAF の主戦場はエッジ（CloudFront）です

## 以前のプラクティスと、いまのプラクティス

私自身、これまで「ALB をインターネットに公開し、その前に CloudFront を置く」構成を何度も作ってきました。当時はそれが一般的な作り方で、**SG で CloudFront の IP に絞る**、**秘密ヘッダーで照合する**、といった対策をプラクティスとして実装していました。

ALB をインターネットに公開する構成は今もよく見かけますし、それ自体が間違いというわけではありません。ただ、ここ数年でオリジンを閉じるためのマネージドな仕組みがそろい、**同じ要件をよりセキュアに、しかも運用負荷を下げて作れる**ようになっています。

| 時期 | 出来事 | 何が変わったか |
|---|---|---|
| 2015年12月 | CloudFront がオリジンへのカスタムヘッダー追加に対応 | 秘密ヘッダーで「CloudFront 経由か」を照合できるように |
| 2016年頃 | AWS Security Blog で、CloudFront の IP 範囲を Lambda で SG に自動反映する方法が紹介される | SG で CloudFront の IP に絞る方式が広まる |
| 2016年8月 | ALB 登場 | CloudFront → ALB の構成が定番に |
| 2020年10月 | AWS Security Blog で、WAF＋Secrets Manager による秘密ヘッダーのローテーションが紹介される | 秘密ヘッダー方式の運用（ローテーション）が整う |
| 2022年2月 | CloudFront のマネージドプレフィックスリスト | IP 範囲を Lambda で更新する仕組みが不要に |
| 2022年8月 | OAC（S3 向け） | OAI の制約（SSE-KMS など）が解消 |
| 2024年4月 | OAC が Lambda 関数 URL に対応 | 関数 URL も CloudFront 専用にできるように |
| 2024年11月 | **VPC オリジン** | ALB / NLB / EC2 をプライベートサブネットに置いたまま CloudFront から使えるように |
| 2025年11月 | VPC オリジンのクロスアカウント対応 | CloudFront とオリジンを別アカウントに分けた構成でも使えるように |

AWS 自身も、VPC オリジンの発表で「これまで VPC 内のオリジンはパブリックサブネットに置く必要があった」と書いています。つまり、10 年近く定番だった「ALB を公開して IP 制限と秘密ヘッダーで守る」方式は、今では**VPC オリジンが使えない場合のフォールバック**という位置付けになりました。

この記事は、そうした変化を踏まえて「いま新しく作るなら、あるいは見直すなら、どう作るか」を整理したものです。

- [Now Add or Modify Request Headers Forwarded From Amazon CloudFront to Origin (2015)](https://aws.amazon.com/about-aws/whats-new/2015/12/now-add-or-modify-request-headers-forwarded-from-amazon-cloudfront-to-origin/)
- [How to Automatically Update Your Security Groups for Amazon CloudFront and AWS WAF by Using AWS Lambda (AWS Security Blog)](https://aws.amazon.com/blogs/security/how-to-automatically-update-your-security-groups-for-amazon-cloudfront-and-aws-waf-by-using-aws-lambda/)
- [Announcing Application Load Balancer for Elastic Load Balancing (2016)](https://aws.amazon.com/about-aws/whats-new/2016/08/announcing-application-load-balancer-for-elastic-load-balancing/)
- [How to enhance Amazon CloudFront origin security with AWS WAF and AWS Secrets Manager (AWS Security Blog, 2020)](https://aws.amazon.com/blogs/security/how-to-enhance-amazon-cloudfront-origin-security-with-aws-waf-and-aws-secrets-manager/)
- [Amazon VPC now supports an AWS-managed prefix list for Amazon CloudFront (2022)](https://aws.amazon.com/about-aws/whats-new/2022/02/amazon-cloudfront-managed-prefix-list/)
- [Amazon CloudFront launches Origin Access Control (OAC) (2022)](https://aws.amazon.com/about-aws/whats-new/2022/08/amazon-cloudfront-origin-access-control/)
- [Amazon CloudFront now supports OAC for Lambda function URL origins (2024)](https://aws.amazon.com/about-aws/whats-new/2024/04/amazon-cloudfront-oac-lambda-function-url-origins/)
- [Amazon CloudFront announces VPC origins (2024)](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-cloudfront-vpc-origins/)
- [Introducing cross-account support for Amazon CloudFront VPC origins](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-cross-account-support-for-amazon-cloudfront-virtual-private-cloud-vpc-origins/)

## 前提知識

### プレフィックスリストだけではダメな理由

`com.amazonaws.global.cloudfront.origin-facing` は「CloudFront 全体」の IP 帯です。自社のディストリビューションに限った IP 帯ではありません。攻撃者が自分の AWS アカウントで CloudFront を作り、オリジンにあなたの ALB を指定すれば、SG は素通りします。WAF は「あなたの CloudFront」に付いているので、当然かかりません。

そのため公式ドキュメントでも、プレフィックスリストは **Optional（追加の防御層）** という位置付けです。本体は、カスタムヘッダーを ALB のリスナールールで照合する仕組みです。

- [Restrict access to Application Load Balancers - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)

### CT Log（Certificate Transparency：証明書の透明性）とは

CT は、公的な CA（Certificate Authority：認証局）が発行した TLS 証明書を、誰でも検索できる公開ログにすべて記録する仕組みです。ブラウザが CT を必須にしているため、**ACM で発行した証明書も必ず記録され、オプトアウトできません**。

そのため `crt.sh` などで検索すれば、`origin.example.com` のように証明書を取ったホスト名を誰でも一覧できます。攻撃者はこれを偵察に使い、証明書の発行から数分でスキャンが来ることも観測されています。

**DNS を公開していなくても、公的な証明書を取った時点でホスト名は知られる** と考えておきましょう。ワイルドカード証明書で個別のホスト名を隠すことはできますが、それは推測しにくくするだけの隠蔽策です。本筋は「名前を知られても届かない」構成にすることです。

CA と CT の仕組み（構成・発行・検証・監視）は、図解付きで別の記事にまとめました。

https://zenn.dev/anfangd/articles/certificate-transparency-ct-log

- [AWS Certificate Manager concepts - Certificate Transparency Logging](https://docs.aws.amazon.com/acm/latest/userguide/acm-concepts.html)

## 選定フロー

![選定フロー](/images/aws-web-ingress-patterns/00-decision-flow.drawio.svg)

DDoS 耐性のホワイトペーパーでは、CloudFront の背後に VPC 内のオリジンを置く場合、**VPC オリジンが最新かつ推奨の方法** とされています。S3・AWS Elemental・Lambda 関数 URL については、引き続き **OAC が推奨** です。

- [Protecting your origin (BP1, BP5) - AWS Best Practices for DDoS Resiliency](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/protecting-your-origin-bp1-bp5.html)

CloudFront の FAQ でも、マネージドな仕組みで要件を満たせない場合の代替として、カスタムオリジンヘッダーが挙げられています。

- [Amazon CloudFront FAQs](https://aws.amazon.com/cloudfront/faqs/)

---

## ① CloudFront + VPC オリジン（内部 ALB）★第一候補

![VPCオリジン](/images/aws-web-ingress-patterns/01-vpc-origin.drawio.svg)

プライベートサブネットにある ALB / NLB / EC2 を、CloudFront のオリジンとして直接使えます。ALB にパブリック IP がないため、**CT Log でホスト名を知られても、Host ヘッダーを付けても、インターネットから届く経路自体がありません**。VPC オリジンの利用に追加料金はかかりません。

### 設定プラクティス

| 項目 | 推奨 |
|---|---|
| ALB | `internal`、プライベートサブネット、HTTPS リスナー（ACM 証明書） |
| ALB の SG | **`CloudFront-VPCOrigins-Service-SG` を許可**（プレフィックスリストより厳密で、自アカウントの CloudFront に限定できる） |
| VPC | **IGW が必要**（存在さえすればよく、ルーティングには使われない） |
| サブネット | サービス管理 ENI 用の空き IP（デュアルスタックなら IPv6 も） |
| NACL | インバウンドは評価されない。アウトバウンドは戻りのエフェメラルポートを許可 |

### 制約・注意点

- オリジンにできないもの：GWLB、TLS リスナーを持つ NLB。NLB には SG の付与が必須です
- 非対応の機能：gRPC、Lambda@Edge の origin request / origin response トリガー
- **VPC Block Public Access** を有効にすると、VPC オリジンの通信も止まります
- VPC オリジンで絞れるのは「自アカウント内で VPC オリジンを設定したディストリビューション」までです。同じアカウントに複数のディストリビューションがあり、特定のディストリビューションだけに限定したい場合は、カスタムヘッダーの検証も併用します（[Origin cloaking](https://aws.amazon.com/developer/application-security-performance/articles/origin-cloaking)）

:::details Terraform 例：VPC オリジンとサービス管理 SG の参照

```hcl:vpc_origin.tf
resource "aws_cloudfront_vpc_origin" "alb" {
  vpc_origin_endpoint_config {
    name                   = "internal-alb"
    arn                    = aws_lb.internal.arn
    http_port              = 80
    https_port             = 443
    origin_protocol_policy = "https-only"

    origin_ssl_protocols {
      items    = ["TLSv1.2"]
      quantity = 1
    }
  }
}

# ディストリビューション側
# origin {
#   domain_name = aws_lb.internal.dns_name
#   origin_id   = "internal-alb"
#   vpc_origin_config {
#     vpc_origin_id = aws_cloudfront_vpc_origin.alb.id
#   }
# }

# VPC オリジン作成後に、AWS が自動作成するサービス管理 SG を参照する
data "aws_security_group" "cf_vpc_origins" {
  vpc_id = aws_vpc.main.id
  filter {
    name   = "group-name"
    values = ["CloudFront-VPCOrigins-Service-SG"]
  }
  depends_on = [aws_cloudfront_vpc_origin.alb]
}

resource "aws_vpc_security_group_ingress_rule" "alb_from_cf_vpc_origins" {
  security_group_id            = aws_security_group.alb.id
  referenced_security_group_id = data.aws_security_group.cf_vpc_origins.id
  ip_protocol                  = "tcp"
  from_port                    = 443
  to_port                      = 443
}
```
:::

:::message alert
`CloudFront-VPCOrigins-Service-SG` は AWS が管理するため、編集してはいけません。参照するだけにしてください。
:::

:::details 確認コマンド
```bash
# ALB が internal になっているか（internet-facing なら見直し）
aws elbv2 describe-load-balancers --names my-alb \
  --query 'LoadBalancers[].[LoadBalancerName,Scheme,DNSName]' --output table

# VPC オリジンの一覧と状態（Deployed になっているか）
aws cloudfront list-vpc-origins \
  --query 'VpcOriginList.Items[].[Id,Name,Status]' --output table

# サービス管理 SG の ID（ALB の SG のインバウンド許可元にする）
aws ec2 describe-security-groups \
  --filters Name=group-name,Values=CloudFront-VPCOrigins-Service-SG \
  --query 'SecurityGroups[].[GroupId,VpcId]' --output table

# インターネットから ALB に直接届かないこと（タイムアウトすれば OK）
curl -sS -m 5 -o /dev/null -w '%{http_code}\n' \
  https://internal-my-alb-123456789.ap-northeast-1.elb.amazonaws.com/

# CloudFront 経由では届くこと（200 など）
curl -s -o /dev/null -w '%{http_code}\n' https://www.example.com/
```
:::

根拠：
- [Restrict access with VPC origins - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [VPC BPA basics - Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa-basics.html)
- [Introducing CloudFront VPC Origins (AWS Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-cloudfront-virtual-private-cloud-vpc-origins-shield-your-web-applications-from-public-internet/)

---

## ② CloudFront + 公開 ALB（プレフィックスリスト＋秘密ヘッダー）※フォールバック

![公開ALB＋秘密ヘッダー](/images/aws-web-ingress-patterns/02-public-alb-secret-header.drawio.svg)

この構成を選ぶのは、ALB を複数の CDN から使う場合や、VPC オリジンが使えない機能に依存している場合です。冒頭の X の話題はこの構成のことです。

図の 2 つの攻撃経路がポイントです。

- **ALB へ直接**：SG（L3/L4）で遮断されます
- **第三者の CloudFront 経由**：SG は**通過します**（同じ CloudFront の IP 帯なので）。ただしヘッダーが一致しないため、ALB のデフォルトルールで 403 になります

### 設定プラクティス

- CloudFront のオリジンカスタムヘッダーに秘密の値（例：`X-Origin-Verify`）を設定します
- ALB リスナーのルールは「ヘッダー一致なら転送」、**デフォルトアクションは固定レスポンス 403**（fail-closed）にします
- **Origin Protocol Policy は HTTPS Only** にします（ヘッダーの盗聴を防ぐため）。この場合、オリジンリクエストポリシーで Host ヘッダーを転送する必要があります（AllViewer が使えます）
- 証明書は、CloudFront 用を `us-east-1`、ALB 用を ALB のリージョンで用意します
- ヘッダーは**資格情報として扱い、定期的にローテーション**します（Secrets Manager＋Lambda で自動化するのが定番）

ローテーションは、公式ドキュメントにある次の 4 段階で行うと無停止で切り替えられます。

1. CloudFront に**新しい**ヘッダーを追加する
2. ALB のルールで新しいヘッダーを許可する
3. CloudFront で**古い**ヘッダーの付与をやめる
4. ALB のルールで古いヘッダーを拒否する

:::details Terraform 例：プレフィックスリストと秘密ヘッダー照合（fail-closed）

```hcl:alb_secret_header.tf
data "aws_ec2_managed_prefix_list" "cloudfront" {
  name = "com.amazonaws.global.cloudfront.origin-facing"
}

resource "aws_vpc_security_group_ingress_rule" "alb_from_cloudfront" {
  security_group_id = aws_security_group.alb.id
  prefix_list_id    = data.aws_ec2_managed_prefix_list.cloudfront.id
  ip_protocol       = "tcp"
  from_port         = 443
  to_port           = 443
}

resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.public.arn
  port              = 443
  protocol          = "HTTPS"
  certificate_arn   = aws_acm_certificate.origin.arn
  ssl_policy        = "ELBSecurityPolicy-TLS13-1-2-2021-06"

  # fail-closed: ヘッダーが一致しなければ 403
  default_action {
    type = "fixed-response"
    fixed_response {
      content_type = "text/plain"
      message_body = "Access denied"
      status_code  = "403"
    }
  }
}

resource "aws_lb_listener_rule" "from_cloudfront" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 1

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.app.arn
  }

  condition {
    http_header {
      http_header_name = "X-Origin-Verify"
      values           = [var.origin_verify_secret] # ローテーション時は新旧2値を並べる
    }
  }
}
```
:::

:::message
秘密の値は Terraform の state に平文で残ります。state の保管場所の暗号化とアクセス制御もセットで考えてください。
:::

:::message alert
公式ドキュメントのサンプル（CloudFormation）は `ViewerProtocolPolicy: allow-all` になっています。そのまま使うと Security Hub の CloudFront.3 に引っかかるので、`redirect-to-https` か `https-only` にしましょう。
:::

:::details 確認コマンド
```bash
# CloudFront のプレフィックスリスト ID（SG のインバウンド許可元にする）
# MaxEntries は SG のルール数として数えられるので、SG のクォータに注意
aws ec2 describe-managed-prefix-lists \
  --filters Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing \
  --query 'PrefixLists[].[PrefixListId,PrefixListName,MaxEntries]' --output table

# リスナールールが「ヘッダー一致なら forward、デフォルトは fixed-response」になっているか
aws elbv2 describe-rules --listener-arn <listener-arn> \
  --query 'Rules[].[Priority,Conditions[0].HttpHeaderConfig.HttpHeaderName,Actions[0].Type]' \
  --output table

# ALB に直接：SG で遮断されてタイムアウトになること
curl -sS -m 5 -o /dev/null -w '%{http_code}\n' \
  https://my-alb-123456789.ap-northeast-1.elb.amazonaws.com/

# 正規の CloudFront 経由：200 になること
curl -s -o /dev/null -w '%{http_code}\n' https://www.example.com/
```
:::

根拠：
- [Restrict access to Application Load Balancers - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/restrict-access-to-load-balancer.html)
- [Optimizing application performance: CloudFront with ALB (AWS Blog)](https://aws.amazon.com/blogs/networking-and-content-delivery/optimizing-application-performance-the-strategic-benefits-of-combining-amazon-cloudfront-with-application-load-balancers/)

---

## ③ CloudFront + S3（OAC）

![S3 OAC](/images/aws-web-ingress-patterns/03-s3-oac.drawio.svg)

S3 なら OAC（Origin Access Control）一択です。OAI はレガシー扱いで、次の点に対応していません。

- 全リージョンの S3 バケット（オプトインリージョンや、2023年1月以降の新リージョンなど）
- SSE-KMS
- 動的リクエスト（PUT / POST / DELETE）

### 設定プラクティス

- バケットは非公開にし、Block Public Access を ON、Object Ownership は **Bucket owner enforced** にします
- バケットポリシーのプリンシパルは `cloudfront.amazonaws.com` にし、`AWS:SourceArn` で**自社のディストリビューションに限定**します
- 署名動作は **Sign requests（always）** にします。この設定のときだけ、CloudFront → S3 の通信が常に HTTPS になります
- SSE-KMS を使う場合は、キーポリシーでも CloudFront を許可します
- S3 のウェブサイトエンドポイントはカスタムオリジン扱いになるため、OAC は使えません
- OAI からの移行では、まずバケットポリシーで OAI と OAC の両方を一時的に許可してから切り替えます

:::details 確認コマンド
```bash
# OAC の署名動作が always になっているか
aws cloudfront list-origin-access-controls \
  --query 'OriginAccessControlList.Items[].[Id,Name,OriginAccessControlOriginType,SigningBehavior]' \
  --output table

# バケットポリシー：Principal が cloudfront.amazonaws.com、AWS:SourceArn が自分のディストリビューションか
aws s3api get-bucket-policy --bucket my-bucket --query Policy --output text | jq .

# S3 に直接アクセスすると 403、CloudFront 経由なら 200
curl -s -o /dev/null -w '%{http_code}\n' https://my-bucket.s3.ap-northeast-1.amazonaws.com/index.html
curl -s -o /dev/null -w '%{http_code}\n' https://www.example.com/index.html
```
:::

根拠：
- [Restrict access to an Amazon S3 origin - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-s3.html)
- [Security Hub CSPM controls for CloudFront（CloudFront.13）](https://docs.aws.amazon.com/securityhub/latest/userguide/cloudfront-controls.html)

---

## ④ CloudFront + Lambda 関数 URL（OAC）

![Lambda関数URL OAC](/images/aws-web-ingress-patterns/04-lambda-url-oac.drawio.svg)

関数 URL 自体は公開エンドポイントのままです。そのため **OAC を付けないと直接叩けてしまいます**。Security Hub でも CloudFront.16 として検知対象になっています。

### 設定プラクティス

- 関数 URL の `AuthType` は **`AWS_IAM`** にします
- リソースベースポリシーでは、`cloudfront.amazonaws.com` に **`lambda:InvokeFunctionUrl` と `lambda:InvokeFunction` の両方**を許可し、`SourceArn` で自社のディストリビューションに限定します
- OAC の署名動作は always にします
- オリジンリクエストポリシーは `AllViewerExceptHostHeader` にします（関数 URL も Host ヘッダーに自分のドメイン名を期待するため）
- POST / PUT でボディを送る場合は、利用者側でペイロードのハッシュ（`x-amz-content-sha256`）を計算する必要がある点に注意してください

:::details 確認コマンド
```bash
# AuthType が AWS_IAM になっているか
aws lambda get-function-url-config --function-name my-fn \
  --query '[FunctionUrl,AuthType]' --output text

# リソースベースポリシーに InvokeFunctionUrl と InvokeFunction の両方があるか
aws lambda get-policy --function-name my-fn --query Policy --output text | jq .

# 関数 URL に直接アクセスすると 403
curl -s -o /dev/null -w '%{http_code}\n' https://abcdefghij.lambda-url.ap-northeast-1.on.aws/
```
:::

根拠：
- [Restrict access to an AWS Lambda function URL origin - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-restricting-access-to-lambda.html)
- [Use various origins with CloudFront distributions](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/DownloadDistS3AndCustomOrigins.html)

---

## ⑤ 自社の CloudFront + Regional API Gateway（公開 API）

![API Gateway Regional](/images/aws-web-ingress-patterns/05-apigw-regional.drawio.svg)

REST API にはエッジ最適化エンドポイント（AWS が管理する CloudFront）もありますが、DDoS 耐性のホワイトペーパーでは **Regional エンドポイント＋自社で管理する CloudFront** が推奨されています。WAF、キャッシュ、地理制限を自分でコントロールできるからです。

### 設定プラクティス

- オリジンリクエストポリシーは **`AllViewerExceptHostHeader`**、キャッシュポリシーは `CachingDisabled` にします。API Gateway は Host ヘッダーに自分のドメイン名を期待するため、ビューワーの Host を転送すると 403 になります（[Use managed origin request policies](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-origin-request-policies.html)）
- **直接アクセス対策**として、CloudFront のオリジンカスタムヘッダーに `x-api-key` を設定し、API Gateway 側で API キーを必須にします
- メソッド単位のスロットリングと Usage Plan で、バックエンドを過剰なトラフィックから守ります
- WAF は、同じリージョンの Regional web ACL をステージに関連付けることもできます（CloudFront の WAF との二段構え）
- カスタムドメインだけで受けたい場合は、デフォルトの `execute-api` エンドポイントを無効化できます。ただし **CloudFront のオリジンを `execute-api` ドメインにしているなら、無効化した時点で到達できなくなる**ので、オリジン側もカスタムドメインにしておく必要があります

:::details 確認コマンド
```bash
# エンドポイントの種類（REGIONAL）と、デフォルトエンドポイントの無効化状態
aws apigateway get-rest-api --rest-api-id abc123 \
  --query '[name,endpointConfiguration.types,disableExecuteApiEndpoint]'

# execute-api に API キーなしで直接アクセスすると 403
curl -s -o /dev/null -w '%{http_code}\n' \
  https://abc123.execute-api.ap-northeast-1.amazonaws.com/prod/items
```
:::

根拠：
- [Protecting API endpoints (BP4) - AWS Best Practices for DDoS Resiliency](https://docs.aws.amazon.com/whitepapers/latest/aws-best-practices-ddos-resiliency/protecting-api-endpoints-bp4.html)
- [Use AWS WAF to protect your REST APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-control-access-aws-waf.html)
- [Disable the default endpoint for REST APIs](https://docs.aws.amazon.com/apigateway/latest/developerguide/rest-api-disable-default-endpoint.html)

---

## ⑥ CloudFront → VPC オリジン → 内部 ALB → Private API（閉域を維持）

![Private API via VPC origin](/images/aws-web-ingress-patterns/06-private-api-vpc-origin.drawio.svg)

API Gateway を Private API のまま CloudFront から公開したい場合の構成です。VPC オリジンで内部 ALB に入り、ALB のターゲットに `execute-api` インターフェイス VPC エンドポイントの IP を登録します。

### 設定プラクティス

- Private API は**リソースポリシーが必須**です。`aws:SourceVpc` / `aws:SourceVpce` で特定の VPC・VPC エンドポイントだけを許可し、全許可にはしません
- VPC エンドポイントポリシーも併用して、データペリメーターを強化します
- ALB のターゲットグループのヘルスチェックは `200,403` を成功扱いにします（VPC エンドポイントはヘッダーなしの呼び出しに 403 を返すため）
- ALB の SG は①と同じく、`CloudFront-VPCOrigins-Service-SG` だけを許可します

:::details 確認コマンド
```bash
# PRIVATE になっているか、リソースポリシーに aws:SourceVpce の条件があるか
aws apigateway get-rest-api --rest-api-id abc123 \
  --query '[endpointConfiguration.types,endpointConfiguration.vpcEndpointIds,policy]'
```
:::

根拠：
- [Accessing private Amazon API Gateway endpoints through custom Amazon CloudFront distribution using VPC Origins (AWS Blog)](https://aws.amazon.com/blogs/compute/accessing-private-amazon-api-gateway-endpoints-through-custom-amazon-cloudfront-distribution-using-vpc-origins/)
- [Private REST APIs in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-apis.html)
- [Create a private API](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-private-api-create.html)

---

## ⑦ Global Accelerator + 内部 ALB（固定 Anycast IP / HTTP 以外も可）

![Global Accelerator](/images/aws-web-ingress-patterns/07-global-accelerator.drawio.svg)

固定 IP が必要な場合や、TCP / UDP を受けたい場合の選択肢です。内部 ALB もエンドポイントにできます。

### ⚠ 要注意ポイント

- 内部 ALB と EC2 のエンドポイントでは、クライアント IP の保持が**常に有効**で、無効化できません
- クライアント IP を保持する場合、**GWLB での検査、AWS Network Firewall、NACL が適用されません**。防御は SG と WAF で設計する必要があります
- SG の許可リストには、Global Accelerator の IP ではなく**クライアント IP** を入れる必要があります
- VPC Block Public Access を有効にすると、Global Accelerator からのインバウンドも止まります

HTTP(S) だけを受けるなら、キャッシュや WAF（Global）の面で CloudFront＋VPC オリジンのほうが有利です。

:::details 確認コマンド
```bash
# Global Accelerator の API は us-west-2 で呼び出す
aws globalaccelerator list-accelerators --region us-west-2 \
  --query 'Accelerators[].[Name,Status,IpSets[0].IpAddresses]'

# エンドポイントごとにクライアント IP 保持が有効か（内部 ALB は常に true）
aws globalaccelerator list-endpoint-groups --listener-arn <listener-arn> --region us-west-2 \
  --query 'EndpointGroups[].EndpointDescriptions[].[EndpointId,ClientIPPreservationEnabled]' \
  --output table
```
:::

根拠：
- [Guidelines and restrictions for client IP address preservation - AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.how-to-enable-preservation.html)
- [How the client IP address is preserved - AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/preserve-client-ip-address.headers.html)

---

## ⑧ 公開 ALB で直接受ける（CloudFront なし）

![公開ALB直受け](/images/aws-web-ingress-patterns/08-public-alb-direct.drawio.svg)

攻撃者のリクエストも ALB までは届く前提で、ALB 自体を固める構成です。

### 設定プラクティス（Security Hub の ELB コントロールに対応）

| コントロール | 内容 |
|---|---|
| ELB.1 | HTTP → HTTPS リダイレクト |
| ELB.4 | `routing.http.drop_invalid_header_fields.enabled = true`（ELB.12 を有効にしているなら無効化してよい） |
| ELB.5 | アクセスログの有効化 |
| ELB.6 | 削除保護 |
| ELB.12 | desync mitigation mode を **defensive（既定）または strictest** にする |
| ELB.16 | WAF web ACL の関連付け |

- `waf.fail_open.enabled` は既定の false（fail-closed）のままにします
- ターゲットの SG は ALB の SG からだけ許可します
- エッジでの吸収やキャッシュがないため、可能なら①への移行を検討しましょう

:::details 確認コマンド
```bash
# desync 対策・不正ヘッダー破棄・WAF fail open・ログ・削除保護の設定値
aws elbv2 describe-load-balancer-attributes --load-balancer-arn <alb-arn> \
  --query "Attributes[?Key=='routing.http.desync_mitigation_mode' || Key=='routing.http.drop_invalid_header_fields.enabled' || Key=='waf.fail_open.enabled' || Key=='access_logs.s3.enabled' || Key=='deletion_protection.enabled']" \
  --output table

# WAF の web ACL が関連付いているか
aws wafv2 get-web-acl-for-resource --resource-arn <alb-arn> \
  --query 'WebACL.Name' --output text

# HTTP が HTTPS にリダイレクトされるか（301 と Location）
curl -sI http://www.example.com/ | grep -iE '^(HTTP|location)'
```
:::

根拠：
- [Edit attributes for your Application Load Balancer](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/edit-load-balancer-attributes.html)
- [Security Hub CSPM controls for Elastic Load Balancing](https://docs.aws.amazon.com/securityhub/latest/userguide/elb-controls.html)

---

## WAF の位置付け

ここまでの①〜⑧は「誰がオリジンに届くか（経路）」の話でした。WAF は層が違い、「届いたリクエストの中身を検査する」ものです。

![防御レイヤーとWAFの位置付け](/images/aws-web-ingress-patterns/09-defense-layers.drawio.svg)

- **B だけ（WAF だけ）の場合**：オリジンに直接来られると素通りです。冒頭の「第三者の CloudFront 経由で WAF を迂回される」話はこれです
- **C・D だけ（経路の限定だけ）の場合**：正規の CloudFront を通って来る SQL インジェクションや HTTP flood は止まりません
- **基本形は A＋B＋C＋D** です（A の Shield Standard は CloudFront や GA を使えば自動で効きます）。E（リージョンの WAF）は、必要なときに足します

WAF の web ACL は CloudFront（Global）と ALB・API Gateway の REST API など（Regional）に付けられますが、**NLB・Global Accelerator・S3・Lambda 関数 URL・API Gateway の HTTP API には直接付けられません**。また AWS は、公開 Web アプリでは**制約がない限り CloudFront と組み合わせて WAF を置く**ことを推奨しています。

### パターンごとの WAF の置き場所

| パターン | WAF の置き場所 | 補足 |
|---|---|---|
| ① VPC オリジン | CloudFront（Global） | 経路が閉じているので ALB 側は基本不要。アプリ固有ルールを分けたいときだけ ALB にも |
| ② 公開 ALB＋秘密ヘッダー | CloudFront（Global）＋必要に応じて ALB（Regional） | 秘密ヘッダーが漏れたときの保険として ALB 側も検討 |
| ③ S3（OAC） | CloudFront（Global） | S3 には付けられない |
| ④ Lambda 関数 URL（OAC） | CloudFront（Global） | 関数 URL には付けられない |
| ⑤ Regional API Gateway | CloudFront（Global）＋必要に応じて REST API ステージ（Regional） | HTTP API には付けられない |
| ⑥ Private API | CloudFront（Global） | 経路は①と同じ |
| ⑦ Global Accelerator | ALB（Regional） | GA には付けられない。クライアント IP が保持されるので、IP ベースのルールがそのまま効く |
| ⑧ 公開 ALB 直受け | ALB（Regional）※必須 | 唯一の L7 防御。Shield Advanced も検討 |

WAF の置き場所ごとの注意点（ALB 側では CloudFront の IP が見える、ボディを検査できるサイズの違い、内部エラー時の挙動）や、ルールの組み方・運用は、別の記事にまとめました。

https://zenn.dev/anfangd/articles/aws-waf-placement

---

## 共通：CloudFront のビューワー側チェックリスト

どのパターンでも CloudFront を前段に置くなら、次の Security Hub（AWS Foundational Security Best Practices）コントロールが目安になります。

| コントロール | 内容 |
|---|---|
| CloudFront.3 | ビューワーの HTTPS を必須にする（`allow-all` は NG） |
| CloudFront.5 | ログの有効化 |
| CloudFront.6 | WAF の関連付け |
| CloudFront.9 | カスタムオリジンへの通信を暗号化する |
| CloudFront.13 | S3 オリジンに OAC を使う |
| CloudFront.15 | 推奨 TLS ポリシー（`TLSv1.2_2025` / `TLSv1.3_2025`）を使う |
| CloudFront.16 | Lambda 関数 URL オリジンに OAC を使う |

:::details 確認コマンド
```bash
# ビューワープロトコル・TLS ポリシー・WAF・各オリジンの閉じ方（VPC オリジン / OAC）を一覧
aws cloudfront get-distribution-config --id E123EXAMPLE \
  --query 'DistributionConfig.{Viewer:DefaultCacheBehavior.ViewerProtocolPolicy,TLS:ViewerCertificate.MinimumProtocolVersion,WebACL:WebACLId,Origins:Origins.Items[].{Id:Id,Domain:DomainName,VpcOrigin:VpcOriginConfig.VpcOriginId,OAC:OriginAccessControlId}}'
```
:::

- [Security Hub CSPM controls for Amazon CloudFront](https://docs.aws.amazon.com/securityhub/latest/userguide/cloudfront-controls.html)

## まとめ

- 「ドメインが知られたら迂回される」問題は、**秘密ヘッダーで照合すれば防げます**。ただしそれはフォールバックで、ローテーションなど運用の負担が残ります
- CT Log によってホスト名は知られる前提です。**「隠す」より「届かない」構成**を選びましょう
- 新規構築なら、まず **VPC オリジン（ALB / NLB / EC2）と OAC（S3 / Lambda 関数 URL）** を検討します。fail-closed で決定論的な、運用しやすい構成になります
- 既存の公開 ALB 構成は、ステージングディストリビューションで検証しながら VPC オリジンへ段階的に移行できます

