# engineering-notes

個人の技術メモ。ハマりポイント・ケーススタディ・ツール運用ノートを蓄積する場所。

実装中に「次同じ罠を踏まないために残す」という観点で書いている。フォーマットは決め打ちせず、テーマ次第で粒度を変える。

## 構成

```
engineering-notes/
├── README.md
├── LICENSE
├── patterns.md       # 設計パターン・コードパターンの気づき（短文メモの集約）
├── gotchas/          # ハマり事例・ケーススタディ（1事例=1ファイル、YYYYMMDD_タイトル.md）
├── tools/            # ツール別運用ノート（ツール名.md）
└── case-studies/     # 自作プロダクトの設計記録（プロダクト名/ 配下に複数ファイル）
```

## 既存のエントリ

### `gotchas/`

- [`20260510_terminal-text-selection.md`](./gotchas/20260510_terminal-text-selection.md) — ターミナル CLI（Ink ベース TUI）でテキスト範囲選択を実装した時の全記録。ANSI / grapheme / 座標変換の絡み合いを解きほぐした記録

### `case-studies/`

- [`tenshot/`](./case-studies/tenshot/) — macOS 用連写スクリーンショットアプリ「TenShot」の設計記録。ソースは [github.com/masayasusuzuki/TenShot](https://github.com/masayasusuzuki/TenShot)

## ライセンス

[MIT](./LICENSE)
