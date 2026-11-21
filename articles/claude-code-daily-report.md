---
title: "日報の下書きを17:00定時で自動化 ― Claude Codeの履歴はJSONLにあり、4つの情報源を集めるだけだった"
emoji: "📝"
type: "tech"
topics: ["claudecode", "automation", "notion", "vscode"]
published: false
---

## はじめに：日報を書くのに、4か所を回っていた

毎日の作業報告を書くのに、散らばった作業を拾い集める手間がかかっていました。チャット、Claude Code、タスク管理。見る場所が4か所に分かれているんです。

1か所ずつ開いて、今日やったことを思い出して、文章にする。この往復が、地味に時間を食います。しかも、拾い忘れが出る。

そこで作ったのが、平日17:00の定時に4つの情報源を集めて、日報の下書きを作る仕組みです。集めるところまでを自動化して、投稿する前には人が確認する。そんな形に落ち着きました。

この記事では、その仕組みを、履歴の保存場所から毎日の習慣まで、原典の記録に沿って書きます。末尾には、そのまま使える導入チェックリストも置きました。

```html-embed
<div class="learnc3">
<div class="ttl">この記事で分かること</div>
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">Claude Codeの履歴の保存場所と読み方</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">日報に集める4つの情報源と取り方</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">17:00定時の下書きの流れ</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">✓</span><span class="txt">devcontainerで履歴が閉じる問題と対処</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">✓</span><span class="txt">全部を自動で拾おうとしない割り切り方</span></div>
<div class="item" style="animation-delay:.75s"><span class="chk">✓</span><span class="txt">そのまま使える導入チェックリスト</span></div>
</div>
<style>
.learnc3 { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .learnc3 { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.learnc3 .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.learnc3 .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:learnfade3 .7s ease both; }
@keyframes learnfade3 { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.learnc3 .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.learnc3 .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .learnc3 * { animation:none !important; } }
</style>
```

---

## まず結論：履歴は端末の中に、JSONLで残っている

Claude Codeの履歴は、claude.aiのチャット検索には含まれません。ここが、最初のつまずきでした。ブラウザのほうを探しても、出てこないんです。

履歴があるのは、作業した端末の中です。場所は `~/.claude/projects/`。1セッションが1つのJSONLファイルになっています。

| 項目 | 内容 |
| --- | --- |
| 保存場所 | `~/.claude/projects/` |
| 形式 | JSONL（1行が1レコード） |
| 単位 | 1セッション＝1ファイル |
| Windows | `%USERPROFILE%\.claude\projects` |

フォルダ名は、作業ディレクトリのパスを変換したものです。パスの中の記号や日本語が `-` に置き換わります。公式ドキュメントの定義は「英数字以外を `-` に置き換える」です。

```html-embed
<div class="histtree">
<div class="ttl">履歴の置き場所 ― 1セッション＝1ファイル</div>
<div class="tree">
<div class="line root">~/.claude/projects/</div>
<div class="line lv1">└─ -home-あなた-作業フォルダ/ <span class="cm">← パスを変換した名前</span></div>
<div class="line lv2">├─ セッションID.jsonl <span class="cm">← 会話の全文</span></div>
<div class="line lv2">└─ セッションID.jsonl <span class="cm">← 別の日の会話</span></div>
</div>
<p class="note">フォルダ名が分からなければ、projects/ の中を一覧すれば見つかる</p>
</div>
<style>
.histtree { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .histtree { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.histtree .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:8px; }
.histtree .tree { font-family:ui-monospace, "SF Mono", Menlo, monospace; font-size:12.5px; line-height:1.7; color:var(--tx); }
.histtree .line.root { font-weight:700; }
.histtree .line.lv1 { padding-left:8px; }
.histtree .line.lv2 { padding-left:24px; }
.histtree .cm { opacity:.65; font-size:11.5px; }
.histtree .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:8px 0 0; }
@media (prefers-reduced-motion: reduce) { .histtree * { animation:none !important; } }
</style>
```

:::message
「claude.aiで検索すれば出てくるはず」という思い込みが、最初の落とし穴でした。Claude Codeの履歴は、端末のローカルファイルです。探す場所を変えるだけで、話は進みます。
:::

