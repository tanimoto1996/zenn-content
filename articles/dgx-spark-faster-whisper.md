---
title: "DGX Sparkにfaster-whisperを載せたら、公開イメージが全部使えなかった ― aarch64×Blackwellの壁を越えてRTF 0.02に届くまで"
emoji: "🖥️"
type: "tech"
topics: ["nvidia", "docker", "whisper", "gpu", "linux"]
published: false
---

## 結論：動くまでの道は「ソースビルド一択」だった

DGX Sparkに faster-whisper を載せようとしました。
結論から言うと、公開イメージも公式wheelも全部使えません。

aarch64とBlackwell（sm_121）の組み合わせが、既存の配布物の想定外だったからです。
最終的に、CTranslate2をC++からソースビルドする道だけが残りました。

```html-embed
<div class="c1learn">
<div class="ttl">この記事で分かること</div>
<div class="item" style="animation-delay:0s"><span class="chk">✓</span><span class="txt">公開イメージが使えない本当の理由</span></div>
<div class="item" style="animation-delay:.15s"><span class="chk">✓</span><span class="txt">sm_121をCMakeに通すためのパッチ</span></div>
<div class="item" style="animation-delay:.3s"><span class="chk">✓</span><span class="txt">「ビルド成功・ロード成功・実音声で落ちる」罠の正体</span></div>
<div class="item" style="animation-delay:.45s"><span class="chk">✓</span><span class="txt">無音ファイルの検証が罠を見逃す理由</span></div>
<div class="item" style="animation-delay:.6s"><span class="chk">✓</span><span class="txt">RTF 0.02に届いた最終構成と実測値</span></div>
</div>
<style>
.c1learn { --acc:#3ea8ff; --bg:#eef6ff; --tx:#1a2733; --bd:#bcd9f5; border:1px solid var(--bd); border-left:4px solid var(--acc); border-radius:8px; padding:12px 16px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; background:var(--bg); }
[data-theme="dark"] .c1learn { --bg:#16324a; --tx:#e8f2fa; --bd:#33506b; }
.c1learn .ttl { color:var(--tx); font-weight:700; font-size:14px; margin-bottom:6px; }
.c1learn .item { display:flex; align-items:flex-start; gap:8px; padding:2px 0; animation:c1fade .7s ease both; }
@keyframes c1fade { from { opacity:0; transform:translateY(5px); } to { opacity:1; transform:translateY(0); } }
.c1learn .chk { flex:none; color:var(--acc); font-weight:700; font-size:13px; line-height:1.55; }
.c1learn .txt { color:var(--tx); font-size:13.5px; line-height:1.55; }
@media (prefers-reduced-motion: reduce) { .c1learn * { animation:none !important; } }
</style>
```

到達点だけ先に置いておきます。
細かい壁の話は、この数字に届くまでの順番で書きます。

| 項目 | 到達値 |
| --- | --- |
| RTF（large-v3-turbo・fp16） | 0.02 |
| 1時間の音声の処理時間 | 約1.2分 |
| モデルロード | 67.9秒 |
| warmup（PTX JIT） | 5.4秒 |
| 最終イメージサイズ | 3.3GB |
| 初回ビルド所要時間 | 30〜60分 |

---

## 相手を知る：DGX Sparkは「aarch64 × sm_121」

まず実機の素性です。
この組み合わせが、後の壁すべての出発点になります。

| 項目 | 値 |
| --- | --- |
| GPU | NVIDIA GB10（Blackwell） |
| CUDA世代 | sm_121（compute capability 12.1） |
| CPU | aarch64（ARM64） |
| メモリ | 128GB（ユニファイド） |
| メモリ帯域 | 273GB/s |
| ドライバ | 580.159.03 |
| OS | DGX OS（Ubuntu 24.04系） |
| ディスク | NVMe 3.7TB |

