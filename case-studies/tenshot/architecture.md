# Architecture — TenShot

このドキュメントは「**何をどう組み立てたか**」の技術判断を残すもの。
UX 判断は [ux-design.md](./ux-design.md) を参照。

---

## 全体像

TenShot は macOS 14（Sonoma）以降向けの **AppKit ベースのメニューバー常駐アプリ**。

主要コンポーネントの依存関係：

```
main.swift                       # エントリポイント（NSApplication を起動）
  └─ AppDelegate                 # メニューバーアイコン管理、ライフサイクル
       └─ PanelWindowController  # 右端パレット（NSPanel）の表示制御
            └─ PanelViewController  # パレット内のすべての UI
                 ├─ ImageStore          # 画像の保持・永続化（singleton）
                 ├─ ScreenshotCapture   # 範囲選択スクリーンショット撮影（singleton）
                 ├─ DropReceivingView   # ドラッグ受け入れ
                 ├─ DraggableImageView  # ドラッグ送り出し
                 └─ ImagePreviewWindow  # 拡大プレビュー
```

ファイル数は 10 個程度、Swift 行数は約 1000 行。最小構成。

---

## 技術スタック選定

### AppKit（SwiftUI ではない）

メニューバーアプリは SwiftUI でも作れる時代だが、AppKit を選んだ。

**理由：**

- **メニューバー常駐の細かい挙動制御** → `NSStatusBar` / `NSStatusItem` 直系の API でしか取れない情報がある（左クリック / 右クリックの判別、メニュー表示制御など）
- **NSPanel の細かなスタイル制御** → `.nonactivatingPanel`（フォーカスを奪わない）、`isFloatingPanel`、`level`、`collectionBehavior` などのプロパティは AppKit 側で直接触る方が素直
- **ScreenCaptureKit との連携** → オーバーレイビューで範囲選択 UI を作る部分は、AppKit のビュー階層と相性が良い
- **`NSFilePromiseProvider` を使ったドラッグ送り出し** → SwiftUI ラッパーがまだ薄い領域

SwiftUI で同等のことをすると、`NSViewRepresentable` でラップする層が増えて、結局 AppKit を露出することになる。
最初から AppKit で組む方がコードが直接的になる。

### ScreenCaptureKit（`/usr/sbin/screencapture` ではない）

最初の実装は `Process` で `/usr/sbin/screencapture -i` を呼ぶ方式だった。
標準 OS ツールをサブプロセスで起動する方法。実装は数行で済む。

```swift
// 旧実装（コミット bbb4b9a 時点）
let process = Process()
process.executableURL = URL(fileURLWithPath: "/usr/sbin/screencapture")
process.arguments = ["-i", tempFile]
try? process.run()
```

しかし問題があった：

- **App Sandbox との相性が悪い** → サンドボックス環境で `Process` を使ってシステムバイナリを呼ぶ動作には制約がある。Mac App Store 配信の前提条件と衝突しうる
- **撮影 UI を制御できない** → `screencapture -i` が提供する黒い半透明オーバーレイをカスタマイズできない（範囲選択中の見た目を統一できない、ガイダンステキストを出せない）
- **連続性のある体験を作りにくい** → サブプロセス起動とコールバックの繋ぎが緩い。撮影完了の検知がファイル書き込み待ちになる

リリース前に **ScreenCaptureKit**（macOS 13+）に切り替えた。

```swift
// 新実装（ScreenshotCapture.swift）
let filter = SCContentFilter(display: display, excludingWindows: [])
let config = SCStreamConfiguration()
config.width = Int(CGFloat(display.width) * scale)
config.height = Int(CGFloat(display.height) * scale)
return try await SCScreenshotManager.captureImage(
    contentFilter: filter, configuration: config
)
```

**メリット：**

- App Sandbox と公式に共存できる API
- オーバーレイ UI を完全自作できる（`SelectionView` でガイダンステキスト「ドラッグして範囲選択 / ESC でキャンセル」を表示）
- マルチディスプレイ対応が API レベルで明示的（`SCShareableContent.displays` で全ディスプレイを取得し、各画面にオーバーレイを配置）
- 撮影前後のタイミング制御が `async/await` で明示的に書ける

`ScreenshotCapture.swift` は約 150 行。`SCScreenshotManager.captureImage` を中心に、オーバーレイの表示・選択範囲の取得・クロップを組み立てている。

