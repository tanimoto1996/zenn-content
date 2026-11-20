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

```html-embed
<div class="core3">
<div class="c3card"><div class="c3num">1</div><div class="c3ttl">ログは数えるか流すか</div><div class="c3sub">出口で、その先の配管が決まる</div></div>
<div class="c3card"><div class="c3num">2</div><div class="c3ttl">検知源は誰が告げるか</div><div class="c3sub">AWS側の異変か、脅威か</div></div>
<div class="c3card"><div class="c3num">3</div><div class="c3ttl">記録は後から掘る</div><div class="c3sub">オブジェクト単位まで残す</div></div>
</div>
<style>
.core3 { --node:#eef6ff; --tx:#1a2733; --sub:#5a6b7a; --acc:#3ea8ff; display:flex; gap:10px; padding:12px 4px; }
[data-theme="dark"] .core3 { --node:#16324a; --tx:#e8f2fa; --sub:#9db4c6; }
.core3 .c3card { flex:1; background:var(--node); border:1.5px solid var(--acc); border-radius:10px; padding:12px 10px; text-align:center; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.core3 .c3num { width:26px; height:26px; margin:0 auto 8px; border-radius:50%; background:var(--acc); color:#fff; font-weight:700; font-size:14px; line-height:26px; animation:c3p 3s ease-in-out infinite; }
.core3 .c3card:nth-child(2) .c3num { animation-delay:1s; }
.core3 .c3card:nth-child(3) .c3num { animation-delay:2s; }
@keyframes c3p { 0%, 100% { transform:scale(1); } 12% { transform:scale(1.18); } 24% { transform:scale(1); } }
.core3 .c3ttl { color:var(--tx); font-size:13px; font-weight:700; line-height:1.5; }
.core3 .c3sub { color:var(--sub); font-size:11px; margin-top:5px; line-height:1.5; }
@media (prefers-reduced-motion: reduce) { .core3 * { animation:none !important; } }
</style>
```

*この地方の核は3つ。まず地図を頭に入れる*

この記事の最後にある反射ドリル8行は、そのまま試験直前の見返しに使えます。

:::message
この記事に出てくる問題パターンは、すべて私が学習用に自分の言葉で再構成したものです。
本番試験の問題文や選択肢そのものは掲載していません。
:::

## まず切り分け：4つの動詞で分ける

「何をしたいか」の動詞で分けます。

```html-embed
<div class="verbs">
<div class="vrow"><span class="vverb b">通知する</span><span class="varr">→</span><span class="vans b">メトリクスフィルター</span></div>
<div class="vrow"><span class="vverb g">別へ送る</span><span class="varr">→</span><span class="vans g">サブスクリプションフィルター</span></div>
<div class="vrow"><span class="vverb s">AWS側の異変</span><span class="varr">→</span><span class="vans s">AWS Health</span></div>
<div class="vrow"><span class="vverb o">脅威を見つける</span><span class="varr">→</span><span class="vans o">GuardDuty</span></div>
</div>
<style>
.verbs { --tx:#1a2733; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .verbs { --tx:#e8f2fa; }
.verbs .vrow { display:flex; align-items:center; gap:10px; margin:8px 0; }
.verbs .vverb { flex:0 0 118px; text-align:center; font-size:12px; font-weight:700; padding:8px 4px; border-radius:8px; color:#fff; }
.verbs .vans { flex:1; font-size:13px; font-weight:700; padding:8px 12px; border-radius:8px; border:1.5px solid; color:var(--tx); }
.verbs .varr { font-size:15px; color:var(--tx); }
.verbs .b { background:#3ea8ff; border-color:#3ea8ff; }
.verbs .g { background:#2fb457; border-color:#2fb457; }
.verbs .s { background:#7a6ff0; border-color:#7a6ff0; }
.verbs .o { background:#e8862e; border-color:#e8862e; }
.verbs span.vverb.b, .verbs span.vverb.g, .verbs span.vverb.s, .verbs span.vverb.o { color:#fff; }
.verbs .vans.b { background:rgba(62,168,255,.12); }
.verbs .vans.g { background:rgba(47,180,87,.12); }
.verbs .vans.s { background:rgba(122,111,240,.12); }
.verbs .vans.o { background:rgba(232,134,46,.12); }
</style>
```