```html-embed
<div class="c1spec">
<div class="box cpu"><div class="bt">CPU</div><div class="bs">aarch64</div></div>
<div class="link"></div>
<div class="box mem"><div class="bt">ユニファイドメモリ</div><div class="bs">128GB / 273GB/s</div></div>
<div class="link"></div>
<div class="box gpu"><div class="bt">GPU GB10</div><div class="bs">Blackwell sm_121</div></div>
</div>
<style>
.c1spec { --tx:#1a2733; --bd:#bcd9f5; --bg1:#eef6ff; --bg2:#e9f7ee; --bg3:#fdf3e3; display:flex; align-items:stretch; gap:0; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1spec { --tx:#e8f2fa; --bg1:#16324a; --bg2:#173d27; --bg3:#453311; }
.c1spec .box { flex:1; border:1.5px solid var(--bd); border-radius:8px; padding:10px 6px; text-align:center; color:var(--tx); }
.c1spec .cpu { background:var(--bg1); } .c1spec .mem { background:var(--bg2); } .c1spec .gpu { background:var(--bg3); }
.c1spec .bt { font-weight:700; font-size:13px; }
.c1spec .bs { font-size:11.5px; margin-top:2px; opacity:.85; }
.c1spec .link { flex:none; width:22px; align-self:center; border-top:3px dashed #3ea8ff; animation:c1beam 1.2s linear infinite; }
@keyframes c1beam { to { border-color:#8ec9ff; } }
@media (prefers-reduced-motion: reduce) { .c1spec * { animation:none !important; } }
</style>
```

ポイントは2つです。
CPUがx86_64ではなくaarch64であること。
そしてGPUの世代がsm_121と、かなり新しいことです。

世の中の配布物は「x86_64 × 少し前のGPU世代」を前提に作られています。
この2点がズレると、既製品は一斉に効かなくなります。

---

## セットアップの壁：電源を入れてからが第1ラウンド

構築以前に、実機セットアップでも小さな壁が続きました。
地味ですが、物理デバイスならではの話です。

```html-embed
<div class="c1setup">
<div class="st" style="animation-delay:0s"><span class="no">1</span><span>ホットスポットとして出現する実機に接続</span></div>
<div class="st" style="animation-delay:1.2s"><span class="no">2</span><span>WiFi接続 → ウェルカム画面 → ユーザー設定</span></div>
<div class="st" style="animation-delay:2.4s"><span class="no">3</span><span>システム更新（約20分）</span></div>
<div class="st" style="animation-delay:3.6s"><span class="no">4</span><span>IPが不明になり、HDMI直結に切り替え</span></div>
<div class="st" style="animation-delay:4.8s"><span class="no">5</span><span>スキャンでIPを特定して運用開始</span></div>
</div>
<style>
.c1setup { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; --acc:#3ea8ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1setup { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c1setup .st { display:flex; align-items:center; gap:10px; border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; animation:c1pulse 6s ease-in-out infinite; }
.c1setup .no { flex:none; width:20px; height:20px; border-radius:50%; background:var(--acc); color:#fff; font-weight:700; font-size:12px; text-align:center; line-height:20px; }
@keyframes c1pulse { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 8%,18% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 30% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c1setup * { animation:none !important; } }
</style>
```

| 壁 | 内容 | 対処 |
| --- | --- | --- |
| W1 | 初期接続はホットスポット経由 | 手順どおり進める |
| W2 | 更新後にIPが不明になる | HDMI・キーボード直結が必須 |
| W3 | DHCPだと再起動でIPが変わる | スキャンスクリプトで特定 |
| W4 | 物理要件がシビア | 下記参照 |

W4は見落としがちです。
アース付き3ピンコンセントとWiFiが必須でした。

ポートはUSB-Cのみ対応です。
キーボードやマウスをつなぐにはハブが要ります。

:::message
SSHが使えれば、NVIDIA Syncアプリは必須ではありません。
セットアップ自体はGUIベースで完結する設計でした。
:::

---

## 本丸の壁：公開イメージが全部使えない（W5）

セットアップが終わり、いよいよ faster-whisper です。
ここで最初の想定が崩れました。

faster-whisperの公開Dockerイメージは、ほぼ全て使用不可でした。
x86_64前提で、arm64タグがあるものでも中身のCTranslate2がCPU-onlyビルドだったからです。

公式のpip wheelも同じでした。
ARM64版は存在しますが、CUDAサポートを含んでいません。

動かそうとすると、このエラーが返ります。

```text
ValueError: This CTranslate2 package was not compiled with CUDA support
```