再開するときは、ファイルを開き直す必要はありません。`claude --resume` でセッションの一覧から選ぶか、`/resume` をセッションの中で使います。公式ドキュメントでは、セッションIDや名前を指定して直接再開する方法も載っています。

```html-embed
<div class="resumeflow">
<div class="step" style="animation-delay:0s"><span class="num">1</span><div class="body"><div class="main">履歴はJSONLで残る</div><div class="sub">~/.claude/projects/ に保存</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:1s"><span class="num">2</span><div class="body"><div class="main">claude --resume</div><div class="sub">一覧から選んで再開</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:2s"><span class="num">3</span><div class="body"><div class="main">/resume</div><div class="sub">セッション内から切り替え</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:3s"><span class="num">4</span><div class="body"><div class="main">続きから作業</div><div class="sub">会話と状態が戻る</div></div></div>
</div>
<style>
.resumeflow { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; display:flex; align-items:stretch; gap:6px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .resumeflow { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.resumeflow .step { flex:1 1 0; min-width:0; display:flex; align-items:flex-start; gap:7px; background:var(--bg); border:1.5px solid var(--bd); border-radius:8px; padding:9px 10px; animation:resumepulse 4s ease-in-out infinite; }
@keyframes resumepulse { 0%, 100% { border-color:var(--bd); } 8% { border-color:var(--acc); box-shadow:0 0 0 2px var(--acc) inset; } 20% { border-color:var(--bd); box-shadow:none; } }
.resumeflow .num { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-size:12px; font-weight:700; text-align:center; line-height:20px; margin-top:1px; }
.resumeflow .main { color:var(--tx); font-size:13px; font-weight:700; line-height:1.4; }
.resumeflow .sub { color:var(--tx); font-size:11px; line-height:1.45; opacity:.8; }
.resumeflow .arr { flex:none; align-self:center; color:var(--acc); font-size:14px; font-weight:700; }
@media (max-width:560px) { .resumeflow { flex-wrap:wrap; } .resumeflow .step { flex:1 1 42%; } .resumeflow .arr { display:none; } }
@media (prefers-reduced-motion: reduce) { .resumeflow * { animation:none !important; } }
</style>
```

:::details JSONLを直接読むときの注意
公式ドキュメントでは、履歴はJSONLで保存されると明記されています。ただ、1行ごとの中身の形式は内部仕様で、バージョンによって変わる可能性があります。日報の下書きに使うなら、ファイルの中身をざっと読む程度に留めて、形式に依存した作り込みは避けるのが無難です。
:::

---

## 情報源は4つ：取り方を並べると、全体像が見える

日報に集める情報源は4つです。どれか1つだけでは、その日の作業は再現できません。4つを並べて、初めて全体像になります。

| 情報源 | 取り方 |
| --- | --- |
| タスク管理 | コネクタで今日の行を取得 |
| claude.aiのチャット | ブラウザで最近の一覧を読む |
| Claude Code | projectsフォルダを読む |
| GitHub | ブラウザで自分のPR／Issueを見る |

タスク管理は、Notionのガントチャートです。コネクタで、今日の日付に該当する行を取得します。今日やるはずだった作業が、ここに並びます。

claude.aiのチャットは、スケジュール実行では検索が使えません。だからブラウザで最近のチャット一覧を開いて読む、という取り方にしています。

Claude Codeは、接続フォルダとしてprojectsフォルダを読みます。端末の履歴を、そのまま情報源にするわけです。

GitHubは、ブラウザで自分のPRとIssueを見ます。コードの動きが、ここに残っています。

