---
title: "昔習った DMZ は、今の AWS ではどこに行ったのか？"
emoji: "🏰"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","VPC","CloudFront","NATGateway","Network"]
published: false
---

:::message
この記事は、Claude と一緒に AWS 公式ドキュメントを調べ、構成図を作りながら整理した内容をまとめたものです。各サービスの仕様は、記事中のリンク先の公式ドキュメントで確認できます。
:::

## はじめに

【ハジメが書く：基本情報技術者試験で DMZ を習ったこと、今の AWS の構成と比べてみようと思ったきっかけ】

## TL;DR

- 昔の DMZ は「サーバーを置く区画」でした。今の AWS では、DMZ にサーバーを置く構成はほぼなくなっています
- 残っている形は 3 つです。**① なくす型**（エッジ＋VPC オリジン＋Regional NAT GW）、**② 小さくする型**（public サブネットにマネージドな入口・出口だけ置く）、**③ 集約する型**（入口・出口・検査の VPC をアカウントごと分ける）
- 変わったのは「境界をどこに置くか」です。① はエッジ、② はサブネット、③ はアカウントが境界になります
- ① でも IGW はなくなりません。ルートではなく「この VPC はインターネットから受ける」という目印として使われます

## 昔の DMZ はどんな構成だったか

基本情報技術者試験などで習う DMZ は、FW を 2 枚重ね、その間の区画に外と接するサーバーを置く構成です。

![昔の DMZ。インターネットと内部ネットワークの間に FW を 2 枚置き、その間の DMZ に Web・DNS・メール中継・外向きプロキシ・踏み台を置く](/images/where-did-the-dmz-go-aws/legacy-dmz.png)

DMZ には、外から来る通信を受けるサーバー（Web、DNS、メール）と、中から外に出るためのサーバー（外向きプロキシ）、管理用の踏み台が同居していました。どの区画にいるかで、どこまで信頼するかが決まる考え方です。

## DMZ にいたサーバーはどこに行ったか

まず、DMZ に置いていた役割ごとに、今の AWS で何が担当しているかを並べます。

| 昔の DMZ の役割 | 今の主な担当 | VPC の中にサーバーが要るか |
| --- | --- | --- |
| Web・リバースプロキシ | CloudFront + WAF（＋VPC オリジン） | 要らない（エッジ） |
| 外部向け DNS | Route 53 の公開ホストゾーン | 要らない |
| メール送信の中継 | Amazon SES | 要らない |
| 踏み台サーバー | Session Manager、EC2 Instance Connect Endpoint | 要らない（IAM で制御） |
| 外向きプロキシ | NAT GW、Network Firewall、Route 53 Resolver DNS Firewall | NAT GW は要る（Regional なら public サブネット不要） |
| HTTP 以外の公開（SFTP、UDP など） | internet-facing NLB、Global Accelerator、Transfer Family | 場合による |

Web・DNS・メール・踏み台は、VPC の外のマネージドサービスか、IAM で制御する仕組みに移りました。最後まで VPC の中に何かが残るのは、外向きの通信と HTTP 以外の公開です。この 2 つが、次の 3 パターンの分かれ目になります。

## 今の構成は 3 つのパターンに分かれる

![今の AWS の 3 パターン。① なくす型、② 小さくする型、③ 集約する型](/images/where-did-the-dmz-go-aws/dmz-three-patterns.png)

### ① なくす型：エッジと IAM が境界になる

VPC に public サブネットを 1 つも作らない構成です。

- **入口**：CloudFront と WAF で受け、[VPC オリジン](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)でプライベートサブネットの内部 ALB に届けます。VPC オリジンは [2024 年 11 月に登場](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-cloudfront-vpc-origins/)しました。それまでは、VPC の中のオリジンを使うなら public サブネットに置き、アクセス制御を自分で作る必要がありました
- **出口**：[Regional NAT GW](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html) を使います。2025 年 11 月に出た機能で、NAT GW をサブネットではなく VPC に作るので、置き場所としての public サブネットが要りません
- **管理**：Session Manager を使います。インバウンドのポートを開ける必要がありません

昔の DMZ のリバースプロキシの役割は、CloudFront が VPC の外で担っています。境界は「VPC の外のエッジ」と「IAM」です。

Regional NAT GW については、AWS も「public サブネットがなくなるので、private にしたいリソースを誤って public なサブネットに置くリスクが減る」という趣旨を説明しています（[ブログ](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-amazon-vpc-regional-nat-gateway/)）。public サブネットは、置いた瞬間に「何かを置けてしまう場所」になるので、作らずに済むなら作らない方が管理が楽です。

気をつけることもあります。