```html-embed
<div class="c1ba">
<div class="panel ng">
<div class="pt">公開イメージ / 公式wheel</div>
<div class="pr">CTranslate2 が CPU-only</div>
<div class="pr bad">CUDAサポートなしでエラー</div>
</div>
<div class="panel ok">
<div class="pt">CTranslate2 をソースビルド</div>
<div class="pr">sm_121 向けに自前でコンパイル</div>
<div class="pr good">GPU で動作（RTF 0.02）</div>
</div>
</div>
<style>
.c1ba { --tx:#1a2733; --bd:#bcd9f5; position:relative; margin:12px 4px; height:118px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1ba { --tx:#e8f2fa; }
.c1ba .panel { position:absolute; inset:0; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; color:var(--tx); background:#fff8f6; }
[data-theme="dark"] .c1ba .panel { background:#2a1a16; }
.c1ba .ok { background:#f2fbf4; animation:c1swap 8s ease-in-out infinite; }
[data-theme="dark"] .c1ba .ok { background:#14281a; }
.c1ba .pt { font-weight:700; font-size:13.5px; margin-bottom:4px; }
.c1ba .pr { font-size:12.5px; line-height:1.6; }
.c1ba .bad { color:#d64545; font-weight:700; }
.c1ba .good { color:#1d9e57; font-weight:700; }
@keyframes c1swap { 0%,42% { opacity:0; } 54%,92% { opacity:1; } 100% { opacity:0; } }
@media (prefers-reduced-motion: reduce) { .c1ba * { animation:none !important; } .c1ba .ok { opacity:0; } }
</style>
```

:::message
「arm64タグがある＝動く」ではありませんでした。
タグは外側の話で、肝心のネイティブライブラリがCPU-onlyという落ちです。
:::

結論は早い段階で出ました。
**CTranslate2をC++からソースビルドするのが、唯一の動作経路** です。

---

## CMakeが「sm_121」を知らない（W6）

ソースビルドを始めると、次の壁がすぐ来ます。
CMake同梱の `select_compute_arch.cmake` が、sm_121のアーキ名を解決できないのです。

```text
Unknown CUDA Architecture Name
```

乗り越え方は力技です。
CMakeLists.txtにgencodeを直接埋め込むsedパッチを当てました。

```bash
# 概念図：実際のパッチは対象バージョンに合わせて調整
sed -i 's|...アーキ検出まわり...|-gencode arch=compute_90,code=compute_90|' CMakeLists.txt
grep -q "gencode" CMakeLists.txt || { echo "patch failed"; exit 1; }
```

```html-embed
<div class="c1build">
<div class="st fail" style="animation-delay:0s"><span>CMake 実行</span><span class="tag">sm_121 を解決できない</span></div>
<div class="st" style="animation-delay:1.4s"><span>sed パッチで gencode を埋め込む</span><span class="tag">力技</span></div>
<div class="st" style="animation-delay:2.8s"><span>grep ガードで適用を確認</span><span class="tag">失敗なら即終了</span></div>
<div class="st ok" style="animation-delay:4.2s"><span>ビルド続行</span><span class="tag">30〜60分</span></div>
</div>
<style>
.c1build { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1build { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c1build .st { display:flex; justify-content:space-between; align-items:center; gap:8px; border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; animation:c1pulse2 5.6s ease-in-out infinite; }
.c1build .tag { flex:none; font-size:11px; opacity:.8; }
.c1build .fail { border-color:#e5a3a3; }
.c1build .ok { border-color:#8fd3a8; }
@keyframes c1pulse2 { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 8%,18% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 30% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c1build * { animation:none !important; } }
</style>
```

grepガードは地味に重要です。
パッチが当たっていないのに進むと、数十分ビルドして最後に失敗します。

「その場で失敗させる」仕掛けは、ビルド時間の長い環境ほど効きます。
笑い話みたいですが、30分待って失敗するのと、3秒で失敗するのでは大違いです。

---

## 一番怖い壁：ビルドもロードも成功するのに、実音声で落ちる（W7）

今回いちばんタチが悪かった壁です。
gencodeを `code=sm_90` にすると、Hopper専用のSASSが生成されます。

この設定だと、ビルドは成功します。
モデルのロードも成功します。
ところが実音声を処理した瞬間に落ちます。