### NSPanel（NSWindow ではない）

パレットの実体は `NSWindow` ではなく `NSPanel`。

```swift
let panel = NSPanel(
    contentRect: NSRect(x: 0, y: 0, width: 300, height: 600),
    styleMask: [.titled, .closable, .nonactivatingPanel],
    backing: .buffered,
    defer: false
)
panel.isFloatingPanel = true
panel.level = .floating
panel.collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]
```

**NSPanel が必要な理由：**

- **`.nonactivatingPanel` スタイルマスク** → 「フォーカスを奪わない」ウィンドウになる。VSCode で作業中にパレットをクリックしても、VSCode のフォーカスは外れない（カーソル位置・選択範囲が維持される）
- **`isFloatingPanel = true`** → 常に他のウィンドウより上に表示
- **`level = .floating`** → アプリ間でも floating 維持（自アプリがバックグラウンドでも他アプリの上に乗る）
- **`collectionBehavior = [.canJoinAllSpaces, .fullScreenAuxiliary]`** → フルスクリーン中の他アプリの上にも乗る、Spaces を切り替えても消えない

通常の `NSWindow` では「フォーカスを奪わずに常に上に乗る」を実現できない。
**「邪魔にならない」UX の核心は NSPanel が握っている**。

---

## データフロー

### 撮影 → 保存 → 表示

```
[ユーザー: SS ボタンクリック]
    ↓
PanelViewController.takeScreenshot()
    ↓
view.window?.orderOut(nil)              ← パレット自身を一時非表示
    ↓
ScreenshotCapture.shared.captureRegion {
    全ディスプレイにオーバーレイを表示
        ↓ ユーザーがドラッグで範囲選択
    SCScreenshotManager.captureImage()  ← ディスプレイ全体をキャプチャ
        ↓ CGImage を取得
    cropping(to: cropRect)              ← 選択範囲に切り抜き
        ↓
    completion(NSImage)
}
    ↓
PanelViewController が画像を受け取る
    ↓
ImageStore.shared.add(image:)
    ↓
JPEG (quality 0.75) に変換
    ↓
最大10枚を超えたら removeFirst() で最古を削除
    ↓
Application Support 配下にファイル保存
    ↓
onChange コールバックを発火
    ↓
PanelViewController.reloadImages() で UI を更新
```

撮影開始時に `view.window?.orderOut(nil)` でパレット自身を非表示にしているのは、**パレット自身が撮影対象に映り込まないため**。
撮影完了後に `orderFrontRegardless()` で復帰させる。

### ドラッグ受け入れ（他アプリ → TenShot）

```
他アプリ（Finder、ブラウザ等）から画像をパレットにドラッグ
    ↓
DropReceivingView.performDragOperation()
    ↓
pasteboard から URL を抽出（ファイルドラッグの場合）
    or pasteboard から NSImage を抽出（画像データドラッグの場合）
    ↓
onDrop コールバック → ImageStore.add()
```

`DropReceivingView` は `[.tiff, .png, .fileURL, "public.image"]` の 4 種類の pasteboard types を受け入れる。
**多様な形式（ファイル参照、画像データ、汎用画像 UTI）に対応**することで、ブラウザのドラッグ・Finder のドラッグ・Slack のスクショドラッグなど、多くのソースをカバーする。

### ドラッグ送り出し（TenShot → 他アプリ）

```
パレット上の画像を他アプリへドラッグ
    ↓
DraggableImageView.mouseDragged()
    ↓
画像を一時ファイルに PNG で書き出し
    ↓
NSFilePromiseProvider で「ファイルを後で渡す」約束を pasteboard に載せる
    ↓
他アプリ側がドロップを受けたタイミングで
filePromiseProvider:writePromiseTo: コールバックが発火
    ↓
一時ファイルを目的地にコピー
```

**`NSFilePromiseProvider` を使う理由：**

- 画像データを直接 pasteboard に載せると重い（複数ピクセル形式に変換しようとして遅くなる）
- ドロップ先が決まってから初めてファイル書き込みする「lazy」な渡し方が標準
- 受け取り側はファイルとして扱えるので、ファイル前提のアプリ（Finder、Slack、Notion 等）と互換性が高い

---

## 永続化設計

### 経緯：UserDefaults → ファイルシステム

**初期実装** は UserDefaults に画像データを直接格納していた：

