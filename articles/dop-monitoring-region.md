---
title: "DOP ⑤監視・観測区 ― ログの2つの出口と、検知源の使い分けで見抜く監視"
emoji: "🔭"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["aws", "dop", "資格", "devops", "aws認定資格"]
published: false
---

## はじめに：ログはあるのに、誰も気づかない

エラーが出ているのに、誰も気づかない。

ログは残っているのに、通知が来ない。

そんな経験はありませんか。

ログは「ある」だけでは、監視になりません。

数えるか、流すか。

その分かれ道を間違えると、監視は動かないままです。

⑤監視・観測区の出題比率は15％。

核になる考え方は3つです。

「ログは数えるか流すか」「検知源は誰が告げるか」「記録は後から掘る」。

この3つを押さえれば、この地方は読みやすくなります。

この記事の最後にある反射ドリル8行は、そのまま試験直前の見返しに使えます。

:::message
この記事に出てくる問題パターンは、すべて私が学習用に自分の言葉で再構成したものです。
本番試験の問題文や選択肢そのものは掲載していません。
:::

## まず切り分け：4つの動詞で分ける

「何をしたいか」の動詞で分けます。

| やりたいこと | 選ぶもの |
| --- | --- |
| ログから通知する | メトリクスフィルター |
| ログを別へ送る | サブスクリプションフィルター |
| AWS側の異変を知る | AWS Health |
| 脅威を見つける | GuardDuty |

ここで一番大事なのは、1行目と2行目です。

ログの出口を間違えると、その後の配管が全部ズレます。

この2つが、この地方のすべての出発点です。

---

## ボス1：ログの2つの出口 ― 数えるか、流すか

**引き金**：「ログにXが出たら通知」「ログを別システムへ配信」。

CloudWatch Logsには、2つの出口があります。

1つはメトリクスフィルターです。

もう1つはサブスクリプションフィルターです。

名前は似ています。

でも、出口の先がまったく違います。

```html-embed
<div class="exits">
<svg viewBox="0 0 700 330" role="img" aria-label="CloudWatch Logsの2つの出口">
<defs>
<marker id="ah" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<path class="beam" marker-end="url(#ah)" d="M112 166 L148 166 L148 70 L176 70"/>
<path class="beam g" marker-end="url(#ah)" d="M112 166 L148 166 L148 250 L164 250"/>
<line class="beam" marker-end="url(#ah)" x1="320" y1="70" x2="376" y2="70"/>
<line class="beam" marker-end="url(#ah)" x1="470" y1="70" x2="516" y2="70"/>
<line class="beam" marker-end="url(#ah)" x1="600" y1="70" x2="628" y2="70"/>
<line class="beam g" marker-end="url(#ah)" x1="352" y1="250" x2="416" y2="250"/>
<circle class="pkt pa" cx="320" cy="70" r="4"/>
<circle class="pkt pb" cx="470" cy="70" r="4"/>
<circle class="pkt pd" cx="352" cy="250" r="4"/>
<rect class="node" x="8" y="140" width="104" height="52" rx="8"/>
<text class="t" x="60" y="162" text-anchor="middle">CloudWatch</text>
<text class="t" x="60" y="180" text-anchor="middle">Logs</text>
<text class="lbl-b" x="250" y="32" text-anchor="middle">数える → 通知</text>
<rect class="node seq" style="animation-delay:0s" x="180" y="48" width="140" height="44" rx="8"/>
<text class="t" x="250" y="75" text-anchor="middle">メトリクスフィルター</text>
<rect class="node seq" style="animation-delay:1.2s" x="380" y="48" width="90" height="44" rx="8"/>
<text class="t" x="425" y="75" text-anchor="middle">メトリクス</text>
<rect class="node seq" style="animation-delay:2.4s" x="520" y="48" width="80" height="44" rx="8"/>
<text class="t" x="560" y="75" text-anchor="middle">アラーム</text>
<rect class="node seq" style="animation-delay:3.6s" x="632" y="48" width="60" height="44" rx="8"/>
<text class="t" x="662" y="75" text-anchor="middle">SNS</text>
<text class="lbl-g" x="260" y="214" text-anchor="middle">流す → 配信</text>
<rect class="node grn" x="168" y="228" width="184" height="44" rx="8"/>
<text class="ts" x="260" y="255" text-anchor="middle">サブスクリプションフィルター</text>
<rect class="node grn" x="420" y="228" width="250" height="44" rx="8"/>
<text class="t" x="545" y="255" text-anchor="middle">Kinesis / Firehose / Lambda</text>
</svg>
</div>
<style>
.exits { --line:#3ea8ff; --line2:#2fb457; --node:#eef6ff; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .exits { --node:#16324a; --tx:#e8f2fa; }
.exits svg { width:100%; height:auto; display:block; }
.exits .beam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:exflow 1.2s linear infinite; }
.exits .beam.g { stroke:var(--line2); }
.exits .ahp { fill:var(--line); }
@keyframes exflow { to { stroke-dashoffset:-14; } }
.exits .node { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.exits .node.grn { stroke:var(--line2); }
.exits text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.exits .t { font-size:13px; }
.exits .ts { font-size:12px; }
.exits .lbl-b { fill:var(--line); font-size:14px; font-weight:700; }
.exits .lbl-g { fill:var(--line2); font-size:14px; font-weight:700; }
.exits .seq { animation:expulse 4.8s ease-in-out infinite; }
@keyframes expulse { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
.exits .pkt { fill:var(--line); opacity:0; }
.exits .pa { animation:expA 1.6s ease-in infinite; }
.exits .pb { animation:expB 1.4s ease-in .5s infinite; }
.exits .pd { fill:var(--line2); animation:expD 1.8s ease-in .2s infinite; }
@keyframes expA { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(56px); opacity:0; } }
@keyframes expB { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(46px); opacity:0; } }
@keyframes expD { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(64px); opacity:0; } }
@media (prefers-reduced-motion: reduce) { .exits * { animation:none !important; } .exits .pkt { display:none; } }
</style>
```

