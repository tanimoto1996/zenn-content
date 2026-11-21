---
title: "ローカルLLMの速度は「賢さ」ではなくメモリ帯域で決まる ― DGX Sparkの実測で決めた、Claudeとの使い分け"
emoji: "🧠"
type: "tech"
topics: ["llm", "ollama", "rag", "nvidia", "claude"]
published: false
---

## 結論：使い分けの物差しは「帯域」と「データの所在」

ローカルLLMとクラウドLLMの使い分けで、いちばん迷いました。
モデル名や評判ではなく、物理の数字で線を引くことにしました。

結論の地図を先に置きます。

| 判断軸 | ローカル（DGX Spark） | クラウド（Claude） |
| --- | --- | --- |
| データの所在 | 外に出さずに完結できる | 外部に送る前提になる |
| 速度の上限 | 帯域273GB/sで頭打ち | 大型モデルでも速い |
| 効く仕事 | 短い回答の文書QA・音声処理 | 深い推論・長い生成 |
| 限界 | 大型モデルは物理的に遅い | 機密データは置けない |

```html-embed
<div class="c2learn">
<div class="ttl">この記事で分かること</div>
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">生成速度の上限を決める「帯域律速」の式</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">モデル規模ごとの実測 tok/s と判定</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">文書QAを17秒→4.3秒にした変更の内訳</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">✓</span><span class="txt">LLMが表の数字を捏造する問題と、その潰し方</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">✓</span><span class="txt">ローカルで足りる条件・足りない条件</span></div>
</div>
<style>
.c2learn { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .c2learn { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.c2learn .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.c2learn .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:c2fade .7s ease both; }
@keyframes c2fade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.c2learn .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.c2learn .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .c2learn * { animation:none !important; } }
</style>
```

:::message
実測値はすべて手元のDGX Sparkでの数字です。
クラウド側の速度は回線やプランで変わるため、物差しにはしていません。
:::

---

## 帯域律速：生成速度には物理の天井がある

ローカルLLMのボトルネックは、モデルの賢さではありません。
**メモリ帯域幅** です。

生成（Decode）は、1トークンごとにモデルの全重みをメモリから読み直す構造をしています。
だから生成速度の上限は、だいたい次の式で決まります。

**生成速度の上限 ≒ メモリ帯域幅 ÷ モデルサイズ**

```html-embed
<div class="c2bw">
<div class="cycle">
<div class="ph" style="animation-delay:0s">重みを読む</div>
<div class="ph" style="animation-delay:1.1s">1トークン生成</div>
<div class="ph" style="animation-delay:2.2s">また重みを読む</div>
<div class="ph" style="animation-delay:3.3s">1トークン生成</div>
</div>
<div class="cap">計算能力（TFLOPS）を上げても、帯域以上には速くならない</div>
</div>
<style>
.c2bw { --tx:#1a2733; --bd:#bcd9f5; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c2bw { --tx:#e8f2fa; background:#101c28; --bd:#33506b; }
.c2bw .cycle { display:flex; gap:6px; flex-wrap:wrap; }
.c2bw .ph { flex:1; min-width:100px; text-align:center; font-size:12px; font-weight:700; border:1.5px solid var(--bd); border-radius:20px; padding:6px 4px; background:#eef6ff; animation:c2ph 4.4s ease-in-out infinite; }
[data-theme="dark"] .c2bw .ph { background:#16324a; }
.c2bw .cap { font-size:12.5px; margin-top:10px; text-align:center; font-weight:700; }
@keyframes c2ph { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 10%,20% { box-shadow:0 0 0 3px rgba(62,168,255,.5); } 32% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c2bw * { animation:none !important; } }
</style>
```

DGX Sparkの帯域は273GB/sです。
この数字が、載せられるモデルの実質的な上限を決めます。

逆に、入力処理（Prefill）は速いマシンです。
「長い入力 → 短い回答」の形は、このマシンの得意分野になります。

---

## 実測：モデル規模ごとの生成速度

式だけでは机上の空論です。
実際にモデル規模を変えて測りました。

