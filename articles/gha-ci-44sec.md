---
title: "GitHub ActionsのCIを3分→44秒に短縮 ― 決め手はキャッシュではなく composer install の廃止だった"
emoji: "⚡"
type: "tech"
topics: ["githubactions", "ci", "laravel", "php"]
published: true
---

## はじめに：3分は、短いようで長い

PRを出すたびに、CIの完了を待つ時間があります。1回で約3分。大したことないように見えます。

でもpushのたびに3分待たされると、作業のリズムは確実に切れます。「終わったら見よう」と思った頃には、頭は別の作業に移っている。

この記事は、LaravelのフォーマットチェックCIを題材にした高速化の記録です。約3分かかっていたGitHub Actionsのワークフローを、44秒まで短縮しました。

やったことは2つだけです。

- composer install をやめて、ツール単体のpharを直接ダウンロードする
- チェックの実行を並列化する

定番のキャッシュは、使いませんでした。その理由も含めて書きます。

```html-embed
<div class="learnc">
<div class="ttl">この記事で分かること</div>
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">CIのボトルネックをステップ単位で特定する方法</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">composer install を排除し、pharを直接実行する構成</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">pharのバージョン整合性を composer.lock で保つ方法</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">✓</span><span class="txt">Experimentalな並列化オプションを採用するときのリスクの見分け方</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">✓</span><span class="txt">キャッシュや差分チェックを、あえて採用しなかった理由</span></div>
</div>
<style>
.learnc { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .learnc { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.learnc .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.learnc .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:learnfade .7s ease both; }
@keyframes learnfade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.learnc .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.learnc .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .learnc * { animation:none !important; } }
</style>
```

---

## まず測る：犯人はステップ単位でしか見えない

最初にやったのは、改善ではなく計測です。`gh run view --log` で実行ログを開き、ステップごとの所要時間を並べました。

改善前の内訳はこうです。

| ステップ | 時間 |
| --- | --- |
| Setup PHP | 約10秒 |
| composer install | 1分40秒 |
| Pint 実行 | 2分17秒 |
| 合計 | 約3分（実行により2〜3分） |

```html-embed
<div class="timeb">
<div class="row"><span class="lab">Setup PHP</span><div class="track"><div class="bar" style="width:8%;animation-delay:0s"></div></div><span class="sec">約10秒</span></div>
<div class="row"><span class="lab">composer install</span><div class="track"><div class="bar hot" style="width:73%;animation-delay:.2s"></div></div><span class="sec">1分40秒</span></div>
<div class="row"><span class="lab">Pint 実行</span><div class="track"><div class="bar hot" style="width:100%;animation-delay:.4s"></div></div><span class="sec">2分17秒</span></div>
</div>
<style>
.timeb { --line:#3ea8ff; --hot:#e08a3c; --tx:#1a2733; --trk:rgba(127,127,127,.16); display:flex; flex-direction:column; gap:8px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .timeb { --tx:#e8f2fa; }
.timeb .row { display:flex; align-items:center; gap:10px; }
.timeb .lab { flex:none; width:104px; color:var(--tx); font-size:12px; white-space:nowrap; }
.timeb .track { flex:1; height:14px; background:var(--trk); border-radius:4px; overflow:hidden; }
.timeb .bar { height:100%; background:var(--line); border-radius:4px; transform-origin:left center; animation:timegrow 1.4s ease both; }
.timeb .bar.hot { background:var(--hot); }
@keyframes timegrow { from { transform:scaleX(0); } to { transform:scaleX(1); } }
.timeb .sec { flex:none; width:48px; color:var(--tx); font-size:12px; font-weight:700; white-space:nowrap; }
@media (prefers-reduced-motion: reduce) { .timeb * { animation:none !important; } }
</style>
```

合計だけ見ても、何も分かりません。ステップに割ると、時間の居場所が見えます。この計測が、後の判断すべての土台になりました。

計測のコツは、合計ではなくステップの境界を見ることです。ログには各ステップの開始と終了の時刻が残るので、差分を取るだけで内訳が出ます。実行ごとにばらつきはあるので、1回で断定せず、数回分を見て傾向を掴みました。

---

## 犯人は2ついた

内訳から見えたボトルネックは2つです。1つ目は composer install です。

フォーマットチェックに必要なツールは、Laravel Pint 1つだけです。それなのに毎回、プロジェクト全体の依存関係をインストールしていました。チェックのためだけに、vendor を丸ごと構築していたわけです。