*動詞で切る。やりたいことが、そのまま答えになる*

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
<line class="beam" x1="112" y1="166" x2="148" y2="166"/>
<line class="beam" x1="148" y1="166" x2="148" y2="70"/>
<line class="beam" marker-end="url(#ah)" x1="148" y1="70" x2="176" y2="70"/>
<line class="beam g" x1="148" y1="166" x2="148" y2="250"/>
<line class="beam g" marker-end="url(#ah)" x1="148" y1="250" x2="164" y2="250"/>
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

```html-embed
<div class="relay">
<svg viewBox="0 0 700 170" role="img" aria-label="メトリクスフィルターの四段リレー">
<defs>
<marker id="rah" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="rahp"/>
</marker>
</defs>
<text class="rlbl" x="520" y="30" text-anchor="middle">閾値：5分で100件</text>
<line class="rbeam" marker-end="url(#rah)" x1="168" y1="95" x2="196" y2="95"/>
<line class="rbeam" marker-end="url(#rah)" x1="336" y1="95" x2="364" y2="95"/>
<line class="rbeam" marker-end="url(#rah)" x1="504" y1="95" x2="532" y2="95"/>
<circle class="rpkt rp1" cx="196" cy="95" r="4"/>
<circle class="rpkt rp2" cx="364" cy="95" r="4"/>
<circle class="rpkt rp3" cx="532" cy="95" r="4"/>
<rect class="rnode rseq" style="animation-delay:0s" x="8" y="70" width="160" height="50" rx="8"/>
<text class="rt" x="88" y="92" text-anchor="middle">1. メトリクスフィルター</text>
<text class="rs" x="88" y="110" text-anchor="middle">ログを数える</text>
<rect class="rnode rseq" style="animation-delay:1.1s" x="196" y="70" width="140" height="50" rx="8"/>
<text class="rt" x="266" y="92" text-anchor="middle">2. メトリクス</text>
<text class="rs" x="266" y="110" text-anchor="middle">数になった状態</text>
<rect class="rnode rseq" style="animation-delay:2.2s" x="364" y="70" width="140" height="50" rx="8"/>
<text class="rt" x="434" y="92" text-anchor="middle">3. アラーム</text>
<text class="rs" x="434" y="110" text-anchor="middle">閾値を超えたら鳴る</text>
<rect class="rnode rseq" style="animation-delay:3.3s" x="532" y="70" width="160" height="50" rx="8"/>
<text class="rt" x="612" y="92" text-anchor="middle">4. SNS</text>
<text class="rs" x="612" y="110" text-anchor="middle">メールで知らせる</text>
</svg>
</div>
<style>
.relay { --line:#3ea8ff; --node:#eef6ff; --tx:#1a2733; --sub:#5a6b7a; padding:12px 4px; }
[data-theme="dark"] .relay { --node:#16324a; --tx:#e8f2fa; --sub:#9db4c6; }
.relay svg { width:100%; height:auto; display:block; }
.relay .rbeam { stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:rflow 1.2s linear infinite; }
.relay .rahp { fill:var(--line); }
@keyframes rflow { to { stroke-dashoffset:-14; } }
.relay .rnode { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.relay text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.relay .rt { font-size:13px; font-weight:700; }
.relay .rs { font-size:11px; fill:var(--sub); }
.relay .rlbl { font-size:12px; font-weight:700; fill:#e8862e; }
.relay .rseq { animation:rpulse 4.4s ease-in-out infinite; }
@keyframes rpulse { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
.relay .rpkt { fill:var(--line); opacity:0; }
.relay .rp1 { animation:rpA 1.5s ease-in .9s infinite; }
.relay .rp2 { animation:rpA 1.5s ease-in 2s infinite; }
.relay .rp3 { animation:rpA 1.5s ease-in 3.1s infinite; }
@keyframes rpA { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(24px); opacity:0; } }
@media (prefers-reduced-motion: reduce) { .relay * { animation:none !important; } .relay .rpkt { display:none; } }
</style>
```