```html-embed
<div class="src4">
<div class="grid">
<div class="card" style="animation-delay:0s"><div class="ico">📋</div><div class="name">タスク管理</div><div class="desc">今日の行を取得</div><div class="how">コネクタ経由</div></div>
<div class="card" style="animation-delay:.2s"><div class="ico">💬</div><div class="name">claude.aiのチャット</div><div class="desc">最近の一覧を読む</div><div class="how">ブラウザ経由</div></div>
<div class="card" style="animation-delay:.4s"><div class="ico">💻</div><div class="name">Claude Code</div><div class="desc">projectsフォルダを読む</div><div class="how">ローカルファイル</div></div>
<div class="card" style="animation-delay:.6s"><div class="ico">🐙</div><div class="name">GitHub</div><div class="desc">自分のPR／Issueを見る</div><div class="how">ブラウザ経由</div></div>
</div>
<p class="note">4つを並べて、初めてその日の全体像になる。1つだけでは足りない</p>
</div>
<style>
.src4 { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .src4 { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.src4 .grid { display:grid; grid-template-columns:1fr 1fr; gap:10px; }
.src4 .card { background:var(--bg); border:1px solid var(--bd); border-top:4px solid var(--acc); border-radius:8px; padding:10px 14px; animation:srcfade .8s ease both; }
@keyframes srcfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.src4 .ico { font-size:18px; line-height:1.2; }
.src4 .name { color:var(--tx); font-weight:700; font-size:14px; margin-top:4px; }
.src4 .desc { color:var(--tx); font-size:12.5px; line-height:1.55; margin-top:2px; }
.src4 .how { color:var(--acc); font-size:11.5px; font-weight:700; margin-top:4px; }
.src4 .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:8px 0 0; text-align:center; }
@media (prefers-reduced-motion: reduce) { .src4 * { animation:none !important; } }
</style>
```

4つのうち、2つがブラウザ経由です。スケジュール実行では過去チャットの検索ツールが使えない、という制約が、その理由です。自動で取れないところは、ブラウザで読む。ここも割り切りの1つです。

---

## 17:00の流れ：定時に集めて、下書きして、人が見る

仕組みの流れは、シンプルです。平日17:00にスケジュールタスクが動いて、4つの情報源を集めます。集めた内容から、定型の日報を下書きして、チャットに投稿します。

| 時刻 | やること |
| --- | --- |
| 17:00 | スケジュールタスクが動く |
| 17:00 | 4つの情報源を集める |
| 17:00 | 定型の日報を下書きする |
| 17:00 | 下書きをチャットに投稿 |

下書きの形は決まっています。本日の作業報告と、次回の作業予定。この2つです。形が決まっているから、集めた情報を当てはめるだけで下書きになります。

投稿は、人が確認してから行います。下書きをそのまま送るのではなく、内容を見てから投稿する。この一手間が、仕組みの最後の砦です。

```html-embed
<div class="flow17">
<svg viewBox="0 0 700 200" role="img" aria-label="17時の日報下書きの流れ">
<defs>
<marker id="ah17" markerWidth="8" markerHeight="8" refX="7" refY="3" orient="auto">
<path d="M0 0 L7 3 L0 6 Z" class="ahp"/>
</marker>
</defs>
<line class="beam" marker-end="url(#ah17)" x1="150" y1="80" x2="178" y2="80"/>
<line class="beam" marker-end="url(#ah17)" x1="332" y1="80" x2="360" y2="80"/>
<line class="beam" marker-end="url(#ah17)" x1="510" y1="80" x2="538" y2="80"/>
<rect class="node seq" style="animation-delay:0s" x="20" y="58" width="130" height="44" rx="8"/>
<text class="t" x="85" y="77" text-anchor="middle">17:00 定時</text>
<text class="ts" x="85" y="94" text-anchor="middle">タスクが動く</text>
<rect class="node seq" style="animation-delay:1.2s" x="182" y="58" width="150" height="44" rx="8"/>
<text class="t" x="257" y="77" text-anchor="middle">4つの情報源を集める</text>
<text class="ts" x="257" y="94" text-anchor="middle">ガント＋履歴＋PR</text>
<rect class="node seq" style="animation-delay:2.4s" x="364" y="58" width="146" height="44" rx="8"/>
<text class="t" x="437" y="77" text-anchor="middle">日報を下書きする</text>
<text class="ts" x="437" y="94" text-anchor="middle">定型の形に当てはめる</text>
<rect class="node seq" style="animation-delay:3.6s" x="542" y="58" width="138" height="44" rx="8"/>
<text class="t" x="611" y="77" text-anchor="middle">人が見て投稿</text>
<text class="ts" x="611" y="94" text-anchor="middle">確認してから送る</text>
<text class="cap" x="350" y="150" text-anchor="middle">集めるのは自動、送るのは手動。この分け方が肝</text>
</svg>
</div>
<style>
.flow17 { --line:#e07a2e; --node:#fdf3e7; --tx:#1a2733; padding:12px 4px; }
[data-theme="dark"] .flow17 { --node:#3a2a16; --tx:#f2ede4; }
.flow17 svg { width:100%; height:auto; display:block; }
.flow17 .beam { fill:none; stroke:var(--line); stroke-width:2; stroke-dasharray:7 7; animation:flow17m 1.2s linear infinite; }
.flow17 .ahp { fill:var(--line); }
@keyframes flow17m { to { stroke-dashoffset:-14; } }
.flow17 .node { fill:var(--node); stroke:var(--line); stroke-width:1.5; }
.flow17 text { fill:var(--tx); font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
.flow17 .t { font-size:13px; font-weight:700; }
.flow17 .ts { font-size:11px; opacity:.75; }
.flow17 .seq { animation:flow17seq 4.8s ease-in-out infinite; }
@keyframes flow17seq { 0%, 100% { stroke-width:1.5; } 10% { stroke-width:4; } 22% { stroke-width:1.5; } }
.flow17 .cap { font-size:13px; font-weight:700; fill:var(--line); }
@media (prefers-reduced-motion: reduce) { .flow17 * { animation:none !important; } }
</style>
```