```html-embed
<div class="c2tps">
<div class="row"><span class="rl">小型MoE（4B級）</span><div class="bar b1"></div><span class="rv">約64 tok/s</span></div>
<div class="row"><span class="rl">30B級</span><div class="bar b2"></div><span class="rv">10〜18 tok/s</span></div>
<div class="row"><span class="rl">70〜120B級</span><div class="bar b3"></div><span class="rv">2.7〜35 tok/s</span></div>
<div class="note">帯域273GB/sの天井どおり、規模が上がるほど頭打ちになる</div>
</div>
<style>
.c2tps { --tx:#1a2733; --bd:#bcd9f5; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c2tps { --tx:#e8f2fa; background:#101c28; --bd:#33506b; }
.c2tps .row { display:flex; align-items:center; gap:8px; margin:6px 0; }
.c2tps .rl { flex:none; width:108px; font-size:12px; font-weight:700; }
.c2tps .bar { height:14px; border-radius:7px; background:linear-gradient(90deg,#3ea8ff,#8ec9ff,#3ea8ff); background-size:200% 100%; animation:c2shimmer 2.4s linear infinite; }
.c2tps .b1 { width:52%; } .c2tps .b2 { width:20%; } .c2tps .b3 { width:26%; }
.c2tps .rv { font-size:12.5px; font-weight:700; }
.c2tps .note { font-size:12.5px; margin-top:8px; }
@keyframes c2shimmer { to { background-position:-200% 0; } }
@media (prefers-reduced-motion: reduce) { .c2tps * { animation:none !important; } }
</style>
```

| モデル規模 | 実測 | 判定 |
| --- | --- | --- |
| 小型MoE（4B級） | 約64 tok/s | 対話・短文なら成立 |
| 30B級 | 10〜18 tok/s | 不足 |
| 70〜120B級 | 2.7〜35 tok/s | 大幅に不足 |

小型MoEは全体でも60 tok/s前後でした。
対話や短文応答なら成立するラインです。

一方で30B級を超えると、待ち時間が支配的になります。
賢さを取りに行くと、速度が物理で負ける構図です。

---

## 得意・不得意は「入力と出力の比率」で決まる

もう1つ、実測で見えた傾向があります。
Prefill（入力処理）とDecode（生成）の得意・不得意です。

```html-embed
<div class="c2pd">
<div class="col ok">
<div class="ct">Spark が得意な形</div>
<div class="flow"><span class="fb">長い入力</span><span class="fa">→</span><span class="fb">短い回答</span></div>
<div class="cb">Prefill 寄りの処理。文書QA（RAG）型がこれ</div>
</div>
<div class="col ng">
<div class="ct">Spark が苦手な形</div>
<div class="flow"><span class="fb">短い入力</span><span class="fa">→</span><span class="fb">長い生成</span></div>
<div class="cb">Decode 寄りの処理。帯域の天井に直撃する</div>
</div>
</div>
<style>
.c2pd { --tx:#1a2733; --bd:#bcd9f5; display:flex; gap:8px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2pd { --tx:#e8f2fa; --bd:#33506b; }
.c2pd .col { flex:1; border:1.5px solid var(--bd); border-radius:8px; padding:10px 12px; color:var(--tx); }
.c2pd .ok { background:#f2fbf4; } .c2pd .ng { background:#fff6f4; }
[data-theme="dark"] .c2pd .ok { background:#14281a; } [data-theme="dark"] .c2pd .ng { background:#2a1a16; }
.c2pd .ct { font-weight:700; font-size:12.5px; margin-bottom:6px; }
.c2pd .flow { display:flex; align-items:center; gap:6px; margin-bottom:6px; }
.c2pd .fb { font-size:12px; border:1px solid var(--bd); border-radius:6px; padding:4px 8px; background:#fff; }
[data-theme="dark"] .c2pd .fb { background:#101c28; }
.c2pd .fa { color:#3ea8ff; font-weight:700; animation:c2flow 1.4s linear infinite; }
.c2pd .cb { font-size:12px; line-height:1.6; }
@keyframes c2flow { 0%,100% { opacity:.35; } 50% { opacity:1; } }
@media (prefers-reduced-motion: reduce) { .c2pd * { animation:none !important; } }
</style>
```

要件から逆算する考え方も置いておきます。
たとえば「回答表示まで3秒、そのうち音声認識が1〜2秒を使う」構成だと、LLMの持ち時間は1〜2秒です。

150トークンを出し切るには75〜150 tok/sが要ります。
ストリーミング表示で体感を緩めても、50 tok/s程度は必要です。