*ログの2つの出口。数えるか、流すかで、その先の配管が決まる*

メトリクスフィルターは、ログを数に変えます。

「ERROR」という文字を数えて、メトリクスにします。

その先にあるのは、メトリクスです。

アラームに直結するわけではありません。

サブスクリプションフィルターは、ログそのものを流します。

マッチしたログイベントを、別のサービスへ送ります。

| フィルター | 出口の先 | 使いどころ |
| --- | --- | --- |
| メトリクス | メトリクス | 数えて通知する |
| サブスクリプション | 他サービス | ログを流して処理する |

サブスクリプションの宛先は、3つです。

Kinesis Data Streams、Firehose、Lambdaです。

SNSは、宛先にできません。

通知を送りたいなら、Lambdaを挟みます。

| ステップ | 内容 |
| --- | --- |
| **最終文** | 聞かれているのは「ログの出口の選択」です |
| **制約** | 決め手は「通知」か「配信」かという言葉 |
| **違反スキャン** | 「サブスク→SNS直結」は実在せん → 消去 |

「通知」と書いてあれば、数える方です。

「配信」「集約」と書いてあれば、流す方です。

動詞が、そのまま答えを教えてくれます。

:::message
両方とも「ログを見る」フィルターです。
名前で混同しないように、出口の先で覚えます。
数えるのがメトリクス、流すのがサブスクリプションです。
:::

:::details Logs Insightsとの違い
- Logs Insightsは、ログの検索です
- トリガーにはなりません
- 「検索して調べる」ならLogs Insights、「見つけたら動く」ならフィルターです
:::

---

## ボス2：メトリクスフィルターの四段リレー ― 5分で100件の鐘

**引き金**：「アプリログに5分間で100件以上のエラー→メール通知」。

5分で100件のエラー。

これをメールで知らせる。

順番は4段です。

| 段 | 役割 |
| --- | --- |
| 1段目 | メトリクスフィルターで数える |
| 2段目 | メトリクスになる |
| 3段目 | アラームが鳴る |
| 4段目 | SNSがメールを送る |

1段も省けません。

フィルターの出力は、必ずメトリクスです。

「フィルター→アラーム」の直結は、実在しません。

双子の選択肢は、段の数で切ります。

アラームの閾値は、5分で100件です。

通知の先は、SNSのメール購読です。

| ステップ | 内容 |
| --- | --- |
| **最終文** | 聞かれているのは「通知までの段取り」です |
| **制約** | 決め手は「5分で100件」という数字 |
| **違反スキャン** | 「一段抜けの直結」は実在せん → 消去 |

ログがすでにCloudWatch Logsにあるなら、話は同じです。

「Critical」という文字を拾って、セキュリティへ通知する。

これも四段のリレーです。

余計な配管は、要りません。

Metric StreamsやFirehoseを足すのは、用途違いです。

ログはもう届いています。

届いているものに、もう1本の管は要りません。

:::message
Beanstalkのヘルスメトリクスは、環境の健全性を表すものです。
ログの中のエラーは、数えられません。
「ログの中身を数える」なら、メトリクスフィルター一択です。
:::

