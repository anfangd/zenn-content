---
title: "そのステータスコード、どの層で返していますか？"
emoji: "🚦"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["AWS","HTTP","CloudFront","WAF","WebSocket"]
published: false
---

:::message
この記事は、Claude と一緒に RFC や AWS の公式ドキュメントを調べ、図を作りながら整理した内容をまとめたものです。各記述の根拠は、記事中のリンク先で確認できます。
:::

## はじめに

【ハジメが書く：書こうと思ったきっかけ。例：「メンテナンス中は何を返すのが普通？」と聞かれて答えたら、ステータスコードそのものより「どこで返すか」のほうが運用では効くと感じた話など。話し言葉や音声の文字起こしで OK】

この記事では、メンテナンス、管理画面、リダイレクト、API、5xx、WebSocket について、ステータスコードを**何にするか**と**どの層で返すか**をまとめます。

## TL;DR

- ステータスコードは、受け取る相手（ブラウザ、クローラー、監視、API クライアント、ロードバランサー）に何をしてほしいかで選ぶ
- メンテナンス中は `503` に `Retry-After` を付ける。`200` や `302` でメンテページを見せない
- 返す層は、手前ほどアプリが止まっていても確実に返せ、奥ほど細かく判断できる。メンテナンスの切り替えは、止める対象より手前の層に置く
- AWS なら WAF のカスタムレスポンスが扱いやすい。管理者の IP を条件から外せて、`Retry-After` も付けられる。ALB の固定レスポンスはヘッダーを付けられない
- ユーザー向けと管理者向けでホスト名やパスが分かれていると、「利用者向けだけ止めて、管理者は動作確認する」が条件 1 つで書ける
- WebSocket は接続済みのコネクションがメンテを素通りする。サーバーから close code を付けて閉じ、クライアントはジッター付きでつなぎ直す

## ステータスコードは、受け取る相手に何をしてほしいかで選ぶ

ステータスコードを読むのは人間ではなく、プログラムです。

| 受け取る相手 | ステータスコードを見て何をするか |
| --- | --- |
| ブラウザ | リダイレクトに従う、キャッシュを使う、認証を求める |
| 検索エンジンのクローラー | インデックスに残す、消す、あとで来直す |
| 監視・アラート | 障害として通知するかを決める |
| API クライアント | リトライするか、ユーザーにエラーを見せるかを決める |
| ロードバランサーや CDN | ターゲットを外す、エラーをキャッシュする |

「メンテ中だから 200 でお知らせページを出す」は人間には伝わりますが、クローラーと監視には「正常」と伝わります。迷ったら「この相手に何をしてほしいか」から逆算します。

## ステータスコードを返せる層は、思っているより多い

AWS で CloudFront と ALB を使う構成だと、リクエストはおおむね次の順に通ります。どの層も、自分でレスポンスを返してそこで止められます。

![手前の層ほど、アプリが止まっていても返せる。WAF、CloudFront Functions、ALB リスナールール、アプリの順にリクエストが流れ、それぞれが自分でレスポンスを返せる](/images/http-status-code-where-to-return/status-code-layers.png)

| 層 | 返し方 | ヘッダーを付けられるか | 判断に使える条件 |
| --- | --- | --- | --- |
| AWS WAF | Block アクションのカスタムレスポンス | 付けられる | IP セット、ホスト、パス、メソッド、ヘッダーなど |
| CloudFront Functions | viewer request でレスポンスを生成 | 付けられる | コードで自由に書ける |
| ALB | リスナールールの `fixed-response` アクション | **付けられない**。ボディは 1,024 文字まで | ホスト、パス、ヘッダー、メソッド、送信元 IP |
| アプリ | 何でも | 付けられる | ユーザーの状態やデータまで見られる |

手前の層ほど、アプリやデータベースが止まっていても確実に返せます。奥の層ほど、「このユーザーは書き込み中か」のような細かい判断ができます。

