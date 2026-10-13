---
title: Marpで画像を量産したらTempに241フォルダ・1.1GBが溜まっていたので、本体を直してPRを出した話
tags:
  - Marp
  - TypeScript
  - OSS
  - Windows
  - Puppeteer
private: false
updated_at: ''
id: null
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- Marp CLI で PNG を書き出すたびに、`%TEMP%` に `marp-cli-xxxxxxxxxx` というフォルダが1つ増え、**消えない**。
- 動画講座のスライドを数か月 Marp で書き出していたら、Windows の Temp に **241フォルダ・約1.1GB** 溜まっていた。
- 原因は、Marp CLI がブラウザ（Chrome/Edge/Firefox）を起動するときに作る**使い捨てのブラウザプロファイル**。
- 本家の Issue（[#678](https://github.com/marp-team/marp-cli/issues/678)）に作者の直し方の設計が書かれたまま10か月放置されていたので、実装して **PR（[#743](https://github.com/marp-team/marp-cli/pull/743)）** を出した。
- PR がマージされるまでの、手元でできる対処（掃除コマンド・サーバーモード）も書いておく。

## 何が起きていたか

動画講座のスライドを Marp で作り、`--images png` で1枚ずつ画像にして、音声と合わせて動画にしています。1本の動画で数十回、作り直しのたびにまた数十回、`marp` を実行します。

ある日、Temp を見たらこうなっていました。

```powershell
PS> $d = Get-ChildItem $env:TEMP -Directory -Filter "marp-cli-*"
PS> $d.Count
241
PS> ($d | % { (Get-ChildItem $_.FullName -Recurse -File | Measure-Object Length -Sum).Sum } | Measure-Object -Sum).Sum / 1MB
1149.9
```

一番古いフォルダは4か月前のものでした。1回あたり数MBのフォルダが、実行回数ぶん、そのまま残っています。

Issue #678 の報告者は、もっと深刻です。PNG を大量に書き出す業務で、**10日で400GB超**が溜まったと書いています（1フォルダ約18MB）。

## なぜ残るのか

Marp CLI は、Markdown を PDF や画像にするとき、Puppeteer でブラウザを起動します。そのとき、普段使いのブラウザと混ざらないように、`os.tmpdir()` の下に**毎回新しいプロファイル用フォルダ**を作ります。

```ts
// marp-cli の src/browser/browser.ts（修正前の要点）
this.#dataDirName = `marp-cli-${nanoid(10)}`
// ...
return path.resolve(os.tmpdir(), this.#dataDirName)
```

このフォルダは、処理が終わっても削除されません。作者のコメントによると、ブラウザが終了後もファイルを書くことがあるため、プロセス終了時にすぐ消すのは危ない、という事情があります。

## 作者が書いていた直し方

Issue には、作者（yhatt さん）による設計が書かれていました。

1. プロファイルは `/tmp/marp-cli-browser/` の下にまとめて作る
2. プロファイル `foo` を作るとき、同じ場所に目印（ミューテックス）の `foo.lock` を作る
3. プロセスが終わったら `foo.lock` だけ消す（プロファイル本体は残す）
4. 次に Marp CLI がブラウザを使うとき、**`.lock` が無いプロファイルは使用済みとみなして、裏で消す**

「終わったプロセスのフォルダは、次のプロセスが掃除する」という形です。変換を止めずに、Temp に残る量を最小にできます。

## 実装して、1つ足したこと

設計どおりに実装したあと、手元で動かすと、**消えない `.lock` が177個**残りました。

原因は、強制終了されたプロセスです。テストの並列実行ワーカーなど、`exit` イベントが走らずに終わったプロセスは `.lock` を消せません。設計どおりだと、そのプロファイルは永久に「使用中」扱いになります。

そこで、`.lock` に書き込むプロセスIDを使って、**そのプロセスがもう生きていなければ、古い目印として扱う**ようにしました。

```ts
const isStaleLock = async (lockFile: string) => {
  try {
    const pid = Number.parseInt(await fs.promises.readFile(lockFile, 'utf8'))
    if (!Number.isInteger(pid) || pid <= 0) return false

    process.kill(pid, 0) // シグナルは送らず、存在だけ確かめる
    return false
  } catch (e: unknown) {
    // ESRCH：そのPIDのプロセスがない（EPERM は「生きている」）
    return (e as NodeJS.ErrnoException)?.code === 'ESRCH'
  }
}
```

ほかに気をつけた点です。

| 点 | 対処 |
| --- | --- |
| 掃除が、作成直後の他プロセスのプロファイルを消す競合 | 目印の `.lock` を**先に**作ってから、プロファイルを作る |
| 掃除で変換が遅くなる | 掃除は待たずに裏で走らせる（失敗しても変換は続ける） |
| 既存テストが「最初の writeFile は出力ファイル」を前提にしていた | 目印の書き込みは小さいので、同期API（`writeFileSync`）にして非同期の書き込みと分けた |
| プロファイルの無い `.lock` だけが残る | プロセスが死んでいれば、`.lock` も消す |

## 結果

修正版をビルドして、同じ変換を何度も実行しました。

| | 修正前 | 修正後 |
| --- | --- | --- |
| 実行のたびに残るフォルダ | 1つずつ増える | 最後の1回分だけ |
| 残った `.lock` | ー | 0 |
| 手元の Temp の `marp-cli-*` | 241フォルダ・約1.1GB | 次の実行で掃除される |

テストは4件追加し、すべて通っています。全体のテストは、修正前と同じものだけが落ちることを確かめました（Firefox が入っていないなど、手元の環境によるもの）。

PR はこちらです。マージされるかどうかは、作者のレビュー次第です。

https://github.com/marp-team/marp-cli/pull/743

## マージまでの対処

PR が取り込まれるまでは、次のどちらかで凌げます。

**1. 溜まったフォルダを定期的に消す（Windows）**

```powershell
# 1日以上前の marp-cli-* を消す
Get-ChildItem $env:TEMP -Directory -Filter "marp-cli-*" |
  Where-Object { $_.LastWriteTime -lt (Get-Date).AddDays(-1) } |
  Remove-Item -Recurse -Force
```

Linux / macOS なら、こうです。

```bash
find "${TMPDIR:-/tmp}" -maxdepth 1 -type d -name 'marp-cli-*' -mtime +1 -exec rm -rf {} +
```

**2. サーバーモードを使う**

`marp -s <フォルダ>` でサーバーとして起動しておき、HTTP で変換すると、ブラウザを毎回起動し直さないため、フォルダも増えません（作者が Issue で案内している方法）。

## まとめ

- 道具を毎日使っていると、こういう「誰も直していない小さな漏れ」に当たる。
- Issue に作者の設計まで書かれているなら、手元の再現と計測の数字を付けて、直してしまうのが早い。
- 作者の設計に、実際に動かして見つかった穴（強制終了したプロセス）を足すと、PR の説明もしやすくなる。