2つ目は Pint の実行そのものです。対象は576ファイル。それをシングルプロセスで、順番にチェックしていました。

セットアップの無駄と、実行の直列。性質の違う2つの遅さが、重なっていたんです。

```html-embed
<div class="bnc">
<div class="card" style="animation-delay:0s">
<div class="head"><span class="name">composer install</span><span class="tag">セットアップの無駄</span><span class="sec">1分40秒</span></div>
<p class="desc">チェックに要るのは Pint 1つだけ。なのに毎回、vendor を丸ごと構築していた</p>
</div>
<div class="card" style="animation-delay:.25s">
<div class="head"><span class="name">Pint 実行</span><span class="tag">実行の直列</span><span class="sec">2分17秒</span></div>
<p class="desc">対象は576ファイル。それをシングルプロセスで、順番にチェックしていた</p>
</div>
</div>
<style>
.bnc { --acc:#e08a3c; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .bnc { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.bnc .card { background:var(--bg); border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:10px 14px; animation:bncfade .8s ease both; }
@keyframes bncfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.bnc .head { display:flex; align-items:center; gap:8px; flex-wrap:wrap; }
.bnc .name { color:var(--tx); font-weight:700; font-size:14px; }
.bnc .tag { color:var(--acc); border:1px solid var(--acc); border-radius:999px; font-size:11px; font-weight:700; padding:1px 8px; white-space:nowrap; }
.bnc .sec { margin-left:auto; color:var(--acc); font-weight:700; font-size:14px; white-space:nowrap; }
.bnc .desc { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
@media (prefers-reduced-motion: reduce) { .bnc * { animation:none !important; } }
</style>
```

---

## 一手目：composer install を捨てて、phar を直接叩く

Pint には、単体で動く phar 版があります。ならば、vendor を構築する必要自体がありません。

phar を直接ダウンロードして実行すれば、composer install は丸ごと消えます。変更後のワークフローはこうなりました。

```yaml
- name: Setup PHP
  uses: shivammathur/setup-php@v2
  with:
    php-version: '8.4'

- name: Get Pint version
  id: pint_version
  run: echo "version=$(jq -r '.["packages-dev"][] | select(.name == "laravel/pint") | .version' src/composer.lock)" >> "$GITHUB_OUTPUT"

- name: Download Pint
  run: curl -sL "https://github.com/laravel/pint/releases/download/${{ steps.pint_version.outputs.version }}/pint.phar" -o /tmp/pint.phar

- name: Laravel Format Check
  run: php /tmp/pint.phar --test --parallel
  working-directory: ./src
```

消したのは、composer 本体のセットアップと、依存関係のキャッシュ指定と、インストールステップです。残したのは Setup PHP だけです。その理由は後で書きます。

```html-embed
<div class="delc">
<div class="col del" style="animation-delay:0s">
<div class="ttl"><span class="mark">✗</span>消したもの</div>
<div class="item">composer 本体のセットアップ</div>
<div class="item">依存関係のキャッシュ指定</div>
<div class="item">インストールステップ</div>
</div>
<div class="col keep" style="animation-delay:.25s">
<div class="ttl"><span class="mark">✓</span>残したもの</div>
<div class="item">Setup PHP のみ</div>
<div class="note">バージョン固定が、保険になる</div>
</div>
</div>
<style>
.delc { --del:#d9534f; --keep:#2f9e5f; --bgd:#fdf0ef; --bgk:#eef7f1; --tx:#1a2733; --bdd:#e5b4b1; --bdk:#a9d4ba; display:flex; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .delc { --bgd:#3a1d1b; --bgk:#173325; --tx:#f2ede4; --bdd:#6e3a37; --bdk:#2c5b40; }
.delc .col { flex:1 1 0; min-width:0; border-radius:8px; padding:10px 14px; animation:delcfade .8s ease both; }
.delc .col.del { background:var(--bgd); border:1px solid var(--bdd); border-top:4px solid var(--del); }
.delc .col.keep { background:var(--bgk); border:1px solid var(--bdk); border-top:4px solid var(--keep); }
@keyframes delcfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.delc .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.delc .del .mark { color:var(--del); margin-right:6px; }
.delc .keep .mark { color:var(--keep); margin-right:6px; }
.delc .item { color:var(--tx); font-size:13px; line-height:1.6; padding:2px 0; }
.delc .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; padding-top:2px; }
@media (max-width:560px) { .delc { flex-direction:column; } }
@media (prefers-reduced-motion: reduce) { .delc * { animation:none !important; } }
</style>
```