注意が要るのは 2 つです。1 つは、ALB の固定レスポンスにはカスタムヘッダーを付けられないことです。メンテナンスで `Retry-After` を付けたいなら、ALB だけでは足りません。もう 1 つは、CloudFront の**カスタムエラーページ**は、オリジンが返したエラーを差し替える機能だということです。自分からメンテナンスに切り替える仕組みではありません。また、オリジンが 400 以上を返したときは、viewer response の CloudFront Functions が動きません。

## メンテナンス中は、503 と Retry-After を返す

メンテナンス中に返すのは `503 Service Unavailable` です。RFC 9110 で、一時的な過負荷や予定されたメンテナンスで処理できない状態として定義されています。`Retry-After` ヘッダーで、再開の目安を秒数か日時で伝えます。

```http
HTTP/1.1 503 Service Unavailable
Retry-After: 3600
Content-Type: text/html; charset=utf-8
Cache-Control: no-store
```

使わないほうがいいものもあります。

| 返し方 | 何が起きるか |
| --- | --- |
| `200` でメンテページ | クローラーには「このページの中身はメンテのお知らせ」と伝わる。監視も正常と判定する |
| `302` でメンテページへ | リダイレクト先が正規のページとして扱われることがある。戻すときに不要なリダイレクトが残りがち |
| `404` | クローラーに「このページは消えた」と伝わる |

Google は、緊急でサイトを 1〜2 日止めるなら 503 の情報ページを返すよう案内しています。長く続けるとインデックスから外れ始めるので、503 で止めるのは短い期間に限ります。

メンテナンスページそのものにも気を付けることがあります。

- `robots.txt` には 503 を返さない。Google によると、すべてのクロールが止まります
- 静的な HTML にする。止めているアプリや DB、外部の JS・フォントに依存させない
- API には HTML ではなく JSON を返す。アプリが「メンテ中」の画面を出せるように、理由と再開予定を入れておく

## メンテナンスは、止めるものより手前の層で切り替える

メンテナンスの切り替えは、止める対象より手前の層に置きます。アプリのメンテナンスをアプリ自身の設定で出していると、アプリが起動しない状態では何も返せません。

AWS で選ぶなら、私は次の順に考えます。

| 置き場所 | 向いているケース | 気になる点 |
| --- | --- | --- |
| WAF のカスタムレスポンス | CloudFront か ALB に WAF が付いている。まずはこれ | 切り替えはルールの追加・変更になる |
| CloudFront Functions | WAF がない。条件をコードで細かく書きたい | 関数の発行と関連付けが必要。判断ロジックがコードに埋もれる |
| ALB のリスナールール | CloudFront も WAF もない | `Retry-After` を付けられない |
| アプリの機能フラグ | 一部の機能だけ止める、読み取り専用にする | アプリが動いていることが前提 |
| DNS の切り替え | 基本は使わない | TTL が残るので、切り替えも戻しもすぐには効かない |

### 管理者は、Allow で通すのではなく Block の条件から外す

メンテナンス中でも、管理者は本番で動作確認をしたいはずです。WAF なら、メンテナンス用の Block ルールに「管理者の IP セットでない」という条件を足します。

![メンテ対象のホスト・パスで、管理者の IP セットでなければ 503 と Retry-After を返す。管理者と対象外のリクエストは通常のルールへ進む](/images/http-status-code-where-to-return/maintenance-admin-bypass.png)

ここで「管理者の IP なら Allow」というルールを先頭に置くと、WAF はその時点で評価を終えます。後ろにあるマネージドルールも素通りになり、管理者のリクエストだけ検査されない状態になります。Block の条件から外すだけにしておけば、管理者のリクエストも通常どおり検査されます。

Terraform で書くと、たとえば次のようになります。`maintenance` 変数で、ルールそのものを出し入れしています。

