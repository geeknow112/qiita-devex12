---
title: 公式ドキュメントゼロ。Claude Codeの秘密テーマ機能を、grep -a十数回・10分のバイナリ解析だけで解明した話
tags:
  - ClaudeCode
  - Claude
  - ReverseEngineering
  - CLI
  - TypeScript
private: false
updated_at: '2026-08-02T01:45:15+09:00'
id: 6d5dab6ec0613afa0c43
organization_url_name: null
slide: false
ignorePublish: false
posting_campaign_uuid: null
agreed_posting_campaign_term: false
---

# きっかけ

Claude Code CLIの入力欄の背景色を紫にしたかった。

`/config` のテーマ設定を見ると、選べるのは `auto` / `dark` / `light` / `light-daltonized` / `dark-daltonized` / `light-ansi` / `dark-ansi` の7つだけ。単色の背景色だけを変える項目はない。

ただし `settings.json` のスキーマを見ると、`theme` フィールドは以下のように定義されていた。

```json
{
  "anyOf": [
    { "enum": ["auto", "dark", "light", ...] },
    { "pattern": "^custom:.*" }
  ]
}
```

`custom:` というプレフィックスを受け付ける形跡がある。だがヘルプにもオンラインドキュメントにも、この仕組みの説明は一切なかった。

# バイナリを直接読むことにした

CLI本体はnpmパッケージとして手元にインストール済みで、実行ファイルはminifyされた単一のJSファイル(exe形式)だった。ソースは公開されていない。

ドキュメントがないなら、動いているものから逆算するしかない。`grep -a`(バイナリをテキストとして読むオプション)で、それらしい識別子を検索した。

```bash
grep -a -o -E "custom[Tt]hemes?[a-zA-Z0-9_./-]*" claude.exe | sort -u
```

```
customThemeRef
customThemes
```

存在した。ここから芋づる式に周辺の文字列を追っていく。

```
getThemesDir:()=>dMt, getCustomThemeBase:()=>Rps,
getCachedCustomThemes:()=>emo, loadCustomThemes:()=>tsr
```

`dMt` という関数の定義を直接grepで引く。

```
function dMt(){return eze.join(pn(),"themes")}
```

`pn()` は設定ディレクトリ(`~/.claude`)を返す関数だった。つまりカスタムテーマの保存場所は `~/.claude/themes/` と判明した。

# ファイル保存処理から構造を割り出す

保存処理の関数もそのまま文字列として埋め込まれていた。

```
function Miy(e){
  return nHe(eze.join(dMt(),`${e}.json`), Piy, {
    defaultValue: () => ({ name: e, base: "dark", overrides: {} }),
    ensureDir: true, indent: 2, trailingNewline: true
  })
}

async function rsr(e){
  await Miy(e.slug).write({ name: e.name, base: e.base, overrides: e.overrides })
}
```

ここで以下が確定した。

- ファイルパス: `~/.claude/themes/<slug>.json`
- 構造: `{ name, base, overrides }`
- `base` は7つの組み込みテーマのいずれかを継承元に指定する

# overridesに書けるキーを特定する

`overrides` に何でも書けるわけではなさそうだった。ロード処理をgrepで追うと、以下のフィルタリングが見つかった。

```
let l = r7(i);
for (let [c, u] of Object.entries(o.overrides))
  if (Object.hasOwn(l, c) && Q$e(u)) a[c] = u;
```

`r7(i)` は継承元テーマ(`base`)の完全なカラーパレットオブジェクトを返す関数。つまり **overridesのキーは、baseにしたテーマが実際に持っているキーだけが有効** という制約だった。存在しないキーを書いても黙って無視される。

では実際のパレットオブジェクトには何があるのか。さらにgrepで実際のオブジェクトリテラルを引き当てた。

```
...clawd_background:"rgb(0,0,0)",
userMessageBackground:"rgb(55, 55, 55)",
userMessageBackgroundHover:"rgb(70, 70, 70)",
composerSidebarBackground:"rgb(38, 38, 38)",
selectionBg:"rgb(38, 79, 120)",
bashMessageBackgroundColor:"rgb(65, 60, 65)",
memoryBackgroundColor:"rgb(55, 65, 70)"...
```

`userMessageBackground` — これがユーザー入力欄の背景色を制御しているキーだった。

# 値のバリデーション形式も特定する

`Q$e` という値検証関数も同じ手順で見つかった。

```
function Q$e(e){
  if (typeof e !== "string") return false;
  if (/^rgb\(\s?\d{1,3},\s?\d{1,3},\s?\d{1,3}\s?\)$/.test(e)) return true;
  if (/^#[0-9a-fA-F]{6}$/.test(e) || /^#[0-9a-fA-F]{3}$/.test(e)) return true;
  if (/^ansi256\(\d{1,3}\)$/.test(e)) return true;
  if (e.startsWith("ansi:")) return ...
}
```

`rgb(r, g, b)` 形式・16進6桁/3桁・`ansi256(n)`・`ansi:` プレフィックスの4種類が許可されている。カンマ後のスペースは0か1個までしか許容しない、という細かい制約もここで判明した。

# 完成したテーマファイル

以上の調査結果から、実際に動く設定を組み立てた。

```json:~/.claude/themes/purple-input.json
{
  "name": "purple-input",
  "base": "dark",
  "overrides": {
    "userMessageBackground": "rgb(88, 28, 135)",
    "userMessageBackgroundHover": "rgb(107, 33, 168)"
  }
}
```

`settings.json` 側は以下のように指定する。

```json
{ "theme": "custom:purple-input" }
```

# まとめ

- ドキュメント・ソースコードが一切公開されていない機能でも、実行バイナリに対する `grep -a` だけで仕様を再現できた
- 使ったコマンドは実質10数回のgrep。所要時間は10分程度
- キーポイントは「保存関数」「ロード時のフィルタリング処理」「値検証関数」の3点を芋づる式に追うこと。単一の巨大なオブジェクトを一気に読もうとせず、処理の入口となる関数名から辿ると効率が良い
- 推測で設定ファイルを書いて壊すより、実際の検証ロジックを先に確認する方が結果的に早い

CLIツールの非公開機能を触るとき、「ドキュメントがないから無理」で終わらせず、動いているバイナリ自体を読みにいく選択肢は覚えておいて損はない。
