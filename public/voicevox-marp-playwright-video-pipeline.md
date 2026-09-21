---
title: 台本とスライドから講座動画を自動生成する ― VOICEVOX + Marp + Playwright + ffmpeg で17本作って踏んだ5つの穴
tags:
  - 自動化
  - VOICEVOX
  - Playwright
  - ffmpeg
  - Marp
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- 台本（テキスト）とMarpスライド（Markdown）から、**ナレーション付きMP4を1コマンドで生成する**構成を組んだ。17レッスン分で合計83MB、1本あたり3〜6MB
- 構成は **VOICEVOX → Marp → Playwright → ffmpeg**。追加の課金は発生しない。かかるのは生成時間だけ
- 動くまでに5つ踏んだ。**台本の形式**、**スライド枚数とのズレ**、**Marpのフロントマター位置**、**Playwrightのプロセス死**、**メモリ**
- とくに効いたのは1つめ。台本が「人が読むための設計書」だったせいで、**見出しやMarkdown記号がそのまま読み上げられた**
- 自動化の肝は生成そのものではなく、**スライド枚数と台本ブロック数が1対1になっていることを機械的に検査する**部分だった

## 全体の流れ

```
台本 (.txt)           スライド (.md)
   │                      │
   ├─ VOICEVOX            ├─ Marp CLI
   │  ページ単位のWAV      │  → HTML
   │  + 再生時間のJSON      │
   │                      │
   └──────────┬───────────┘
              │
        Playwright で録画
      （タイミングJSONに従って
        ArrowRight でページ送り）
              │
           ffmpeg
      （無音の録画 + 連結WAV → MP4）
```

台本は `---` で区切る。**1ブロック＝スライド1ページ**。この対応が全部の前提になる。

```
こんにちは、このコースへようこそ。
このコースでは、AWSのふかテストのほうほうをまなんでいきます。
---
このコースでまなぶないようをしょうかいします。
まず、がいようとアーキテクチャ。
---
（以下、ページごとに続く）
```

## 1. 音声：VOICEVOXをローカルで立てる

VOICEVOXは `localhost:50021` にHTTPで待ち受ける。GUIを起動してもいいが、**エンジンだけ単体で起動できる**。

```bash
# Windows。インストール先の vv-engine にエンジン本体が入っている
"C:/Program Files/VOICEVOX/vv-engine/run.exe" --host 127.0.0.1 --port 50021
```

起動確認。

```bash
$ curl -s http://localhost:50021/version
"0.25.2"
```

合成は2段階。`audio_query` でパラメータを作り、`synthesis` でWAVにする。

```bash
QUERY=$(curl -s -X POST "http://localhost:50021/audio_query?speaker=2" \
  --get --data-urlencode "text=こんにちは")

curl -s -X POST "http://localhost:50021/synthesis?speaker=2" \
  -H "Content-Type: application/json" -d "$QUERY" -o page01.wav
```

ページごとにWAVを作り、**それぞれの再生時間をJSONに残す**。これが後のページ送りのタイミングになる。

```json
{
  "lecture_id": "1-1",
  "total_pages": 7,
  "pages": [
    { "page": 1, "wav": ".../1-1_page01.wav", "duration": 7.125, "text": "..." },
    { "page": 2, "wav": ".../1-1_page02.wav", "duration": 21.077, "text": "..." }
  ]
}
```

## 2. スライド：Marpで HTML にする

```bash
npx @marp-team/marp-cli@latest slide.md -o slide.html
```

Marpは `---` でページを区切る。出力HTMLでは1ページが `<section>` になるので、**ページ数は `document.querySelectorAll('section').length` で数えられる**。この数と、台本のブロック数を突き合わせる。

## 3. 録画：Playwrightでページ送り

無音の画面録画を取る。音は後からffmpegで合わせる。

```ts
const context = await browser.newContext({
  viewport:    { width: 1920, height: 1080 },
  recordVideo: { dir: tempDir, size: { width: 1920, height: 1080 } },
});

const page = await context.newPage();
await page.goto(`file://${htmlPath}`);
await page.waitForLoadState('networkidle');

const slideCount = await page.evaluate(
  () => document.querySelectorAll('section').length
);

for (const [i, slide] of timings.slides.entries()) {
  await page.waitForTimeout(slide.duration * 1000);   // その頁の音声ぶん待つ
  if (i < slideCount - 1) {
    await page.keyboard.press('ArrowRight');
    await page.waitForTimeout(50);
  }
}

await page.close();        // close しないと動画ファイルが確定しない
await context.close();
```

`page.close()` を忘れると、録画ファイルが書き出されない。ここは何度か踏んだ。

## 4. 結合：ffmpeg

ページごとのWAVを連結し、無音の録画と合わせる。

```bash
# WAVの連結
ffmpeg -y -f concat -safe 0 -i list.txt -c copy voice.wav

