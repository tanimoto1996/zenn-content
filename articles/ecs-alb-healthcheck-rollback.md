---
title: "ECSの起動58秒とALBヘルスチェック45秒が競合して、デプロイが毎回ロールバックした話 ― 原因はグレース期間の勘違いだった"
emoji: "🚢"
type: "tech"
topics: ["ecs", "alb", "fargate", "docker", "aws"]
published: false
---

## はじめに：デプロイするたびに、コードが戻る

デプロイしたのに、コードが変わらない。タスク定義は更新されているのに、しばらくすると古いコードが戻ってくる。そんな現象に、開発環境で遭遇しました。

アプリのコードは正しく動いています。Nginxのログを見ても、ヘルスチェックのパスには200を返している。それなのにECSは「Task failed ELB health checks」と言って、タスクを止め続ける。

原因は、コードのバグではありませんでした。コンテナの起動に58秒かかることと、ALBが45秒でUNHEALTHYと判定すること。この2つの数字が競合していたんです。

この記事では、原因の切り分けから修正までを、実際のタイムラインと設定値つきで書きます。末尾には、デプロイ前にそのまま使えるチェックリストも置きました。コピーして使ってもらえると嬉しいです。

```html-embed
<div class="learnc2">
<div class="ttl">この記事で分かること</div>
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">3層のヘルスチェックの役割分担</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">起動58秒と判定45秒がぶつかる仕組み</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">healthCheckGracePeriodSeconds の本当の意味</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">✓</span><span class="txt">コンテナの startPeriod の決め方</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">✓</span><span class="txt">デプロイ前にそのまま使えるチェックリスト</span></div>
</div>
<style>
.learnc2 { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .learnc2 { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.learnc2 .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.learnc2 .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:learnfade2 .7s ease both; }
@keyframes learnfade2 { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.learnc2 .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.learnc2 .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .learnc2 * { animation:none !important; } }
</style>
```

---

## まず整理する：ヘルスチェックは3層ある

ECSのデプロイには、名前の似たヘルスチェックが3つ絡みます。まずこの3つを分けて考えるのが、切り分けの出発点です。

| チェック | 見るもの | 設定する場所 |
| --- | --- | --- |
| ALB | HTTPの応答 | ターゲットグループ |
| コンテナ | 内部コマンド | タスク定義 |
| ECSサービス | 結果の扱い | サービス定義 |

3つは独立して動きます。ALBはECSのグレース期間を知りません。コンテナの startPeriod も、ALBのチェックには影響しません。

この独立性こそ、今回の話の核心です。1つの層だけ見ても、原因は見えません。

切り分けの手がかりは、見る場所が層ごとに違うことです。ALBの判定は、ターゲットグループの「ターゲットの正常性」で確認します。

コンテナの状態は、タスク詳細のヘルスステータスで見ます。ECSサービス側の判断は、サービスのイベントに残ります。

| 層 | 確認する場所 | 残る記録 |
| --- | --- | --- |
| ALB | ターゲットの正常性 | UNHEALTHY・理由コード |
| コンテナ | タスク詳細 | ヘルスステータス |
| ECSサービス | サービスイベント | タスクの停止・置き換え |

今回のケースでは、サービスのイベントに「Task failed ELB health checks」という停止理由が残っていました。アプリのログには異常がなく、Nginxは200を返している。

それでもECSがタスクを止める。この組み合わせが、「アプリではなく、判定のタイミングを疑う」合図でした。

```html-embed
<div class="layer3">
<div class="flow">
<div class="node app" style="animation-delay:0s"><div class="t">コンテナ起動</div><div class="s">アプリが起きるまで58秒</div></div>
<div class="arr">↓ 同時に動く</div>
<div class="node alb" style="animation-delay:.3s"><div class="t">ALBヘルスチェック</div><div class="s">15秒ごと・3回失敗で判定</div></div>
<div class="node cnt" style="animation-delay:.6s"><div class="t">コンテナヘルスチェック</div><div class="s">30秒ごと・猶予は10秒だけ</div></div>
<div class="arr">↓ 結果を受け取る</div>
<div class="node ecs" style="animation-delay:.9s"><div class="t">ECSサービス</div><div class="s">グレース60秒の間だけ結果を無視</div></div>
</div>
<p class="note">3層は独立して動く。1層だけ見ても、競合は見えない</p>
</div>
<style>
.layer3 { --acc:#3ea8ff; --alb:#e08a3c; --cnt:#2f9e5f; --ecs:#7b5fc7; --tx:#1a2733; --bd:#bcd9f5; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .layer3 { --tx:#e8f2fa; }
.layer3 .flow { display:flex; flex-direction:column; gap:6px; }
.layer3 .node { border-radius:8px; padding:10px 14px; animation:layerfade .8s ease both; }
@keyframes layerfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.layer3 .node.app { background:#eef6ff; border:1.5px solid var(--acc); border-left:4px solid var(--acc); }
.layer3 .node.alb { background:#fdf3e7; border:1.5px solid var(--alb); border-left:4px solid var(--alb); }
.layer3 .node.cnt { background:#eef7f1; border:1.5px solid var(--cnt); border-left:4px solid var(--cnt); }
.layer3 .node.ecs { background:#f2eefb; border:1.5px solid var(--ecs); border-left:4px solid var(--ecs); }
[data-theme="dark"] .layer3 .node.app { background:#16324a; }
[data-theme="dark"] .layer3 .node.alb { background:#3a2a16; }
[data-theme="dark"] .layer3 .node.cnt { background:#173325; }
[data-theme="dark"] .layer3 .node.ecs { background:#2a2140; }
.layer3 .t { color:var(--tx); font-weight:700; font-size:14px; }
.layer3 .s { color:var(--tx); font-size:12.5px; line-height:1.55; opacity:.85; }
.layer3 .arr { color:var(--tx); font-size:12px; text-align:center; opacity:.7; line-height:1.4; }
.layer3 .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:8px 0 0; text-align:center; }
@media (prefers-reduced-motion: reduce) { .layer3 * { animation:none !important; } }
</style>
```