```swift
// 旧（〜4/14 のコミット時点）
private let key = "TenShot.images"
UserDefaults.standard.set(images, forKey: key)  // [Data] をそのまま放り込む
```

これには 3 つの問題があった：

1. **UserDefaults の本来用途と違う** → `UserDefaults` は「設定値」を入れる場所であり、メガバイト級のデータを入れる用途ではない。Apple もガイドラインで「小さい設定値のみ」と明記している
2. **起動時のメモリスパイク** → 起動時に UserDefaults からすべての画像データをロードするとメモリスパイクが起きる
3. **想定外のシステム同期に巻き込まれる** → UserDefaults のバックアップに画像データが含まれる（iCloud 同期、Time Machine など）

**リリース直前の改修**（4/20）で **Application Support 配下のファイル保存** に切り替えた：

```swift
// 新（ImageStore.swift 現行）
private func imagesDirectory() -> URL {
    let base = FileManager.default.urls(
        for: .applicationSupportDirectory, in: .userDomainMask
    ).first!
    return base.appendingPathComponent("TenShot/images", isDirectory: true)
}

private func save() {
    // 既存ディレクトリを丸ごと消して書き直す
    try? FileManager.default.removeItem(at: dir)
    try? FileManager.default.createDirectory(at: dir, withIntermediateDirectories: true)
    for (i, data) in images.enumerated() {
        let name = String(format: "%03d.jpg", i)
        try? data.write(to: dir.appendingPathComponent(name))
    }
    // ファイル名一覧（インデックス）だけを UserDefaults に持つ
    UserDefaults.standard.set(names, forKey: indexKey)
}
```

**Application Support を選んだ理由：**

- macOS の慣例として「アプリの永続データ」はここに置く
- サンドボックス環境では `~/Library/Containers/com.suzukimasayasu.TenShot/Data/Library/Application Support/TenShot/` に隔離されるため、他アプリから干渉されない
- Time Machine バックアップ対象として正当（Documents ほどユーザー領域ではないが、設定よりは重要なデータ）

**UserDefaults には「ファイル名のリスト」だけを残す。** 画像本体はファイル、インデックスは UserDefaults という役割分担。
これで起動時は「インデックスを読む → 該当ファイルを順番に読む」だけで済み、UserDefaults は軽量に保たれる。

### PNG → JPEG（quality 0.75）

容量削減のため。
スクリーンショット用途なら JPEG で十分。テキストや UI のスクショで多少のブロックノイズが出る場合があるが、一時的な「メモ用」の用途であれば許容範囲。

10 枚 × 数 MB の PNG を貯めると簡単に 50 MB 級になる。JPEG quality 0.75 なら 1/3〜1/5 まで圧縮できる。

### 「ディレクトリ全消しして書き直す」方式

`save()` のたびに `removeItem` → `createDirectory` → 全件再書き込み。

差分管理（追加されたものだけ書き、削除されたものだけ消す）より **冪等で単純**。
最大 10 枚しか入らないので、書き直しのコストは無視できる。

「複雑性を持ち込まない」判断。

### マイグレーション処理

旧 UserDefaults キー（`TenShot.images`）に画像データが残っていれば削除する：

```swift
private func migrateFromLegacyIfNeeded() {
    let defaults = UserDefaults.standard
    guard defaults.object(forKey: legacyKey) != nil else { return }
    defaults.removeObject(forKey: legacyKey)
}
```

旧データの中身を新形式に変換する実装は入れていない（旧データは破棄）。
**理由：旧バージョンのストック画像はあくまで「直近の作業用」であり、長期保存価値はない**。
アップデート後にユーザーが「あれ、ストック消えた？」と思っても、すぐ新しいスクショを溜め始めれば実害はない。

---

## サンドボックス権限

`TenShot.entitlements`：

```xml
<key>com.apple.security.app-sandbox</key>
<true/>
<key>com.apple.security.files.user-selected.read-write</key>
<true/>
<key>com.apple.security.network.client</key>
<true/>
```

| キー | 役割 |
|---|---|
| `app-sandbox` | Mac App Store 必須。アプリがサンドボックス内で動くことを宣言 |
| `files.user-selected.read-write` | ユーザーが明示的に選んだファイル（ドラッグ受け入れた画像）への読み書き |
| `network.client` | 現状未使用。将来的なアップデート確認や（許される範囲での）外部連携を想定 |