毎日の習慣も、仕組みの一部です。17:00にPCとブラウザを開いておく。コンテナ内でやった作業は要約を貼る。下書きを確認してから投稿する。この3つが回って、初めて仕組みが回ります。

```html-embed
<div class="habit3">
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">17:00にPCとブラウザを開いておく</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">コンテナ内の作業は要約を貼る</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">下書きを確認してから投稿する</span></div>
</div>
<style>
.habit3 { --acc:#2f9e5f; --bg:#eef7f1; --tx:#1a2733; --bd:#a9d4ba; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .habit3 { --bg:#173325; --tx:#e8f2fa; --bd:#2c5b40; }
.habit3 .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:habitfade .7s ease both; }
@keyframes habitfade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.habit3 .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.habit3 .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .habit3 * { animation:none !important; } }
</style>
```

:::message
自動化の範囲は「下書きまで」です。投稿は人の手に残す。この線引きがあるから、下書きの間違いがそのまま外に出ません。自動化するほど、最後の人の目が大事になる、という話でもあります。
:::

---

## devcontainerの壁：履歴がコンテナの中に閉じる

ここで、1つ大きな壁があります。devcontainerの中でClaude Codeを使っていると、履歴はコンテナの中に閉じてしまうんです。ホスト側の `~/.claude/projects/` からは、見えません。

原因はシンプルです。Claude Codeが動いている場所が、コンテナの中だからです。履歴も、同じ場所に書かれます。ホストから読もうとしても、そもそもファイルがそこにありません。

| 場所 | 履歴は見えるか |
| --- | --- |
| ホストで実行 | 見える |
| コンテナ内で実行 | ホストからは見えない |

対処は、bind mountです。`devcontainer.json` に `mounts` を足して、ホストの履歴フォルダをコンテナの中に見せます。VS Codeの公式ドキュメントでは、`mounts` に `source=ホスト側のパス,target=コンテナ側のパス,type=bind` の形で書く方法が載っています。

```json
"mounts": [
  "source=${localEnv:HOME}/.claude/projects,target=/home/コンテナのユーザー/.claude/projects,type=bind"
]
```

targetは、コンテナのユーザーに合わせるのが原典の注意点です。ユーザー名が違えば、パスも変わります。ここを間違えると、mountしても履歴がつながりません。

