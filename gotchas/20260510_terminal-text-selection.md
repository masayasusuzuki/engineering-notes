# ターミナル CLI でテキスト範囲選択を実装する — ANSI / grapheme / 座標変換の格闘記

| 項目 | 内容 |
|------|------|
| 日付 | 2026-05-10 |
| 対象プロジェクト | [CHACRAM](https://github.com/masayasusuzuki/chacram) — Anthropic Claude Code 派生の OSS TUI（OpenClaude をさらにフォーク） |
| 主要コミット | [`9be5e14`](https://github.com/masayasusuzuki/chacram/commit/9be5e14)（パイプライン全体） / [`8f8659b`](https://github.com/masayasusuzuki/chacram/commit/8f8659b)（ANSI レンダリング修正） |
| 設計書 | [`docs/architecture/text-selection.md`](https://github.com/masayasusuzuki/chacram/blob/main/docs/architecture/text-selection.md) |

> **このドキュメントの位置付け**: Claude Code 系の TUI に「マウスドラッグでプロンプト入力欄のテキストを範囲選択して削除／コピー」を実装した時の全記録。同じく Ink ベースの CLI を作っている人、ターミナルで grapheme と ANSI が絡む UI を組んでいる人、エディタ系 UI 実装者向け。本文では対象プロダクトを「対象 CLI」と表記し、再現性のある技術知見に焦点を当てる。

---

## 0. 用語の前置き

このドキュメントを技術背景の薄い人でも追えるよう、出てくる用語を最初に固める。

| 用語 | 意味 |
|------|------|
| **TUI** | Terminal User Interface。ターミナル上で動く GUI 風アプリの総称 |
| **Ink** | React を CLI に流し込むレンダラ。`<Box>` `<Text>` 等を JSX で書ける。対象 CLI の UI 基盤 |
| **CSI** | Control Sequence Introducer。`\x1b[` で始まり英字で終わる ANSI エスケープシーケンスのカテゴリ |
| **SGR** | Select Graphic Rendition。CSI のうち色・反転・太字等を制御するもの。`\x1b[7m`（反転 ON）/ `\x1b[27m`（反転 OFF）が今回の主役 |
| **インバース／反転** | 文字色と背景色を入れ替える表示。SGR-7。テキスト選択のハイライトはこれで描画する |
| **grapheme（書記素）** | 「ユーザーが見て1文字に見える単位」。例：👨‍👩‍👧‍👦 は4つのコードポイントの合成だが grapheme としては1文字 |
| **`Intl.Segmenter`** | ECMAScript 標準。文字列を grapheme / word / sentence 単位で分割するイテレータを返す |
| **alt-screen** | Alternate Screen Buffer。`\x1b[?1049h` で入る「フルスクリーンモード」。`vim` や `less` が使うやつ。Ink のマウスイベントもこのモードでしか取れない |
| **SGR mouse tracking** | ターミナルにマウス座標を SGR 形式で送らせるプロトコル（`\x1b[?1006h`）。Ink がこれを使ってマウス選択を取る |
| **`Cursor.selection`** | 対象 CLI で導入した概念。`Cursor` クラス内のフィールドで、選択範囲のアンカーを保持する |

---

## 1. 背景: なぜこの機能が無かったのか

### 1.1. 系譜

対象 CLI のコードベースは3段階を経ている。

```
Anthropic Claude Code (上流・閉源)
        ↓ fork（OpenAI 互換シム追加）
OpenClaude (gnanam1990 系の OSS フォーク)
        ↓ fork & rebrand
対象 CLI（本ドキュメントの作業対象）
```

`git log` で系譜の節目が見える:

| commit | 意味 |
|--------|------|
| `d2542c9 asdf` | OpenClaude の最初期コミット（Claude Code から bundling 抜き出し） |
| `619b5fb feat: add OpenAI-compatible provider shim` | OpenAI 互換 API を Claude Code に喋らせるシム |
| `2d7aa9c feat: rebrand as Open Claude` | OpenClaude へリブランド |
| `9be5e14 feat: add text selection (mouse drag) pipeline` | 今回の機能追加 |
| `8f8659b fix(Cursor.render): preserve selection inverse...` | レンダリング修正 |

### 1.2. なぜ Claude Code には範囲選択削除が無かったか

Claude Code はもともとプロンプト入力欄を「シングル行的なテキスト編集」として設計していた。`Cursor` クラスにはカーソル位置 `offset` があるだけで、範囲を表現する手段がそもそも無かった。

**コアの非対応点**（`9be5e14` 直前の状態）:

1. `Cursor` クラスに `selection: number = 0` というフィールドはあったが、**0 は「位置 0 を選択」と「選択なし」の区別がつかず、実質未使用**
2. 範囲選択を処理するメソッド（`hasSelection()`／`selectionStart()`／`selectionEnd()`／`deleteSelection()` など）が**1つも無い**
3. レンダラ `render()` は単一カーソルしか描画しない。**選択範囲を反転表示するロジックが無い**
4. 入力欄の `useTextInput` フックは Shift+矢印を「カーソル移動 with shift」として無視する。**選択拡大の概念が無い**
5. マウスドラッグ → 選択は Ink の出力エリア（会話ログ）専用。**入力欄では機能しない**

OpenClaude 段階でもここは触られていなかった（エージェント・プロバイダ周りに集中していた）。本作業で初めて手を入れた領域。

### 1.3. なぜ「やらない」設計だったかの妥当性

ターミナルでテキスト範囲選択は**実装難度が高い**。理由:

- ターミナルは「文字セルの集合」であって「テキストノード」ではない。文字列とのマッピングを自前で持つ必要がある
- フォント幅が等幅とは限らない（CJK の全角は 2 セル、絵文字は 2 セル、ZWJ 結合済みは 1 セル）
- グラフェム単位（👨‍👩‍👧‍👦）を1文字として扱わないと、Backspace で家族絵文字が壊れる
- マウスイベントは alt-screen（フルスクリーンモード）でしか取れない。スクロール可能なログのある環境では制約多
- ANSI エスケープ（色・反転）と地のテキストが同居するため、文字列スライス操作と表示位置が常にずれる

これらを全部受けに行く設計が今回のテキスト選択パイプライン。

---

## 2. 設計の幹: なぜ Cursor.selection を Single Source of Truth にしたか

### 2.1. 候補だった2つの設計

**案 X — Ink の selection.ts に乗っかる**

Ink にはそもそも `useSelection()` というフックがあって、出力エリア（会話ログ）でマウスドラッグ → コピー、を扱っている。これを入力欄にも流用する案。

却下理由:
- Ink の selection は**画面セル単位**の選択。文字列内の「offset 5..10」みたいなアプリケーション概念に変換するロジックが無い
- 削除・置換のような編集操作と相性が悪い（Ink は基本「読み取りコピー」用途）
- 入力欄のテキストは React state で持っているが、Ink selection の「画面セル選択」とテキストマッチで結びつけると、同じ文字列が2回出てきた時に曖昧になる（CJK・全角・絵文字でさらに歪む）

**案 Y — Cursor クラスに selection 状態を持たせる**（採用）

`Cursor` は元々カーソル位置 `offset` を持っていて、文字列との対応関係は厳密。ここに「アンカー offset」を一個増やすだけで「`offset` と `anchor` の間が選択」と表現できる。

採用理由:
- 文字列オフセット同士なので CJK・絵文字・ANSI 影響なし
- キーボード選択（Shift+Arrow）は anchor を固定して offset だけ動かせば素直に書ける
- 削除・コピーは `text.slice(min, max)` で済む
- マウスは「画面座標 → offset 変換」さえ用意すれば、同じ Cursor.selection に流し込める

### 2.2. 責任境界の言語化

設計書で明文化したルール:

```
┌─ 出力エリア（会話ログ）────────┐
│  選択方式: Ink selection.ts    │  ← マウスドラッグ→コピーのみ
└────────────────────────────────┘

┌─ 入力エリア（プロンプト）──────┐
│  選択方式: Cursor.selection    │  ← 削除/コピー/置換
│  マウス→offset: render() 時に  │
│  構築した逆引きマップで変換    │
└────────────────────────────────┘
```

**両者をブリッジしない**。テキストマッチで結びつけない。Ink 選択は出力エリアのテキスト → クリップボード、Cursor 選択は入力欄の文字列操作。それぞれ独立に動かす。

---

## 3. 出発点 — 元の Cursor.ts にあった「形だけのフィールド」

`9be5e14` 直前の `src/utils/Cursor.ts` を抜粋:

```typescript
export class Cursor {
  readonly offset: number
  constructor(
    readonly measuredText: MeasuredText,
    offset: number = 0,
    readonly selection: number = 0,    // ← 形だけある
  ) {
    this.offset = Math.max(0, Math.min(this.text.length, offset))
  }
  // hasSelection / selectionStart / selectionEnd / deleteSelection ... なし
  // renderLineWithSelection ... なし
  // renderWithMapping ... なし

  render(cursorChar, mask, invert, ghostText, maxVisibleLines) {
    // カーソル1個を描くだけ。selection は完全無視。
    ...
  }
}
```

`selection: number = 0` というフィールドは存在するが、**0 が「位置 0 を選択している」のか「選択していない」のかの区別がつかない**。コンストラクタ以外で書き換えるコードもメソッドも無い。読まれもしない。

これを直すのが第一歩。

---

## 4. Phase 1: Sentinel の意味付け + キーボード選択

### 4.1. selection の sentinel 化

**変更**: `selection` のデフォルトを `0` → `-1` にし、「`-1` = 選択なし、それ以外 = アンカー offset」と意味付け。

```diff
- readonly selection: number = 0,
+ /** -1 = no selection. Non-negative and ≠ offset = selection anchor active. */
+ readonly selection: number = -1,
```

なぜ `null | undefined` ではなく `-1`?
- TypeScript の Optional は immutable update（`new Cursor(measuredText, offset, anchor)`）でユニオン書き分けが面倒
- `number` 一本で扱えれば算術がそのまま使える（`Math.min(offset, selection)` 等）
- `-1` は文字列オフセットが取らない値なので衝突しない

### 4.2. 選択判定・取得ヘルパ

```typescript
hasSelection(): boolean {
  return this.selection >= 0 && this.selection !== this.offset
}
selectionStart(): number { return Math.min(this.offset, this.selection) }
selectionEnd():   number { return Math.max(this.offset, this.selection) }
selectedText():   string {
  return this.hasSelection() ? this.text.slice(this.selectionStart(), this.selectionEnd()) : ''
}
deleteSelection(): Cursor {
  if (!this.hasSelection()) return this
  const start = this.selectionStart()
  const end = this.selectionEnd()
  const newText = this.text.slice(0, start) + this.text.slice(end)
  return new Cursor(new MeasuredText(newText, this.measuredText.columns), start, -1)
}
```

これで「アプリケーション層」では `cursor.hasSelection()` `cursor.deleteSelection()` `cursor.selectedText()` が使える。

### 4.3. Shift+Arrow（キーボード選択拡大）

`useTextInput.ts` の `mapKey()` 内で、Shift 修飾子付きの矢印キーをハンドル:

```typescript
// 疑似コード
if (key.shift && key.name === 'right') {
  const anchor = cursor.hasSelection() ? cursor.selection : cursor.offset
  const next = cursor.right()  // 1 文字右へ
  return new Cursor(measuredText, next.offset, anchor)
}
```

**ポイント**: アンカーは「選択開始時の offset」を保持する。Shift を押し始めた瞬間に anchor を `cursor.offset` で固定し、その後 Shift+矢印を押すたびに `offset` だけが動く → `anchor !== offset` になり選択範囲ができる。Shift を離してから別キーを押すと anchor を `-1` に戻す。

### 4.4. 削除・コピーの結線

- **Backspace / Delete**: `cursor.hasSelection()` なら `cursor.deleteSelection()` を呼ぶ。なければ通常の1文字削除
- **Ctrl+C**: `cursor.hasSelection()` なら `cursor.selectedText()` をクリップボードに送る（`copyToClipboard` ユーティリティ経由）

ここまでで「キーボードによるテキスト選択 → 削除／コピー」が完成。

---

## 5. Phase 2: マウスドラッグ選択

ここが本番。「画面座標」と「文字列オフセット」のあいだに翻訳機を作る作業。

### 5.1. 描画と同時に逆引きマップを返す（renderWithMapping）

**問題**: マウスは画面の `(行, 列)` を返す。Cursor は文字列の `offset` で選択を持つ。両者を変換する関数がいる。

**naive な解**: `cursor.text` を毎回スキャンして `(行, 列)` から offset を計算する。
**問題**: viewport scroll、折り返し継続行の先頭空白トリム、CJK の全角幅、ANSI 制御文字の不可視幅、すべてを再現しないといけない。`render()` が既に持っている計算をもう1回やることになる。

**採用した解**: `render()` のレンダリングと**同時に**、画面座標→offset 変換のクロージャを返す `renderWithMapping()` を生やす。

```typescript
renderWithMapping(
  cursorChar: string,
  mask: string,
  invert: (text: string) => string,
  ghostText?: { text: string; dim: (text: string) => string },
  maxVisibleLines?: number,
): {
  text: string
  offsetAt: (viewportRow: number, viewportCol: number) => number | null
} {
  const text = this.render(cursorChar, mask, invert, ghostText, maxVisibleLines)
  const startLine = this.getViewportStartLine(maxVisibleLines)
  const totalLines = this.measuredText.getWrappedLines().length
  const measuredText = this.measuredText  // クロージャ用キャプチャ

  const offsetAt = (viewportRow, viewportCol) => {
    if (viewportCol < 0) return null
    const absoluteLine = startLine + viewportRow
    if (absoluteLine < 0 || absoluteLine >= totalLines) return null
    return measuredText.getOffsetFromPosition({
      line: absoluteLine,
      column: viewportCol,
    })
  }

  return { text, offsetAt }
}
```

**効くポイント**:
- `MeasuredText.getOffsetFromPosition({ line, column })` が既に表示幅→文字列インデックス・行頭空白・CJK・絵文字を全部考慮して書かれている。流用する
- `render()` が持っている viewport 情報（`startLine`）をクロージャで閉じ込めるだけで、変換が `O(1)` 近似（実際は1行内の幅累積で `O(line_length)`）
- 出力テキストは触らないので既存のレンダリング挙動はゼロ変更

### 5.2. テスト先行

[`src/__tests__/cursor-render-mapping.test.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/__tests__/cursor-render-mapping.test.ts) を 28 test で先に通した:
- 単一行
- 複数行（改行）
- 折り返し（カラム数より長い行が継続行に分かれる）
- CJK（全角幅）
- viewport scroll（スクロール下方向で `(0,0)` が文字列の途中を指す）
- 範囲外（負の row、line.length 超過、空行）
- 選択は offsetAt の結果に影響しない（pure function 性）

これがあると次の Phase で「マウスから来た座標が間違ってるのか、変換ロジックが間違ってるのか」の切り分けが効く。

### 5.3. Ink の useSelection() で drag-end を取る

設計時の想定 → **Ink は React 層に raw mousedown/mousemove/mouseup を露出していない**。`ClickEvent` は単発クリック専用、ドラッグはドラッグでフックが分かれる。

実装で使ったのは `useSelection()` フック。Ink の出力エリア用の選択メカニズムだが、`subscribe()` で「選択状態が変化した時」のコールバックを取れる。

```typescript
// src/hooks/useTextInput.ts
const inkSel = useSelection()
useEffect(() => {
  if (!focus) return
  let prevDragging = false
  const unsub = inkSel.subscribe(() => {
    const state = inkSel.getState()
    if (!state) return
    const isDragging = state.isDragging
    const dragJustEnded = !isDragging && prevDragging
    prevDragging = isDragging

    if (!dragJustEnded) return
    if (!state.anchor || !state.focus) return
    // ... 座標変換 ...
  })
  return unsub
}, [focus, inkSel, ...])
```

**狙い**:
- `dragJustEnded` の瞬間（ドラッグ中の transition 終わり）で1回だけ処理する
- `state.anchor`（ドラッグ開始の絶対画面座標）と `state.focus`（終了座標）が両方揃ったタイミングで反映
- ドラッグ中は Ink が画面レベルで反転表示してくれる（後で `inkSel.clearSelection()` で剥がして Cursor の反転に引き継ぐ）

### 5.4. 座標変換パイプライン全段

```
[Ink useSelection]
  state.anchor.row, state.anchor.col  ← 絶対画面座標（ターミナル左上原点）
  state.focus.row,  state.focus.col

      ↓ 入力 Box の左上座標で減算

  relAnchorRow = state.anchor.row - box.y
  relAnchorCol = state.anchor.col - box.x
  relFocusRow  = state.focus.row  - box.y
  relFocusCol  = state.focus.col  - box.x
                              ← Box 相対座標

      ↓ viewport scroll 加算

  anchorLine = relAnchorRow + viewportStartLine
  focusLine  = relFocusRow  + viewportStartLine
                              ← 折り返し済み行番号（絶対）

      ↓ MeasuredText.getOffsetFromPosition で文字列 offset へ

  anchorOffset = measuredText.getOffsetFromPosition({ line: anchorLine, column: relAnchorCol })
  focusOffset  = measuredText.getOffsetFromPosition({ line: focusLine,  column: relFocusCol })

      ↓ Cursor 選択に反映

  updateRenderedInput(value, focusOffset, anchorOffset)
  // → Cursor.offset = focusOffset, Cursor.selection = anchorOffset
```

この全段で「画面の (絶対 row, col)」が「文字列の offset」に翻訳される。

### 5.5. Box の画面座標を取る

`box = { x, y }` は入力 Box の左上の絶対画面座標。これを取らないと相対座標が出ない。

最初の実装では `handleInputClick` で `e.col - e.localCol` を使って取得していた:
```typescript
// e は Ink の ClickEvent
// e.col = 絶対画面座標
// e.localCol = Box ローカル座標
// → e.col - e.localCol = Box の左 x
```

**問題**: ClickEvent は**ドラッグなしクリック**でしか発火しない。最初のマウスドラッグ時には `selectionSyncRef.current` が `null` のまま → 変換失敗 → 選択が反映されない。

**修正**: `useEffect` + `inputBoxRef` + Ink の `nodeCache` で、毎レンダー後に Box 位置を取得する経路を追加。

```typescript
useEffect(() => {
  if (!inputBoxRef.current) return
  const { x, y } = nodeCache.getPosition(inputBoxRef.current)
  selectionSyncRef.current = { x, y }
})
```

これで「ドラッグ前にクリックしてないと動かない」を解消。

---

## 6. 詰まったバグと修正の全記録

ここからが本ドキュメントの本番。実装中に踏んだバグを実物コードで残す。

### 6.1. Bug 1: onOffsetChange ↔ useLayoutEffect 競合

**症状**: マウスでドラッグ選択 → 一瞬選択範囲が反転表示される → **直後に勝手にリセットされる**

**原因の連鎖**:
1. マウス選択 sync が `updateRenderedInput(value, focusOffset, anchorOffset)` を呼ぶ
   → `Cursor.selection = anchorOffset, Cursor.offset = focusOffset`（選択ON）
2. 直後に `onOffsetChange?.(focusOffset)` を呼ぶ → 親の `externalOffset` が変わる
3. `useLayoutEffect` が `externalOffset` の変更を検知
4. `useLayoutEffect` 内で `updateRenderedInput(originalValue, externalOffset, /* anchor */ -1)` を実行
5. **`Cursor.selection = -1` で選択が消える**

つまりマウス選択処理の中で自分自身がトリガーした offset 変更が、別経路で選択を上書きしていた。

**修正**: `updateRenderedInput` 内で `lastSeenPropsRef` も同期更新する。

```typescript
// 疑似コード
function updateRenderedInput(value, offset, anchor) {
  setCursor(new Cursor(measuredText, offset, anchor))
  // ↓ これが追加された行
  lastSeenPropsRef.current = { value, offset }
  // → 次の useLayoutEffect が「props と一致してる」と判定して上書きをスキップ
}
```

`lastSeenPropsRef` は元々「親 props と当コンポーネントの内部状態が同期済みかどうか」を判定するためのフラグ。`updateRenderedInput` が同期した直後に props 由来の `useLayoutEffect` が空走して選択を消す、を防ぐ。

**教訓**: React の useLayoutEffect は同フレーム内で発火する。1つのイベントで「内部状態を更新→親に通知」を両方やる時、親→子の同期ループに巻き込まれて自分の更新を打ち消す。
**対策パターン**: 親通知の前に「直近の親 props は何だったか」を ref で記録して、useLayoutEffect 側で「変わってない」と判定させる。

### 6.2. Bug 2: 初回ドラッグで Box 位置が null

5.5 で書いた話。`handleInputClick` 経由でしか Box 位置を取らない設計だったので、最初のドラッグが効かなかった。`useEffect` + `nodeCache` で毎レンダー取得する経路を併設して解決。

**教訓**: クリック前提のフックで「初回」が抜ける典型パターン。ライフサイクル系の値（DOM 位置・レイアウト寸法）は **render 後に常に更新**しておくのが安全。クリックは「最新化のトリガー」として補完的に使う。

### 6.3. Bug 3: ANSI escape 分断による `[7m` 露出

**症状**: マウスでテキストを選択すると、選択範囲付近に **`[7m`** という生テキストが見える。

**原因**:

`Cursor.render()` 内のカーソル配置ループは、`renderLineWithSelection()` で生成された **ANSI 制御文字を含む文字列** を、`Intl.Segmenter`（grapheme 単位）で1文字ずつ走査していた。

```typescript
// 修正前（簡略化）
for (const { segment } of getGraphemeSegmenter().segment(displayText)) {
  if (cursorFound) {
    afterCursor += segment
    continue
  }
  const nextWidth = currentWidth + stringWidth(segment)
  if (nextWidth > cursorCol) {
    atCursor = segment
    cursorFound = true
  } else {
    currentWidth = nextWidth
    beforeCursor += segment
  }
}
```

`displayText` は `before + \x1b[7m + selected + \x1b[27m + after` の形。これを grapheme segmenter で iterate すると **ANSI シーケンスがバラバラに分解される**:

| segment | 視覚幅 (`stringWidth`) | 想定 |
|---------|-----------------------|------|
| `\x1b` | 0（制御文字） | 不可視 |
| `[`    | **1** | ❌ ANSI の一部なのに可視扱い |
| `7`    | **1** | ❌ |
| `m`    | **1** | ❌ |
| 's'    | 1 | 可視 |

`stringWidth('[')` は CSI の中身でも単独だと「ASCII の左角括弧（幅 1）」として扱われる。結果、ANSI 4文字を可視幅 3 として数えてしまう。

カーソル列が `before` 直後（つまり `\x1b[7m` の中）に来ると:

```
beforeCursor = "before\x1b"   ← 前半が ANSI 途中で切られる
atCursor     = "["            ← ANSI の中身が atCursor
afterCursor  = "7mselected\x1b[27mafter"  ← 後半に "7m..." がそのまま残る
                ↑↑
                これが画面に「[7m」として見える
```

`atCursor` を `invert("[")` でラップすると:
```
"before\x1b" + "\x1b[7m[\x1b[27m" + "7mselected\x1b[27mafter"
       ↑↑                            ↑↑
       不正な ESC ESC のシーケンス     地のテキストとして "7m" が表示される
```

ターミナルは `\x1b\x1b[7m` を「壊れた ESC + 反転ON」として処理し、続く `[`（反転）→ `\x1b[27m`（反転OFF）→ **`7m`（地のテキスト）**→ 残り、と読む。これが `[7m` 露出の正体。

**修正**: ANSI CSI シーケンスを atomic な zero-width トークンとして扱う tokenizer を導入。grapheme segmentation はそれ以外の地のテキスト部分にのみ適用。

```typescript
// src/utils/Cursor.ts L137-166（追加）
const ANSI_CSI_REGEX = /\x1b\[[\x30-\x3F]*[\x20-\x2F]*[\x40-\x7E]/g

type AnsiAwareToken = { value: string; isAnsi: boolean }

function* tokenizeAnsi(text: string): Generator<AnsiAwareToken> {
  let lastIdx = 0
  ANSI_CSI_REGEX.lastIndex = 0
  let m: RegExpExecArray | null
  while ((m = ANSI_CSI_REGEX.exec(text)) !== null) {
    if (m.index > lastIdx) {
      for (const { segment } of getGraphemeSegmenter().segment(
        text.slice(lastIdx, m.index),
      )) {
        yield { value: segment, isAnsi: false }
      }
    }
    yield { value: m[0], isAnsi: true }
    lastIdx = m.index + m[0].length
  }
  if (lastIdx < text.length) {
    for (const { segment } of getGraphemeSegmenter().segment(
      text.slice(lastIdx),
    )) {
      yield { value: segment, isAnsi: false }
    }
  }
}
```

正規表現 `\x1b\[[\x30-\x3F]*[\x20-\x2F]*[\x40-\x7E]` は VT100/ECMA-48 の CSI 仕様そのまま:
- `\x1b\[` — CSI 導入
- `[\x30-\x3F]*` — パラメータバイト群（数字、`;`、`?` 等）
- `[\x20-\x2F]*` — intermediate バイト群（ほぼ空）
- `[\x40-\x7E]` — final バイト（`A-Z` `a-z` `@` 等の終端）

カーソル配置ループも書き換え:

```typescript
for (const tok of tokenizeAnsi(displayText)) {
  if (cursorFound) {
    afterCursor += tok.value
    continue
  }
  if (tok.isAnsi) {
    beforeCursor += tok.value   // ← ANSI は幅 0 で常に before/after に直行
    continue
  }
  const nextWidth = currentWidth + stringWidth(tok.value)
  if (nextWidth > cursorCol) {
    atCursor = tok.value
    cursorFound = true
  } else {
    currentWidth = nextWidth
    beforeCursor += tok.value
  }
}
```

これで `[7m` の露出は消える。

### 6.4. Bug 4: カーソルが選択範囲内に着地すると inverse が早期終了

**症状**: 後ろから前にドラッグ（5→2）して `12345678` の `345` を選択すると、`[7m` は出ないが **`345` の `45` の部分が反転表示されない**（`3` だけ反転、`45` は通常表示）。

**原因**: Cursor.offset は選択範囲のどちらかの端点。バックワードドラッグだと cursor は **selStart**（選択範囲の左端）に着地する。

```
displayText = "12" + \x1b[7m + "345" + \x1b[27m + "678"
                     ↑                  ↑
                  反転 ON           反転 OFF
カーソル列 = 2（"3" の位置）
```

カーソル配置ループの結果:
```
beforeCursor = "12\x1b[7m"
atCursor     = "3"
afterCursor  = "45\x1b[27m678"
```

`renderedCursor = invert("3") = "\x1b[7m3\x1b[27m"`

最終出力:
```
"12\x1b[7m" + "\x1b[7m3\x1b[27m" + "45\x1b[27m678"
= "12\x1b[7m\x1b[7m3\x1b[27m45\x1b[27m678"
```

ターミナル解釈:
| chunk | 状態 |
|-------|------|
| `12` | 通常 |
| `\x1b[7m\x1b[7m` | 反転 ON（`\x1b[7m` は idempotent） |
| `3` | 反転（カーソル） |
| `\x1b[27m` | 反転 OFF ← **ここが罠** |
| `45` | **通常**（選択範囲のはずなのに） |
| `\x1b[27m` | no-op |
| `678` | 通常 |

カーソル文字を `invert(...)` で囲んだ閉じタグ `\x1b[27m` が、本来 `\x1b[27m`（after の前にある）まで持続するはずだった「選択範囲の inverse」を**前倒しで終了**させてしまっている。

**修正**: ANSI トークン走査中に「inverse が ON か」を追跡し、カーソルが ON 状態で着地したら `afterCursor` の頭に `\x1b[7m` を再挿入する。

```typescript
// 状態追跡を追加
let inverseCurrent = false
let inverseAtCursor = false

for (const tok of tokenizeAnsi(displayText)) {
  if (cursorFound) {
    afterCursor += tok.value
    continue
  }
  if (tok.isAnsi) {
    if (tok.value === '\x1b[7m') inverseCurrent = true
    else if (tok.value === '\x1b[27m') inverseCurrent = false
    beforeCursor += tok.value
    continue
  }
  const nextWidth = currentWidth + stringWidth(tok.value)
  if (nextWidth > cursorCol) {
    atCursor = tok.value
    inverseAtCursor = inverseCurrent  // ← カーソル位置の inverse 状態をスナップショット
    cursorFound = true
  } else {
    currentWidth = nextWidth
    beforeCursor += tok.value
  }
}

// レンダリング側
const reopenInverse = inverseAtCursor && cursorChar ? '\x1b[7m' : ''

return (
  beforeCursor +
  renderedCursor +
  ghostSuffix +
  reopenInverse +    // ← ここで再開
  afterCursor.trimEnd()
)
```

修正後のバックワード出力:
```
"12\x1b[7m\x1b[7m3\x1b[27m\x1b[7m45\x1b[27m678"
                          ↑↑↑↑↑↑↑↑
                          再開された inverse
```

ターミナル解釈:
- `12` 通常
- `\x1b[7m\x1b[7m` 反転 ON
- `3` 反転（カーソル）
- `\x1b[27m` 反転 OFF
- `\x1b[7m` 反転 ON（再開）
- `45` 反転 ✓
- `\x1b[27m` 反転 OFF
- `678` 通常

`345` 全部反転表示。✓

---

## 7. 検証

### 7.1. 単体テスト

[`src/__tests__/cursor-render-mapping.test.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/__tests__/cursor-render-mapping.test.ts)（28 test）と
[`src/__tests__/cursor-render-selection.test.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/__tests__/cursor-render-selection.test.ts)（9 test）の2ファイル構成。

**特に重要**: `cursor-render-selection.test.ts` 内に「ターミナル状態シミュレータ」を仕込んだ。

```typescript
function inverseMap(rendered: string): { char: string; inv: boolean }[] {
  // SGR-7/27 を辿って各可視文字の「反転 ON か」を返す
  ...
}
```

これで **「最終出力の各位置でターミナルが反転表示するかどうか」** を assert できる。raw bytes ではなく視覚状態を直接検査できるので、将来 ANSI 方式が変わっても test 意図がブレない。

### 7.2. 全体回帰

`bun test` で 2310 pass / 33 fail。33 fail は無関係領域（branding／Wiki／Secure Storage 等）の元から落ちているテスト。Cursor 関連は全 pass。

### 7.3. 実機

`bun run build` で `dist/cli.mjs` を再生成 → 起動 → `12345678` を入力 → 5→2 にバックワードドラッグで選択 → `345` 全反転表示を確認。

---

## 8. 再利用可能なテクニカル教訓

このセクションが本ドキュメントの目的。「次同じ罠を踏まないため」のチェックリスト。

### 教訓 1: ANSI を含む文字列を grapheme で走査するな

`Intl.Segmenter` は ANSI を知らない。`\x1b[7m` を `\x1b` `[` `7` `m` の4 grapheme として吐く。`stringWidth()` は **完全な ANSI シーケンス**には 0 を返すが、**バラされた個別文字**（`[` `7` `m`）には 1 を返す。

→ ANSI を含む可能性がある文字列を視覚幅で走査するなら、**まず ANSI を atomic に切り出してから** grapheme segmenter にかける。

### 教訓 2: SGR-7 と SGR-27 はネストできない

ANSI SGR は **状態遷移**であって **stack 操作**ではない。`\x1b[7m\x1b[7m` は「2回反転 ON」（idempotent）、`\x1b[27m` 1個で全部 OFF になる。

→ 反転テキストの中にさらに反転が必要な要素（カーソル等）を入れる時、内側の閉じタグが外側を巻き込んで OFF にする。**内側を閉じた直後に外側を再オープン**する後処理が必要。

### 教訓 3: 画面座標↔文字列 offset の変換は描画と同じ場所に置く

「描画用に既に計算した値」と「マウスから来た座標を offset に変換する値」は同じデータ。別々に書くと乖離する。

→ 描画関数（`render()`）が逆引き関数（`offsetAt`）も同時に返す方式（`renderWithMapping()`）が一番ズレない。クロージャでスコープを閉じれば追加 state は不要。

### 教訓 4: 状態は1箇所に集約。ブリッジ禁止

Ink の selection と Cursor の selection を「同じものとして」扱おうとするとテキストマッチで結びつけることになり、CJK・絵文字・全角・改行で必ず壊れる。

→ 「画面選択」と「テキスト選択」は責務が違う。**入力欄は Cursor.selection、出力欄は Ink selection、両者は完全に分離。役割境界をコードコメント＆設計書に明文化**する。

### 教訓 5: useLayoutEffect の同フレーム上書き対策

「内部状態を更新 → 親に通知 → 親 props が変わる → useLayoutEffect 発火 → さっきの更新を上書き」は React で頻発する。

→ 内部状態と親 props の **同期スナップショット** を ref で持っておく。useLayoutEffect 内で「props が前回スナップショットと一致するならスキップ」のガードを入れる。

### 教訓 6: フォールバック取得経路を必ず併設する

「最初のクリック」「最初のフォーカス」「初回 mount」のような one-off イベントで初期化する設計は、その瞬間が来ないと動かない。

→ レイアウト系の値（DOM 位置・サイズ）は **毎レンダー後の useEffect** で常に更新する経路をデフォルトにし、クリックなどはそこに「最新化を強制する」補助として置く。

### 教訓 7: テストは「ターミナルが何を表示するか」で書く

raw bytes（`\x1b[7m345\x1b[27m`）を assert するテストは、同じ視覚効果を別の SGR 順序で作った時に壊れる。

→ ANSI 出力をパースして「各文字の反転状態」「色」「太字」など**視覚状態の配列**を返すヘルパを書き、それで assert する。実装が変わっても意図が変わらない限りテストは生きる。

### 教訓 8: 設計書は「設計時の判断と差分」を残す

設計書には「設計時の想定」と「実装で発覚した差分」を別表で残す。Ink の API 形状（`useSelection().subscribe()` しか取れない）が想定と違ったので、**設計→実装の差分テーブル**を作って後から追跡可能にした。

→ 実装中に設計が変わったら即座にドキュメントに反映。コミット時の差分メッセージだけだと埋もれる。

---

## 9. 関連ファイル早見表

| 役割 | パス |
|------|------|
| 選択状態本体 | [`src/utils/Cursor.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/utils/Cursor.ts) |
| マウス座標 sync | [`src/hooks/useTextInput.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/hooks/useTextInput.ts) |
| Box 位置取得 | [`src/components/PromptInput/PromptInput.tsx`](https://github.com/masayasusuzuki/chacram/blob/main/src/components/PromptInput/PromptInput.tsx) |
| selectionSyncRef pass-through | [`src/components/TextInput.tsx`](https://github.com/masayasusuzuki/chacram/blob/main/src/components/TextInput.tsx) |
| Props 型 | [`src/types/textInputTypes.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/types/textInputTypes.ts) |
| マッピングテスト | [`src/__tests__/cursor-render-mapping.test.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/__tests__/cursor-render-mapping.test.ts) |
| ANSI/inverse テスト | [`src/__tests__/cursor-render-selection.test.ts`](https://github.com/masayasusuzuki/chacram/blob/main/src/__tests__/cursor-render-selection.test.ts) |
| 設計書 | [`docs/architecture/text-selection.md`](https://github.com/masayasusuzuki/chacram/blob/main/docs/architecture/text-selection.md) |

## 10. 関連コミット早見表

| commit | 役割 |
|--------|------|
| [`9be5e14`](https://github.com/masayasusuzuki/chacram/commit/9be5e14) | テキスト選択パイプライン全体（Cursor.selection sentinel 化、renderWithMapping、useTextInput sync、テスト 28本、設計書） |
| [`8f8659b`](https://github.com/masayasusuzuki/chacram/commit/8f8659b) | Cursor.render の ANSI atomic 化と選択 inverse 再オープン（本ドキュメントの 6.3／6.4） |

---

## 11. 残タスク

- ダブルクリック → 単語選択（`prevWord()` / `nextWord()` で実装予定）
- トリプルクリック → 行選択（`startOfLine()` / `endOfLine()` で実装予定）
- 単体での ClickEvent 間隔計測（〜300ms 以内をダブルクリック判定）
