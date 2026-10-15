---
title: Udemy講座の動画でスライドと音声が最大15秒ずれていた原因は「録画」だった ― 全37本を測って、コマ単位で作り直した話
tags:
  - Udemy
  - ffmpeg
  - Playwright
  - TypeScript
  - Marp
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- Udemyで公開しているAWS CI/CDの講座に、「後半ずっとスライドと説明がずれている」という星1つのレビューがついた。
- 全37レクチャーを、ffmpegで「スライドが切り替わった時刻」を取り出し、音声の予定の時刻と突き合わせて測った。**ずれは最大で約15秒**。原因は3つあった。
  1. スライドを **Playwrightで実時間で録画**していて、待ち時間が積み重なる
  2. **7本で、スライドの枚数と台本のページ数が合っていない**（途中から別のスライドを映しながら話していた）
  3. **4本で、台本ではなく制作メモを読み上げていた**（これは次の記事で書く）
- 録画をやめ、スライドを1枚ずつ画像にして、**音声に1/30秒単位のコマ数で並べる**形に作り直した。ずれは **最大0.02秒** になった。
- 作り直した仕組みには、**動画を作ったあとに切り替わりの時刻を測り直し、1コマ以上ずれていたら止める**検査を入れた。

## 講座動画の作りかた（直す前）

講座の動画は、次の流れで自動で作っています。

1. 台本（テキスト）を `---` でページに分け、VOICEVOXでページごとに音声（WAV）を作る
2. 音声を1本につなぎ、ページごとの長さを `timings.json` に書く
3. スライドを Marp で HTML にする
4. **Playwright でスライドを開いて録画**し、ページの長さだけ待っては次へ送る
5. 録画（webm）と音声を ffmpeg で重ねる

4 の部分は、こういうコードでした。

```ts
const context = await browser.newContext({
  recordVideo: { dir: tempDir, size: { width: 1920, height: 1080 } },
});
const page = await context.newPage();
await page.goto(htmlUrl);
await page.waitForLoadState('networkidle');

for (let i = 0; i < timings.slides.length; i++) {
  const hold = timings.slides[i].duration + timings.silence_duration;
  await page.waitForTimeout(hold * 1000);
  await page.keyboard.press('ArrowRight');
  await page.waitForTimeout(50);
}
```

「音声の長さだけ待って、次へ送る」ので、合いそうに見えます。

## 測りかた

感覚ではなく数字で確かめるため、全レクチャーで次のように測りました。

- **予定の時刻**：`timings.json` の、各ページの長さと間を足していった時刻
- **実際の時刻**：ffmpeg の `select='gt(scene,…)'` と `showinfo` で、画面が切り替わったコマの時刻

```bash
ffmpeg -i lecture.mp4 -vf "select='gt(scene,0.01)',showinfo" -an -f null - 2>&1 \
  | grep -o "pts_time:[0-9.]*"
```

予定の時刻ごとに、一番近い実際の切り替わりとの差を取ります。

## 結果（直す前）

| 範囲 | ずれ |
| --- | --- |
| 5〜8章（後半の大部分） | 0.5秒前後。最初は約0.5秒早く、だんだん遅れていく |
| 1〜4章、7-3・7-4 | 数秒〜**最大約15秒** |

さらに、スライドの枚数と台本のページ数を数えると、**7本で合っていません**でした。録画の仕組みは、枚数が合わなくても警告を出すだけで、動画を作っていました。合っていないレクチャーは、途中から1〜2枚ずれたスライドを映しながら話し続けます。

## なぜ録画だとずれるのか

- 録画は `newContext` の時点で始まる。ページの読み込みと `networkidle` を待つ間も録画されているので、**最初から音声とずれる**。
- `waitForTimeout` は「少なくともその時間待つ」で、ぴったりではない。さらに切り替えごとに 50ms 待っている。**1枚ごとの小さな誤差が、枚数ぶん積み重なる**。
- 実時間の録画なので、PCの負荷でコマが落ちる。

「音声の長さだけ待つ」という作りそのものが、ずれを生む構造でした。

## 直しかた：録画をやめ、コマ単位で並べる

方針は、**時間を「待つ」のではなく「コマ数で数える」**ことです。

1. Playwright では、スライドを1枚ずつ**スクリーンショット**するだけにする
2. 各スライドを何コマ表示するかを、音声の時刻から**計算で**決める
3. ffmpeg で、静止画を決めたコマ数だけの動画にし、つなぐ
4. 音声は1本のまま、最後に1回だけ符号化して重ねる

### コマ数は「累積の時刻」を丸めて決める

スライドごとに「長さ × 30」を丸めると、丸めの誤差が積み重なります。そこで、**切り替わりの時刻（累積）を丸めて**、その差をコマ数にします。

```ts
export function frameCounts(timings: SlideTimings, audioSeconds: number, fps = 30): number[] {
  const boundaries = [0];
  let t = 0;
  for (let i = 0; i < timings.slides.length - 1; i++) {
    t += timings.slides[i].duration + timings.silence_duration;
    boundaries.push(t);
  }
  boundaries.push(Math.max(audioSeconds, t + 1 / fps)); // 最後は音声の終わりまで
  const frames = boundaries.map((b) => Math.round(b * fps));
  return frames.slice(1).map((f, i) => Math.max(1, f - frames[i]));
}
```

こうすると、どれだけ枚数があっても、各切り替わりは予定から **半コマ（約0.017秒）以内** に収まります。

### 静止画をコマ数ぶんの動画にする

```bash
ffmpeg -loop 1 -framerate 30 -i slide.png -frames:v 123 \
  -c:v libx264 -tune stillimage -g 123 -pix_fmt yuv420p -an seg.mp4
```

`-g`（キーフレームの間隔）をコマ数と同じにしているのは、後で説明する検査で、画質の揺れを「切り替わり」と見誤らないためです。

### 枚数が合わなければ止める

スライドの枚数（Marp の HTML の `section[id]` の数）と、台本のページ数が違えば、動画を作らずにエラーで止めます。背景画像（`![bg]`）のあるスライドでは、Marp が `section` を3つ作るので、`id` のある `section` だけを数えます。

### 作ったあとに、測り直して止める

最後に、出来上がった動画で切り替わりの時刻を測り直し、予定から1コマ（約0.033秒）以上ずれていたら失敗にします。最初に「測りかた」で使った方法を、そのまま検査にしました。

## 結果（直した後）

| | 直す前 | 直した後 |
| --- | --- | --- |
| 全37本のずれの最大 | 約15秒 | **0.02秒** |
| 5〜8章の典型的なずれ | 0.5秒前後 | 0.01秒前後 |
| 枚数が合わないレクチャー | 7本 | 0本 |

## まとめ

- 「待ち時間で合わせる」録画は、ずれを生む構造だった。時間は「待つ」のではなく「コマ数で数える」。
- 切り替わりの時刻は、累積で丸める。
- 作ったものは、測り直して、外れていたら止める。
- 感覚で「ずれてないと思う」ではなく、全本を測ると、どこが壊れているかがはっきりする。
