---
title: "CloudFront の裏の ALB、本当に閉じていますか？ CSPM では見えない組み合わせの穴と、検知・移行の進め方"
emoji: "🚪"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","CloudFront","ALB","WAF","Security"]
published: false
---

:::message
この記事は、Claude と一緒に AWS 公式ドキュメントを調べ、個人の検証用アカウントで構成を再現しながら整理した内容をまとめたものです。各記述の根拠は、記事中のリンク先で確認できます。
:::

## はじめに

「CloudFront の前段に WAF、後ろに公開 ALB」という構成は、2015〜2017 年頃から定番でした。新しく作る構成は VPC オリジンに移りつつありますが、**当時作った構成がそのまま動き続けている環境**は今でもよく見かけます。

そうした環境を見直すときに、つまずきやすい点がいくつかあります。

- CloudFront に WAF を付けているので、ALB も守られていると思っている
- ALB の SG に CloudFront のプレフィックスリストを入れたので、閉じたつもりになっている
- 秘密ヘッダーを作った当時から、一度もローテーションしていない
- Security Hub のスコアは高いのに、実は ALB にインターネットから直接届く

どれも、リソースを 1 つずつ見ると問題がないように見えるのが厄介なところです。問題は**組み合わせ**にあります。

この記事では、こうした「あるある」を整理したうえで、次の 3 つを扱います。

1. なぜ CSPM（Security Hub CSPM のような設定チェック）では拾えないのか
2. 「WAF があるから大丈夫」は、どこまで正しいのか
3. 仕組みでどう検知し、どう安全に移行するか

あるべき構成（VPC オリジンなど）そのものは、こちらの記事にまとめています。

https://zenn.dev/anfangd/articles/aws-web-ingress-origin-protection

## TL;DR

- CSPM のチェックは**リソース単体**の設定を見ます。「CloudFront のオリジンになっている ALB に、CloudFront を通らずに届くか」という**組み合わせ**の問題は判定できません
- 経路を **構造で閉じる**（VPC オリジン）のと、**ルールで閉じる**（秘密ヘッダー、WAF）のは別物です。ルールは書き間違い・漏洩・Count モードのままで簡単に開きます
- ネットワーク到達性のツール（Network Access Analyzer、Security Hub の Network Scanning）は、「CloudFront の IP 帯から届く」ことまでしか分かりません。**自分の CloudFront か他人の CloudFront か**は区別できません
- 最後の確認は、**攻撃者と同じことを自分でやる**のが確実です。別アカウントに検証用の CloudFront を作り、自分の ALB をオリジンに指定して届くかを試します
- 移行は「秘密ヘッダーで応急処置 → 新しい内部 ALB と VPC オリジンを並行稼働 → ステージングディストリビューションで切り替え」の順に進めます。**既存の `aws_lb` の `internal` を書き換えるとリソースが作り直される**ので注意が必要です

## よくある 4 つの状態

まず、CloudFront の裏にある ALB がとりうる状態を 4 つに分けます。この記事では以降、この分類を使います。

| 状態 | ALB の SG | ALB のリスナー | 他人の CloudFront から届くか | 直接届くか |
|---|---|---|---|---|
| A. 公開のまま | `0.0.0.0/0` | 何でも forward | 届く | **届く** |
| B. プレフィックスリストだけ | CloudFront のプレフィックスリスト | 何でも forward | **届く** | 届かない |
| C. 秘密ヘッダー | CloudFront のプレフィックスリスト | ヘッダー一致なら forward、デフォルトは 403 | 届くが 403 | 届かない |
| D. VPC オリジン | `CloudFront-VPCOrigins-Service-SG` | 内部 ALB | 届かない | 届かない |

A は論外に見えますが、「CloudFront 経由でしか使わないから大丈夫」と思われたまま残っていることがあります。いちばん見落とされやすいのは B です。CloudFront のプレフィックスリストには**他人のディストリビューションも含む CloudFront 全体の IP** が入っているので、第三者が自分の CloudFront のオリジンにあなたの ALB を指定すれば、SG を通過してしまいます。