---

## 現場のタイムライン：58秒と45秒の競合

まず、起きていたことを時系列で並べます。数字だけ見ると、競合の構図が見えてきます。

| 時刻 | 出来事 |
| --- | --- |
| T+0秒 | コンテナ起動 |
| T+15秒 | ALB 1回目 → 失敗 |
| T+30秒 | ALB 2回目 → 失敗 |
| T+45秒 | ALB 3回目 → 失敗、UNHEALTHY |
| T+55秒 | PHP-FPM 起動完了 |
| T+58秒 | Nginx 起動完了 |
| T+60秒 | グレース期間の終了 |
| T+62秒 | ALB → 200、1回目の成功 |
| T+77秒 | ALB → 失敗、HEALTHYに戻れない |
| T+77秒 | ECS がタスクの停止を開始 |
| T+131秒 | タスク停止の確定 |

起動が58秒、UNHEALTHY判定が45秒。13秒の差が、すべての始まりです。Nginxが起きる前に、ALBはもう見切りをつけていたわけです。

グレース期間が終わるT+60秒の時点で、ターゲットはすでにUNHEALTHYでした。そこからHEALTHYに戻るには、2回連続の成功が必要です。T+62秒で1回成功したものの、T+77秒で失敗して、戻れませんでした。

```html-embed
<div class="raceflow">
<svg viewBox="0 0 700 300" role="img" aria-label="起動58秒と判定45秒の競合タイムライン">
<defs>
<marker id="ahr" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<line class="track" x1="40" y1="60" x2="660" y2="60"/>
<circle class="pt seq" style="animation-delay:0s" cx="40" cy="60" r="7"/>
<text class="tl" x="40" y="34" text-anchor="middle">T+0秒</text>
<text class="td" x="40" y="84" text-anchor="middle">起動</text>
<circle class="pt seq bad" style="animation-delay:1s" cx="180" cy="60" r="7"/>
<text class="tl" x="180" y="34" text-anchor="middle">T+15秒</text>
<text class="td" x="180" y="84" text-anchor="middle">1回目 失敗</text>
<circle class="pt seq bad" style="animation-delay:2s" cx="320" cy="60" r="7"/>
<text class="tl" x="320" y="34" text-anchor="middle">T+30秒</text>
<text class="td" x="320" y="84" text-anchor="middle">2回目 失敗</text>
<circle class="pt seq bad" style="animation-delay:3s" cx="460" cy="60" r="7"/>
<text class="tl" x="460" y="34" text-anchor="middle">T+45秒</text>
<text class="td" x="460" y="84" text-anchor="middle">3回目 失敗</text>
<text class="td hot" x="460" y="100" text-anchor="middle">UNHEALTHY</text>
<circle class="pt seq" style="animation-delay:4s" cx="600" cy="60" r="7"/>
<text class="tl" x="600" y="34" text-anchor="middle">T+58秒</text>
<text class="td" x="600" y="84" text-anchor="middle">Nginx 起動</text>
<line class="track" x1="40" y1="180" x2="660" y2="180"/>
<circle class="pt seq" style="animation-delay:5s" cx="80" cy="180" r="7"/>
<text class="tl" x="80" y="154" text-anchor="middle">T+60秒</text>
<text class="td" x="80" y="204" text-anchor="middle">グレース終了</text>
<circle class="pt seq ok" style="animation-delay:6s" cx="240" cy="180" r="7"/>
<text class="tl" x="240" y="154" text-anchor="middle">T+62秒</text>
<text class="td" x="240" y="204" text-anchor="middle">200・成功1回目</text>
<circle class="pt seq bad" style="animation-delay:7s" cx="420" cy="180" r="7"/>
<text class="tl" x="420" y="154" text-anchor="middle">T+77秒</text>
<text class="td" x="420" y="204" text-anchor="middle">失敗・停止開始</text>
<circle class="pt seq bad" style="animation-delay:8s" cx="600" cy="180" r="7"/>
<text class="tl" x="600" y="154" text-anchor="middle">T+131秒</text>
<text class="td" x="600" y="204" text-anchor="middle">停止が確定</text>
<text class="cap" x="350" y="262" text-anchor="middle">Nginxが起きる前に、ALBは見切りをつけていた</text>
</svg>
</div>
<style>
.raceflow { --line:#3ea8ff; --bad:#d9534f; --ok:#2f9e5f; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .raceflow { --tx:#e8f2fa; }
.raceflow svg { width:100%; height:auto; display:block; }
.raceflow .track { stroke:var(--line); stroke-width:2; opacity:.5; }
.raceflow .pt { fill:#fff; stroke:var(--line); stroke-width:2; }
.raceflow .pt.bad { stroke:var(--bad); }
.raceflow .pt.ok { stroke:var(--ok); }
.raceflow text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.raceflow .tl { font-size:13px; font-weight:700; }
.raceflow .td { font-size:11.5px; opacity:.8; }
.raceflow .td.hot { fill:var(--bad); font-weight:700; opacity:1; }
.raceflow .cap { font-size:13px; font-weight:700; fill:var(--bad); }
.raceflow .seq { animation:raceseq 9s ease-in-out infinite; }
@keyframes raceseq { 0%, 100% { stroke-width:2; } 6% { stroke-width:5; } 14% { stroke-width:2; } }
@media (prefers-reduced-motion: reduce) { .raceflow * { animation:none !important; } }
</style>
```