```html-embed
<div class="dcmorph">
<div class="panel before" style="animation-delay:0s">
<div class="ttl">mountなし ― 履歴がコンテナに閉じる</div>
<div class="bar"><span class="seg bad">ホスト：履歴が見えない</span></div>
<div class="bar"><span class="seg bad">コンテナ：履歴は中にだけある</span></div>
<div class="total">日報の情報源が1つ欠ける</div>
</div>
<div class="panel after" style="animation-delay:.4s">
<div class="ttl">mountあり ― 履歴をホストと共有する</div>
<div class="bar"><span class="seg ok">ホスト：projectsフォルダが見える</span></div>
<div class="bar"><span class="seg ok">コンテナ：同じ場所に履歴を書く</span></div>
<div class="total">日報の情報源がそろう</div>
</div>
<p class="note">やるのは設定1つ。履歴の場所を変えるのではなく、見せる場所を増やす</p>
</div>
<style>
.dcmorph { --bad:#d9534f; --ok:#2f9e5f; --tx:#1a2733; --bgb:#fdf0ef; --bga:#eef7f1; --bdb:#e5b4b1; --bda:#a9d4ba; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .dcmorph { --bgb:#3a1d1b; --bga:#173325; --tx:#f2ede4; --bdb:#6e3a37; --bda:#2c5b40; }
.dcmorph .panel { border-radius:8px; padding:10px 14px; animation:dcmorphfade 1s ease both; }
@keyframes dcmorphfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.dcmorph .panel.before { background:var(--bgb); border:1px solid var(--bdb); border-top:4px solid var(--bad); }
.dcmorph .panel.after { background:var(--bga); border:1px solid var(--bda); border-top:4px solid var(--ok); }
.dcmorph .ttl { color:var(--tx); font-weight:700; font-size:13.5px; margin-bottom:8px; }
.dcmorph .bar { margin:4px 0; }
.dcmorph .seg { display:inline-block; color:#fff; font-size:12px; font-weight:700; border-radius:4px; padding:3px 8px; line-height:1.5; }
.dcmorph .seg.bad { background:var(--bad); }
.dcmorph .seg.ok { background:var(--ok); }
.dcmorph .total { color:var(--tx); font-size:13px; font-weight:700; margin-top:6px; }
.dcmorph .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:4px 0 0; text-align:center; }
@media (prefers-reduced-motion: reduce) { .dcmorph * { animation:none !important; } }
</style>
```

反映にはRebuildが必要です。VS Codeの公式ドキュメントでも、mountの変更後は「Dev Containers: Rebuild Container」で反映させると書かれています。それまでの履歴は、Rebuildの前に先にコピーしておきます。Rebuildでコンテナが作り直されると、中の履歴は消えてしまうからです。

:::details Rebuildの前にやること
Rebuildをする前に、コンテナ内の履歴をホストへコピーしておくのが原典の手順です。mountを設定してからRebuildすれば、それ以降の履歴はホスト側に残ります。順番を間違えると、それまでの履歴が消えるので注意が必要です。
:::

---

## 貼る習慣：拾えない分は、要約で補う

mountを設定しても、拾えない作業は残ります。コンテナ内でやった作業のうち、mountの前に終わった分は、ホストの履歴にはありません。そういう分は、要約を「作業ログ」チャットに貼る運用で補います。

| 拾い方 | 対象 |
| --- | --- |
| 自動で読む | ホストの履歴・ガント・PR |
| 貼って補う | コンテナ内の作業の要約 |
| ブラウザで読む | claude.aiの最近のチャット |

貼るのは、作業の要約です。全文を貼るのではなく、「何をやったか」が分かる要約にします。要約なら、貼る手間も小さいし、読むほうも負担になりません。

この運用のいいところは、設定を増やさずに済むことです。拾えない分を全部自動化しようとすると、設定がどんどん重くなります。貼る習慣で補えば、仕組みは軽いまま保てます。

```html-embed
<div class="pasteflow">
<div class="step" style="animation-delay:0s"><span class="num">1</span><div class="body"><div class="main">コンテナで作業する</div><div class="sub">履歴はコンテナの中</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:1s"><span class="num">2</span><div class="body"><div class="main">要約を作る</div><div class="sub">何をやったか、短く</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:2s"><span class="num">3</span><div class="body"><div class="main">作業ログに貼る</div><div class="sub">チャットに残す</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:3s"><span class="num">4</span><div class="body"><div class="main">17:00に集める</div><div class="sub">他の情報源と一緒に</div></div></div>
</div>
<style>
.pasteflow { --acc:#2f9e5f; --bg:#eef7f1; --tx:#1a2733; --bd:#a9d4ba; display:flex; align-items:stretch; gap:6px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .pasteflow { --bg:#173325; --tx:#e8f2fa; --bd:#2c5b40; }
.pasteflow .step { flex:1 1 0; min-width:0; display:flex; align-items:flex-start; gap:7px; background:var(--bg); border:1.5px solid var(--bd); border-radius:8px; padding:9px 10px; animation:pastepulse 4s ease-in-out infinite; }
@keyframes pastepulse { 0%, 100% { border-color:var(--bd); } 8% { border-color:var(--acc); box-shadow:0 0 0 2px var(--acc) inset; } 20% { border-color:var(--bd); box-shadow:none; } }
.pasteflow .num { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-size:12px; font-weight:700; text-align:center; line-height:20px; margin-top:1px; }
.pasteflow .main { color:var(--tx); font-size:13px; font-weight:700; line-height:1.4; }
.pasteflow .sub { color:var(--tx); font-size:11px; line-height:1.45; opacity:.8; }
.pasteflow .arr { flex:none; align-self:center; color:var(--acc); font-size:14px; font-weight:700; }
@media (max-width:560px) { .pasteflow { flex-wrap:wrap; } .pasteflow .step { flex:1 1 42%; } .pasteflow .arr { display:none; } }
@media (prefers-reduced-motion: reduce) { .pasteflow * { animation:none !important; } }
</style>
```