```hcl
variable "maintenance" {
  type    = bool
  default = false
}

resource "aws_wafv2_web_acl" "main" {
  name  = "main"
  scope = "CLOUDFRONT"

  default_action {
    allow {}
  }

  custom_response_body {
    key          = "maintenance"
    content      = file("${path.module}/maintenance.html")
    content_type = "TEXT_HTML"
  }

  dynamic "rule" {
    for_each = var.maintenance ? [1] : []
    content {
      name     = "maintenance"
      priority = 0

      action {
        block {
          custom_response {
            response_code            = 503
            custom_response_body_key = "maintenance"
            response_header {
              name  = "Retry-After"
              value = "3600"
            }
          }
        }
      }

      # 管理者の IP セット「以外」を Block する
      statement {
        not_statement {
          statement {
            ip_set_reference_statement {
              arn = aws_wafv2_ip_set.admin.arn
            }
          }
        }
      }

      visibility_config {
        cloudwatch_metrics_enabled = true
        metric_name                = "maintenance"
        sampled_requests_enabled   = true
      }
    }
  }

  # マネージドルールなど、通常のルールが続く

  visibility_config {
    cloudwatch_metrics_enabled = true
    metric_name                = "main"
    sampled_requests_enabled   = true
  }
}
```

ホストやパスで範囲を絞るときは、`not_statement` と並べて `and_statement` でホスト（`Host` ヘッダー）やパスの条件を足します。

WAF がない構成で ALB を使う場合は、リスナールールを 2 本にします。優先度の高いほうに「送信元 IP が管理者なら forward」、次に「それ以外は fixed-response で 503」を置きます。ただし、前の節のとおり `Retry-After` は付けられません。

## ユーザー向けと管理者向けでエンドポイントを分けておくと、止め方を選べる

あなたのシステムでは、利用者向けと管理者向けの入口はどう分かれていますか？ ホスト名（`www.example.com` と `admin.example.com`）やパス（`/admin`）で分かれていれば、メンテナンスの範囲を条件 1 つで選べます。同じ URL で権限だけが違う作りだと、手前の層からは区別できず、アプリで切り替えるしかなくなります。

実運用でよくあるのは、次のようなケースです。

| やりたいこと | 切り替える層 | 条件 | 返すもの |
| --- | --- | --- | --- |
| 全体を止めるが、管理者は動作確認したい | WAF | すべてのホストかつ管理者の IP でない | 503 と `Retry-After`、メンテページ |
| 利用者向けだけ止め、管理画面は動かす | WAF | `Host` が `www` または `api` | 503 |
| 管理画面だけ止める（データ移行中など） | WAF か ALB | `Host` が `admin` | 503 |
| モバイルアプリの API だけ止める | WAF | `Host` が `api`、またはパスが `/api/` | 503 と JSON ボディ |
| 書き込みだけ止める（読み取り専用モード） | WAF かアプリ | メソッドが `POST`・`PUT`・`DELETE` | 書き込みは 503、読み取りは通常どおり |
| 決済など一部の機能だけ止める | アプリ | 機能フラグ | その API だけ 503 |

管理画面を利用者向けと同じ CloudFront や ALB に載せているなら、この機会に入口を分けるかどうかも考えどころです。分かれていれば、管理画面にだけ IP 制限や別の認証をかけるのも簡単になります。

## 管理画面へのアクセスは、401・403・404 を使い分ける

管理者以外がアクセスしてきたときに返すステータスは、状況で分けます。

| ステータス | 使う場面 |
| --- | --- |
| `401 Unauthorized` | まだ認証していない。`WWW-Authenticate` で認証を求める。Basic 認証で守るときはこれ |
| `403 Forbidden` | 認証は済んでいるが権限がない。IP 制限ではじく場合も普通はこれ。WAF の既定の Block も 403 |
| `404 Not Found` | 管理画面があること自体を隠したい。`/admin` を探索してくるボットに存在を教えない |

隠したいなら WAF のカスタムレスポンスで 404 を返せます。あわせて、管理画面には `X-Robots-Tag: noindex` を付けて、万一クロールされてもインデックスされないようにしておきます。

## リダイレクトと削除は、恒久か一時か、メソッドを保つかで選ぶ