C は AWS のドキュメントにある公開 ALB 向けの構成で、今でも正しい対策です。ただし「ルールで防いでいる」構成なので、後で述べるように運用で穴が開く余地が残ります。

## なぜ CSPM では拾えないのか

Security Hub CSPM（AWS Foundational Security Best Practices）で、この構成に関係しそうなコントロールを並べてみます。

| コントロール | 見ているもの | 状態 A〜C で合格するか |
|---|---|---|
| CloudFront.6 | CloudFront に WAF が関連付いているか | 合格（CloudFront に WAF があれば） |
| CloudFront.9 | カスタムオリジンへの通信が暗号化されているか | 合格（HTTPS なら） |
| ELB.16 | ALB に WAF が関連付いているか | ALB に WAF を付けていれば合格 |
| EC2.18 | `0.0.0.0/0` を許可している SG が、許可されたポートだけか | **既定値では 80 / 443 は許可ポート**なので、状態 A でも合格 |

EC2.18 は既定値の `authorizedTcpPorts` が `[80, 443]` なので、`0.0.0.0/0` からの 443 を許可していても失敗になりません。Web サーバーとしては自然な判定ですが、「CloudFront の裏にいるはずの ALB」という文脈は持っていません。

つまり、**状態 A のまま全部緑**ということが起こりえます。CSPM が悪いのではなく、リソース単体の設定チェックという仕組みの性質です。「この ALB は CloudFront のオリジンである」という関係と、「その ALB の SG・リスナー・スキーム」を突き合わせないと判定できません。