```text
cudaErrorNoKernelImageForDevice
```

「カーネルのイメージが無い」というエラーです。
ロード時点では検証されず、実際にカーネルが走る段になって初めて表面化します。

```html-embed
<div class="c1sass">
<div class="col ng">
<div class="ct">code=sm_90（SASS埋め込み）</div>
<div class="row">ビルド <b class="ok2">成功</b></div>
<div class="row">モデルロード <b class="ok2">成功</b></div>
<div class="row">実音声の処理 <b class="ng2">エラーで停止</b></div>
</div>
<div class="col ok">
<div class="ct">code=compute_90（PTXのみ）</div>
<div class="row">ビルド <b class="ok2">成功</b></div>
<div class="row">起動時にドライバが JIT 変換</div>
<div class="row">実音声の処理 <b class="ok2">動作</b></div>
</div>
</div>
<style>
.c1sass { --tx:#1a2733; --bd:#bcd9f5; display:flex; gap:8px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1sass { --tx:#e8f2fa; --bd:#33506b; }
.c1sass .col { flex:1; border:1.5px solid var(--bd); border-radius:8px; padding:10px 12px; color:var(--tx); }
.c1sass .ng { background:#fff6f4; } .c1sass .ok { background:#f2fbf4; }
[data-theme="dark"] .c1sass .ng { background:#2a1a16; } [data-theme="dark"] .c1sass .ok { background:#14281a; }
.c1sass .ct { font-weight:700; font-size:12.5px; margin-bottom:6px; }
.c1sass .row { font-size:12.5px; line-height:1.7; }
.c1sass .ok2 { color:#1d9e57; } .c1sass .ng2 { color:#d64545; animation:c1blink 1.6s ease-in-out infinite; }
@keyframes c1blink { 0%,100% { opacity:1; } 50% { opacity:.35; } }
@media (prefers-reduced-motion: reduce) { .c1sass * { animation:none !important; } }
</style>
```

乗り越え方は、SASSを埋め込むのをやめることです。
`code=compute_90` に固定してPTXだけを埋め込み、ドライバの実行時JITでsm_121向けに変換させます。

JIT変換のコストは起動時に寄せました。
warmupとして5.4秒で消化するので、初回リクエストに遅延が乗りません。

```html-embed
<div class="c1warm">
<div class="seg s1" style="animation-delay:0s"><span class="sl">モデルロード</span><span class="sv">67.9秒</span></div>
<div class="seg s2" style="animation-delay:1.5s"><span class="sl">warmup（PTX JIT）</span><span class="sv">5.4秒</span></div>
<div class="seg s3" style="animation-delay:3s"><span class="sl">リクエスト受付開始</span><span class="sv">遅延なし</span></div>
</div>
<style>
.c1warm { --tx:#1a2733; --bd:#bcd9f5; display:flex; gap:6px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1warm { --tx:#e8f2fa; --bd:#33506b; }
.c1warm .seg { border:1.5px solid var(--bd); border-radius:8px; padding:10px 8px; color:var(--tx); text-align:center; animation:c1pulse3 4.5s ease-in-out infinite; }
.c1warm .s1 { flex:5; background:#fdf3e3; } .c1warm .s2 { flex:1.4; background:#eef6ff; } .c1warm .s3 { flex:2; background:#e9f7ee; }
[data-theme="dark"] .c1warm .s1 { background:#453311; } [data-theme="dark"] .c1warm .s2 { background:#16324a; } [data-theme="dark"] .c1warm .s3 { background:#173d27; }
.c1warm .sl { display:block; font-size:11.5px; font-weight:700; }
.c1warm .sv { display:block; font-size:12px; margin-top:2px; }
@keyframes c1pulse3 { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 10%,22% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 34% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c1warm * { animation:none !important; } }
</style>
```

:::message
PTX JITの経路は公式サポート構成ではありません。
ドライバ更新で挙動が変わるリスクは残ります。この点は正直に書いておきます。
:::

---

## 検証の罠：無音ファイルでは罠に気づけない（W8）

W7には、もう1つ落とし穴が重なっていました。
検証に無音ファイルを使うと、VADが全区間を除去してしまいます。