```html-embed
<div class="racebar">
<div class="row"><span class="lab">起動にかかる時間</span><div class="track"><div class="bar hot" style="width:100%;animation-delay:0s"></div></div><span class="sec">58秒</span></div>
<div class="row"><span class="lab">UNHEALTHY判定まで</span><div class="track"><div class="bar" style="width:78%;animation-delay:.3s"></div></div><span class="sec">45秒</span></div>
<div class="row"><span class="lab">その差</span><div class="track"><div class="bar gap" style="width:22%;animation-delay:.6s"></div></div><span class="sec">13秒</span></div>
<p class="note">判定のほうが13秒早い。起動が終わる前に、負けが決まっていた</p>
</div>
<style>
.racebar { --line:#3ea8ff; --hot:#d9534f; --gap:#e08a3c; --tx:#1a2733; --trk:rgba(127,127,127,.16); display:flex; flex-direction:column; gap:8px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .racebar { --tx:#e8f2fa; }
.racebar .row { display:flex; align-items:center; gap:10px; }
.racebar .lab { flex:none; width:118px; color:var(--tx); font-size:12px; white-space:nowrap; }
.racebar .track { flex:1; height:14px; background:var(--trk); border-radius:4px; overflow:hidden; }
.racebar .bar { height:100%; background:var(--line); border-radius:4px; transform-origin:left center; animation:racegrow 1.4s ease both; }
.racebar .bar.hot { background:var(--hot); }
.racebar .bar.gap { background:var(--gap); }
@keyframes racegrow { from { transform:scaleX(0); } to { transform:scaleX(1); } }
.racebar .sec { flex:none; width:40px; color:var(--tx); font-size:12px; font-weight:700; white-space:nowrap; }
.racebar .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:4px 0 0; }
@media (prefers-reduced-motion: reduce) { .racebar * { animation:none !important; } }
</style>
```

:::details T+77秒のチェックが失敗した理由について
原典の記録では、T+77秒のチェックが失敗した理由は分かっていません。ここでは推測で補わず、記録された通りに書いています。1回目の成功（T+62秒）のあと、2回目の成功が返らなかったことだけが事実です。
:::

---

## 原因1：起動のたびに composer install を実行していた

なぜ起動に58秒もかかったのか。原因は、コンテナの起動スクリプトにありました。起動スクリプトの中で、composer install を実行していたんです。

依存パッケージのインストールに、約40〜50秒かかっていました。起動時間の大半が、ここに消えていたわけです。

```bash
# 起動スクリプト（修正前）
composer install --no-dev -o --no-interaction  # ← 約40〜50秒かかる
php artisan migrate --force
php artisan config:cache
...
php-fpm8.4 &
sleep 3
exec nginx -g 'daemon off;'
```

背景には、イメージの作り方があります。`.dockerignore` で `src/vendor` を除外していたため、イメージの中に依存パッケージがありません。だから起動のたびに、毎回ダウンロードしてインストールしていたんです。

起動時にやる必要のない作業を、起動時にやっていた。これが58秒の正体です。

```html-embed
<div class="startmorph">
<div class="panel before" style="animation-delay:0s">
<div class="ttl">修正前 ― 起動時に全部やる</div>
<div class="bar"><span class="seg big">composer install 約40〜50秒</span></div>
<div class="bar"><span class="seg small">マイグレーション・設定キャッシュ</span></div>
<div class="bar"><span class="seg small">PHP-FPM・Nginx 起動</span></div>
<div class="total">合計 約58秒</div>
</div>
<div class="panel after" style="animation-delay:.4s">
<div class="ttl">修正後 ― ビルド時に寄せる</div>
<div class="bar"><span class="seg done">composer install はビルド時へ</span></div>
<div class="bar"><span class="seg small">マイグレーション・設定キャッシュ</span></div>
<div class="bar"><span class="seg small">PHP-FPM・Nginx 起動</span></div>
<div class="total">合計 約15〜20秒</div>
</div>
<p class="note">起動時の処理を減らす。速くするのではなく、やる場所を変える</p>
</div>
<style>
.startmorph { --bad:#d9534f; --ok:#2f9e5f; --tx:#1a2733; --bgb:#fdf0ef; --bga:#eef7f1; --bdb:#e5b4b1; --bda:#a9d4ba; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .startmorph { --bgb:#3a1d1b; --bga:#173325; --tx:#f2ede4; --bdb:#6e3a37; --bda:#2c5b40; }
.startmorph .panel { border-radius:8px; padding:10px 14px; animation:morphfade 1s ease both; }
@keyframes morphfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.startmorph .panel.before { background:var(--bgb); border:1px solid var(--bdb); border-top:4px solid var(--bad); }
.startmorph .panel.after { background:var(--bga); border:1px solid var(--bda); border-top:4px solid var(--ok); }
.startmorph .ttl { color:var(--tx); font-weight:700; font-size:13.5px; margin-bottom:8px; }
.startmorph .bar { margin:4px 0; }
.startmorph .seg { display:inline-block; color:#fff; font-size:12px; font-weight:700; border-radius:4px; padding:3px 8px; line-height:1.5; }
.startmorph .seg.big { background:var(--bad); }
.startmorph .seg.small { background:rgba(127,127,127,.55); }
.startmorph .seg.done { background:var(--ok); }
.startmorph .total { color:var(--tx); font-size:13px; font-weight:700; margin-top:6px; }
.startmorph .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:4px 0 0; text-align:center; }
@media (prefers-reduced-motion: reduce) { .startmorph * { animation:none !important; } }
</style>
```

---

## 原因2：グレース期間は「チェックが止まる時間」ではなかった

次に、ALB側の設定です。間隔15秒、unhealthy閾値3回。つまり3回連続で失敗すると、45秒でUNHEALTHYになります。

healthy閾値は2回で、UNHEALTHYから戻るには2回連続の成功が必要です。

| 設定 | 値 |
| --- | --- |
| 間隔 | 15秒 |
| unhealthy閾値 | 3回 |
| healthy閾値 | 2回 |
| グレース期間 | 60秒 |