| ステータス | 意味 | メソッド | 使う場面 |
| --- | --- | --- | --- |
| `301` | 恒久的に移動 | POST が GET に変わることがある | ドメイン移転、URL 変更。検索の評価が新 URL に移る |
| `308` | 恒久的に移動 | 保つ | API のエンドポイント移転など、POST を保ちたいとき |
| `302` | 一時的に移動 | POST が GET に変わることがある | ログイン後の戻り先など |
| `307` | 一時的に移動 | 保つ | 一時的な振り分けで POST を保ちたいとき |
| `410` | 恒久的に削除 | ― | 意図して消したページ。404 より明確に「もう戻らない」と伝わる |

HTTPS への移行は 301 か 308 に加えて `Strict-Transport-Security`（HSTS）を付けます。2 回目以降はブラウザが最初から HTTPS で接続します。

もう 1 つ、SPA でよく起きるのが**ソフト 404** です。存在しない URL でも 200 で「ページが見つかりません」を表示してしまう状態です。CloudFront のカスタムエラーページで 403 や 404 を `index.html` と 200 に差し替えていると起きやすく、クローラーには存在しないページが大量にあるように見えます。

## API は、4xx を細かく分けるとクライアントが楽になる

| ステータス | 使う場面 |
| --- | --- |
| `400 Bad Request` | JSON が壊れているなど、リクエストの形がおかしい |
| `422 Unprocessable Content` | 形は正しいが、バリデーションに通らない |
| `409 Conflict` | 楽観ロックの衝突、重複登録 |
| `429 Too Many Requests` | レート制限。`Retry-After` を付ける |
| `201 Created` | 作成した。`Location` で作ったリソースの URL を返す |
| `204 No Content` | 成功したが返すボディがない |

WAF のレートベースルールは、既定のままだと 403 を返します。API なら、カスタムレスポンスで 429 と `Retry-After` に変えておくと、クライアントが「拒否された」のか「待てばいい」のかを区別できます。

エラーのボディは、RFC 9457（Problem Details、`application/problem+json`）の形にそろえておくと、クライアントが共通の処理で扱えます。

## 5xx は、どの層が返したかで意味が変わる

同じ 503 でも、自分でメンテとして返したのか、ALB が返したのかで意味が違います。ALB が自分で返す 5xx は、おおむね次の意味です。

| ステータス | ALB が返す主な理由 |
| --- | --- |
| `502 Bad Gateway` | ターゲットとの接続や TLS に失敗した。ターゲットが不正な応答を返した、接続を途中で切った |
| `503 Service Unavailable` | ターゲットグループに登録済みのターゲットがない |
| `504 Gateway Timeout` | ターゲットが時間内に応答しなかった |

ALB のアクセスログでは、`elb_status_code`（ALB が返したもの）と `target_status_code`（ターゲットが返したもの）が分かれています。障害の切り分けでは、まずどちらが返したかを見ます。

運用で効くのは次の 2 つです。

- **ヘルスチェック用のエンドポイントを分ける**：`/healthz` のような軽いものを用意し、DB など依存先まで見るかは意図して決める。依存先まで見ていると、DB が止まった瞬間に全ターゲットが unhealthy になり、全体が 503 になる
- **監視は 5xx と 4xx を分ける**：5xx はサーバー側の問題なのでアラートの対象にする。4xx は急増したときだけ見る。攻撃や、クライアントのリリース不具合の兆候になる

## WebSocket は、つながったあとのことを別に考える

WebSocket は、最初のハンドシェイクだけが HTTP です。成功すると `101 Switching Protocols` が返り、そこから先は HTTP のステータスコードがありません。

### ハンドシェイクの失敗理由は、ブラウザの JS から読めない

ハンドシェイクの段階では、ここまでの話がそのまま使えます。メンテ中は 503、未認証は 401、権限なしは 403、レート制限は 429 です。

ただし、ブラウザの `WebSocket` API では、ハンドシェイクで返ったステータスコードを JS から読めません。どれで失敗しても `onerror` が呼ばれ、close code は 1006 になるだけです。理由をクライアントに伝えたいなら、いったん接続を受けてから close code を付けて閉じるか、接続の前に HTTP の API で状態を確かめさせます。