---

## ボス3：Health vs GuardDuty ― 検知源は誰が告げるか

**引き金**：「認証情報が公開GitHubに漏洩」「EKSの不正アクセスを監視」。

検知源は、誰が告げるかで分けます。

AWS Healthは、AWS側の異変を告げます。

GuardDutyは、脅威を見つけます。

漏れた認証情報の知らせは、Healthから来ます。

イベント名はAWS_RISK_CREDENTIALS_EXPOSEDです。

EventBridgeで受けて、Step Functionsへ渡します。

Step Functionsが、後始末の手順を回します。

無効化、レポート作成、通知。

実行の履歴が、すべて残ります。

| 知らせの主 | 告げるもの | 受け取る側 |
| --- | --- | --- |
| AWS Health | AWS側の異変 | EventBridge |
| GuardDuty | 脅威の検知 | EventBridge |

「すべての実行が監査可能」の一文が、Step Functionsの合図です。

Lambda単発では、監査の要件に負けます。

CloudTrailは、発信元ではありません。

レポート作りに使う部品です。

| ステップ | 内容 |
| --- | --- |
| **最終文** | 聞かれているのは「検知源と後始末の型」です |
| **制約** | 決め手は「漏洩」「監査可能」という言葉 |
| **違反スキャン** | 「ソースはCloudTrail」は部品の混同 → 消去 |

次に、GuardDuty側です。

EKSの不正アクセス監視は、GuardDuty EKS Protectionです。

EKS監査ログとランタイムを自動で分析します。

視覚的な根本原因分析は、Detectiveです。

EKS監査ログをデータソースに追加して、グラフで追います。

MacieはS3の機密データ専門です。

「視覚的」「根本原因分析」と書いてあれば、Detectiveです。

:::message
検知源を間違えると、配管が全部ズレます。
「誰が告げるか」を先に決めてください。
HealthはAWS側の異変、GuardDutyは脅威の検知です。
:::

---

## ボス4：CloudTrailのデータイベント＋Insights ― オブジェクト単位の目

**引き金**：「オブジェクトレベルAPIを監視」「PutObject急増を自動検知」。

管理イベントだけでは、PutObjectは見えません。

オブジェクト単位の操作は、データイベントです。

別途、有効化が必要です。

| イベントの種類 | 記録するもの |
| --- | --- |
| 管理イベント | リソースの作成・変更・削除 |
| データイベント | オブジェクト単位の操作 |

急増の自動検知は、CloudTrail Insightsです。

コールレートのベースラインを学習して、異常を知らせます。

サーバーアクセスログ＋Lambdaの自作計算では、同じことはできません。

手作りのレート計算は、工数が最大です。

Macieは機密データの検出です。

コールの急増とは、役割が違います。

| ステップ | 内容 |
| --- | --- |
| **最終文** | 聞かれているのは「記録と検知の2点セット」です |
| **制約** | 決め手は「オブジェクトレベル」「急増」という言葉 |
| **違反スキャン** | 「管理イベントだけ」はPutObjectが載らん → 消去 |

「継続監視＋急増の自動検知」は、2点セットです。

データイベントで記録して、Insightsで異常を検知する。

この組み合わせを、1つの反射にしてください。

---

## 反射ドリル：この地方の8行

最後にこの地方の反射ドリルを置いておきます。

左の引き金を見て、右の即答が口から出るまで回すやつです。

| 引き金 | 即答 |
| --- | --- |
| ログにXが出たら通知 | メトリクスフィルター＋アラーム→SNS |
| ログを別システムへ配信 | サブスクリプションフィルター。SNSは宛先にできん |
| 5分で100件のエラー | MF→メトリクス→アラーム→SNS（四段） |
| 漏洩認証情報の知らせ | AWS_RISK_CREDENTIALS_EXPOSED |
| EKSの不正アクセス監視 | GuardDuty EKS＋Detective |
| S3のPutObject急増 | CloudTrailデータイベント＋Insights |
| ログ内の機密IDをマスク | データ保護ポリシー＋logs:Unmask |
| メトリクスを15ヶ月超保持 | Metric Streams→Firehose→S3 |

---

## おわりに：次は⑥障害対応・事件区へ

⑤監視・観測区の核は3つでした。

「ログは数えるか流すか」「検知源は誰が告げるか」「記録は後から掘る」です。

ここまで読めば、この地方のボスはもう怖くありません。

次回は⑥障害対応・事件区へ進みます。

なお、反射ドリルとボス図鑑の完全版は、Zenn Booksで準備中です。

まとまり次第このアカウントでお知らせします。