# 録画 + 音声
ffmpeg -y -i recorded.webm -i voice.wav \
  -c:v libx264 -c:a aac -b:a 192k -pix_fmt yuv420p -shortest out.mp4
```

`-shortest` を付けておくと、どちらかが長くても尻尾が切れる。

## 踏んだ5つの穴

### 穴1：台本が「人が読むための文書」だった

いちばん時間を溶かしたのがこれ。台本ファイルがこうなっていた。

```md
# セクション1 レクチャー1: コース紹介と学習ゴール

## 動画情報
- **時間**: 約5分
- **形式**: スライド

---

## 台本

### オープニング（0:00-0:30）

こんにちは、このコースへようこそ。
```

これをそのまま合成すると、1ページ目のナレーションがこうなる。

> 「シャープ、セクション1 レクチャー1 コース紹介と学習ゴール。シャープシャープ、動画情報。時間、約5分。形式、スライド」

**生成は成功する。** エラーも警告も出ない。音声ファイルもできる。中身だけが使い物にならない。

台本は制作用の設計書であって、読み上げ原稿ではなかった。**書式を先に決めておかないと、あとで全本数を書き直すことになる**（実際に13本書き直した）。

読み上げ用は、本文だけを `---` で区切ったプレーンテキストにする。日本語の読み間違いを避けるため、固有名詞と英字以外はかな書きにしておくと安定する。

### 穴2：スライド枚数と台本ブロック数がズレる

ズレていても**生成は通る**。足りないページが無音になるだけだ。

```
HTML のスライド数: 8 / タイミングの件数: 7
警告: スライド数(8)とタイミング件数(7)が一致しません。
```

警告を出すようにしたが、最初は見落とした。**生成前に落とす検査にすべきだった。**

```js
const pages  = slideHtml.match(/<section/g).length;
const blocks = script.split(/^---$/m).map(s => s.trim()).filter(Boolean).length;

if (pages !== blocks) {
  throw new Error(`${lessonId}: スライド${pages}枚 / 台本${blocks}ブロック`);
}
```

### 穴3：Marpのフロントマターの前に1行あると、空ページが増える

これが穴2の原因だった。

```md
# セクション1-1「コース紹介」スライド     ← これ

---
marp: true
theme: default
---

# AWS Distributed Load Testing
```

Marpのフロントマターは**ファイルの先頭になければならない**。前に何か書くと、フロントマターとして解釈されず、その部分が1ページ目として描画される。結果、意図より1枚多くなる。

12ファイルすべてで同じことが起きていた。

```js
// 先頭の見出し行を落として、フロントマターを1行目へ
s = s.replace(/^#[^\n]*\n\n?---\n/, '---\n');
```

### 穴4：Playwrightのプロセスが落ちる

長時間の生成をシェル越しに走らせていたら、途中でこうなった。

```
[error] 2-1: page.waitForTimeout: Target page, context or browser has been closed
```

親プロセスが終了したときに、Chromiumが道連れになっていた。**最初からバックグラウンドのジョブとして起動する**か、`nohup` 相当で切り離す。13本連続だと30分前後かかるので、前面で回すものではない。

### 穴5：メモリ

1920x1080の録画をChromiumで回しながら、VOICEVOXのエンジンも常駐している。長時間走らせると空きメモリが削れていく。実際、13本の途中で外側の仕組みに停止させられた。

対策としては、**レッスン単位でプロセスを立て直す**のが素直だった。1本終わるごとにブラウザを閉じ、コンテキストも破棄する。まとめて1プロセスで13本回すより、少し遅いが落ちない。

## 費用

追加課金はない。

| 要素 | ライセンス | 費用 |
|---|---|---|
| VOICEVOX | 利用規約に従えば商用可（キャラごとに条件あり） | 無料 |
| Marp CLI | MIT | 無料 |
| Playwright | Apache-2.0 | 無料 |
| ffmpeg | LGPL/GPL | 無料 |

かかるのは生成時間と電気代だけ。17レッスン分で83MB、1本あたり3〜6MBのMP4になった。

**VOICEVOXは音声ライブラリごとに利用条件が違う**ので、商用で使うなら使用するキャラクターの規約を必ず確認すること。クレジット表記が必要なものが多い。

## まとめ

- 台本とスライドから動画を作る部分は、既存のツールを繋ぐだけで組める
- 難しいのは生成ではなく、**入力の形式を守らせること**。台本は読み上げ原稿であって設計書ではない
- **スライド枚数と台本ブロック数の一致は、生成前に例外で落とす**。警告では見落とす
- Marpのフロントマターは必ずファイル先頭に置く
- 長時間の生成はバックグラウンドで、レッスン単位にプロセスを分ける