### バージョンの整合性は composer.lock が担保する

phar を直接落とすと聞くと、バージョンがズレる心配があります。そこで、落とすバージョンは固定値にしませんでした。

composer.lock の中に記録された laravel/pint のバージョンを、jq で読み取って使います。これで、ローカルの `vendor/bin/pint` と CI の pint.phar は、常に同じバージョンになります。

lock を更新すれば、次回のCIから新しい phar が自動で落ちてきます。

```html-embed
<div class="lockf">
<svg viewBox="0 0 700 104" role="img" aria-label="composer.lockからpint.phar実行までの流れ">
<defs>
<marker id="ah3" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<line class="beam" marker-end="url(#ah3)" x1="160" y1="52" x2="188" y2="52"/>
<line class="beam" marker-end="url(#ah3)" x1="342" y1="52" x2="370" y2="52"/>
<line class="beam" marker-end="url(#ah3)" x1="520" y1="52" x2="548" y2="52"/>
<rect class="node seq" style="animation-delay:0s" x="20" y="30" width="140" height="44" rx="8"/>
<text class="t" x="90" y="49" text-anchor="middle">composer.lock</text>
<text class="ts" x="90" y="66" text-anchor="middle">記録された版</text>
<rect class="node seq" style="animation-delay:1.2s" x="192" y="30" width="150" height="44" rx="8"/>
<text class="t" x="267" y="49" text-anchor="middle">jq で抽出</text>
<text class="ts" x="267" y="66" text-anchor="middle">pint の版を読む</text>
<rect class="node seq" style="animation-delay:2.4s" x="374" y="30" width="146" height="44" rx="8"/>
<text class="t" x="447" y="49" text-anchor="middle">pint.phar 取得</text>
<text class="ts" x="447" y="66" text-anchor="middle">同じ版をダウンロード</text>
<rect class="node seq" style="animation-delay:3.6s" x="552" y="30" width="128" height="44" rx="8"/>
<text class="t" x="616" y="49" text-anchor="middle">実行</text>
<text class="ts" x="616" y="66" text-anchor="middle">--test でチェック</text>
</svg>
</div>
<style>
.lockf { --line:#3ea8ff; --node:#eef6ff; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .lockf { --node:#16324a; --tx:#e8f2fa; }
.lockf svg { width:100%; height:auto; display:block; }
.lockf .beam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:lockflow 1.2s linear infinite; }
.lockf .ahp { fill:var(--line); }
@keyframes lockflow { to { stroke-dashoffset:-14; } }
.lockf .node { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.lockf text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.lockf .t { font-size:13px; font-weight:700; }
.lockf .ts { font-size:11px; opacity:.75; }
.lockf .seq { animation:lockseq 4.8s ease-in-out infinite; }
@keyframes lockseq { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
@media (prefers-reduced-motion: reduce) { .lockf * { animation:none !important; } }
</style>
```

「キャッシュは整合性が危ないから避ける」という判断は、裏を返せば「整合性の担保さえできれば、手段は自由」ということでもあります。担保の仕組みを lock に寄せたのが、この一手目の肝でした。

---

## でも、半分しか速くならなかった

phar化だけを入れた時点の結果です。

| ステップ | phar化のみ |
| --- | --- |
| Setup PHP | 約10秒 |
| composer install | 0秒 |
| phar ダウンロード | 0.4秒 |
| Pint 実行 | 2分17秒 |
| 合計 | 約2分34秒 |

composer install は消えました。ダウンロードは、わずか0.4秒です。

でも合計は、3分から2分34秒までしか縮みません。Pint 実行の2分17秒が、そのまま残ったからです。

一番大きいステップを消しても、次に大きいステップが残れば、そこで止まります。当たり前やけど、これは実測するまで体感できませんでした。

「最大の無駄を消したのに、まだ遅い」。この状態が、二手目の必要性をはっきりさせました。

---

## 二手目：--parallel で最後の2分を消す

Pint には `--parallel` という並列実行のオプションがあります。ただし Experimental、つまり実験的機能の扱いです。採用するかどうかは、リスクの種類で判断しました。

今回のコマンドは `php /tmp/pint.phar --test --parallel` です。`--test` はチェック専用の読み取りモードで、ファイルを書き換えません。