ここで、よくある勘違いがあります。グレース期間60秒を「60秒間はヘルスチェックが止まる」と思い込んでいたんです。でも公式ドキュメントの定義は違います。

healthCheckGracePeriodSeconds は「ECSのサービススケジューラが、タスク開始後の一定時間、ELBのヘルスチェック結果を無視する」設定です。無視するのはECS側の判断だけで、ALB自体のチェックは止まりません。

```html-embed
<div class="gracevs">
<div class="cols">
<div class="card ng" style="animation-delay:0s">
<div class="ttl">思い込み</div>
<p class="desc">グレース期間の60秒間は、ヘルスチェックが止まる</p>
<div class="judge">→ 実際は違う</div>
</div>
<div class="card ok" style="animation-delay:.25s">
<div class="ttl">実際</div>
<p class="desc">ECSが結果を無視するだけ。ALBは最初からチェックし続ける</p>
<div class="judge">→ 状態は動き続ける</div>
</div>
</div>
<p class="basis">根拠：公式ドキュメントの定義は「サービススケジューラが結果を無視する期間」。ALBのチェック自体を止める設定ではない</p>
</div>
<style>
.gracevs { --ng:#d9534f; --ok:#2f9e5f; --bgn:#fdf0ef; --bgo:#eef7f1; --tx:#1a2733; --bdn:#e5b4b1; --bdo:#a9d4ba; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .gracevs { --bgn:#3a1d1b; --bgo:#173325; --tx:#f2ede4; --bdn:#6e3a37; --bdo:#2c5b40; }
.gracevs .cols { display:flex; gap:10px; }
.gracevs .card { flex:1 1 0; min-width:0; border-radius:8px; padding:10px 14px; animation:gracefade .8s ease both; }
.gracevs .card.ng { background:var(--bgn); border:1px solid var(--bdn); border-top:4px solid var(--ng); }
.gracevs .card.ok { background:var(--bgo); border:1px solid var(--bdo); border-top:4px solid var(--ok); }
@keyframes gracefade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.gracevs .ttl { color:var(--tx); font-weight:700; font-size:14px; }
.gracevs .desc { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
.gracevs .judge { font-size:13px; font-weight:700; margin-top:6px; }
.gracevs .ng .judge { color:var(--ng); }
.gracevs .ok .judge { color:var(--ok); }
.gracevs .basis { color:var(--tx); font-size:12px; line-height:1.55; opacity:.85; margin:8px 0 0; }
@media (max-width:560px) { .gracevs .cols { flex-direction:column; } }
@media (prefers-reduced-motion: reduce) { .gracevs * { animation:none !important; } }
</style>
```

だから、こうなります。T+45秒でALBはUNHEALTHYに遷移する。T+60秒にグレース期間が終わり、ECSが結果を見始めた時点で、ターゲットはすでにUNHEALTHYです。

そこからHEALTHYに戻るには、2回連続の成功が必要です。T+62秒で1回成功したものの、T+77秒で失敗して、戻れませんでした。

グレース期間は「安全な猶予」ではなく「ECSが見ない時間」です。ALBの状態は、その間も動き続けます。

:::message
グレース期間を長くしても、ALBがUNHEALTHYになること自体は防げません。防げるのは、ECSがその結果でタスクを止めることだけです。この区別が、今回の核心でした。
:::

なお、グレース期間の既定値は、APIでは0秒です。CDKでLB付きのサービスを作ると、未指定の場合に60秒が入ります。

今回の構成の60秒は、そのCDKの既定値でした。つまり「60秒あれば足りるはず」という前提そのものが、起動58秒に対しては余裕のない数字だったわけです。

---

## 原因3：コンテナの startPeriod が10秒しかなかった

3つ目の要因は、コンテナのヘルスチェックです。startPeriod が10秒しかありませんでした。

startPeriod は、起動直後の失敗をカウントしない猶予です。公式の定義では「コンテナがブートストラップする時間を与えるための、任意の猶予期間」です。

起動に58秒かかるのに、猶予が10秒しかない。30秒ごとの最初のチェックは、必ず失敗してカウントされていました。

```ts
// タスク定義のヘルスチェック（修正前）
healthCheck: {
  command: ["CMD-SHELL", "curl -f http://localhost:80/healthcheck || exit 1"],
  interval: cdk.Duration.seconds(30),
  retries: 3,
  startPeriod: cdk.Duration.seconds(10), // ← 短すぎ！起動に58秒かかるのに
}
```

```html-embed
<div class="startp">
<div class="row"><span class="lab">startPeriod</span><div class="track"><div class="bar short" style="width:17%;animation-delay:0s"></div></div><span class="sec">10秒</span></div>
<div class="row"><span class="lab">実際の起動時間</span><div class="track"><div class="bar long" style="width:100%;animation-delay:.3s"></div></div><span class="sec">58秒</span></div>
<div class="ticks">
<span class="tick" style="left:52%">T+30秒 1回目のチェック → 失敗をカウント</span>
<span class="tick" style="left:100%">T+58秒 起動完了</span>
</div>
<p class="note">猶予が切れたあとのチェックは、起動が終わる前から失敗を数え始める</p>
</div>
<style>
.startp { --short:#d9534f; --long:#3ea8ff; --tx:#1a2733; --trk:rgba(127,127,127,.16); display:flex; flex-direction:column; gap:8px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .startp { --tx:#e8f2fa; }
.startp .row { display:flex; align-items:center; gap:10px; }
.startp .lab { flex:none; width:104px; color:var(--tx); font-size:12px; white-space:nowrap; }
.startp .track { flex:1; height:14px; background:var(--trk); border-radius:4px; overflow:hidden; }
.startp .bar { height:100%; border-radius:4px; transform-origin:left center; animation:spgrow 1.4s ease both; }
.startp .bar.short { background:var(--short); }
.startp .bar.long { background:var(--long); }
@keyframes spgrow { from { transform:scaleX(0); } to { transform:scaleX(1); } }
.startp .sec { flex:none; width:40px; color:var(--tx); font-size:12px; font-weight:700; white-space:nowrap; }
.startp .ticks { position:relative; height:34px; margin-left:114px; }
.startp .tick { position:absolute; top:0; color:var(--tx); font-size:11.5px; line-height:1.45; transform:translateX(-50%); text-align:center; }
.startp .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:4px 0 0; }
@media (prefers-reduced-motion: reduce) { .startp * { animation:none !important; } }
</style>
```