この逆算に通るのは、Spark上では小型モデルだけでした。
「**応答要件を満たせる最小モデルで、品質が耐えるか**」が、検証の焦点はこの1点に集約されます。

---

## 文書QAの実測：17秒 → 4.3秒、誤答2 → 0

モデルの話だけでは実践になりません。
文書QA（RAG）の構成を詰めた実測がこちらです。

受け入れテスト12問で、構成変更の前後を比べました。

```html-embed
<div class="c2qa">
<div class="panel before">
<div class="pt">初期構成（デフォルト寄り）</div>
<div class="pr">正答 7 / 誤答 2</div>
<div class="pr">平均応答 約17秒</div>
</div>
<div class="panel after">
<div class="pt">最終構成（専用カスタムモデル）</div>
<div class="pr good">正答 11（+部分正解1）/ 誤答 0</div>
<div class="pr good">平均応答 4.3秒</div>
</div>
</div>
<style>
.c2qa { --tx:#1a2733; --bd:#bcd9f5; position:relative; margin:12px 4px; height:112px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2qa { --tx:#e8f2fa; }
.c2qa .panel { position:absolute; inset:0; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; color:var(--tx); background:#fff8f6; }
[data-theme="dark"] .c2qa .panel { background:#2a1a16; }
.c2qa .after { background:#f2fbf4; animation:c2swap 8s ease-in-out infinite; }
[data-theme="dark"] .c2qa .after { background:#14281a; }
.c2qa .pt { font-weight:700; font-size:13.5px; margin-bottom:4px; }
.c2qa .pr { font-size:12.5px; line-height:1.6; }
.c2qa .good { color:#1d9e57; font-weight:700; }
@keyframes c2swap { 0%,42% { opacity:0; } 54%,92% { opacity:1; } 100% { opacity:0; } }
@media (prefers-reduced-motion: reduce) { .c2qa * { animation:none !important; } .c2qa .after { opacity:0; } }
</style>
```

変更の内訳はこの通りです。
派手な変更は1つもありません。

| 変更点 | 内容 |
| --- | --- |
| 埋め込みモデル | デフォルト → bge-m3 |
| 検索クエリ生成 | オフ |
| temperature | 0に固定 |
| think | "low" に抑制 |
| Top K | 5 → 8 |
| チャンク | 800文字 / 重なり100文字 |
| background_tasks | 全オフ |
| システムプロンプト | 抜粋のみ・根拠明記・不明は「見つかりません」 |

```html-embed
<div class="c2rag">
<div class="st" style="animation-delay:0s">文書をチャンク分割</div>
<div class="st" style="animation-delay:1s">bge-m3 でベクトル化</div>
<div class="st" style="animation-delay:2s">類似検索（Top K = 8）</div>
<div class="st" style="animation-delay:3s">根拠つきで回答生成</div>
</div>
<style>
.c2rag { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2rag { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c2rag .st { border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; font-weight:700; animation:c2pulse 4s ease-in-out infinite; }
@keyframes c2pulse { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 10%,22% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 34% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c2rag * { animation:none !important; } }
</style>
```

埋め込みの差し替えは、日本語対策です。
Ollamaのデフォルト埋め込みモデルは日本語に弱く、検索の入口で落としていました。

なお、埋め込みモデルを変えたら既存文書は再インデックスが必須です。
ここを忘れると、静かに検索が壊れたままになります。

---

## Top Kは「勘」ではなく検索を叩いて決めた

変更の中で、個人的にいちばん効いたと思うのがTop Kです。
正解チャンクが検索順位の6位に沈んでいたからです。

K=5のままでは、正解が永遠にプロンプトへ届きません。
検索APIを直接叩いて順位を確認し、K=8に決めました。