### 接続後は close code で理由を伝える

| close code | 意味 | 使う場面 |
| --- | --- | --- |
| `1000` | Normal Closure | 正常に終わった |
| `1001` | Going Away | サーバーの停止、ページ遷移 |
| `1008` | Policy Violation | 権限違反など |
| `1011` | Internal Error | サーバー内部のエラー |
| `1012` | Service Restart | 再起動するので、つなぎ直してほしい |
| `1013` | Try Again Later | 過負荷やメンテ中で、あとでつなぎ直してほしい |
| `4000`〜`4999` | アプリ独自 | トークン期限切れ（例：`4401`）など |

1012 と 1013 は RFC 6455 本体ではなく、IANA のレジストリに登録されたコードです。4000 番台は HTTP のコードに寄せた番号（`4401`、`4403` など）にしておくと、意味が伝わりやすくなります。

### メンテナンスに入っても、接続済みのコネクションは切れない

WAF や ALB でメンテナンスに切り替えても、それは新しいリクエストにしか効きません。すでにつながっている WebSocket はそのまま残ります。サーバーから閉じて、クライアントにつなぎ直させる必要があります。

![WebSocket はサーバーから close 1012 で閉じ、クライアントはジッターを足して待ってからつなぎ直す。503 や接続失敗なら待ち時間を倍にして再試行する](/images/http-status-code-where-to-return/websocket-reconnect.png)

- サーバーは、SIGTERM を受けたりメンテに入ったりしたら、全接続に 1001 か 1012 を送ってから終了する。ALB のターゲット登録解除の遅延（deregistration delay）が終わると残った接続は切られるので、アプリ側で先に閉じておく
- クライアントは、**指数バックオフにジッターを足して**つなぎ直す。全員が同じ間隔でつなぎ直すと、デプロイ直後の新しいタスクに接続が殺到して落ちる

### 認証は、ヘッダーを付けられない前提で考える

ブラウザの `WebSocket` API では、`Authorization` などのカスタムヘッダーを付けられません。主な選択肢は次の 3 つです。

| 方法 | 気を付けること |
| --- | --- |
| クエリパラメータにトークンを載せる | URL がアクセスログに残る。短命なワンタイムチケットにする |
| Cookie | 別のサイトから接続される（Cross-Site WebSocket Hijacking）。サーバーで `Origin` ヘッダーを必ず確かめる |
| 接続後の最初のメッセージで認証する | 数秒以内に認証が来なければ切る |

接続中にトークンの期限が切れたら、`4401` のような独自コードで閉じて、クライアントにトークンを取り直してからつなぎ直させます。

### AWS のタイムアウトに合わせて、ハートビートを送る

- ALB のアイドルタイムアウトは既定で 60 秒。ping/pong のハートビートを送るか、タイムアウトを延ばす
- API Gateway の WebSocket API は、接続の最長が 2 時間、アイドルタイムアウトが 10 分で、どちらも引き上げられない。長時間つなぐ用途は、つなぎ直す前提で作る

## メンテ明けにも、詰まりやすいところがある

- **CloudFront がエラーをキャッシュしている**：CloudFront はオリジンのエラーレスポンスを既定で 10 秒キャッシュします。オリジン側で 503 を返していた場合、戻したあともしばらく 503 が見えることがあります。メンテで長めの `Cache-Control` を付けていると、その分だけ長く残ります
- **メンテのフラグを戻し忘れる**：WAF のルールを外し忘れると、503 のままです。戻したあとに管理者以外の環境から確認する手順を、メンテの手順書に入れておきます
- **監視が鳴りっぱなしになる**：メンテ中の 503 でアラートが鳴らないよう、メンテの時間帯はアラームを止めるか、メンテ由来の 503（WAF のルール名で区別できる）を除外します
- **つなぎ直しが殺到する**：WebSocket だけでなく、モバイルアプリの API リトライも同じです。メンテ明けの数分は、スケールを多めにしておきます

## 試した範囲と、まだ試していない範囲