*四段リレー。1段も省けない*

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

```html-embed
<div class="srcs">
<svg viewBox="0 0 700 250" role="img" aria-label="検知源は2つ、受け手はEventBridge">
<defs>
<marker id="sah" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="sahp"/>
</marker>
</defs>
<line class="sbeam" x1="218" y1="70" x2="262" y2="70"/>
<line class="sbeam" x1="262" y1="70" x2="262" y2="118"/>
<line class="sbeam" marker-end="url(#sah)" x1="262" y1="118" x2="292" y2="118"/>
<line class="sbeam o" x1="218" y1="180" x2="262" y2="180"/>
<line class="sbeam o" x1="262" y1="180" x2="262" y2="132"/>
<line class="sbeam" marker-end="url(#sah)" x1="448" y1="125" x2="492" y2="125"/>
<circle class="spkt sp1" cx="292" cy="118" r="4"/>
<circle class="spkt sp2" cx="448" cy="125" r="4"/>
<rect class="snode s" x="8" y="45" width="210" height="50" rx="8"/>
<text class="st" x="113" y="67" text-anchor="middle">AWS Health</text>
<text class="ss" x="113" y="85" text-anchor="middle">AWS側の異変を告げる</text>
<rect class="snode o" x="8" y="155" width="210" height="50" rx="8"/>
<text class="st" x="113" y="177" text-anchor="middle">GuardDuty</text>
<text class="ss" x="113" y="195" text-anchor="middle">脅威を見つけて告げる</text>
<rect class="snode" x="292" y="100" width="156" height="50" rx="8"/>
<text class="st" x="370" y="122" text-anchor="middle">EventBridge</text>
<text class="ss" x="370" y="140" text-anchor="middle">受けて、次へ渡す</text>
<rect class="snode" x="492" y="100" width="200" height="50" rx="8"/>
<text class="st" x="592" y="122" text-anchor="middle">対応・通知</text>
<text class="ss" x="592" y="140" text-anchor="middle">Step Functions / SNS など</text>
</svg>
</div>
<style>
.srcs { --line:#3ea8ff; --org:#e8862e; --pur:#7a6ff0; --node:#eef6ff; --tx:#1a2733; --sub:#5a6b7a; padding:12px 4px; }
[data-theme="dark"] .srcs { --node:#16324a; --tx:#e8f2fa; --sub:#9db4c6; }
.srcs svg { width:100%; height:auto; display:block; }
.srcs .sbeam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:sflow 1.2s linear infinite; }
.srcs .sbeam.o { stroke:var(--org); }
.srcs .sahp { fill:var(--line); }
@keyframes sflow { to { stroke-dashoffset:-14; } }
.srcs .snode { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.srcs .snode.s { stroke:var(--pur); }
.srcs .snode.o { stroke:var(--org); }
.srcs text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.srcs .st { font-size:13px; font-weight:700; }
.srcs .ss { font-size:11px; fill:var(--sub); }
.srcs .spkt { fill:var(--line); opacity:0; }
.srcs .sp1 { animation:spA 1.6s ease-in infinite; }
.srcs .sp2 { animation:spB 1.6s ease-in .6s infinite; }
@keyframes spA { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(120px); opacity:0; } }
@keyframes spB { 0% { transform:translateX(0); opacity:0; } 15% { opacity:1; } 85% { opacity:1; } 100% { transform:translateX(40px); opacity:0; } }
@media (prefers-reduced-motion: reduce) { .srcs * { animation:none !important; } .srcs .spkt { display:none; } }
</style>
```

*検知源は2つ。受け手はEventBridgeで共通*

AWS Healthは、AWS側の異変を告げます。

GuardDutyは、脅威を見つけます。

漏れた認証情報の知らせは、Healthから来ます。

イベント名はAWS_RISK_CREDENTIALS_EXPOSEDです。

EventBridgeで受けて、Step Functionsへ渡します。

Step Functionsが、後始末の手順を回します。

無効化、レポート作成、通知。

実行の履歴が、すべて残ります。