区間がゼロだと、GPUカーネルは一度も実行されません。
つまり「エラーが出なかった」のではなく「何も試していなかった」状態です。

```html-embed
<div class="c1vad">
<div class="lane">
<div class="lt">無音ファイルで検証</div>
<div class="flow"><span class="fb">入力</span><span class="fa">→</span><span class="fb">VADが全区間を除去</span><span class="fa">→</span><span class="fb ng">カーネル未実行（未検証）</span></div>
</div>
<div class="lane">
<div class="lt">実音声で検証</div>
<div class="flow"><span class="fb">入力</span><span class="fa">→</span><span class="fb">区間が残る</span><span class="fa">→</span><span class="fb ok">カーネル実行（検証OK）</span></div>
</div>
</div>
<style>
.c1vad { --tx:#1a2733; --bd:#bcd9f5; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1vad { --tx:#e8f2fa; --bd:#33506b; }
.c1vad .lane { border:1.5px solid var(--bd); border-radius:8px; padding:10px 12px; margin:6px 0; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c1vad .lane { background:#101c28; }
.c1vad .lt { font-weight:700; font-size:12.5px; margin-bottom:6px; }
.c1vad .flow { display:flex; align-items:center; gap:6px; flex-wrap:wrap; }
.c1vad .fb { font-size:12px; border:1px solid var(--bd); border-radius:6px; padding:4px 8px; background:#eef6ff; }
[data-theme="dark"] .c1vad .fb { background:#16324a; }
.c1vad .fb.ng { background:#fdecec; border-color:#e5a3a3; font-weight:700; }
[data-theme="dark"] .c1vad .fb.ng { background:#3a1d1d; }
.c1vad .fb.ok { background:#e9f7ee; border-color:#8fd3a8; font-weight:700; }
[data-theme="dark"] .c1vad .fb.ok { background:#173d27; }
.c1vad .fa { color:#3ea8ff; font-weight:700; animation:c1flow 1.4s linear infinite; }
@keyframes c1flow { 0%,100% { opacity:.35; } 50% { opacity:1; } }
@media (prefers-reduced-motion: reduce) { .c1vad * { animation:none !important; } }
</style>
```

対処は決め打ちです。
検証とwarmupは、必ず実音声（またはホワイトノイズ）を使います。

あわせて `vad_filter=False` にします。
generatorは `list()` で消費しないとカーネルが走らない、という点も地味に踏みました。

---

## 依存関係と再現性の壁（W9〜W13）

GPUで動き出しても、まだ終わりません。
ビルド成果物を「壊さずに保つ」側の壁が続きました。

最大の罠はW9です。
`pip install faster-whisper` が、依存としてPyPIのCTranslate2（CPU-only）を引き込みます。

せっかく自前ビルドしたwheelが、上書きされて壊れるのです。
しかも壊れ方が静かなので、気づくのが遅れます。

```html-embed
<div class="c1ord">
<div class="st" style="animation-delay:0s"><span>CTranslate2 をソースビルド</span><span class="tag">自前 wheel 4.6.0</span></div>
<div class="st warn" style="animation-delay:1.4s"><span>faster-whisper をインストール</span><span class="tag">PyPI版 4.8.1 が上書き</span></div>
<div class="st ok" style="animation-delay:2.8s"><span>自前 wheel を最後に強制上書き</span><span class="tag">--force-reinstall --no-deps</span></div>
</div>
<style>
.c1ord { --tx:#1a2733; --bd:#bcd9f5; --bg:#f4f9ff; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1ord { --tx:#e8f2fa; --bg:#12293d; --bd:#33506b; }
.c1ord .st { display:flex; justify-content:space-between; align-items:center; gap:8px; border:1.5px solid var(--bd); background:var(--bg); color:var(--tx); border-radius:8px; padding:8px 12px; margin:6px 0; font-size:13px; animation:c1pulse4 4.2s ease-in-out infinite; }
.c1ord .tag { flex:none; font-size:11px; opacity:.8; }
.c1ord .warn { border-color:#e8c97a; }
.c1ord .ok { border-color:#8fd3a8; }
@keyframes c1pulse4 { 0%,100% { box-shadow:0 0 0 0 rgba(62,168,255,0); } 10%,22% { box-shadow:0 0 0 3px rgba(62,168,255,.45); } 34% { box-shadow:0 0 0 0 rgba(62,168,255,0); } }
@media (prefers-reduced-motion: reduce) { .c1ord * { animation:none !important; } }
</style>
```