```html-embed
<div class="c2topk">
<div class="rank"><span class="rn">1〜5位</span><div class="rb in"></div><span class="rt">K=5 でも届く範囲</span></div>
<div class="rank hit"><span class="rn">6位</span><div class="rb hit"></div><span class="rt">正解チャンクはここ</span></div>
<div class="rank"><span class="rn">7〜8位</span><div class="rb in"></div><span class="rt">K=8 で拾える範囲</span></div>
<div class="note">大きくしすぎるとプロンプトが伸びて遅くなり、無関係なチャンクに釣られる</div>
</div>
<style>
.c2topk { --tx:#1a2733; --bd:#bcd9f5; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c2topk { --tx:#e8f2fa; background:#101c28; --bd:#33506b; }
.c2topk .rank { display:flex; align-items:center; gap:8px; margin:6px 0; }
.c2topk .rn { flex:none; width:52px; font-size:12px; font-weight:700; }
.c2topk .rb { height:14px; border-radius:7px; background:#bcd9f5; }
.c2topk .rb.in { width:44%; }
.c2topk .rb.hit { width:44%; background:linear-gradient(90deg,#ffb020,#ffd97a,#ffb020); background-size:200% 100%; animation:c2shimmer2 2.4s linear infinite; }
.c2topk .rt { font-size:12px; }
.c2topk .note { font-size:12px; margin-top:8px; }
@keyframes c2shimmer2 { to { background-position:-200% 0; } }
@media (prefers-reduced-motion: reduce) { .c2topk * { animation:none !important; } }
</style>
```

「パラメータは勘で決めず、1つ下の層を直接叩いて決める」。
RAGの調整は、この姿勢がほぼ全部だと思います。

---

## 隠れたLLM呼び出しが、回答の後に走っていた

速度の内訳を追って、もう1つ見つけました。
Open WebUIでは、回答の後にタイトル・タグ・フォローアップ生成のLLM呼び出しが裏で走ります。

`background_tasks` をオフにしないと、1回答ごとに+4〜10秒が乗ります。
ユーザーから見ると「答え終わったのに、まだ何かしている」時間です。

```html-embed
<div class="c2bg">
<div class="lane"><span class="lt">オンのまま</span><div class="seg ans">回答</div><div class="seg hid">タイトル生成</div><div class="seg hid">タグ生成</div><div class="seg hid">フォローアップ</div><span class="sum">+4〜10秒</span></div>
<div class="lane"><span class="lt">全オフ</span><div class="seg ans">回答</div><span class="sum">ここで終わる</span></div>
</div>
<style>
.c2bg { --tx:#1a2733; --bd:#bcd9f5; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); }
[data-theme="dark"] .c2bg { --tx:#e8f2fa; --bd:#33506b; }
.c2bg .lane { display:flex; align-items:center; gap:5px; margin:8px 0; flex-wrap:wrap; }
.c2bg .lt { flex:none; width:64px; font-size:12px; font-weight:700; }
.c2bg .seg { font-size:11.5px; border:1.5px solid var(--bd); border-radius:6px; padding:5px 8px; }
.c2bg .ans { background:#e9f7ee; font-weight:700; }
[data-theme="dark"] .c2bg .ans { background:#173d27; }
.c2bg .hid { background:#fdecec; animation:c2blink 1.8s ease-in-out infinite; }
[data-theme="dark"] .c2bg .hid { background:#3a1d1d; }
.c2bg .sum { font-size:12px; font-weight:700; }
@keyframes c2blink { 0%,100% { opacity:1; } 50% { opacity:.4; } }
@media (prefers-reduced-motion: reduce) { .c2bg * { animation:none !important; } }
</style>
```

ついでに踏んだ作法もメモしておきます。
`chat_id` が無いと400になること。
モデル定義をAPIで更新するときは、read-modify-writeが必須なこと。

部分更新のつもりで書くと、システムプロンプトやナレッジ紐付けが消えます。
UIから文書を削除してもベクトルが残ることがあり、リセットAPIで一掃してから入れ直すのが確実でした。

---

## LLMは、表の数字を「それらしく」捏造する

ハイブリッド以前に、信頼性の問題も潰しておく必要がありました。
表形式の文書をLLMに読ませると、数字を捏造するのです。

まず入口の問題です。
表PDFをデフォルト抽出のまま投入すると、列同士が分離した「ゴミチャンク」になります。

検索が壊れるだけならまだしも、壊れたテキストが文脈に入り、LLMが無限復唱を始める生成暴走まで起きました。
900行を超えて、手動で止めています。

犯人はPDFではなく、内蔵の抽出エンジンでした。
`pdftotext -layout` でテキスト層を抽出すると、対象文書は表も崩れずに取れました。

:::message
表PDFの原本は投入禁止、が結論です。
テキスト層が無い表は、次に書く分割OCRパイプラインに回します。
:::

---

## OCRの捏造を、パイプラインで潰す