---

## 下書きの形は2つだけ：本日の報告と、次回の予定

下書きの形は、決まっています。本日の作業報告と、次回の作業予定。この2つだけです。形が決まっているから、集めた情報を当てはめるだけで下書きになります。

| 項目 | 書くこと |
| --- | --- |
| 本日の作業報告 | 今日やったこと |
| 次回の作業予定 | 次にやること |

この2つに絞るのには、理由があります。日報の目的は、その日の作業を伝えることです。感想や背景を広げ始めると、下書きを作る側も、読む側も重くなります。

形が固定されていると、17:00のタスクも迷いません。4つの情報源から拾った内容を、報告と予定に振り分けるだけです。振り分けに迷う内容があれば、それは下書きの外に置くか、人が確認するときに足します。

```html-embed
<div class="tpl2">
<div class="cols">
<div class="card" style="animation-delay:0s">
<div class="ttl">本日の作業報告</div>
<p class="desc">今日やったことを、集めた情報源から並べる</p>
<div class="src">ガント＋履歴＋PRから</div>
</div>
<div class="card" style="animation-delay:.25s">
<div class="ttl">次回の作業予定</div>
<p class="desc">次にやることを、今日の続きから書く</p>
<div class="src">ガントの明日の行から</div>
</div>
</div>
<p class="note">形は2つだけ。迷わない形が、毎日回るコツ</p>
</div>
<style>
.tpl2 { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .tpl2 { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.tpl2 .cols { display:flex; gap:10px; }
.tpl2 .card { flex:1 1 0; min-width:0; background:var(--bg); border:1px solid var(--bd); border-top:4px solid var(--acc); border-radius:8px; padding:10px 14px; animation:tplfade .8s ease both; }
@keyframes tplfade { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.tpl2 .ttl { color:var(--tx); font-weight:700; font-size:14px; }
.tpl2 .desc { color:var(--tx); font-size:13px; margin:6px 0 0; line-height:1.6; }
.tpl2 .src { color:var(--acc); font-size:12px; font-weight:700; margin-top:6px; }
.tpl2 .note { color:var(--tx); font-size:12px; line-height:1.55; opacity:.8; margin:8px 0 0; text-align:center; }
@media (max-width:560px) { .tpl2 .cols { flex-direction:column; } }
@media (prefers-reduced-motion: reduce) { .tpl2 * { animation:none !important; } }
</style>
```

---

## 効かなかったこと：全部を自動で拾おうとすると、設定が重くなる

この仕組みを作る中で、いちばん効かなかった考え方は「全部を自動で拾う」ことです。最初は、すべての情報源を自動で集めたかったんです。でも、やればやるほど設定が重くなりました。

| 考え方 | 結果 |
| --- | --- |
| 全部を自動で拾う | 設定が重くなる |
| 拾えない分は貼る | 仕組みが軽いまま |
| 最後は人が見る | 間違いが外に出ない |

スケジュール実行では、過去チャットの検索ツールが使えません。ここを無理に自動化しようとすると、別の仕組みを足すことになります。足すたびに、壊れる場所が増えるんです。

だから、拾えないところは「貼る」習慣で補うことにしました。設定を重くするより、人の習慣に寄せたほうが、長く回ります。自動化の目的は「手間をゼロにすること」ではなく、「手間を軽くすること」です。

:::message
全部を自動化したくなる気持ちは分かります。でも、拾えない分を無理に拾おうとすると、仕組み自体が重くなって、結局回らなくなります。軽く保つための割り切りが、この仕組みの核心でした。
:::