- Regional NAT GW が新しい AZ に広がるまで、平均 15〜20 分、最大 60 分かかります（同じブログより）。[AWS re:Post の解説](https://repost.aws/articles/AR77CnVH2zR6KjoD4SvgNVAQ/understanding-amazon-vpc-regional-nat-gateway)によると、その間の通信は既存の AZ の NAT が AZ をまたいで処理します
- サブネットに紐づかないので、サブネット単位の設定は効きません

### ② 小さくする型：public サブネットに機器だけ置く

public サブネットは残すものの、置くのはマネージドな入口・出口の機器だけで、EC2 は置かない構成です。

- **zonal NAT GW**：今までの NAT GW です。AZ ごとに public サブネットに置きます
- **internet-facing の ALB・NLB**：internet-facing の LB はノードが public IP を持ち（[ELB のドキュメント](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)）、IGW へのルートがあるサブネットに置きます。CloudFront を使えない HTTP の公開や、SFTP・MQTT・UDP などの公開がここに当たります

昔の DMZ から、サーバーを抜いて機器だけを残した形です。境界はサブネットのままです。

HTTP 以外の公開でも、② を避ける道はあります。[Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/secure-vpc-connections.html) は、内部 ALB や NLB、プライベートサブネットの EC2 を直接エンドポイントにできます。LB や EC2 に public IP は要らず、サブネットに IGW へのルートも要りません。SFTP なら Transfer Family に任せる手もあります。

② になりやすいのは、むしろ意図していない場合です。EKS や ECS で Ingress コントローラーに任せていると、internet-facing の ALB が作られ、気づかないうちに public サブネットに入口ができていることがあります。

### ③ 集約する型：アカウントが境界になる

マルチアカウント構成で、入口・出口・検査を担う VPC をネットワーク用のアカウントに集め、Transit Gateway で各アカウントの VPC とつなぐ構成です。

- **入口 VPC**：ALB などで受け、Transit Gateway 経由で各アプリ VPC に届けます
- **出口 VPC**：NAT GW を集約し、各アプリ VPC には IGW を置きません
- **検査**：真ん中に Network Firewall や GWLB を置き、出入りの通信と VPC 間の通信を検査します

[AWS のブログ](https://aws.amazon.com/blogs/networking-and-content-delivery/choosing-the-right-inspection-architecture-for-aws-network-firewall/)によると、Network Firewall には 2025 年に、専用の検査 VPC を作らずに Transit Gateway へ直接つなぐ形などが加わり、検査 VPC を自分で作る以外の選択肢が増えています。

FW を真ん中に置いて全部の出入りをそこで見る、という考え方は昔の DMZ とほぼ同じです。違いは、それが 1 つのネットワークの区画ではなく、アカウントの単位で切り出されていることです。

### どう組み合わせるか

3 つは排他ではなく、組み合わせて使います。

- 新しく作る Web アプリは ① を基本にする
- HTTP 以外の公開や、Regional NAT GW を使えない事情があるところだけ ② にする
- 組織全体で出入りをまとめて検査したい場合は、③ を土台にして、その上で各アプリを ① の形にする

【ハジメが書く：個人的にはどれを基本にするか、その理由】

## IGW の意味が変わった

① の構成を作ると、「public サブネットはないのに IGW は要る」という状態になります。

[VPC オリジン](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)も [Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/secure-vpc-connections.html) も、プライベートサブネットのリソースに届けるのに、VPC に IGW がアタッチされていることを求めます。どちらのドキュメントも、IGW は「この VPC がインターネットからの通信を受けることを示すため」に必要で、ルートテーブルに IGW へのルートを書く必要はない、と説明しています。

つまり IGW は、昔の「外に出るための出口」だけでなく、「この VPC はインターネットから受ける」という宣言の役割も持つようになっています。「IGW がアタッチされているか」で外との接点を把握する、という従来のチェックをそのまま使えるようにした設計とも言えます。

逆に言うと、「public サブネットがない＝外と接していない」とは限りません。① の VPC も、IGW がある以上はインターネットからの入口を持っています。

## ハマりどころ：VPC Block Public Access は VPC オリジンも止める

「public サブネットを作らない」を強制したいなら、[VPC Block Public Access（BPA）](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa.html)が候補になります。IGW を通るインターネットの通信を、アカウントやリージョン単位でまとめて止める機能です。

ただし、[影響範囲の一覧](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa-basics.html)を見ると、CloudFront の VPC オリジンも Global Accelerator も、BPA の対象として通信が止まると書かれています。inbound だけを止める ingress-only モードでも、VPC オリジンへの通信は止まります。

① の構成は「public サブネットがない」だけで、IGW 経由の受け口は持っています。そのため BPA を有効にするなら、VPC オリジンを使う VPC やサブネットを除外する必要があります。「public サブネットがないから BPA を入れても影響ない」と考えると、サービスが止まります。

## ここは書かない

- IPv6（egress-only IGW を使えば、外向きの IPv6 に NAT も public サブネットも要りません）
- Cloud WAN を使った集約型の詳細
- 各パターンの料金比較

## おわりに

【ハジメが書く：本音・次にやりたいこと】

## 関連する記事

「public サブネットをなくす」構成そのものは、すでに次の記事で詳しく紹介されています。

- [Amazon VPC からパブリックサブネットを無くしたい!（サーバーワークス）](https://blog.serverworks.co.jp/no-public-subnet)
- [パブリックサブネットレスなVPCにECS Express Mode環境を構築してみた（DevelopersIO）](https://dev.classmethod.jp/articles/ecs-express-mode-public-subnet-less/)

## 参考

- [Restrict access with VPC origins - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/private-content-vpc-origins.html)
- [Amazon CloudFront announces VPC origins](https://aws.amazon.com/about-aws/whats-new/2024/11/amazon-cloudfront-vpc-origins/)
- [Regional NAT gateways for automatic multi-AZ expansion - Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/nat-gateways-regional.html)
- [Introducing Amazon VPC Regional NAT Gateway](https://aws.amazon.com/blogs/networking-and-content-delivery/introducing-amazon-vpc-regional-nat-gateway/)
- [Secure VPC connections in AWS Global Accelerator](https://docs.aws.amazon.com/global-accelerator/latest/dg/secure-vpc-connections.html)
- [How Elastic Load Balancing works](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/how-elastic-load-balancing-works.html)
- [Choosing the right inspection architecture for AWS Network Firewall](https://aws.amazon.com/blogs/networking-and-content-delivery/choosing-the-right-inspection-architecture-for-aws-network-firewall/)
- [VPC BPA basics - Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/security-vpc-bpa-basics.html)