```html-embed
<div class="clean">
<div class="crow">
<span class="cstep warn">漏洩の知らせ</span><span class="carr">→</span>
<span class="cstep">EventBridge</span><span class="carr">→</span>
<span class="cstep main">Step Functions</span><span class="carr">→</span>
<span class="cacts"><span class="cact">無効化</span><span class="cact">レポート作成</span><span class="cact">通知</span></span>
</div>
<div class="cbadge">監査：実行の履歴が、すべて残る</div>
</div>
<style>
.clean { --node:#eef6ff; --tx:#1a2733; --acc:#3ea8ff; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .clean { --node:#16324a; --tx:#e8f2fa; }
.clean .crow { display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
.clean .cstep { background:var(--node); border:1.5px solid var(--acc); color:var(--tx); border-radius:8px; padding:9px 12px; font-size:12px; font-weight:700; }
.clean .cstep.warn { border-color:#e8862e; }
.clean .cstep.main { background:var(--acc); color:#fff; animation:cpulse 3s ease-in-out infinite; }
@keyframes cpulse { 0%, 100% { transform:scale(1); } 10% { transform:scale(1.06); } 22% { transform:scale(1); } }
.clean .carr { color:var(--tx); font-size:14px; }
.clean .cacts { display:flex; gap:6px; }
.clean .cact { border:1.5px dashed var(--acc); color:var(--tx); border-radius:8px; padding:9px 10px; font-size:12px; }
.clean .cbadge { margin-top:10px; display:inline-block; background:rgba(47,180,87,.14); border:1.5px solid #2fb457; color:var(--tx); border-radius:8px; padding:7px 12px; font-size:12px; font-weight:700; }
@media (prefers-reduced-motion: reduce) { .clean * { animation:none !important; } }
</style>
```

*後始末はStep Functions。「監査可能」が合図*

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

```html-embed
<div class="gran">
<svg viewBox="0 0 700 270" role="img" aria-label="管理イベントとデータイベントの粒度">
<text class="gt b" x="170" y="30" text-anchor="middle">管理イベント：リソース単位</text>
<rect class="gbox" x="30" y="48" width="280" height="180" rx="10"/>
<rect class="gblk" x="52" y="70" width="236" height="38" rx="6"/>
<text class="gs" x="170" y="94" text-anchor="middle">バケットの作成</text>
<rect class="gblk" x="52" y="122" width="236" height="38" rx="6"/>
<text class="gs" x="170" y="146" text-anchor="middle">ポリシーの変更</text>
<rect class="gblk" x="52" y="174" width="236" height="38" rx="6"/>
<text class="gs" x="170" y="198" text-anchor="middle">リソースの削除</text>
<text class="gt o" x="530" y="30" text-anchor="middle">データイベント：オブジェクト単位</text>
<rect class="gbox o" x="390" y="48" width="280" height="180" rx="10"/>
<g class="gcells">
<rect x="412" y="70" width="34" height="26" rx="4"/><rect x="458" y="70" width="34" height="26" rx="4"/><rect x="504" y="70" width="34" height="26" rx="4"/><rect x="550" y="70" width="34" height="26" rx="4"/><rect x="596" y="70" width="34" height="26" rx="4"/>
<rect x="412" y="108" width="34" height="26" rx="4"/><rect x="458" y="108" width="34" height="26" rx="4"/><rect class="ghot" x="504" y="108" width="34" height="26" rx="4"/><rect x="550" y="108" width="34" height="26" rx="4"/><rect x="596" y="108" width="34" height="26" rx="4"/>
<rect x="412" y="146" width="34" height="26" rx="4"/><rect x="458" y="146" width="34" height="26" rx="4"/><rect x="504" y="146" width="34" height="26" rx="4"/><rect x="550" y="146" width="34" height="26" rx="4"/><rect x="596" y="146" width="34" height="26" rx="4"/>
<rect x="412" y="184" width="34" height="26" rx="4"/><rect x="458" y="184" width="34" height="26" rx="4"/><rect x="504" y="184" width="34" height="26" rx="4"/><rect x="550" y="184" width="34" height="26" rx="4"/><rect x="596" y="184" width="34" height="26" rx="4"/>
</g>
<text class="ghotlbl" x="521" y="125" text-anchor="middle">Put</text>
<text class="gs" x="530" y="252" text-anchor="middle">1つ1つのオブジェクト操作まで記録</text>
</svg>
</div>
<style>
.gran { --line:#3ea8ff; --org:#e8862e; --node:#eef6ff; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .gran { --node:#16324a; --tx:#e8f2fa; }
.gran svg { width:100%; height:auto; display:block; }
.gran .gbox { fill:none; stroke:var(--line); stroke-width:1.5; }
.gran .gbox.o { stroke:var(--org); }
.gran .gblk { fill:var(--node); stroke:var(--line); stroke-width:1.2; }
.gran .gcells rect { fill:var(--node); stroke:var(--org); stroke-width:1.2; }
.gran .gcells .ghot { fill:var(--org); animation:ghot 2s ease-in-out infinite; }
@keyframes ghot { 0%, 100% { opacity:1; } 50% { opacity:.45; } }
.gran text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.gran .gt { font-size:14px; font-weight:700; }
.gran .gt.b { fill:var(--line); }
.gran .gt.o { fill:var(--org); }
.gran .gs { font-size:12px; }
.gran .ghotlbl { font-size:10px; font-weight:700; fill:#fff; }
@media (prefers-reduced-motion: reduce) { .gran * { animation:none !important; } }
</style>
```

