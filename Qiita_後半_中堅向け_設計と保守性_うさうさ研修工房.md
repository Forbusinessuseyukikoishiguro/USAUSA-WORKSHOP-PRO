# 【後半・中堅向け】HTML一枚の実務ツールを“設計”で読み解く ― 文書生成・5形式出力・AI連携・保守性

> タグ: `JavaScript` `設計` `フロントエンド` `AI` `リファクタリング`

うさうさ先生🐰です。[前半（新人向け）](#)では、HTML一枚で動く全体像と「定義 → 状態 → 画面 → 文章」の一本道を解説しました。

後半（中堅向け）では、**設計判断**に踏み込みます。文書生成エンジンの構造、1つの中間表現から5形式に書き出すエクスポート、AIを“任意・透明・安全”に組み込む作法、そして保守性・可読性の勘どころを、原則名と参考文献つきで整理します。

:::note info
**この記事で分かること（後半）**
- 1ディスパッチ＋ビルダー群による文書生成エンジンの構造
- 共通の中間表現から5形式へ書き出す DRY なエクスポート設計
- AIを“任意・透明・BYOK”で組み込むときの実装上の勘どころ
- データ駆動・単方向・関心の分離が効く理由（原則名つき）
:::

## 🦅 対象読者

- 小さなツールを「動く」だけでなく「変えやすく・読みやすく」したい中堅エンジニア
- 設計判断のトレードオフを言語化したい方
- AIをプロダクトに“任意機能”として安全に組み込みたい方

## 🧩 おさらい：データ駆動という土台

前半で触れたとおり、このツールは「どんな文書を・どんな項目で作るか」を定義データ（`VARIANTS` / `META` / `FIELDS` / `SAMPLES`）として宣言します。これにより、**新しい文書タイプの追加は、原則としてデータを追記するだけ**で済みます。

```js
// 「日報（簡易）」を足す例：ロジックは触らない
VARIANTS.report.push({ v: "nippo", label: "日報（簡易）" });
META.report.nippo  = { title: "日報", aim: "今日の要点を3行で" };
FIELDS.report.nippo = [ /* 件名・やったこと・所感… */ ];
// 専用の文面が不要なら、汎用ビルダー(buildGeneric)が out 指定で整形する
```

これは **Open/Closed 原則**（拡張に開き、修正に閉じる）の素朴な実践です。`switch` の分岐を増やすのではなく、データの宣言を増やす。新タイプ追加で既存ロジックを触らないので、回帰リスクが小さくなります。

> 参考: Meyer, B. (1988) *Object-Oriented Software Construction*（Open/Closed原則）／ Martin, R.C. *Clean Architecture* (2017)

## 📝 文書生成エンジン：1ディスパッチ＋ビルダー群

完成文を組み立てる中心は `buildDoc()` です。選ばれているタイプに応じて専用ビルダーへ振り分け、専用が無いタイプは汎用ビルダーが整形します。

```js
function buildDoc() {
  if (lesson === "report") {
    if (variant === "naibu")    return buildNaibu();    // 社内（専用）
    if (variant === "shagai")   return buildShagai();   // 社外（専用）
    if (variant === "original") return buildOriginal();
    return buildGeneric(fieldList());                   // 障害/研修/SES… は汎用
  }
  if (variant === "step")     return buildStep();       // 自分用Step
  if (variant === "original") return buildOriginal();
  return buildManual();                                 // ISO手順書
}
```

各ビルダーは「値があるときだけ見出しと本文を積む」だけのシンプルな関数です。ガード節（早期 return）でネストを浅く保っているのもポイントです。

```js
function sec(L, label, val, mode) {
  if (!val) return;                 // ガード節：空ならスキップ
  L.push("■ " + label);
  lines(val).forEach(x =>           // 小さな関数を合成
    L.push(mode === "bullets" ? "・" + x : "　" + x));
  L.push("");
}
```

手順書では `(i + 1) + ". "` で自動採番し、自分用 Step では予定/実績を数値で受け取って合計と差を算出します。こうした“気が利く整形”が、現場での使い心地を決めます。

> 参考: Fowler, M. (2018) *Refactoring* 2nd ed.（Replace Nested Conditional with Guard Clauses）／ McCabe, T.J. (1976) "A Complexity Measure", *IEEE TSE* SE-2:308–320（循環的複雑度）

## 📦 エクスポート：1つの中間表現から5形式へ（DRY）

ここが設計のいちばんの勘どころです。完成文を一度だけ `previewToBlocks()` で構造化（タイトル/見出し/リスト/段落のブロック列）し、**共通の素から全形式を作ります**。だから5形式の出力がブレません。

```js
function previewToBlocks(text) {
  const lines = text.split("\n");
  const title = lines[0];
  const blocks = [];
  let list = null;
  for (const raw of lines.slice(1)) {
    const s = raw.replace(/\u3000/g, " ").trimEnd();
    if (s.startsWith("■")) blocks.push({ t: "h", text: s.slice(1).trim() });
    else if (s.startsWith("・")) { (list ??= { t: "list", items: [] }).items.push(s.slice(1)); }
    else { if (list) { blocks.push(list); list = null; } if (s) blocks.push({ t: "p", text: s }); }
  }
  if (list) blocks.push(list);
  return { title, blocks };
}
```

各エクスポータは、この共通ブロック列を入力にします。

| 形式 | 実装メモ |
| --- | --- |
| Markdown | 見出し→`##`、リスト→`- ` |
| Word(.doc) | Office名前空間つきHTMLを Word として保存（**厳密なOOXMLではない**） |
| PDF | 印刷用領域に流して `window.print()`（`@media print`） |
| CSV | 先頭に `\uFEFF`（BOM）＋CRLF。Excelで開いても文字化けしない |
| Excel(.xls) | 表形式HTMLを Excel として保存 |

保存はすべて端末内です。`Blob` を作って `URL.createObjectURL` でリンク化し、`<a download>` を擬似クリックします（外部送信なし）。

:::note warn
⚠️ **正直な補足**
Word/Excel出力は「HTMLを `.doc` / `.xls` として保存」する方式で、厳密な OOXML ではありません。依存ゼロで互換性を確保する代わりに、高度な書式は持てません。割り切りとして理解しておくと、後で「なぜ罫線が…」と悩まずに済みます。
:::

> 参考: Hunt, A. & Thomas, D. (1999) *The Pragmatic Programmer*（DRY 原則）

## 🤖 AIを“任意・透明・安全”に組み込む

AI整形は「あったら便利」の追加機能です。設計の前提として、**無くても本体は動くこと**、**送る内容を利用者が確認できること**を最初に決めています。

送信前に①本文が短い→中断、②鍵が空→中断、というガードを入れ、③プロンプトを組んで送信します。送信先は1か所だけです。

```js
const res = await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "x-api-key": key,                  // 利用者が入力（保存しない＝BYOK）
    "anthropic-version": "2023-06-01",
    "anthropic-dangerous-direct-browser-access": "true"
  },
  body: JSON.stringify({
    model, max_tokens: 1500,
    messages: [{ role: "user", content: buildPrompt(mode) }]
  })
});
const data = await res.json();
// 結果は“位置でなく型”で取り出す（将来ブロックが増えても壊れにくい）
const text = (data.content || [])
  .filter(b => b.type === "text")
  .map(b => b.text).join("\n");
```

:::note info
🦅 **中堅向けポイント：レスポンスは型で取り出す**
`data.content[0].text` のように先頭固定で取らず、`filter(b => b.type === "text")` で取り出します。`tool_use` などが混ざっても壊れにくい、地味だけど大事な作法です。
:::

:::note warn
⚠️ **ブラウザ直送＋BYOKの鍵管理**
`anthropic-dangerous-direct-browser-access` はブラウザから直接APIを叩く方式です。鍵をコードに埋め込んで配布すると盗用されえます。このツールは「鍵を保存せず、その場で入力、リロードで消える」設計で回避しています。配布時は利用上限つきの鍵を勧める運用を徹底しましょう。
:::

エラーは原因を伏せて1つの安全な日本語メッセージに集約し、`finally` でボタンを再有効化しています。AIには文体を整えてもらうだけで、数字や固有名を変えないようプロンプトで指示し、出力は必ず人が目視確認する——この運用を前提にしているのも、安全側に倒した設計判断です。

## 🔒 セキュリティ・プライバシー（事実ベース）

- HTML化する出力経路（Word/Excel/印刷）では入力値を `esc()`（`& < > "`）でエスケープ。プレビューは `textContent` 代入でHTMLとして解釈させない
- 永続化（`localStorage` 等）を使わない。入力は端末内のみで、リロードで消える
- 外部送信はAI使用時の1か所だけ。送る内容は「プロンプトを見る」で事前に確認できる

```js
const esc = s => s.replace(/&/g, "&amp;").replace(/</g, "&lt;")
                  .replace(/>/g, "&gt;").replace(/"/g, "&quot;");
```

## 🧭 設計の勘どころ（原則名で整理）

| 観点 | このツールでの現れ方 | 原則 |
| --- | --- | --- |
| 保守性 | 型をデータ追記で増やす | Open/Closed |
| 保守性 | 状態は store の1か所（単方向） | Single Source of Truth |
| 保守性 | データと振る舞いを分離 | 関心の分離（SoC） |
| 保守性 | 共通の中間表現で重複排除 | DRY |
| 保守性 | 核は純粋関数（esc/previewToBlocks…） | 参照透過性・テスト容易性 |
| 可読性 | 動詞＋対象の命名（buildNaibu…） | 命名 |
| 可読性 | ガード節でネストを浅く | 複雑度の低減 |

> 参考: Dijkstra, E.W. (1974) "On the role of scientific thought"（関心の分離）／ Martin, R.C. (2008) *Clean Code*（命名）／ Buse, R. & Weimer, W. (2010) "Learning a Metric for Code Readability", *IEEE TSE* 36(4):546–558

## 🧱 改善余地（正直に）

単一HTML・依存ゼロは強みですが、引き換えもあります。

- 純粋関数のユニットテストを独立ファイルへ外部化したい
- CSSの多重定義（`:root` の再定義）を最終テーマへ一本化したい
- 純粋な生成と DOM 結線（副作用）の分離をもう一段進めたい
- AI失敗の原因別表示（認証/通信/モデル）を足したい

「何を捨てて何を得たか」を言語化しておくと、次の改修やレビューでブレません。改善余地は欠点ではなく、保守性を上げる地図です。

> 参考: Cunningham, W. (1992) "The WyCash Portfolio Management System", OOPSLA（技術的負債）

## 🎁 後半のまとめ

- 文書生成は **1ディスパッチ＋ビルダー群**。ガード節で素直に積む
- エクスポートは **共通の中間表現から5形式**（DRYで出力がブレない）
- AIは **任意・透明・BYOK**。レスポンスは型で取り出し、鍵は保存しない
- 設計の芯は **データ駆動 × 単方向 × 関心の分離**。原則名で言語化できる

小さなツールでも、設計の芯が一本通っていれば、新人にもやさしく、中堅にも拡張しやすいものになります。

:::note info
**◀ 前半（新人向け）もあわせてどうぞ**
全体像と「動く仕組みの土台」は前半で解説しています。これから読む方は前半からがおすすめです。
:::

最後まで読んでいただきありがとうございました。

*面白きこともなき世を面白く（高杉晋作）— うさうさ先生🐰*

---

> 本記事の技術的な記述は、ツールのソースコードで確認できる実装にもとづいています。フッターに記載の「商用レベルテスト済」等のツール自身の主張については、本記事では独自に検証していません。