つまり最悪の場合でも、起こるのは「チェックが失敗する」までです。コードが壊れることはありません。失敗したら、直して再実行すれば済みます。

「壊れるリスク」と「失敗するだけのリスク」は、別物です。読み取り専用の処理なら、Experimental でも踏み込める。そう判断して採用しました。

```html-embed
<div class="rskc">
<div class="cols">
<div class="card ng" style="animation-delay:0s">
<div class="ttl">壊れるリスク</div>
<p class="desc">最悪の場合、コードが壊れる</p>
<div class="judge">→ 踏み込まない</div>
</div>
<div class="card ok" style="animation-delay:.25s">
<div class="ttl">失敗するだけのリスク</div>
<p class="desc">最悪でも、チェックが失敗するだけ</p>
<div class="judge">→ 読み取り専用なら踏み込める</div>
</div>
</div>
<p class="basis">根拠：<code>--test</code> は読み取り専用で、ファイルを書き換えません</p>
</div>
<style>
.rskc { --ng:#d9534f; --ok:#2f9e5f; --bgn:#fdf0ef; --bgo:#eef7f1; --tx:#1a2733; --bdn:#e5b4b1; --bdo:#a9d4ba; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .rskc { --bgn:#3a1d1b; --bgo:#173325; --tx:#f2ede4; --bdn:#6e3a37; --bdo:#2c5b40; }
.rskc .cols { display:flex; gap:10px; }
.rskc .card { flex:1 1 0; min-width:0; border-radius:8px; padding:10px 14px; animation:rskcfade .8s ease both; }
.rskc .card.ng { background:var(--bgn); border:1px solid var(--bdn); border-top:4px solid var(--ng); }
.rskc .card.ok { background:var(--bgo); border:1px solid var(--bdo); border-top:4px solid var(--ok); }
@keyframes rskcfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.rskc .ttl { color:var(--tx); font-weight:700; font-size:14px; }
.rskc .desc { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
.rskc .judge { font-size:13px; font-weight:700; margin-top:6px; }
.rskc .ng .judge { color:var(--ng); }
.rskc .ok .judge { color:var(--ok); }
.rskc .basis { color:var(--tx); font-size:12px; line-height:1.55; opacity:.85; margin:8px 0 0; }
.rskc .basis code { font-size:11.5px; background:rgba(127,127,127,.18); border-radius:4px; padding:1px 5px; }
@media (max-width:560px) { .rskc .cols { flex-direction:column; } }
@media (prefers-reduced-motion: reduce) { .rskc * { animation:none !important; } }
</style>
```

結果、Pint の実行は2分17秒から23秒になりました。

### 結果：44秒の全体像

3段階の比較です。

| ステップ | 改善前 | phar化のみ | 最終 |
| --- | --- | --- | --- |
| Setup PHP | 約10秒 | 約10秒 | 12秒 |
| composer install | 1分40秒 | 0秒 | 0秒 |
| phar ダウンロード | ― | 0.4秒 | 0.4秒 |
| Pint 実行 | 2分17秒 | 2分17秒 | 23秒 |
| 合計 | 約3分 | 約2分34秒 | 約44秒 |

流れを図にすると、こうなります。上のバーが伸びきる間に、下のバーはとっくに終わっています。