なお、startPeriod にはもう1つ注意点があります。猶予期間中に1回でもチェックが成功すると、コンテナはhealthyと見なされ、それ以降の失敗はカウントされます。猶予は「起動が終わるまでの盾」であって、成功した後の免罪符ではありません。

---

## なぜ同じイメージの別サービスは動いたのか

不思議な点が1つあります。同じDockerイメージを使うAPI系のサービスは、正常にデプロイできていたんです。

イメージが同じなら、起動時間も同じはず。それでも片方だけこける。

違いは、設定にありました。

| 項目 | 管理画面系（問題あり） | API系（問題なし） |
| --- | --- | --- |
| startPeriod | 10秒 | 60秒 |
| LB種別 | ALB | NLB |
| チェック間隔 | 15秒 | 30秒 |
| unhealthy閾値 | 3回 | 5回 |
| UNHEALTHYまで | 45秒 | 150秒 |

API系はNLBで、UNHEALTHYになるまで150秒かかります。グレース期間60秒の間に走るチェックは最大2回で、閾値5回には届きません。だからグレース終了の時点でUNHEALTHYになっておらず、Nginxが起きれば素直にHEALTHYへ遷移できたわけです。

同じイメージでも、判定の厳しさが違えば結果が変わる。これが「設定の競合」という話の裏付けです。

原典の教訓にもある通り、ALBはNLBより判定が厳しくなりがちです。ALBはHTTPの応答内容まで見るぶん、間隔・閾値・グレース期間の組み合わせがそのまま判定の速さに出ます。

NLBに変えれば緩和できますが、それは「判定を甘くする」選択です。今回の修正では、LBの種類は変えず、起動を速くする方を選びました。

```html-embed
<div class="twinsvc">
<div class="cols">
<div class="panel ng" style="animation-delay:0s">
<div class="svc">管理画面系 ― ALB</div>
<div class="line"><span class="k">間隔</span><span class="v">15秒</span></div>
<div class="line"><span class="k">unhealthy閾値</span><span class="v">3回</span></div>
<div class="line"><span class="k">UNHEALTHYまで</span><span class="v hot">45秒</span></div>
<div class="line"><span class="k">startPeriod</span><span class="v hot">10秒</span></div>
<div class="result ng">起動58秒より先に判定 → ロールバック</div>
</div>
<div class="panel ok" style="animation-delay:.3s">
<div class="svc">API系 ― NLB</div>
<div class="line"><span class="k">間隔</span><span class="v">30秒</span></div>
<div class="line"><span class="k">unhealthy閾値</span><span class="v">5回</span></div>
<div class="line"><span class="k">UNHEALTHYまで</span><span class="v">150秒</span></div>
<div class="line"><span class="k">startPeriod</span><span class="v">60秒</span></div>
<div class="result ok">起動58秒が先に終わる → デプロイ成功</div>
</div>
</div>
<p class="note">同じイメージ、同じ起動時間。結果を分けたのは設定だけ</p>
</div>
<style>
.twinsvc { --ng:#d9534f; --ok:#2f9e5f; --tx:#1a2733; --bgn:#fdf0ef; --bgo:#eef7f1; --bdn:#e5b4b1; --bdo:#a9d4ba; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .twinsvc { --bgn:#3a1d1b; --bgo:#173325; --tx:#f2ede4; --bdn:#6e3a37; --bdo:#2c5b40; }
.twinsvc .cols { display:flex; gap:10px; }
.twinsvc .panel { flex:1 1 0; min-width:0; border-radius:8px; padding:10px 14px; animation:twinfade .8s ease both; }
.twinsvc .panel.ng { background:var(--bgn); border:1px solid var(--bdn); border-top:4px solid var(--ng); }
.twinsvc .panel.ok { background:var(--bgo); border:1px solid var(--bdo); border-top:4px solid var(--ok); }
@keyframes twinfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.twinsvc .svc { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.twinsvc .line { display:flex; justify-content:space-between; gap:8px; color:var(--tx); font-size:12.5px; line-height:1.6; }
.twinsvc .v { font-weight:700; white-space:nowrap; }
.twinsvc .v.hot { color:var(--ng); }
.twinsvc .result { font-size:12.5px; font-weight:700; line-height:1.55; margin-top:8px; padding-top:8px; border-top:1px dashed rgba(127,127,127,.4); }
.twinsvc .result.ng { color:var(--ng); }
.twinsvc .result.ok { color:var(--ok); }
.twinsvc .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:8px 0 0; text-align:center; }
@media (max-width:560px) { .twinsvc .cols { flex-direction:column; } }
@media (prefers-reduced-motion: reduce) { .twinsvc * { animation:none !important; } }
</style>
```

:::details ALBとNLBの既定値について
ALB・NLBとも、公式ドキュメントの既定値は間隔30秒、healthy閾値5回、unhealthy閾値2回です。上の表の値は、この構成で変えたものです。既定のままでも、healthy閾値5回なら復帰に5回連続の成功が必要になる点には注意が必要です。
:::

---

## 修正は3つ：ビルド時に寄せて、猶予を合わせる

修正は3つです。どれも「起動を短くする」「猶予を起動に合わせる」という方向で、設定を緩める方向には振りませんでした。

### 修正1：Dockerfileに composer install を追加する

