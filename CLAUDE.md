# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## リポジトリの性質

ビルド不要のバニラJS/HTMLによるWebGPUシェーダープレイグラウンド。package.json・ビルドツール・リンター・テストフレームワークは一切ない。

- 各シェーダーは`shaders/*.html`の**1ファイルに完結**する(WGSLはJS内のテンプレート文字列として埋め込む)。外部JS/CSS/アセットへの依存は作らない
- サーバー不要で`file://`で直接開いて動作することが必須要件(このためPNGアセットの読み込みは`fetch`ではなく`<input type=file>`+`createImageBitmap`によるインポートボタン方式を採る)
- `index.html`がシェーダー一覧のホームページ。**新しいシェーダーを追加したら`index.html`のギャラリーと`README.md`の構成リストにもエントリを追加する**
- デモはGitHub Pages (https://shugo15.github.io/webgpu-with-ai/) で`main`ブランチから配信される
- ドキュメント(README)・UI文言は日本語。ユーザーへの回答も日本語で行う

## 動作確認(サンドボックス環境でのヘッドレス実行)

ブラウザでの手動確認の代わりに、swiftshader(ソフトウェアレンダリング)経由のPlaywrightでWebGPUを動かせる。`playwright install`は不要・実行禁止。

```js
const { chromium } = require('playwright'); // NODE_PATH=/opt/node22/lib/node_modules が必要
const browser = await chromium.launch({
  executablePath: '/opt/pw-browsers/chromium',
  env: { ...process.env, VK_ICD_FILENAMES: '/opt/pw-browsers/chromium-1194/chrome-linux/vk_swiftshader_icd.json' },
  args: ['--enable-unsafe-webgpu', '--enable-features=Vulkan', '--use-vulkan=swiftshader',
         '--use-angle=swiftshader', '--disable-vulkan-surface', '--ignore-gpu-blocklist',
         '--enable-webgpu-developer-features'],
});
```

実行は `NODE_PATH=/opt/node22/lib/node_modules node script.js`。

サンドボックス特有の注意:

- 表示canvasのスクリーンショットや`drawImage`読み出しは黒/透明になることがある。確実なのはオフスクリーンテクスチャに描いて`copyTextureToBuffer`でピクセル値を直接読む方法。`canvas.toDataURL('image/png')`によるキャプチャは動く場合が多い
- **重いレンダリング(高解像度の焼き込み・長時間待機)でブラウザが落ちる**。対策: ビューポートを小さくする(220px程度)、待機を短く分割する、定期的にキャプチャして途中経過を保存するループにする、ダウンロードイベントではなく`toDataURL`で画像を取り出す
- `page.on('console')`と`page.on('pageerror')`を必ず仕込み、WGSLコンパイルエラーを拾う
- 画像の数値分析はpython(numpy/PILは`pip install`で入る)が便利

シェーダーの一部を単体で数値検証したいときは、WGSLと同じ計算をNode.jsに複製して決定的に積分する方法が有効(過去にバグ切り分けの決定打になった)。ただし入力ベクトル(カメラの向き・太陽方向など)は実際のレンダリングで発生する値をそのまま使うこと(README「落とし穴」参照)。

### `toDataURL()`などの応答が返ってこないとき: 「ハング」と「単に遅い」を必ず切り分ける

このサンドボックスのswiftshader(ソフトウェアレンダリング)は、複雑なシェーダーの初回コンパイル・初回描画が実機より大幅に(数十秒〜100秒以上)遅くなることがある。応答が無いからといって即座にプロセスを殺したり「壊れている」と判断せず、まず**該当プロセスのCPU使用率を確認する**:

```bash
ps -o etimes=,pcpu= -C chrome 2>/dev/null | sort -rn -k2 | head -1
```

- **CPU使用率が高い(数百%)** → 単に計算が重いだけ。タイムアウトを伸ばして待つ(`Bash`の`timeout`コマンドや`run_in_background`+`Monitor`を使い、60秒で諦めず100秒以上待ってみる)。実際、一度「50秒待って応答なし」と判断してプロセスを強制終了させたら、裏では既に正常終了していたことがあった
- **CPU使用率がほぼ0%のまま動かない** → 本当のハング(無限ループ・デッドロック)を疑ってよい。この場合は疑わしいパラメータ(ループ回数・配列サイズなど)を2分探索して閾値を特定するのが早い。実際に波の本数`NUM_WAVES`を段階的に増やしたところ、**16は必ず成功し17は必ずハングする**という明確な閾値があった(swiftshaderのシェーダーコンパイラかドライバの上限と見られ、WGSL側からは根本原因を特定できなかった)。実機の本物のGPUで同じ制限があるとは限らないが、検証できない以上は安全側の値に留める
- 切り分けが終わるまでは、**ビューポートを小さく・描画距離やループ回数を控えめにした「正しさだけを確認する」構成**で先に動作確認し、フル設定でのパフォーマンス検証は分けて行うと、原因の切り分けが速い

### 「暗黙の前提」を埋め込んだ定数はコメントを残す

あるパラメータ(カメラの高さなど)向けにチューニングした定数(レイマーチングの近距離探索範囲など)を、その前提を書かずに埋め込むと、後で別の箇所だけ変更したときに性能や見た目が予告なく崩れる。実際、`shaders/ocean-mirror.html`のレイと波面の交差判定で、近距離探索範囲`nearDist`を「カメラの高さ2.5m程度」を前提に固定値5.0mにしていたところ、後からカメラの高さだけを引き上げたら大半のレイが近距離域を素通りするようになり、レンダリングが極端に遅くなった(README「カメラを高くしたら近距離域の設計前提が崩れて激重になった」参照)。値が他のパラメータに依存する場合は、可能なら式で導出する(`nearDist = max(cameraHeight * 2.0, 5.0)`のように)か、最低限その前提をコメントに残すこと。

## アーキテクチャ: 各シェーダー共通のパターン

全シェーダーが概ね同じ骨格を持つ。新規シェーダーは既存ファイル(特に`shaders/sky-precise.html`が最新かつ最も完成度が高い)を雛形にするのが早い。

1. **フルスクリーン三角形**のvertex shader + 本体のfragment shader
2. **時間積分による収束**(パストレーシング系): カメラ固定のまま毎フレーム新しいモンテカルロサンプルを生成し、ping-pongの蓄積テクスチャに累積平均 `newAvg = prev + (sample - prev) / n` で書き込む。スライダー等のパラメータ変更で蓄積カウンタをリセット
3. **resolve/displayパス**: HDR蓄積結果をトーンマップ(Reinhard)+ガンマ補正して表示
4. UIはページ右上のスライダー群(太陽時刻、反射回数など)

シェーダー間の共有ライブラリは意図的に無い。太陽軌道の式 `hourToSunDir()` や大気係数などは複数ファイルに複製されているので、変更する際は他ファイルとの整合性を確認する。

### 蓄積バッファは rgba32float を使う

累積平均方式は n が大きくなるほど更新量が小さくなり、`rgba16float`では**n≈2000で更新が量子化幅を下回って平均が凍結し、縞(バンディング)が出る**(実際に起きたバグ)。新規の蓄積バッファは`rgba32float`にすること。`cornell-box-*.html`と`ocean-gerstner.html`は歴史的にrgba16floatのままだが、実害が視認されにくいため放置している(触る機会があれば同じ修正を適用)。

## READMEの「開発中に見つかった落とし穴」は必読

`README.md`末尾のセクションに、このリポジトリで実際に踏んだバグと教訓が蓄積されている。特に再発しやすいもの:

- WGSL予約語との衝突(`target`など) — 変数名に注意
- 非uniformな分岐内では`textureSample`不可 — `textureSampleLevel(tex, sampler, uv, 0.0)`を使う
- NEEと位相関数/BSDFサンプリングで光源寄与を二重カウントしない
- 「サンプル数を増やすと悪化する」症状は、サンプリング理論のバグとは限らず蓄積側の数値精度のこともある
- 仮説を実装したら症状がbefore/afterで変わるかを必ず確認する。変わらないならその仮説は間違い
- 原因不明のアーティファクトをぼかし等で隠さない。1コミットには1つの論理的変更のみ(revert巻き添え防止)
- equirectangular画像上の幾何学的分析は投影の歪みを補正してから行う

**重要な失敗や教訓が新たに得られたら、このセクションに追記する**のがこのリポジトリの慣習。同様に、新しいシェーダーの技術解説もREADMEに節を追加する。