```html-embed
<div class="ciflow">
<svg viewBox="0 0 700 248" role="img" aria-label="CIパイプラインのBeforeとAfter">
<defs>
<marker id="ah2" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<text class="lbl-o" x="20" y="28">Before ― 直列で約3分</text>
<line class="beam o" marker-end="url(#ah2)" x1="124" y1="62" x2="138" y2="62"/>
<line class="beam o" marker-end="url(#ah2)" x1="340" y1="62" x2="354" y2="62"/>
<line class="beam o" marker-end="url(#ah2)" x1="556" y1="62" x2="570" y2="62"/>
<rect class="node nd seq" style="animation-delay:0s" x="20" y="40" width="104" height="44" rx="8"/>
<text class="t" x="72" y="67" text-anchor="middle">Setup PHP</text>
<rect class="node nd seq" style="animation-delay:1.2s" x="142" y="40" width="198" height="44" rx="8"/>
<text class="t" x="241" y="59" text-anchor="middle">composer install</text>
<text class="ts" x="241" y="76" text-anchor="middle">1分40秒</text>
<rect class="node nd seq" style="animation-delay:2.4s" x="358" y="40" width="198" height="44" rx="8"/>
<text class="t" x="457" y="59" text-anchor="middle">Pint 実行（直列）</text>
<text class="ts" x="457" y="76" text-anchor="middle">2分17秒</text>
<rect class="node nd seq" style="animation-delay:3.6s" x="574" y="40" width="106" height="44" rx="8"/>
<text class="t b" x="627" y="67" text-anchor="middle">約3分</text>
<text class="ts" x="680" y="93" text-anchor="end">約3分</text>
<rect class="barb growb" x="20" y="98" width="660" height="9"/>
<text class="lbl-b2" x="20" y="152">After ― 消す＋並列で44秒</text>
<line class="beam" marker-end="url(#ah2)" x1="124" y1="188" x2="138" y2="188"/>
<line class="beam" marker-end="url(#ah2)" x1="292" y1="188" x2="306" y2="188"/>
<line class="beam" marker-end="url(#ah2)" x1="486" y1="188" x2="500" y2="188"/>
<rect class="node seq" style="animation-delay:0s" x="20" y="166" width="104" height="44" rx="8"/>
<text class="t" x="72" y="193" text-anchor="middle">Setup PHP</text>
<rect class="node seq" style="animation-delay:1.2s" x="142" y="166" width="150" height="44" rx="8"/>
<text class="t" x="217" y="185" text-anchor="middle">pint.phar 取得</text>
<text class="ts" x="217" y="202" text-anchor="middle">0.4秒</text>
<rect class="node seq" style="animation-delay:2.4s" x="310" y="166" width="176" height="44" rx="8"/>
<text class="t" x="398" y="185" text-anchor="middle">Pint 実行（並列）</text>
<text class="ts" x="398" y="202" text-anchor="middle">23秒</text>
<rect class="node seq" style="animation-delay:3.6s" x="504" y="166" width="106" height="44" rx="8"/>
<text class="t b" x="557" y="193" text-anchor="middle">44秒</text>
<rect class="bara growa" x="20" y="224" width="161" height="9"/>
<text class="ts" x="190" y="233">合計 44秒</text>
</svg>
</div>
<style>
.ciflow { --line:#3ea8ff; --lineo:#e08a3c; --node:#eef6ff; --nodeo:#fdf3e7; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .ciflow { --node:#16324a; --nodeo:#3a2a16; --tx:#e8f2fa; }
.ciflow svg { width:100%; height:auto; display:block; }
.ciflow .beam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:ciflowm 1.2s linear infinite; }
.ciflow .beam.o { stroke:var(--lineo); }
.ciflow .ahp { fill:var(--line); }
@keyframes ciflowm { to { stroke-dashoffset:-14; } }
.ciflow .node { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.ciflow .node.nd { fill:var(--nodeo); stroke:var(--lineo); }
.ciflow text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.ciflow .t { font-size:13px; }
.ciflow .t.b { font-weight:700; }
.ciflow .ts { font-size:11.5px; opacity:.75; }
.ciflow .lbl-o { fill:var(--lineo); font-size:14px; font-weight:700; }
.ciflow .lbl-b2 { fill:var(--line); font-size:14px; font-weight:700; }
.ciflow .seq { animation:ciseq 4.8s ease-in-out infinite; }
@keyframes ciseq { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
.ciflow .barb { fill:var(--lineo); opacity:.55; }
.ciflow .bara { fill:var(--line); opacity:.7; }
.ciflow .growb { transform-box:fill-box; transform-origin:left center; animation:cigrowb 4.8s linear infinite; }
.ciflow .growa { transform-box:fill-box; transform-origin:left center; animation:cigrowa 4.8s linear infinite; }
@keyframes cigrowb { 0% { transform:scaleX(0); } 88% { transform:scaleX(1); } 100% { transform:scaleX(1); } }
@keyframes cigrowa { 0% { transform:scaleX(0); } 24% { transform:scaleX(1); } 100% { transform:scaleX(1); } }
@media (prefers-reduced-motion: reduce) { .ciflow * { animation:none !important; } }
</style>
```

*Before は直列で約3分。After は「やらないこと」を決めて44秒*

---

## 効かなかったこと・見送ったこと

速くするためなら、何でも入れればいいわけではありません。今回、あえて手を付けなかった定番対策が3つあります。