```docker
# ソースを /var/www/html にコピー
COPY --chown=www-data:www-data ./src /var/www/html/

# Composerの依存関係をビルド時にインストール（起動時間短縮のため）
# .dockerignore で src/vendor を除外しているため、ここで改めてインストールする
RUN composer install --no-dev -o --no-interaction

# フロントエンド（Vite）ビルド
RUN npm ci && npm run build
```

`.dockerignore` で除外している分を、ビルド時に改めてインストールします。イメージに依存パッケージが入るので、起動時のダウンロードは不要になります。

### 修正2：起動スクリプトから composer install を削除する

```bash
# 削除した行
export COMPOSER_ALLOW_SUPERUSER=1
composer install --no-dev -o --no-interaction
```

ビルド時に実行済みのため、起動時には不要です。起動スクリプトは、マイグレーションとプロセス起動だけに絞ります。

### 修正3：コンテナの startPeriod を10秒から60秒にする

```ts
// タスク定義のヘルスチェック（修正後）
healthCheck: {
  command: ["CMD-SHELL", "curl -f http://localhost:80/healthcheck || exit 1"],
  interval: cdk.Duration.seconds(30),
  retries: 3,
  startPeriod: cdk.Duration.seconds(60), // 10s → 60s（API系に合わせる）
}
```

目安はシンプルです。startPeriod は、実際の起動時間以上にする。起動が15〜20秒になった今、60秒の猶予には十分な余裕があります。

```html-embed
<div class="afterflow">
<svg viewBox="0 0 700 200" role="img" aria-label="修正後のタイムライン">
<defs>
<marker id="aha" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<line class="beam" marker-end="url(#aha)" x1="150" y1="80" x2="178" y2="80"/>
<line class="beam" marker-end="url(#aha)" x1="332" y1="80" x2="360" y2="80"/>
<line class="beam" marker-end="url(#aha)" x1="510" y1="80" x2="538" y2="80"/>
<rect class="node seq" style="animation-delay:0s" x="20" y="58" width="130" height="44" rx="8"/>
<text class="t" x="85" y="77" text-anchor="middle">T+0秒 起動</text>
<text class="ts" x="85" y="94" text-anchor="middle">composer 不要に</text>
<rect class="node seq" style="animation-delay:1.2s" x="182" y="58" width="150" height="44" rx="8"/>
<text class="t" x="257" y="77" text-anchor="middle">T+5〜8秒</text>
<text class="ts" x="257" y="94" text-anchor="middle">PHP-FPM・Nginx 起動</text>
<rect class="node seq" style="animation-delay:2.4s" x="364" y="58" width="146" height="44" rx="8"/>
<text class="t" x="437" y="77" text-anchor="middle">T+15秒 200</text>
<text class="ts" x="437" y="94" text-anchor="middle">1回目の成功</text>
<rect class="node seq" style="animation-delay:3.6s" x="542" y="58" width="138" height="44" rx="8"/>
<text class="t" x="611" y="77" text-anchor="middle">T+30秒 HEALTHY</text>
<text class="ts" x="611" y="94" text-anchor="middle">2回連続で成功</text>
<text class="cap" x="350" y="150" text-anchor="middle">判定ラインの45秒より、ずっと前に HEALTHY へ</text>
<text class="cap2" x="350" y="172" text-anchor="middle">※ このタイムラインは原典の「想定」としての記録です</text>
</svg>
</div>
<style>
.afterflow { --line:#2f9e5f; --node:#eef7f1; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .afterflow { --node:#173325; --tx:#e8f2fa; }
.afterflow svg { width:100%; height:auto; display:block; }
.afterflow .beam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:afterflowm 1.2s linear infinite; }
.afterflow .ahp { fill:var(--line); }
@keyframes afterflowm { to { stroke-dashoffset:-14; } }
.afterflow .node { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.afterflow text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.afterflow .t { font-size:13px; font-weight:700; }
.afterflow .ts { font-size:11px; opacity:.75; }
.afterflow .seq { animation:afterseq 4.8s ease-in-out infinite; }
@keyframes afterseq { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
.afterflow .cap { font-size:13px; font-weight:700; fill:var(--line); }
.afterflow .cap2 { font-size:11px; opacity:.7; }
@media (prefers-reduced-motion: reduce) { .afterflow * { animation:none !important; } }
</style>
```

修正後の想定タイムラインはこうなります。T+5秒でPHP-FPM、T+8秒でNginxが起動。T+15秒の1回目のチェックで200が返り、T+30秒の2回目でHEALTHYになります。

---

## 修正後の結果：起動15〜20秒、デプロイが通るようになった

| 項目 | 修正前 | 修正後 |
| --- | --- | --- |
| 起動時間 | 約58秒 | 約15〜20秒 |
| ALB 1回目 | 失敗 | 200 |
| デプロイ | ロールバック | 成功 |
| startPeriod | 10秒 | 60秒 |

起動が15〜20秒になれば、45秒の判定ラインを十分に下回ります。競合そのものが消えたわけです。

設定を緩めたのではなく、起動を速くした。これが、今回の修正の肝でした。

修正が効いたかどうかの確認は、サービスのデプロイ状態で見ます。デプロイが定常状態に達すれば、rolloutState は COMPLETED になります。

サーキットブレーカーが FAILED にせず、ロールバックも走らなければ、修正は効いていると判断できます。逆に言えば、デプロイの成否は「新しいコードが動いたか」だけでなく、この状態遷移でも確認できるということです。