---

## 見落としやすい勘違い3つ

今回の仕組みで、実際に引っかかった思い込みを3つ残しておきます。

```html-embed
<div class="pitc3">
<div class="card" style="animation-delay:0s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">履歴はclaude.aiで検索できる</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">履歴は端末のJSONL。ブラウザの検索には出てこない</span></div>
</div>
<div class="card" style="animation-delay:.3s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">コンテナの中の履歴も、ホストから見える</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">履歴はコンテナに閉じる。mountで見せる必要がある</span></div>
</div>
<div class="card" style="animation-delay:.6s">
<div class="row"><span class="tag ng">勘違い</span><span class="txt">下書きができたら、そのまま投稿してよい</span></div>
<div class="arrow">↓</div>
<div class="row"><span class="tag ok">実際</span><span class="txt">下書きは人が確認してから投稿する。最後は人の目</span></div>
</div>
</div>
<style>
.pitc3 { --ng:#e08a3c; --ok:#2f9e5f; --bg:#fdf6ec; --tx:#1a2733; --bd:#e3cfa8; display:flex; flex-direction:column; gap:10px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .pitc3 { --bg:#3a2c16; --tx:#f2ede4; --bd:#6e5a33; }
.pitc3 .card { background:var(--bg); border:1px solid var(--bd); border-radius:8px; padding:10px 14px; animation:pitfade3 .8s ease both; }
@keyframes pitfade3 { from { opacity:0; transform:translateY(8px); } to { opacity:1; transform:translateY(0); } }
.pitc3 .row { display:flex; align-items:flex-start; gap:8px; }
.pitc3 .tag { flex:none; border-radius:999px; font-size:11px; font-weight:700; padding:2px 9px; margin-top:1px; white-space:nowrap; }
.pitc3 .tag.ng { color:var(--ng); border:1px solid var(--ng); }
.pitc3 .tag.ok { color:var(--ok); border:1px solid var(--ok); }
.pitc3 .txt { color:var(--tx); font-size:13px; line-height:1.6; }
.pitc3 .arrow { color:var(--ng); font-size:13px; text-align:center; line-height:1.4; }
@media (prefers-reduced-motion: reduce) { .pitc3 * { animation:none !important; } }
</style>
```

もう1つ、公式ドキュメントで確認した注意点があります。履歴は既定で30日で削除されます。`cleanupPeriodDays` を `settings.json` で変えれば、期間を延ばせます。

| 項目 | 内容 |
| --- | --- |
| 既定の保存期間 | 30日 |
| 変える場所 | `settings.json` の `cleanupPeriodDays` |

日報の材料として古い履歴を残したいなら、ここは先に確認しておくのがおすすめです。30日を過ぎると、履歴は消えてしまいます。

---

## まとめ

- 履歴はclaude.aiではなく、端末の `~/.claude/projects/` にJSONLで残る
- 日報の情報源は4つ。タスク管理・チャット・Claude Code・GitHubを並べる
- 17:00定時に集めて下書きし、投稿は人が確認してから行う
- devcontainerの履歴はmountでホストに見せる。反映にはRebuildが必要
- 拾えない分は要約を貼る習慣で補う。全部の自動化は設定を重くするだけ
- 履歴は既定30日で消える。残したいなら `cleanupPeriodDays` を確認する

判断の順番を、基準として並べるとこうなります。

| 順番 | やること | この記事での例 |
| --- | --- | --- |
| 1 | 履歴の場所を確認する | `~/.claude/projects/` |
| 2 | 情報源を4つ並べる | 取り方を表にする |
| 3 | 定時に集めて下書きする | 平日17:00 |
| 4 | 拾えない分は貼る | 作業ログに要約を貼る |
| 5 | 最後は人が見る | 確認してから投稿 |