対処は、インストール順序の固定です。
自前wheelを `--force-reinstall --no-deps` で最後に強制上書きします。

自前4.6.0がPyPI 4.8.1を置き換える形で、動作確認できました。
バージョン番号が若い方が勝つ、という気持ち悪さは残りますが、これが現実解です。

残りの壁は表にまとめます。

| 壁 | 内容 | 対処 |
| --- | --- | --- |
| W10 | ベースイメージのタグ運用で前提が崩れる | digest固定 |
| W11 | ARGの宣言位置でイメージパースエラー | 最初のFROMより前に宣言 |
| W12 | x86専用ライブラリが使えない | 下記参照 |
| W13 | 再起動後にコンテナが自動で起きない場合がある | 手動起動を運用に組み込む |

W10の効き目の確認方法も決めました。
「ビルドが通ること」ではありません。

再ビルドが数秒で終わり、イメージIDが変わらないことです。
実測では再ビルド2.8秒で、キャッシュが効いていることを確認しました。

W12はx86前提の掃除です。
MKLはOFFにしてOpenBLASを使います。

Intel OpenMP（libiomp5）は使えず、`-DOPENMP_RUNTIME=COMP` でGCCのlibgompに寄せました。
`-DCMAKE_INSTALL_PREFIX=/usr/local` でないと、Pythonバインディングがヘッダを見つけられず失敗します。

---

## 効かなかったこと

遠回りを正直に並べます。
どれも「それっぽい」からこそ踏みました。

```html-embed
<div class="c1fail">
<div class="card"><div class="ct">gencodeを sm_90 に固定</div><div class="cb">ロード成功に騙された。実音声で初めて落ちる</div></div>
<div class="card"><div class="ct">無音ファイルでの動作確認</div><div class="cb">VADが全区間を消し、何も検証できていなかった</div></div>
<div class="card"><div class="ct">ベースイメージのタグ運用</div><div class="cb">同一タグの再pushでビルド前提が崩れた</div></div>
</div>
<style>
.c1fail { --tx:#1a2733; display:flex; gap:8px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1fail { --tx:#e8f2fa; }
.c1fail .card { flex:1; border:1.5px solid #e5a3a3; background:#fff6f4; border-radius:8px; padding:10px 12px; color:var(--tx); }
[data-theme="dark"] .c1fail .card { background:#2a1a16; }
.c1fail .ct { font-weight:700; font-size:12.5px; margin-bottom:4px; }
.c1fail .cb { font-size:12px; line-height:1.6; }
@media (prefers-reduced-motion: reduce) { .c1fail * { animation:none !important; } }
</style>
```

- 公開イメージのarm64タグを信じて時間を溶かした
- 「ロードできた＝動いた」と早合点した
- 検証音声に無音ファイルを使い、未検証を「正常」と読んだ

共通点は1つです。
**「成功したように見える」段階で止めてしまった** ことです。

実機の壁は、エラーを出す壁より、静かに騙す壁の方が厄介でした。
これは今回いちばんの学びです。

---

## 到達した構成と実測値

最終構成は、Dockerのマルチステージビルドです。
builderでCTranslate2をソースビルドし、runtimeに成果物だけを載せます。

```html-embed
<div class="c1arch">
<div class="node"><div class="nt">builder</div><div class="ns">CT2 ソースビルド</div></div>
<div class="arw">→</div>
<div class="node"><div class="nt">runtime（3.3GB）</div><div class="ns">自前 wheel を搭載</div></div>
<div class="arw">→</div>
<div class="node"><div class="nt">バッチ文字起こし</div><div class="ns">OpenAI互換API</div></div>
<div class="node"><div class="nt">リアルタイム文字起こし</div><div class="ns">WebSocket</div></div>
</div>
<style>
.c1arch { --tx:#1a2733; --bd:#bcd9f5; display:flex; align-items:stretch; gap:6px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; }
[data-theme="dark"] .c1arch { --tx:#e8f2fa; --bd:#33506b; }
.c1arch .node { flex:1; border:1.5px solid var(--bd); border-radius:8px; padding:10px 6px; text-align:center; color:var(--tx); background:#eef6ff; }
[data-theme="dark"] .c1arch .node { background:#16324a; }
.c1arch .nt { font-weight:700; font-size:12px; }
.c1arch .ns { font-size:11px; margin-top:2px; opacity:.85; }
.c1arch .arw { flex:none; align-self:center; color:#3ea8ff; font-weight:700; animation:c1flow2 1.4s linear infinite; }
@keyframes c1flow2 { 0%,100% { opacity:.35; } 50% { opacity:1; } }
@media (prefers-reduced-motion: reduce) { .c1arch * { animation:none !important; } }
</style>
```