| 対策 | 判断 | 理由 |
| --- | --- | --- |
| 依存関係のキャッシュ | 見送り | phar の取得が0.4秒で済み、キャッシュする意味自体が消えた |
| 差分チェック（--diff） | 見送り | 変更ファイルだけでは、既存の違反を見逃す |
| Setup PHP の削除 | 見送り | ランナー側のPHP版が変わると、CIが突然壊れる |

```html-embed
<div class="rejc">
<div class="card" style="animation-delay:0s">
<div class="head"><span class="mark">✗</span><span class="name">依存関係のキャッシュ</span><span class="badge">見送り</span></div>
<p class="reason">phar の取得が0.4秒で済み、キャッシュする意味自体が消えた</p>
</div>
<div class="card" style="animation-delay:.25s">
<div class="head"><span class="mark">✗</span><span class="name">差分チェック（--diff）</span><span class="badge">見送り</span></div>
<p class="reason">変更ファイルだけでは、既存の違反を見逃す</p>
</div>
<div class="card" style="animation-delay:.5s">
<div class="head"><span class="mark">✗</span><span class="name">Setup PHP の削除</span><span class="badge">見送り</span></div>
<p class="reason">ランナー側のPHP版が変わると、CIが突然壊れる</p>
</div>
</div>
<style>
.rejc { --acc:#d9534f; --bg:#fdf0ef; --tx:#1a2733; --bd:#e5b4b1; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .rejc { --bg:#3a1d1b; --tx:#f2e9e8; --bd:#6e3a37; }
.rejc .card { background:var(--bg); border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:10px 14px; animation:rejfade .8s ease both; }
@keyframes rejfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.rejc .head { display:flex; align-items:center; gap:8px; }
.rejc .mark { color:var(--acc); font-weight:700; font-size:15px; }
.rejc .name { color:var(--tx); font-weight:700; font-size:14px; }
.rejc .badge { margin-left:auto; color:var(--acc); border:1px solid var(--acc); border-radius:999px; font-size:11px; padding:1px 8px; white-space:nowrap; }
.rejc .reason { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
@media (prefers-reduced-motion: reduce) { .rejc * { animation:none !important; } }
</style>
```

1つずつ見ます。

キャッシュは、CI高速化の定番です。でも今回は、そもそもインストール自体を消しました。残ったダウンロードは0.4秒です。

0.4秒をキャッシュで削るために、整合性の管理を持ち込むのは割に合いません。キャッシュは速くする手段であって、目的ではないんです。

差分チェックは、PRで変更したファイルだけを対象にする方法です。確かに速くなります。でも、既存ファイルの違反は検出できません。

フォーマットチェックの役割は「全体が揃っていることの保証」です。速さと引き換えに、その役割を手放す判断はしませんでした。

Setup PHP の削除も検討しました。ランナーにはPHPが入っているので、消せば約10秒縮みます。でも、ランナー側のPHPバージョンは予告なく変わります。

setup系のアクションは、外部要因で突然壊れるリスクと隣り合わせです。逆に言えば、バージョンを固定する Setup PHP は、そのリスクへの保険です。12秒は、安全料として払うことにしました。

:::message
「効かなかったこと」として正直に書くと、最初は phar化だけで終わらせるつもりでした。結果は2分34秒。
一手で終わらせず、内訳を見直したことが44秒への分かれ道でした。
:::

---

## 落とし穴2つ：置き場所とキー指定

つまずいた点を2つ残しておきます。

1つ目は、phar の置き場所です。リポジトリのルートにダウンロードすると、実行時の working-directory とパスが合わずに失敗しました。`/tmp/pint.phar` に置くことで解決しています。

2つ目は、jq のキー指定です。composer.lock の `packages-dev` にはハイフンが含まれます。ドット記法では辿れないので、`.["packages-dev"][]` の形で書く必要があります。

