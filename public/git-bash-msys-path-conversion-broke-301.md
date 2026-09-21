---
title: Git Bashから渡した「/path」が「C:/Program Files/Git/path」に化けて、本番に壊れた301リダイレクトを2本登録した
tags:
  - Windows
  - GitBash
  - MSYS2
  - bash
  - WordPress
private: false
organization_url_name: null
slide: false
ignorePublish: false
---

## TL;DR

- Git Bashから `node script.js add "/old/path/" "/new/path/"` を実行したら、スクリプトが受け取った値が **`/C:/Program Files/Git/old/path/`** になっていた
- 原因はMSYS2（Git Bash）の**POSIXパス変換**。`/` で始まる引数を「Windowsパスに直してやろう」と変換してから、Windowsネイティブのexe（`node.exe`）へ渡している
- 気づかずAPIを叩いたので、本番のWordPressに**転送元が壊れた301リダイレクトが2本登録された**
- 回避は `MSYS_NO_PATHCONV=1`。あるいは先頭を `//` にする
- **エラーは出ない。** 正常終了して、間違った値が入る。これがいちばん厄介

## 何をしていたか

WordPressで、内容がほぼ同じ記事3本を1本に統合していた。残す1本へ、他の2本から301リダイレクトを張る作業だ。

Redirectionプラグインが入っていたので、管理画面ではなくREST APIから登録することにした。Node.jsで小さなCLIを書いて、Git Bashから叩く。

```bash
node redirect.js add "/investigation/old-article/" \
                     "/investigation/new-article/"
```

スクリプト側はこうなっている。

```js
const [, , cmd, from, to] = process.argv;

const payload = {
  url: from,                    // 転送元
  action_data: { url: to },     // 転送先
  match_type: 'url',
  action_type: 'url',
  action_code: 301,
  group_id: 1,
  status: 'enabled',
};
```

HTTPは200。エラーなし。登録できたように見えた。

## 登録されたものを見て気づく

念のため一覧を取り直した。

```
60 301 /C:/Program Files/Git/investigation/old-article-2/
       -> C:/Program Files/Git/investigation/new-article/   hits=0 enabled=true
59 301 /C:/Program Files/Git/investigation/old-article/
       -> C:/Program Files/Git/investigation/new-article/   hits=0 enabled=true
```

渡したはずの `/investigation/...` が、`/C:/Program Files/Git/investigation/...` になっている。

Git Bashのインストール先が、そのまま前に付いていた。

## 原因：MSYSのPOSIXパス変換

MSYS2（Git Bashの土台）は、**Windowsネイティブのプログラムへ引数を渡すとき、POSIXパスに見える文字列をWindowsパスへ変換する**。

`node.exe` はMSYSのプログラムではないので、この変換の対象になる。

```bash
# Git Bash 上で
$ node -e 'console.log(process.argv[1])' "/investigation/foo/"
C:/Program Files/Git/investigation/foo/
```

`/investigation/foo/` は、MSYSから見れば「ルート直下のパス」だ。MSYSのルートはインストールディレクトリ（`C:\Program Files\Git`）なので、そこを起点にWindowsパスへ直している。**親切心である。**

引っかかるのは、パスのつもりがない文字列も対象になる点だ。

```bash
$ node -e 'console.log(process.argv[1])' "/api/v2/posts"
C:/Program Files/Git/api/v2/posts

$ node -e 'console.log(process.argv[1])' "/c=1"
/c=1                      # = が含まれると変換されない

$ node -e 'console.log(process.argv[1])' "//investigation/foo/"
/investigation/foo/       # 先頭を // にすると変換されない
```

REST APIのパス、URLのパス部分、cron式、正規表現、S3のプレフィックス。`/` で始まるものは全部候補になる。

## 回避方法

### 1. `MSYS_NO_PATHCONV=1`（推奨）

そのコマンドだけ無効にする。

```bash
MSYS_NO_PATHCONV=1 node redirect.js add "/investigation/old/" "/investigation/new/"
```

スクリプト全体で無効にするなら、先頭で `export` する。

```bash
#!/bin/bash
export MSYS_NO_PATHCONV=1
```

### 2. 先頭を `//` にする

```bash
node redirect.js add "//investigation/old/" "//investigation/new/"
```

変換は止まるが、**受け取り側に `//` がそのまま渡る**。スクリプト側で1つ落とす処理が要る。API仕様によっては、これが別の不具合になる。

### 3. そもそも引数で渡さない

いちばん確実なのはこれだった。JSONファイルに書いて、ファイルパスだけ渡す。

```bash
node redirect.js add ./redirects.json
```

ファイルパスは変換されても正しく解決されるので、事故らない。**パスに見える文字列を引数に置かない**のが根本対策になる。

## 同じ罠を踏む場所

| コマンド | 化ける引数 |
|---|---|
| `docker run -v /data:/data` | コンテナ側のパス |
| `aws s3api ... --prefix /logs/` | プレフィックス |
| `gh api /repos/:owner/:repo` | エンドポイントのパス |
| `openssl req -subj "/CN=example"` | `-subj` の値 |
| `kubectl ... --path=/healthz` | パス指定 |

`openssl` の `-subj` は昔から知られているが、**原因が同じだと知らないと、別の問題に見える。**

## いちばんの問題は「成功すること」

この事故の質が悪いのは、**どこもエラーにならない**点だ。

- Git Bash: 変換して渡すだけ。警告なし
- Node: 受け取った文字列をそのまま使う。異常なし
- REST API: 文字列としては妥当。**HTTP 200 を返す**
- プラグイン: 指定どおりのルールを作る。仕事をしている

全員が正しく動いて、結果だけが間違っている。ログを見ても気づけない。

今回たまたま気づけたのは、**登録直後に一覧を取り直したから**だった。書き込み系のAPIを叩いたら、レスポンスのステータスではなく、**入ったものを読み直して確認する**。これに尽きる。

## 後始末

壊れたルールは、同じIDに正しい値を入れて上書きした。

```bash
export MSYS_NO_PATHCONV=1
node redirect.js fix 59 "/investigation/old-article/"   "/investigation/new-article/"
node redirect.js fix 60 "/investigation/old-article-2/" "/investigation/new-article/"
```

削除して作り直すとIDが変わり、既存ルールの並び順（優先度）に影響する可能性があったため、上書きを選んだ。

確認は `curl` で。

```bash
$ curl -s -o /dev/null -w "%{http_code} -> %{redirect_url}\n" \
    "https://example.com/investigation/old-article/"
301 -> https://example.com/investigation/new-article/
```

## まとめ

- Git Bashは、Windowsネイティブのexeへ渡す引数のうち、`/` で始まるものをWindowsパスへ変換する
- 変換されると `C:/Program Files/Git` が前に付く。**エラーは出ない**
- 回避は `MSYS_NO_PATHCONV=1`、先頭 `//`、または引数にパス形式の文字列を置かない
- 書き込み系のAPIは、200が返っても信用しない。**入ったものを読み直す**