- [Security Hub CSPM controls for Amazon CloudFront](https://docs.aws.amazon.com/securityhub/latest/userguide/cloudfront-controls.html)
- [Security Hub CSPM controls for Elastic Load Balancing](https://docs.aws.amazon.com/securityhub/latest/userguide/elb-controls.html)
- [Security Hub CSPM controls for Amazon EC2（EC2.18）](https://docs.aws.amazon.com/securityhub/latest/userguide/ec2-controls.html)

## 「WAF があるから大丈夫」を分解する

この話をすると、「でも WAF で防いでいるから」という反応がよく返ってきます。ここは、**経路を閉じる層**と**中身を検査する層**を分けて考えると整理できます。

| 経路の閉じ方 | WAF はエッジ（CloudFront）だけ | WAF はエッジ＋ALB |
|---|---|---|
| 開いている（状態 A・B） | ✕ 迂回されると WAF を一度も通らない | △ 迂回されても ALB の WAF は効く。ただし後述の制約あり |
| ルールで閉じる（状態 C） | ○ ヘッダーが漏れない限りは安全 | ○ ヘッダーが漏れたときの保険になる |
| 構造で閉じる（状態 D） | ◎ 推奨 | ◎ アプリ固有のルールを分けたいときだけ ALB にも |

ポイントは 3 つあります。

**1. エッジの WAF は、エッジを通ったリクエストにしか効かない**

当たり前ですが、いちばんよく混同される点です。CloudFront に付けた WAF は、ALB に直接来たリクエストや、他人の CloudFront 経由で来たリクエストを一切見ていません。「CloudFront.6 が緑」は「ALB が守られている」ことを意味しません。

**2. ALB に WAF を足しても、エッジと同じ防御にはならない**

迂回への保険として ALB にも WAF を付けるのは有効です。ただし、ALB 側の WAF には次の制約があります（詳しくは [WAF の記事](https://zenn.dev/anfangd/articles/aws-waf-placement)にまとめています）。

- CloudFront 経由のリクエストでは、送信元 IP が CloudFront の IP になります。IP・国・レートベースのルールを効かせるには forwarded IP の設定が必要で、`X-Forwarded-For` の先頭はクライアントが書き換えられます
- ボディの検査上限が 8 KB 固定です
- ALB 側だけに頼ると、L3/L4 の攻撃や大量のリクエストを ALB まで受けることになり、LCU の課金もかかります

**3. 秘密ヘッダーの照合は「ルールで閉じている」だけ**

AWS Security Blog（2020 年）で紹介された構成のように、ALB の WAF で秘密ヘッダーを照合する方法もあります。これは正しい対策ですが、ルールで閉じている以上、次のような理由で簡単に開きます。

- ルールを Count モードのまま運用している
- ルールの優先度やデフォルトアクション（Allow）の組み方を間違えている
- ヘッダーの値が、ログや設定のエクスポートなどから漏れる
- ローテーションしていないので、過去に漏れた値がそのまま使える

VPC オリジンのように**経路そのものがない**構成なら、こうした運用ミスで穴が開くことはありません。fail-closed とはこのことです。WAF は「経路の代わり」ではなく、**届いたリクエストの中身を検査することに専念させる**のが正しい位置付けです。

- [How to enhance Amazon CloudFront origin security with AWS WAF and AWS Secrets Manager (AWS Security Blog)](https://aws.amazon.com/blogs/security/how-to-enhance-amazon-cloudfront-origin-security-with-aws-waf-and-aws-secrets-manager/)
- [Using forwarded IP addresses in AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/waf-rule-statement-forwarded-ip-address.html)

## 仕組みで検知する

では、どうすれば状態 A〜C を見つけられるでしょうか。手段ごとに「何が分かって、何が分からないか」を整理すると、次のようになります。

| 手段 | A. 公開のまま | B. プレフィックスリストだけ | C. 秘密ヘッダー | D. VPC オリジン |
|---|---|---|---|---|
| Security Hub CSPM | 見逃しうる（EC2.18 は 443 を許可） | 見逃す | 見逃す | — |
| Network Access Analyzer | 検出できる | CloudFront の IP 帯からの経路として出る（自他の区別なし） | B と同じに見える | 経路なし |
| Security Hub の Network Scanning | 検出できる | 届かないと判定されるはず（後述） | 同左 | 対象外 |
| 構成の突き合わせ（スクリプト） | 分類できる | 分類できる | 分類できる | 分類できる |
| 攻撃者視点のテスト | 確定できる | **確定できる** | **確定できる** | 確定できる |

ネットワークのツールは、パケットが届くかどうかを判定するものです。状態 B と C の違いは「届いたあとに ALB が 403 を返すかどうか」という L7 の話なので、ネットワークのツールでは区別できません。ここが「組み合わせの穴」が見つけにくい理由です。

そこで、次の 3 段で検知します。

### 1 段目：構成を突き合わせて分類する

まずは設定を読むだけで分類します。CloudFront のプレフィックスリストを SG に入れている ALB は、「CloudFront の裏にいるつもりの ALB」だと判断できます。そうした ALB について、リスナーにヘッダー条件があるか、デフォルトアクションが 403 になっているかを見ます。

```bash:audit-cf-alb.sh
#!/usr/bin/env bash
# internet-facing な ALB を洗い出し、CloudFront の裏にいる前提でどう閉じているかを分類する
set -euo pipefail
REGION="${AWS_REGION:-ap-northeast-1}"

CF_PL=$(aws ec2 describe-managed-prefix-lists --region "$REGION" \
  --filters Name=prefix-list-name,Values=com.amazonaws.global.cloudfront.origin-facing \
  --query 'PrefixLists[0].PrefixListId' --output text)

{
printf '%s\t%s\t%s\t%s\t%s\t%s\n' NAME SG_SOURCE HEADER_RULE DEFAULT_ACTION WAF VERDICT

aws elbv2 describe-load-balancers --region "$REGION" \
  --query "LoadBalancers[?Type=='application' && Scheme=='internet-facing'].[LoadBalancerArn,LoadBalancerName,join(',',SecurityGroups)]" \
  --output text |
while IFS=$'\t' read -r arn name sgs; do
  # SG のインバウンド許可元：/0 があれば open、CloudFront のプレフィックスリストがあれば cf-pl
  # shellcheck disable=SC2086
  src=$(aws ec2 describe-security-groups --region "$REGION" --group-ids ${sgs//,/ } \
    --query 'SecurityGroups[].IpPermissions[]' --output json |
    jq -r --arg pl "$CF_PL" '
      [ .[] | (.IpRanges[]?.CidrIp), (.Ipv6Ranges[]?.CidrIpv6), (.PrefixListIds[]?.PrefixListId) ] as $s
      | if any($s[]; . == "0.0.0.0/0" or . == "::/0") then "open"
        elif any($s[]; . == $pl) then "cf-pl"
        else "other" end')

  # リスナールール：http-header 条件があるか、デフォルトルールの最終アクションは何か
  hdr=no; def=""
  for l in $(aws elbv2 describe-listeners --region "$REGION" --load-balancer-arn "$arn" \
               --query 'Listeners[].ListenerArn' --output text); do
    rules=$(aws elbv2 describe-rules --region "$REGION" --listener-arn "$l" --output json)
    jq -e '[.Rules[].Conditions[]? | select(.Field == "http-header")] | length > 0' <<<"$rules" >/dev/null && hdr=yes
    def+="$(jq -r '.Rules[] | select(.IsDefault) | .Actions[-1].Type' <<<"$rules") "
  done

  # ALB に関連付いた WAF
  waf=$(aws wafv2 get-web-acl-for-resource --region "$REGION" --resource-arn "$arn" \
          --query 'WebACL.Name' --output text 2>/dev/null || true)
  if [ -z "$waf" ] || [ "$waf" = "None" ]; then waf="-"; fi

  case "$src/$hdr" in
    open/*)    verdict="A: インターネットから直接届く" ;;
    cf-pl/no)  verdict="B: 他人の CloudFront で迂回できる可能性" ;;
    cf-pl/yes) verdict="C: 秘密ヘッダー（デフォルトが 403 か、ローテーションを確認）" ;;
    *)         verdict="要確認" ;;
  esac

  printf '%s\t%s\t%s\t%s\t%s\t%s\n' "$name" "$src" "$hdr" "${def% }" "$waf" "$verdict"
done
} | column -t -s $'\t'
```

出力はこのようになります。

```text
NAME          SG_SOURCE  HEADER_RULE  DEFAULT_ACTION   WAF      VERDICT
legacy-web    open       no           forward          -        A: インターネットから直接届く
legacy-api    cf-pl      no           forward          api-acl  B: 他人の CloudFront で迂回できる可能性
fallback-web  cf-pl      yes          fixed-response   -        C: 秘密ヘッダー（デフォルトが 403 か、ローテーションを確認）
```

<!-- TODO: 検証環境で実行した実際の出力に差し替える -->

注意点が 2 つあります。

- 秘密ヘッダーを ALB の WAF で照合している場合（2020 年のブログの構成）は、リスナーにヘッダー条件が出てこないので B と判定されます。WAF 列に名前が出ていたら、WAF のルールも確認してください
- このスクリプトは 1 リージョン・1 アカウント分です。マルチアカウントで回すなら、組織の各アカウントに AssumeRole してループさせます

### 2 段目：ネットワーク到達性を確かめる

次に、ネットワークとして本当に届くかを確かめます。手段は 2 つあります。

**Network Access Analyzer** は、IGW・ルートテーブル・NACL・SG を評価して、インターネットから届く経路を洗い出します。ロードバランサーも経路上のリソースとして扱えます。

```bash
# インターネットゲートウェイから入ってくる経路をすべて対象にするスコープ
SCOPE_ID=$(aws ec2 create-network-insights-access-scope \
  --match-paths '[{"Source":{"ResourceStatement":{"ResourceTypes":["AWS::EC2::InternetGateway"]}}}]' \
  --query 'NetworkInsightsAccessScope.NetworkInsightsAccessScopeId' --output text)

ANALYSIS_ID=$(aws ec2 start-network-insights-access-scope-analysis \
  --network-insights-access-scope-id "$SCOPE_ID" \
  --query 'NetworkInsightsAccessScopeAnalysis.NetworkInsightsAccessScopeAnalysisId' --output text)

# 数分待ってから、ALB を経由する経路だけを取り出す
aws ec2 get-network-insights-access-scope-analysis-findings \
  --network-insights-access-scope-analysis-id "$ANALYSIS_ID" --output json |
  jq -r '.AnalysisFindings[]
         | select(tostring | test("loadbalancer/app/"))
         | [ .. | strings | select(test("loadbalancer/app/")) ] | unique | .[]' | sort -u
```

<!-- TODO: 状態 B の ALB で、送信元アドレスが CloudFront の IP 帯として出ることを確認する -->

状態 A の ALB は、送信元が `0.0.0.0/0` 相当の経路として出てきます。状態 B・C の ALB も、送信元が CloudFront の IP 帯に絞られた経路として出てきます。**ここで B と C は区別できません**し、その CloudFront が自分のものか他人のものかも分かりません。

**Security Hub の Network Scanning**（2026 年 7 月に追加）は、AWS が所有する IP アドレスから実際にポートスキャンをかけて、外から本当に届くかを確かめる機能です。ALB にも対応していて、DNS 名を解決してスキャンします。

```bash
aws securityhub enable-security-hub-feature-v2 --feature-name NETWORK_SCANNING
```

状態 A の ALB は、443 番ポートが届くという検出結果として出てきます。一方、スキャンは CloudFront を経由しないので、プレフィックスリストで絞った B・C の ALB は「届かない」と判定されるはずです。つまり Network Scanning が静かでも、B の穴は残っている可能性があります。

<!-- TODO: 検証環境で、状態 B の ALB が Network Scanning で Informational（開いているポートなし）になることを確認する -->

- [How Network Access Analyzer works](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/how-network-access-analyzer-works.html)
- [Getting started with Network Access Analyzer using the AWS CLI](https://docs.aws.amazon.com/vpc/latest/network-access-analyzer/getting-started-cli.html)
- [Network Scanning in Security Hub](https://docs.aws.amazon.com/securityhub/latest/userguide/securityhub-v2-network-scanning.html)
- [AWS Security Hub now offers Network Scanning to identify publicly reachable resources (2026)](https://aws.amazon.com/about-aws/whats-new/2026/07/aws-security-hub-network-scanning/)

### 3 段目：攻撃者と同じことを自分でやる

最後は、B と C を確実に見分けるテストです。やることは攻撃者と同じで、**別のアカウントに検証用の CloudFront を作り、自分の ALB をオリジンに指定して**届くかを試します。

オリジンのホスト名は、CT Log で調べられるものを使います（[CT Log の記事](https://zenn.dev/anfangd/articles/ct-log-hostname-exposure-ca-ct)の「手を動かして確かめる」の 1 を参照）。攻撃者もまさにこの手順でホスト名を見つけます。

:::message alert
このテストは、必ず**自分が管理しているリソース**に対してだけ行ってください。AWS の侵入テストのポリシーも確認しておきましょう。
- [Penetration Testing - AWS](https://aws.amazon.com/security/penetration-testing/)
:::

```hcl:probe.tf
# 検証用の「別アカウント」に作る、攻撃者役の CloudFront
variable "target_origin_domain" {
  description = "確認したい ALB のオリジン用ホスト名（例: origin.example.com）"
  type        = string
}

data "aws_cloudfront_cache_policy" "disabled" {
  name = "Managed-CachingDisabled"
}

resource "aws_cloudfront_distribution" "probe" {
  enabled = true
  comment = "origin bypass probe (own resources only)"

  origin {
    domain_name = var.target_origin_domain
    origin_id   = "probe"

    custom_origin_config {
      http_port              = 80
      https_port             = 443
      origin_protocol_policy = "https-only"
      origin_ssl_protocols   = ["TLSv1.2"]
    }
  }

  default_cache_behavior {
    target_origin_id       = "probe"
    viewer_protocol_policy = "https-only"
    allowed_methods        = ["GET", "HEAD"]
    cached_methods         = ["GET", "HEAD"]
    cache_policy_id        = data.aws_cloudfront_cache_policy.disabled.id
  }

  restrictions {
    geo_restriction {
      restriction_type = "none"
    }
  }

  viewer_certificate {
    cloudfront_default_certificate = true
  }
}

output "probe_url" {
  value = "https://${aws_cloudfront_distribution.probe.domain_name}/"
}
```

```bash
# 検証用 CloudFront 経由
curl -s -o /dev/null -w '%{http_code}\n' "$(terraform output -raw probe_url)"

# ALB に直接（DNS 名で、Host ヘッダーはオリジン用ホスト名にする）
curl -s -o /dev/null -w '%{http_code}\n' --max-time 10 \
  --connect-to origin.example.com:443:my-alb-123456.ap-northeast-1.elb.amazonaws.com:443 \
  https://origin.example.com/
```

期待する結果は次のとおりです。

| 状態 | ALB に直接 | 検証用 CloudFront 経由 |
|---|---|---|
| A. 公開のまま | 200 | 200 |
| B. プレフィックスリストだけ | タイムアウト | **200（迂回できてしまう）** |
| C. 秘密ヘッダー | タイムアウト | 403 |
| D. VPC オリジン | 届かない（内部 ALB） | 502 / 504 などのエラー |

<!-- TODO: 検証環境で実際に試した結果（ステータスコードとスクショ）に差し替える -->

B で 200 が返ってきたら、その ALB には CloudFront の WAF を通らずに誰でもアクセスできる状態です。テストが終わったら、検証用の CloudFront は無効化して削除しておきましょう。

## 安全に移行する

見つけたあとの直し方です。いきなり VPC オリジンに切り替えるのではなく、穴をふさぐ応急処置と、構成を変える本移行を分けて進めます。

### ステップ 1：秘密ヘッダーで応急処置する（A・B → C）

まず B の穴をふさぎます。順番を間違えると正規のリクエストまで 403 になるので、次の順で進めます。

1. CloudFront のオリジン設定に、ランダムな値のカスタムヘッダー（例：`X-Origin-Verify`）を追加し、デプロイが終わるのを待つ
2. ALB のリスナーに、「そのヘッダーが一致したら forward」というルールを優先度の高い位置に追加する
3. ALB のアクセスログで、正規のリクエストがすべてヘッダー付きで来ていることを確かめる
4. リスナーのデフォルトアクションを `fixed-response`（403）に変える
5. A の場合は、SG の `0.0.0.0/0` を CloudFront のプレフィックスリストに置き換える

ヘッダーの値は秘密情報なので、Secrets Manager で管理し、定期的にローテーションします。ローテーションも「新しい値を追加 → ALB で両方許可 → 古い値を CloudFront から外す → ALB で古い値を拒否」の順です。

### ステップ 2：VPC オリジンへ移行する（C → D）

ここで Terraform を使っている人がはまりやすい点があります。ALB のスキーム（internet-facing / internal）は**作成後に変更できず、変えるとリソースが作り直されます**。既存の `aws_lb` の `internal = false` を `true` に書き換えて `apply` すると、旧 ALB が削除されて新しい ALB が作られるので、その間はサービスが止まります。

そのため、既存の ALB は残したまま、**新しい内部 ALB を別のリソースとして作り、並行稼働させてから切り替えます**。

1. プライベートサブネットに新しい内部 ALB を作る（既存の `aws_lb` は書き換えない）
2. アプリのターゲット（ECS サービスなど）を、新しい ALB 側のターゲットグループにも登録する
3. 新しい ALB を VPC オリジンとして作成し、状態が `Deployed` になるのを待つ（最大 15 分ほど）
4. 新しい ALB の SG に `CloudFront-VPCOrigins-Service-SG` からのインバウンドを許可する
5. **ステージングディストリビューション**を作り、オリジンを VPC オリジンに差し替える
6. 継続的デプロイポリシーで、まずはヘッダー指定（`aws-cf-cd-` で始まるヘッダー）で自分のリクエストだけを流して確かめ、次に重み付けで一部のトラフィックを流す
7. 問題がなければ、ステージングの設定をプライマリに昇格（promote）する
8. しばらく様子を見てから、旧 ALB の SG を閉じ、旧 ALB を削除する

```hcl
# NG：既存の ALB を書き換えると作り直しになる
resource "aws_lb" "web" {
  internal = true # false から変更すると replace
  # ...
}

# OK：新しい ALB を別リソースとして作り、並行稼働させる
resource "aws_lb" "web_internal" {
  name               = "web-internal"
  internal           = true
  load_balancer_type = "application"
  subnets            = var.private_subnet_ids
  security_groups    = [aws_security_group.alb_internal.id]
}
```

切り戻しは、昇格前ならステージングへの振り分けを止めるだけで済みます。昇格後は、旧 ALB が残っている間なら、プライマリのオリジンを旧 ALB に戻せばすぐに戻せます。旧 ALB を消すのは、切り戻す可能性がなくなってからにしましょう。

- [Restrict access with VPC origins - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [Use CloudFront continuous deployment to safely test CDN configuration changes](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/continuous-deployment.html)
- [AWS::ElasticLoadBalancingV2::LoadBalancer（Scheme は Update requires: Replacement）](https://docs.aws.amazon.com/AWSCloudFormation/latest/TemplateReference/aws-resource-elasticloadbalancingv2-loadbalancer.html)

### ステップ 3：再発を防ぐ

直したあとは、同じ構成が新しく生まれないようにします。

- **作る段階で止める**：Terraform の plan に対して、internet-facing な ALB を作るときは理由を明記させるルールを conftest などで入れる
- **定期的に見る**：1 段目のスクリプトを定期実行し、Network Scanning を有効にしておく
- **変更のたびに確かめる**：オリジンまわりを変えたときは、3 段目の検証用 CloudFront でテストする

```rego:policy/alb.rego
package main

import rego.v1

# internet-facing な ALB には、公開する理由をタグで明記させる
deny contains msg if {
  rc := input.resource_changes[_]
  rc.type == "aws_lb"
  rc.change.after.internal == false
  not rc.change.after.tags.public_alb_reason
  msg := sprintf("%s: internet-facing な ALB には public_alb_reason タグで公開する理由を書いてください", [rc.address])
}
```

```bash
terraform plan -out tfplan && terraform show -json tfplan > tfplan.json
conftest test tfplan.json --policy policy/
```

## AI に棚卸しさせるときの使いどころ

最近は、生成 AI に構成を読ませてセキュリティ上の問題を探してもらうことが増えました。こうした「組み合わせの穴」は、リソースをまたいで設定を読み比べる作業なので、AI に棚卸しさせると気づきやすい種類の問題です。

一方で、AI が見つけた結果を毎回人が確かめるのでは続きません。**発見は AI に任せてもいいけれど、再発防止は仕組みでやる**。この記事の 3 段の検知は、そのための仕組みです。

## まとめ

- CloudFront の裏にある ALB の問題は**組み合わせ**にあり、リソース単体を見る CSPM では「全部緑なのに直接届く」ということが起こりえます
- WAF は中身を検査する層で、経路を閉じる代わりにはなりません。**構造で閉じる（VPC オリジン）**のが fail-closed な答えで、秘密ヘッダーや WAF は「ルールで閉じる」フォールバックです
- 検知は「構成の突き合わせ → ネットワーク到達性 → 攻撃者視点のテスト」の 3 段で行います。ネットワークのツールだけでは、自分の CloudFront と他人の CloudFront を区別できません
- 移行は秘密ヘッダーで応急処置してから、新しい内部 ALB と VPC オリジンを並行稼働させ、ステージングディストリビューションで切り替えます。既存の ALB のスキームを書き換えないことが大事です

関連する記事はこちらです。

https://zenn.dev/anfangd/articles/aws-web-ingress-origin-protection

https://zenn.dev/anfangd/articles/aws-waf-placement

https://zenn.dev/anfangd/articles/ct-log-hostname-exposure-ca-ct