**画面収録権限は entitlements ではない**。
これは macOS の **TCC（Transparency, Consent, and Control）** の枠組みで管理される、ユーザー同意制の権限。
初回 ScreenCaptureKit 呼び出し時に macOS が許可ダイアログを出す。
ユーザーが拒否すると撮影機能だけが動かなくなり、ドラッグ＆ドロップ機能は引き続き使える（フォールバック動作）。

---

## ファイル分割の方針

10 ファイルしかないので、機能ごとに 1 ファイル：

| ファイル | 責務 | 行数目安 |
|---|---|---|
| `main.swift` | NSApplication 起動 | 8 |
| `AppDelegate.swift` | メニューバーアイコン、左/右クリック振り分け、ライフサイクル | 50 |
| `PanelWindowController.swift` | パレットの NSPanel 生成、位置決め、表示/非表示トグル | 40 |
| `PanelViewController.swift` | パレット内 UI 全部（ヘッダー、スライダー、ドロップゾーン、画像リスト、各種ボタンアクション） | 350 |
| `ImageStore.swift` | 画像配列の保持、永続化、変更通知 | 100 |
| `ScreenshotCapture.swift` | 範囲選択撮影（オーバーレイ + ScreenCaptureKit + クロップ） | 150 |
| `DraggableImageView.swift` | パレット画像をドラッグで他アプリへ送り出す | 100 |
| `DropReceivingView.swift` | 他アプリからドラッグされた画像を受け入れる | 55 |
| `ImagePreviewWindow.swift` | 拡大プレビュー（クリック時のフルサイズ表示） | 50 |
| `CorkTextureView.swift` | （未使用）コルクボード風テクスチャの試作残骸 | 180 |

`PanelViewController.swift` はやや肥大化（〜350行）。
本来はさらにコンポーネント分割の余地がある（ヘッダー、画像リスト、スライダー行など）が、現状の機能数（〜5機能）であれば 1 ファイルで見渡せる方が読み手にとって楽。**「分割の前に見渡せるサイズで止めておく」** 判断。

`CorkTextureView.swift` と `WoodFrameView` / `TapeView` は最終 UI には使われていない。
**コルクボードに写真を留めるという初期デザイン案の残骸**。グラスモーフィズム路線に切り替わる前の試作物。

意図的に削除していない理由：**設計プロセスの記録としての価値**を残すため。
「最初はコルクボードを考えた」という事実は、アプリの形が決まる過程を残す上で意味がある。OSS で公開する以上、そういう試行錯誤も読み手にとって学びになる。

---

## ドキュメントと実装のズレ（要修正メモ）

`doc/phase4-support.md`（App Store サポートページ草案、4/22 時点）には以下の FAQ がある：

> Q. アプリを終了するとストックした画像は消えますか？
> はい。TenShotはメモリ内にのみ画像を保持するため、アプリ終了時に画像は破棄されます。

これは **UserDefaults 時代の古い記述**。
現在の実装（4/20 改修以降）は **Application Support にファイル永続化されている** ため、アプリを終了しても画像は残る。

サポートページを公開する前に修正が必要：

> Q. アプリを終了するとストックした画像は消えますか？
> いいえ。最大 10 枚の直近スクショは Application Support 配下にファイル保存されており、アプリを終了・再起動しても保持されます。

---

## まとめ：技術判断を貫く 3 原則

### 1. 「OS 標準 API に寄せる」

AppKit / NSPanel / NSStatusBar / NSFilePromiseProvider / ScreenCaptureKit / Application Support…
すべて Apple 公式の枠組みに乗っかっている。
独自実装やサードパーティ依存を避けることで、**OS アップデート時の生存率が上がる**。サンドボックス審査も通りやすい。

### 2. 「複雑性を持ち込まない」

差分保存ではなく全件書き直し。マイグレーションは破棄方式。ファイル分割も「まだ見渡せるなら分けない」。
最大 10 枚という小さなドメインだからこそ、**「賢く実装する」よりも「単純に実装する」** ことを優先する。

### 3. 「ユーザーフォーカスを最優先で守る」

`.nonactivatingPanel` も `Buy Me a Coffee 削除` も、根っこは **「TenShot がユーザーの作業の主役を奪わない」** 一点に集約される。
脇役としての品格を、技術選定のレベルから維持する。