バッチとリアルタイムは別サービスに分けました。
同じプロセスに同居させると、長いバッチ処理がGPUを占有し、リアルタイム側の遅延が跳ねるからです。

実測値はこうなりました。
モデルはlarge-v3-turbo、fp16です。

```html-embed
<div class="c1perf">
<div class="row"><span class="rl">音声の長さ</span><div class="bar b1"></div><span class="rv">417.4秒</span></div>
<div class="row"><span class="rl">処理時間</span><div class="bar b2"></div><span class="rv">7.0秒</span></div>
<div class="note">RTF 0.02 ＝ 1時間の音声を約1.2分で処理</div>
</div>
<style>
.c1perf { --tx:#1a2733; --bd:#bcd9f5; border:1.5px solid var(--bd); border-radius:8px; padding:12px 14px; margin:12px 4px; font-family:-apple-system, "Hiragino Sans", "Noto Sans JP", sans-serif; color:var(--tx); background:#fbfdff; }
[data-theme="dark"] .c1perf { --tx:#e8f2fa; background:#101c28; --bd:#33506b; }
.c1perf .row { display:flex; align-items:center; gap:8px; margin:6px 0; }
.c1perf .rl { flex:none; width:74px; font-size:12px; font-weight:700; }
.c1perf .bar { height:14px; border-radius:7px; background:linear-gradient(90deg,#3ea8ff,#8ec9ff,#3ea8ff); background-size:200% 100%; animation:c1shimmer 2.4s linear infinite; }
.c1perf .b1 { width:58%; } .c1perf .b2 { width:12%; }
.c1perf .rv { font-size:12.5px; font-weight:700; }
.c1perf .note { font-size:12.5px; margin-top:8px; font-weight:700; }
@keyframes c1shimmer { to { background-position:-200% 0; } }
@media (prefers-reduced-motion: reduce) { .c1perf * { animation:none !important; } }
</style>
```

| 項目 | 実測値 |
| --- | --- |
| RTF | 0.02（417.4秒 → 7.0秒） |
| モデルロード | 67.9秒 |
| warmup（PTX JIT） | 5.4秒 |
| イメージサイズ | 3.3GB |
| ネットワーク転送 | 10.6MB/s |

モデルロードの67.9秒は短くありません。
だからこそ、常駐前提の設計とwarmupの先行消化が効いてきます。

---

## まとめ：新しい実機は「配布物の想定外」から始まる

壁を並べ直します。

| 分類 | 壁 | 教訓 |
| --- | --- | --- |
| セットアップ | W1〜W4 | 物理の前提を先に潰す |
| 配布物 | W5 | arm64タグを信じない |
| ビルド | W6・W12 | 新しい世代はパッチで通す |
| 実行時 | W7・W8 | 静かに騙す壁が一番怖い |
| 再現性 | W9〜W11・W13 | 成果物を壊さない仕掛けが要る |

判断基準として残すのは、この3つです。

- 既製品が無い前提で、ソースビルドの覚悟を最初から持つ
- 「成功したように見える」段階で止めず、実データで必ず叩く
- 効いた対処には、再発を防ぐガード（grep・digest固定・順序固定）をセットで付ける

DGX Spark自体は、動いてしまえば速いマシンです。
417.4秒の音声を7.0秒で処理する性能は、文句のない数字でした。

壁の高さは「マシンが新しいこと」の裏返しです。
同じaarch64 × Blackwellの海に漕ぎ出す人の、地図になれば幸いです。