```html-embed
<div class="judge3">
<div class="step" style="animation-delay:0s"><span class="num">1</span><div class="body"><div class="main">場所を確認</div><div class="sub">~/.claude/projects/</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:1s"><span class="num">2</span><div class="body"><div class="main">情報源を並べる</div><div class="sub">4つの取り方</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:2s"><span class="num">3</span><div class="body"><div class="main">定時に集める</div><div class="sub">平日17:00</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:3s"><span class="num">4</span><div class="body"><div class="main">貼って補う</div><div class="sub">要約を作業ログへ</div></div></div>
<div class="arr">→</div>
<div class="step" style="animation-delay:4s"><span class="num">5</span><div class="body"><div class="main">人が見て投稿</div><div class="sub">確認してから送る</div></div></div>
</div>
<style>
.judge3 { --acc:#e07a2e; --bg:#fdf3e7; --tx:#1a2733; --bd:#ecd3ac; display:flex; align-items:stretch; gap:6px; padding:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .judge3 { --bg:#3a2a16; --tx:#f2ede4; --bd:#6e5a33; }
.judge3 .step { flex:1 1 0; min-width:0; display:flex; align-items:flex-start; gap:7px; background:var(--bg); border:1.5px solid var(--bd); border-radius:8px; padding:9px 10px; animation:judge3pulse 5s ease-in-out infinite; }
@keyframes judge3pulse { 0%, 100% { border-color:var(--bd); } 8% { border-color:var(--acc); box-shadow:0 0 0 2px var(--acc) inset; } 20% { border-color:var(--bd); box-shadow:none; } }
.judge3 .num { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-size:12px; font-weight:700; text-align:center; line-height:20px; margin-top:1px; }
.judge3 .main { color:var(--tx); font-size:13px; font-weight:700; line-height:1.4; }
.judge3 .sub { color:var(--tx); font-size:11px; line-height:1.45; opacity:.8; }
.judge3 .arr { flex:none; align-self:center; color:var(--acc); font-size:14px; font-weight:700; }
@media (max-width:560px) { .judge3 { flex-wrap:wrap; } .judge3 .step { flex:1 1 42%; } .judge3 .arr { display:none; } }
@media (prefers-reduced-motion: reduce) { .judge3 * { animation:none !important; } }
</style>
```

### そのまま使える実用物：導入チェックリスト

下のチェックリストは、この記事の内容をそのまま使える形にしたものです。導入するときに、上から順に確認してください。ブックマークして、設定するときに開いてもらえると嬉しいです。

```html-embed
<div class="depcheck3">
<div class="ttl">日報の下書き自動化 ― 導入チェックリスト</div>
<div class="item" style="animation-delay:0s"><span class="chk">☐</span><span class="txt">~/.claude/projects/ に履歴があるか確認した</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">☐</span><span class="txt">4つの情報源と取り方を決めた</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">☐</span><span class="txt">平日17:00のスケジュールタスクを作った</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">☐</span><span class="txt">日報の定型（本日の作業報告／次回の作業予定）を決めた</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">☐</span><span class="txt">devcontainer利用時はmountとRebuildを確認した</span></div>
<div class="item" style="animation-delay:.75s"><span class="chk">☐</span><span class="txt">コンテナ内の作業は要約を貼る習慣を決めた</span></div>
<div class="item" style="animation-delay:.9s"><span class="chk">☐</span><span class="txt">下書きを確認してから投稿する流れを決めた</span></div>
<div class="item" style="animation-delay:1.05s"><span class="chk">☐</span><span class="txt">cleanupPeriodDays（既定30日）を確認した</span></div>
</div>
<style>
.depcheck3 { --acc:#2f9e5f; --bg:#eef7f1; --tx:#1a2733; --bd:#a9d4ba; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .depcheck3 { --bg:#173325; --tx:#e8f2fa; --bd:#2c5b40; }
.depcheck3 .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.depcheck3 .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:dep3fade .7s ease both; }
@keyframes dep3fade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.depcheck3 .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.depcheck3 .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .depcheck3 * { animation:none !important; } }
</style>
```

- [ ] `~/.claude/projects/` に履歴があるか確認した
- [ ] 4つの情報源と取り方を決めた
- [ ] 平日17:00のスケジュールタスクを作った
- [ ] 日報の定型（本日の作業報告／次回の作業予定）を決めた
- [ ] devcontainer利用時はmountとRebuildを確認した
- [ ] コンテナ内の作業は要約を貼る習慣を決めた
- [ ] 下書きを確認してから投稿する流れを決めた
- [ ] `cleanupPeriodDays`（既定30日）を確認した

次回は、アプリ出身がインフラ未経験からAWS CDKに着手した話です。