```html-embed
<div class="afterbar">
<div class="row"><span class="lab">修正前の起動</span><div class="track"><div class="bar hot" style="width:100%;animation-delay:0s"></div></div><span class="sec">58秒</span></div>
<div class="row"><span class="lab">修正後の起動</span><div class="track"><div class="bar ok" style="width:31%;animation-delay:.3s"></div></div><span class="sec">15〜20秒</span></div>
<div class="row"><span class="lab">判定ライン</span><div class="track"><div class="bar line" style="width:78%;animation-delay:.6s"></div></div><span class="sec">45秒</span></div>
<p class="note">起動が判定ラインを下回れば、競合は起きない。削ったのは43秒ではなく、競合そのもの</p>
</div>
<style>
.afterbar { --hot:#d9534f; --ok:#2f9e5f; --line:#e08a3c; --tx:#1a2733; --trk:rgba(127,127,127,.16); display:flex; flex-direction:column; gap:8px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .afterbar { --tx:#e8f2fa; }
.afterbar .row { display:flex; align-items:center; gap:10px; }
.afterbar .lab { flex:none; width:104px; color:var(--tx); font-size:12px; white-space:nowrap; }
.afterbar .track { flex:1; height:14px; background:var(--trk); border-radius:4px; overflow:hidden; }
.afterbar .bar { height:100%; border-radius:4px; transform-origin:left center; animation:aftergrow 1.4s ease both; }
.afterbar .bar.hot { background:var(--hot); }
.afterbar .bar.ok { background:var(--ok); }
.afterbar .bar.line { background:var(--line); }
@keyframes aftergrow { from { transform:scaleX(0); } to { transform:scaleX(1); } }
.afterbar .sec { flex:none; width:56px; color:var(--tx); font-size:12px; font-weight:700; white-space:nowrap; }
.afterbar .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:4px 0 0; }
@media (prefers-reduced-motion: reduce) { .afterbar * { animation:none !important; } }
</style>
```

---

## 効かなかったこと・見送ったこと

速くするためなら、何でも入れればいいわけではありません。今回、あえて手を付けなかった案が3つあります。

| 案 | 判断 | 理由 |
| --- | --- | --- |
| グレース期間の延長だけ | 見送り | 起動の遅さは残る |
| ALBの間隔・閾値を緩めるだけ | 見送り | 検出が遅れる |
| サーキットブレーカーを切る | 見送り | 壊れたデプロイが残る |

```html-embed
<div class="rejc2">
<div class="card" style="animation-delay:0s">
<div class="head"><span class="mark">✗</span><span class="name">グレース期間の延長だけ</span><span class="badge">見送り</span></div>
<p class="reason">その場は通るかもしれない。でも起動が58秒のままでは、少し遅れただけでまた同じことが起きる</p>
</div>
<div class="card" style="animation-delay:.25s">
<div class="head"><span class="mark">✗</span><span class="name">ALBの間隔・閾値を緩めるだけ</span><span class="badge">見送り</span></div>
<p class="reason">判定を遅らせる代わりに、本当に壊れたときの検出も遅れる。安全装置を鈍らせることになる</p>
</div>
<div class="card" style="animation-delay:.5s">
<div class="head"><span class="mark">✗</span><span class="name">サーキットブレーカーを切る</span><span class="badge">見送り</span></div>
<p class="reason">ロールバックしなくなるだけで、壊れたデプロイが残る。問題を隠すだけで、解決にはならない</p>
</div>
</div>
<style>
.rejc2 { --acc:#d9534f; --bg:#fdf0ef; --tx:#1a2733; --bd:#e5b4b1; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .rejc2 { --bg:#3a1d1b; --tx:#f2e9e8; --bd:#6e3a37; }
.rejc2 .card { background:var(--bg); border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:10px 14px; animation:rejfade2 .8s ease both; }
@keyframes rejfade2 { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.rejc2 .head { display:flex; align-items:center; gap:8px; }
.rejc2 .mark { color:var(--acc); font-weight:700; font-size:15px; }
.rejc2 .name { color:var(--tx); font-weight:700; font-size:14px; }
.rejc2 .badge { margin-left:auto; color:var(--acc); border:1px solid var(--acc); border-radius:999px; font-size:11px; padding:1px 8px; white-space:nowrap; }
.rejc2 .reason { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
@media (prefers-reduced-motion: reduce) { .rejc2 * { animation:none !important; } }
</style>
```

どれも「競合を先送りする」案で、原因を消していません。グレース期間を延ばせば、その場は通るかもしれません。でも起動の遅さが残る限り、少し遅れただけでまた同じことが起きます。

間隔や閾値を緩めるのも同じです。判定を遅らせる代わりに、本当に壊れたときの検出も遅れます。サーキットブレーカーを切るのは、問題を隠すだけで、解決にはなりません。

:::message
速くするのではなく、競合そのものを消す。今回の判断軸はこれでした。設定を緩める案は、すべてこの軸で却下しています。
:::

---

## 見落としやすい勘違い3つ

今回の調査で、実際に引っかかった思い込みを3つ残しておきます。

```html-embed
<div class="pitc2">
<div class="card" style="animation-delay:0s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">グレース期間は、チェックが止まる時間</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">ECSが結果を無視するだけ。ALBは最初からチェックし続ける</span></div>
</div>
<div class="card" style="animation-delay:.3s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">ログに200があるから、ヘルスチェックは通っている</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">200を返したのは、Nginxが起きた後の話。ALBが見ていたのは起きる前の45秒</span></div>
</div>
<div class="card" style="animation-delay:.6s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">同じイメージなら、同じ挙動になる</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">判定の厳しさは設定で決まる。間隔・閾値・猶予が違えば結果も変わる</span></div>
</div>
</div>
<style>
.pitc2 { --ng:#e08a3c; --ok:#2f9e5f; --bg:#fdf6ec; --tx:#1a2733; --bd:#e3cfa8; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .pitc2 { --bg:#3a2c16; --tx:#f2ede4; --bd:#6e5a33; }
.pitc2 .card { background:var(--bg); border:1px solid var(--bd); border-radius:8px; padding:10px 14px; animation:pitfade2 .8s ease both; }
@keyframes pitfade2 { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.pitc2 .row { display:flex; align-items:flex-start; gap:8px; }
.pitc2 .tag { flex:none; border-radius:999px; font-size:11px; font-weight:700; padding:2px 9px; margin-top:1px; white-space:nowrap; }
.pitc2 .tag.ng { color:var(--ng); border:1px solid var(--ng); }
.pitc2 .tag.ok { color:var(--ok); border:1px solid var(--ok); }
.pitc2 .txt { color:var(--tx); font-size:13px; line-height:1.6; }
.pitc2 .arrow { color:var(--ng); font-size:13px; text-align:center; line-height:1.4; }
@media (prefers-reduced-motion: reduce) { .pitc2 * { animation:none !important; } }
</style>
```