この記事は、RFC と AWS の公式ドキュメントを読んで整理したもので、手元のアカウントではまだ試していません。次のことは、個人のアカウントで確かめてから追記する予定です。

- WAF のカスタムレスポンスで、503 と `Retry-After` が CloudFront 経由でそのまま返るか
- 記事中の Terraform で、`maintenance` を切り替えたときに想定どおり 503 になり、管理者の IP からは通常どおり通るか
- WAF が返した 503 が、CloudFront の Error Caching Minimum TTL の影響を受けるか

【ハジメが書く：試した結果と感想（検証したら）】

## HTTP/2・gRPC、ブルーグリーンデプロイはこの記事では扱わない

- gRPC のステータスコード（HTTP のステータスとは別体系）
- 止めずにリリースするための、ブルーグリーンやカナリアのデプロイ
- 機能フラグの実装方法

## おわりに

【ハジメが書く：本音・次にやりたいこと】

## 関連する記事

WAF をどこに置くか、オリジンをどう閉じるかは、前に書いた記事で扱っています。

- [AWS WAF、どこに置いてどう組んでいますか？ エッジとリージョンの使い分けと注意点](https://zenn.dev/anfangd/articles/aws-waf-placement)
- [AWSでWebリクエストを受ける9パターンと「オリジンを裸にしない」設定プラクティス](https://zenn.dev/anfangd/articles/aws-web-ingress-origin-protection)

## 参考

- [RFC 9110 - HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [RFC 6585 - Additional HTTP Status Codes（429）](https://www.rfc-editor.org/rfc/rfc6585)
- [RFC 9457 - Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc9457)
- [RFC 6455 - The WebSocket Protocol](https://www.rfc-editor.org/rfc/rfc6455)
- [WebSocket Close Code Number Registry - IANA](https://www.iana.org/assignments/websocket/websocket.xhtml)
- [CloseEvent: code property - MDN](https://developer.mozilla.org/docs/Web/API/CloseEvent/code)
- [ウェブサイトを一時停止または無効にする - Google 検索セントラル](https://developers.google.com/search/docs/crawling-indexing/pause-online-business?hl=ja)
- [Sending custom responses for Block actions - AWS WAF](https://docs.aws.amazon.com/waf/latest/developerguide/customizing-the-response-for-blocked-requests.html)
- [Action types for listener rules - Elastic Load Balancing](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/rule-action-types.html)
- [AWS::ElasticLoadBalancingV2::ListenerRule FixedResponseConfig - AWS CloudFormation](https://docs.aws.amazon.com/AWSCloudFormation/latest/UserGuide/aws-properties-elasticloadbalancingv2-listenerrule-fixedresponseconfig.html)
- [How do I send custom HTTP responses from my Application Load Balancer? - AWS re:Post](https://repost.aws/knowledge-center/elb-send-custom-http-responses-from-alb)
- [CloudFront Functions event structure - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/functions-event-structure.html)
- [Control how long CloudFront caches errors - Amazon CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/custom-error-pages-expiration.html)
- [How do I troubleshoot 503 errors from my Application Load Balancer? - AWS re:Post](https://repost.aws/knowledge-center/alb-troubleshoot-503-errors)
- [Quotas for configuring and running a WebSocket in API Gateway](https://docs.aws.amazon.com/apigateway/latest/developerguide/apigateway-execution-service-websocket-limits-table.html)

<!-- 公開前に削除する
## 公開前の確認事項

- 手を動かした検証がまだない。上の「まだ試していない範囲」の 3 つを個人アカウントで確かめると記事の芯になる
- WAF（CloudFront に関連付け）が CloudFront Functions より先に評価される、という図の順番を公式ドキュメントで確かめる
- Terraform の例は terraform validate していない。aws プロバイダーのバージョンを決めて validate する
- 図は Python で .drawio.svg を生成したもの。draw.io で開いて見た目を整えてから PNG を書き出し直してもよい
- 社内の事例は書いていない（ケースはすべて一般化した「あるある」）
-->
