# TenShot — Case Study

macOS 用の連写スクリーンショットアプリ「TenShot」の設計記録。

## このディレクトリ

- 本体ソースは別リポ（下記）
- ここには「**なぜこう作ったか**」の設計思想だけを置く

## ソースコード（自由に Fork してください）

👉 https://github.com/masayasusuzuki/TenShot

オープンソース推奨派です。Fork して自由にチューニングしてください。
PR や Issue は歓迎しますが、対応をお約束するものではありません。

## ドキュメント

設計を 3 つのレイヤーで分けて書いている：

- **[ux-design.md](./ux-design.md)** — ユーザーにどう使ってほしいか（プロダクト設計）
  - 「消しちゃいけない画像を消した日」から始まる動機
  - なぜ 10 枚なのか / なぜ FIFO なのか / なぜ右端パレットなのか
  - 「邪魔しない」「溜めない」「クリップボードを壊さない」の 3 原則
- **[architecture.md](./architecture.md)** — それを実現するために何をどう組んだか（技術設計）
  - AppKit / NSPanel / ScreenCaptureKit の選定理由
  - 永続化を UserDefaults からファイルシステムに切り替えた経緯
  - サンドボックス権限・ファイル分割の方針
- **[development-process.md](./development-process.md)** — プロダクトと技術を生み出す「作り方」（開発プロセス設計）
  - Claude Code との協働で得た最大の学び
  - 「人間がボトルネックにならない設計」の組み立て方
  - Chrome 拡張試作からの方針転換、要件定義のあり方

## ライセンス

- ドキュメント：MIT（engineering-notes リポに準拠）
- ソースコード：[MIT](https://github.com/masayasusuzuki/TenShot/blob/main/LICENSE)（TenShot リポ）