ログに200が残っているのに「ヘルスチェック失敗」と言われるのは、納得しにくい現象でした。でも200を返したのは、Nginxが起きた後の話です。ALBが見ていたのは、起きる前の45秒だったんです。

時間を切り分けて見れば、矛盾はありません。ログと判定が違う時間を指しているだけでした。

---

## まとめ

- 起動時の処理はビルド時に寄せる。起動時のダウンロードはアンチパターン
- グレース期間は「ECSが無視する時間」。ALBの状態はリセットされない
- startPeriod は実際の起動時間以上にする。目安は startPeriod ≥ 起動時間
- ALBとNLBでは判定の厳しさが違う。間隔・閾値・グレース期間をセットで見る
- 競合は「設定を緩める」のではなく「起動を速くする」で消す

判断の順番を、基準として並べるとこうなります。

| 順番 | やること | この記事での例 |
| --- | --- | --- |
| 1 | 起動時間を測る | 58秒と判明 |
| 2 | 起動時の処理を減らす | composer install をビルド時へ |
| 3 | 猶予を起動に合わせる | startPeriod 10秒→60秒 |
| 4 | 判定ラインを確認する | 45秒 vs 15〜20秒 |

```html-embed
<div class="judgeflow">
<div class="step" style="animation-delay:0s"><span class="num">1</span><div class="body"><div class="main">測る</div><div class="sub">起動時間と判定ライン</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:1s"><span class="num">2</span><div class="body"><div class="main">起動を短くする</div><div class="sub">処理をビルド時へ</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:2s"><span class="num">3</span><div class="body"><div class="main">猶予を合わせる</div><div class="sub">startPeriod ≥ 起動時間</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:3s"><span class="num">4</span><div class="body"><div class="main">判定を確認する</div><div class="sub">起動 ＜ 判定ライン</div></div></div>
</div>
<style>
.judgeflow { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; display:flex; align-items:stretch; gap:6px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .judgeflow { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.judgeflow .step { flex:1 1 0; min-width:0; display:flex; align-items:flex-start; gap:7px; background:var(--bg); border:1.5px solid var(--bd); border-radius:8px; padding:9px 10px; animation:judgepulse 4s ease-in-out infinite; }
@keyframes judgepulse { 0%, 100% { border-color:var(--bd); } 8% { border-color:var(--acc); box-shadow:0 0 0 2px var(--acc) inset; } 20% { border-color:var(--bd); box-shadow:none; } }
.judgeflow .num { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-size:12px; font-weight:700; text-align:center; line-height:20px; margin-top:1px; }
.judgeflow .main { color:var(--tx); font-size:13px; font-weight:700; line-height:1.4; }
.judgeflow .sub { color:var(--tx); font-size:11px; line-height:1.45; opacity:.8; }
.judgeflow .arr { flex:none; align-self:center; color:var(--acc); font-size:14px; font-weight:700; }
@media (max-width:560px) { .judgeflow { flex-wrap:wrap; } .judgeflow .step { flex:1 1 42%; } .judgeflow .arr { display:none; } }
@media (prefers-reduced-motion: reduce) { .judgeflow * { animation:none !important; } }
</style>
```

### デプロイ前のチェックリスト（そのまま使えます）

下のチェックリストは、この記事の内容をそのまま使える形にしたものです。デプロイ前の確認に1分だけ使えば、58秒と45秒の競合は防げます。ブックマークして、次回のデプロイ前に開いてください。

```html-embed
<div class="depcheck">
<div class="ttl">デプロイ前のチェックリスト</div>
<div class="item" style="animation-delay:0s"><span class="chk">☐</span><span class="txt">起動時間を実測したか（目安：判定ラインの半分以下）</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">☐</span><span class="txt">起動時にパッケージのダウンロードをしていないか</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">☐</span><span class="txt">startPeriod ≥ 実測の起動時間か</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">☐</span><span class="txt">グレース期間の終了時点で、ALBがUNHEALTHYになっていないか</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">☐</span><span class="txt">healthy閾値分の連続成功に必要な時間を見込んだか</span></div>
<div class="item" style="animation-delay:.75s"><span class="chk">☐</span><span class="txt">同じイメージの別サービスと、設定を比べたか</span></div>
</div>
<style>
.depcheck { --acc:#2f9e5f; --bg:#eef7f1; --tx:#1a2733; --bd:#a9d4ba; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .depcheck { --bg:#173325; --tx:#e8f2fa; --bd:#2c5b40; }
.depcheck .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.depcheck .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:depfade .7s ease both; }
@keyframes depfade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.depcheck .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.depcheck .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .depcheck * { animation:none !important; } }
</style>
```

- [ ] 起動時間を実測したか（目安：判定ラインの半分以下）
- [ ] 起動時にパッケージのダウンロードをしていないか
- [ ] startPeriod ≥ 実測の起動時間か
- [ ] グレース期間の終了時点で、ALBがUNHEALTHYになっていないか
- [ ] healthy閾値分の連続成功に必要な時間を見込んだか
- [ ] 同じイメージの別サービスと、設定を比べたか

次回は、Claude Codeで日報の下書きを自動化した仕組みを書きます。