テキスト層が無い表は、画像として読むしかありません。
ところがビジョンLLMのOCRは、読めない部分を前半のパターン外挿で埋めます。

**存在しない数値が、きれいな表の形で出てくる** のです。
出力がきれいに見えるほど疑わしい、という逆説でした。

```html-embed
<div class="c2ocr">
<div class="st" style="animation-delay:0s">画像を分割し、300dpiに高解像度化</div>
<div class="st" style="animation-delay:1.1s">帯状に分割（境界は白い行間へ）</div>
<div class="st" style="animation-delay:2.2s">全体読みと帯読みを突き合わせ</div>
<div class="st" style="animation-delay:3.3s">不一致セルだけズームし、別モデルで再読</div>
<div class="st" style="animation-delay:4.4s">算術検証（年額＝月額×12 など）</div>
</div>
<style>
.c2ocr { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2ocr { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c2ocr .st { border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; font-weight:700; animation:c2pulse2 5.5s ease-in-out infinite; }
@keyframes c2pulse2 { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 8%,18% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 30% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c2ocr * { animation:none !important; } }
</style>
```

```html-embed
<div class="c2acc">
<div class="row"><span class="rl">分割＋高解像度化</span><div class="bar b1"></div><span class="rv">97.6%</span></div>
<div class="row"><span class="rl">最終パイプライン</span><div class="bar b2"></div><span class="rv">42セル全問正解</span></div>
<div class="note">多様性は「見せ方（ズーム）」と「読み手（別モデル）」で作る</div>
</div>
<style>
.c2acc { --tx:#1a2733; --bd:#bcd9f5; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c2acc { --tx:#e8f2fa; background:#101c28; --bd:#33506b; }
.c2acc .row { display:flex; align-items:center; gap:8px; margin:6px 0; }
.c2acc .rl { flex:none; width:118px; font-size:12px; font-weight:700; }
.c2acc .bar { height:14px; border-radius:7px; background:linear-gradient(90deg,#1d9e57,#7ad79f,#1d9e57); background-size:200% 100%; animation:c2shimmer3 2.4s linear infinite; }
.c2acc .b1 { width:46%; } .c2acc .b2 { width:50%; }
.c2acc .rv { font-size:12.5px; font-weight:700; }
.c2acc .note { font-size:12.5px; margin-top:8px; }
@keyframes c2shimmer3 { to { background-position:-200% 0; } }
@media (prefers-reduced-motion: reduce) { .c2acc * { animation:none !important; } }
</style>
```

読み手は12B級と31B級のビジョンLLMを併用しました。
クロスチェックの多様性は、モデルを変えて初めて生まれます。

確定した表データの投入方法にも工夫が要ります。
Markdown表のまま入れず、1セル1文に展開し、レンジ要約文を足しました。

質問文とチャンクがほぼ一致する形になり、ベクトル検索が外れにくくなります。
地味ですが、検索精度はこういう所で決まります。

---

## 効かなかったこと

いちばん期待して、いちばん効かなかったのがこれです。

- 温度0で複数回実行し、多数決で誤読を消す作戦

同じモデル・同じ画像だと、誤読まで決定論的に再現されました。
多数決は全員が同じ間違いをする会議で、意味がありませんでした。

多様性は「回数」では作れません。
**見せ方と読み手を変える** ことでしか作れない、が結論です。

もう1つ、見送った判断も書いておきます。
推論エンジンの載せ替えです。

単一ユーザーならllama.cpp直、マルチユーザーならSGLangやvLLMが速いとされます。
ただし載せ替え先でAPIの400問題を踏んだ経緯があり、Ollamaネイティブ経路に固定しました。

載せ替えは、十分な検証期間がある場合だけやる。
これも実測側の教訓です。

---

## 使い分けの確定版：ローカルで足りる条件、足りない条件

実測を踏まえた、最終的な線引きです。