*管理はリソース単位、データはオブジェクト単位*

| イベントの種類 | 記録するもの |
| --- | --- |
| 管理イベント | リソースの作成・変更・削除 |
| データイベント | オブジェクト単位の操作 |

急増の自動検知は、CloudTrail Insightsです。

コールレートのベースラインを学習して、異常を知らせます。

```html-embed
<div class="insp">
<svg viewBox="0 0 700 250" role="img" aria-label="CloudTrail Insightsの異常検知">
<rect class="iband" x="60" y="80" width="580" height="70"/>
<line class="ithr" x1="60" y1="80" x2="640" y2="80"/>
<text class="ilbl" x="66" y="70">いつもの範囲（ベースライン）</text>
<polyline class="iline" points="60,120 130,112 200,124 270,110 340,118 410,114 470,116 520,60 570,108 640,114"/>
<circle class="idot" cx="520" cy="60" r="7"/>
<text class="ialert" x="520" y="34" text-anchor="middle">急増 → 異常として検知</text>
<line class="iaxis" x1="60" y1="200" x2="640" y2="200"/>
<text class="isub" x="60" y="224">コールレートを継続監視</text>
<text class="isub" x="640" y="224" text-anchor="end">時間 →</text>
</svg>
</div>
<style>
.insp { --line:#3ea8ff; --tx:#1a2733; --sub:#5a6b7a; padding:12px 4px; }
[data-theme="dark"] .insp { --tx:#e8f2fa; --sub:#9db4c6; }
.insp svg { width:100%; height:auto; display:block; }
.insp .iband { fill:rgba(62,168,255,.13); }
.insp .ithr { stroke:var(--line); stroke-width:1.5; stroke-dasharray:6 5; }
.insp .iline { fill:none; stroke:var(--line); stroke-width:2.5; stroke-linejoin:round; stroke-linecap:round; }
.insp .idot { fill:#e5484d; transform-box:fill-box; transform-origin:center; animation:idot 2s ease-in-out infinite; }
@keyframes idot { 0%, 100% { transform:scale(1); opacity:1; } 50% { transform:scale(1.55); opacity:.55; } }
.insp text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.insp .ilbl { font-size:12px; fill:var(--sub); }
.insp .ialert { font-size:13px; font-weight:700; fill:#e5484d; }
.insp .iaxis { stroke:var(--sub); stroke-width:1; }
.insp .isub { font-size:11px; fill:var(--sub); }
@media (prefers-reduced-motion: reduce) { .insp * { animation:none !important; } .insp .iline { stroke-dashoffset:0; } .insp .idot { opacity:1; } }
</style>
```

*Insightsは「いつもの範囲」を覚えて、はみ出しを告げる*

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