```html-embed
<div class="pitc">
<div class="card" style="animation-delay:0s">
<div class="row"><span class="tag ng">落とし穴</span><span class="txt">phar をリポジトリのルートにダウンロード → 実行時の working-directory とパスが合わず失敗</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">解決</span><span class="txt"><code>/tmp/pint.phar</code> に置く</span></div>
</div>
<div class="card" style="animation-delay:.3s">
<div class="row"><span class="tag ng">落とし穴</span><span class="txt">jq をドット記法で書く → <code>packages-dev</code> のハイフンで辿れない</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">解決</span><span class="txt"><code>.["packages-dev"][]</code> の形で書く</span></div>
</div>
</div>
<style>
.pitc { --ng:#e08a3c; --ok:#2f9e5f; --bg:#fdf6ec; --bg2:#eef7f1; --tx:#1a2733; --bd:#e3cfa8; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .pitc { --bg:#3a2c16; --bg2:#173325; --tx:#f2ede4; --bd:#6e5a33; }
.pitc .card { background:var(--bg); border:1px solid var(--bd); border-radius:8px; padding:10px 14px; animation:pitfade .8s ease both; }
@keyframes pitfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.pitc .row { display:flex; align-items:flex-start; gap:8px; }
.pitc .tag { flex:none; border-radius:999px; font-size:11px; font-weight:700; padding:2px 9px; margin-top:1px; white-space:nowrap; }
.pitc .tag.ng { color:var(--ng); border:1px solid var(--ng); }
.pitc .tag.ok { color:var(--ok); border:1px solid var(--ok); }
.pitc .txt { color:var(--tx); font-size:13px; line-height:1.6; }
.pitc .txt code { font-size:12px; background:rgba(127,127,127,.18); border-radius:4px; padding:1px 5px; }
.pitc .arrow { color:var(--ng); font-size:13px; text-align:center; line-height:1.4; }
@media (prefers-reduced-motion: reduce) { .pitc * { animation:none !important; } }
</style>
```

:::details コミット分割の基準
変更は「phar化」と「--parallel 追加」の2コミットに分けました。
基準は、中間状態でCIが壊れないことです。
phar化の途中の状態（composer を消したのに phar がまだ無い）は壊れるので、そこは分割しませんでした。
:::

---

## まとめ

- 改善の前に、ステップ単位で測る。合計時間だけでは判断できない
- 一番効いたのは「速くする」ではなく「やめる」判断。composer install の廃止が最大の削減だった
- 並列化は、読み取り専用の処理から踏み込む。リスクは「壊れるか」「失敗するだけか」で分けて見る
- キャッシュは最後の手段。ダウンロードが0.4秒なら、そもそも要らない
- バージョンの整合性は composer.lock に寄せれば、phar の直接実行でも保てる

CI高速化の進め方を、判断基準として並べるとこうなります。

| 順番 | やること | この記事での例 |
| --- | --- | --- |
| 1 | ステップ単位で測る | 実行ログで内訳を出す |
| 2 | 不要なステップを消す | composer install の廃止 |
| 3 | 重い処理を並列化する | --parallel の採用 |
| 4 | キャッシュは最後に検討する | 今回は不要と判断 |

```html-embed
<div class="stepd">
<div class="step" style="animation-delay:0s"><span class="num">1</span><div class="body"><div class="main">測る</div><div class="sub">ステップ単位の内訳</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:1s"><span class="num">2</span><div class="body"><div class="main">消す</div><div class="sub">不要ステップを消す</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:2s"><span class="num">3</span><div class="body"><div class="main">並列化</div><div class="sub">重い処理を並列化</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:3s"><span class="num">4</span><div class="body"><div class="main">キャッシュは最後</div><div class="sub">今回は不要と判断</div></div></div>
</div>
<style>
.stepd { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; display:flex; align-items:stretch; gap:6px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .stepd { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.stepd .step { flex:1 1 0; min-width:0; display:flex; align-items:flex-start; gap:7px; background:var(--bg); border:1.5px solid var(--bd); border-radius:8px; padding:9px 10px; animation:steppulse 4s ease-in-out infinite; }
@keyframes steppulse { 0%, 100% { border-color:var(--bd); } 8% { border-color:var(--acc); box-shadow:0 0 0 2px var(--acc) inset; } 20% { border-color:var(--bd); box-shadow:none; } }
.stepd .num { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-size:12px; font-weight:700; text-align:center; line-height:20px; margin-top:1px; }
.stepd .main { color:var(--tx); font-size:13px; font-weight:700; line-height:1.4; }
.stepd .sub { color:var(--tx); font-size:11px; line-height:1.45; opacity:.8; }
.stepd .arr { flex:none; align-self:center; color:var(--acc); font-size:14px; font-weight:700; }
@media (max-width:560px) { .stepd { flex-wrap:wrap; } .stepd .step { flex:1 1 42%; } .stepd .arr { display:none; } }
@media (prefers-reduced-motion: reduce) { .stepd * { animation:none !important; } }
</style>
```

次回は、ECSの起動58秒とALBヘルスチェック45秒が競合して、デプロイがロールバックした話を書きます。