```html-embed
<div class="c2judge">
<div class="st" style="animation-delay:0s"><span>データを外に出せない？</span><span class="tag">Yes → ローカル完結</span></div>
<div class="st" style="animation-delay:1.3s"><span>小型MoEで品質が足りる？</span><span class="tag">Yes → ローカル</span></div>
<div class="st" style="animation-delay:2.6s"><span>回答は短く、ストリーミングで緩和できる？</span><span class="tag">Yes → ローカル</span></div>
<div class="st" style="animation-delay:3.9s"><span>どれか No なら</span><span class="tag">クラウド（Claude）へ</span></div>
</div>
<style>
.c2judge { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2judge { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c2judge .st { display:flex; justify-content:space-between; align-items:center; gap:8px; border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; animation:c2pulse3 5.2s ease-in-out infinite; }
.c2judge .tag { flex:none; font-size:11.5px; font-weight:700; }
@keyframes c2pulse3 { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 8%,18% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 30% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c2judge * { animation:none !important; } }
</style>
```

ローカル側の構成は、役割ごとに分けています。

| 役割 | 採用 |
| --- | --- |
| 回答生成LLM | gpt-oss:20B（20B級MoE） |
| 埋め込み | bge-m3 |
| チャットUI＋RAG | Open WebUI 0.9.5 |
| 実行基盤 | Ollama |
| 音声認識（バッチ） | faster-whisper large-v3-turbo |
| 音声認識（リアルタイム） | faster-whisper medium |

音声認識をバッチとリアルタイムで分けたのは、GPUの占有対策です。
同居させると、長いバッチ処理の間、リアルタイム側の遅延が数秒単位で跳ねます。

fp16で1インスタンス約1.6GBなので、統合メモリ上では2インスタンス常駐が問題ありません。
窓口はOpen WebUIのカスタムモデルで分け、モデル本体は共有したまま用途ごとに性格を変えています。

```html-embed
<div class="c2hyb">
<div class="lane">
<div class="lt">ローカル（DGX Spark）</div>
<div class="flow"><span class="fb">文書・音声データ</span><span class="fa">→</span><span class="fb">RAG / Whisper</span><span class="fa">→</span><span class="fb">小型MoEで回答</span></div>
</div>
<div class="lane">
<div class="lt">クラウド（Claude）</div>
<div class="flow"><span class="fb">外に出せる相談</span><span class="fa">→</span><span class="fb">深い推論・長い生成</span><span class="fa">→</span><span class="fb">結果を持ち帰る</span></div>
</div>
</div>
<style>
.c2hyb { --tx:#1a2733; --bd:#bcd9f5; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c2hyb { --tx:#e8f2fa; --bd:#33506b; }
.c2hyb .lane { border:1.5px solid var(--bd); border-radius:8px; padding:10px 12px; margin:6px 0; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c2hyb .lane { background:#101c28; }
.c2hyb .lt { font-weight:700; font-size:12.5px; margin-bottom:6px; }
.c2hyb .flow { display:flex; align-items:center; gap:6px; flex-wrap:wrap; }
.c2hyb .fb { font-size:12px; border:1px solid var(--bd); border-radius:6px; padding:4px 8px; background:#eef6ff; }
[data-theme="dark"] .c2hyb .fb { background:#16324a; }
.c2hyb .fa { color:#3ea8ff; font-weight:700; animation:c2flow2 1.4s linear infinite; }
@keyframes c2flow2 { 0%,100% { opacity:.35; } 50% { opacity:1; } }
@media (prefers-reduced-motion: reduce) { .c2hyb * { animation:none !important; } }
</style>
```

「大きいモデル × 速い生成」を両立できる変数は、メモリ帯域だけです。
Sparkの273GB/sでは埋まらない部分は、素直にクラウドへ渡します。

これが「ハイブリッド」の実態で、流行りの言葉より物理の話でした。
帯域の太いGPUでは大型モデルでも速いという報告もありますが、それは他環境の値なので、ここでは判断材料に含めていません。

---

## まとめ：線引きは、モデルの評判ではなく物理で決める

- 生成速度の上限 ≒ 帯域 ÷ モデルサイズ。実測もこの式どおりだった
- Sparkの主戦場は「長い入力 → 短い回答」。文書QAは17秒→4.3秒、誤答2→0まで詰められた
- 表の数字は、検証パイプラインなしにLLMへ渡さない
- ローカルで足りない条件は、帯域の壁として素直に認め、クラウドへ渡す

検証の焦点は最後まで1つでした。
**応答要件を満たせる最小モデルで、回答品質が用途に耐えるか** です。

この1点が見えれば、追加のハード投資が要るかどうかも自然に決まります。
使い分けに迷う時間を、実測に変えるのがいちばんの近道でした。
