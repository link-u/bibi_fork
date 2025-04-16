# Bibi 追加解析メモ (機能別)

## 1. Bibi の特定機能の詳細解析

### 1.5. 指定位置へのスクロール (`R.focusOn`)

-   **目的**: 指定された「目的地」(`Par.Destination`) が表示されるように、メイン表示領域 (`R.Main`) をスクロールします。ページ単位移動、初期表示、リンク/目次からのジャンプなどで使用されます。
-   **主な処理**:
    1.  **目的地の正規化 (`R.hatchDestination`)**: 様々な形式の入力 (`Par.Destination`: ページ番号、アイテム番号、要素セレクタ、CFI、"head"/"foot" など) を解釈し、最終的なターゲットページ (`Page`) や座標 (`Point`) を含むオブジェクトに変換します。内部で `hatchPage`, `hatchItem`, `hatchNearestPageOfElement`, `hatchPoint`, `getPDestination` などのヘルパー関数や、CFI 処理拡張 (`X['EPUBCFI']`) を利用します。
    2.  **スクロール目標座標計算**: 正規化された目的地 (`Page`, `Side`, `Point` など) と、書籍のレイアウト (リフロー/固定)、ビューモード (`RVM`, `SLD`) を考慮して、スクロール先の座標 (`Par.FocusPoint`) を計算します。
    3.  **テキスト選択 (任意)**: 目的地情報にテキストノード情報が含まれる場合、`R.selectTextLocation()` で該当テキストを選択・ハイライトします。
    4.  **スクロール実行**: `sML.scrollTo()` を使用し、計算した座標へスクロールします。アニメーション時間 (`Duration`) やイージングも適用可能です。
    5.  **完了**: スクロール完了後、正規化された目的地情報を返し、`bibi:focused-on` イベントを発行します。

-   **関連ヘルパー関数**: `R.hatchDestination`, `R.hatchPage`, `R.hatchItem`, `R.hatchSpread`, `R.hatchNearestPageOfElement`, `R.hatchPoint`, `R.getPDestination` などは、目的地情報を解決するための重要な内部関数群です。

### 1.1. 拡大縮小 (ピンチ操作とズーム)

#### `I.PinchObserver`
-   **目的**: 2本指でのタッチ操作 (ピンチイン/アウト) を監視し、ジェスチャー情報 (中心点、指間距離の変化率) を取得して `I.Loupe.scale` を呼び出します。
-   **処理**: `touchstart`, `touchmove`, `touchend` イベントを監視。
    -   `touchmove` 中に、開始時からの指間距離の変化率 (`Ratio`) を計算し、`I.Loupe.scale(開始時拡大率 * Ratio, { Center: 開始時中心点, Stepless: true })` を連続的に呼び出すことで、リアルタイムな拡大縮小を実現。

#### `I.Loupe`
-   **目的**: ピンチ操作やタップ操作に応じて、メイン表示領域 (`R.Main`) に CSS `transform` (scale, translate) を適用し、拡大縮小と表示位置調整を行います。
-   **状態**: `CurrentTransformation` (現在の Scale, TranslateX/Y), `Transforming` (変形中フラグ), `IsZoomedOutForUtilities` (メニュー表示用縮小フラグ) などを管理。
-   **コア処理 (`transform`, `scale`)**: 指定された拡大率や変形情報を元に、実際に適用する `transform` スタイル値を計算し、`R.Main` に適用。画面外への移動制限や、メニュー表示時の縮小変形 (`getActualTransformation`) も考慮。変形完了後に `zoomed-in`/`zoomed-out` クラスを `<html>` に付与。
-   **タップズーム (`onTap`)**: ダブルタップ (または Space+シングルタップ) を検知し、設定された倍率 (`S['loupe-scale-per-step']`) で段階的に拡大 (`scale()` を呼び出し)。最大倍率に達したらデフォルト (1) に戻る。タップ位置がズームの中心になるように計算。
-   **ドラッグ移動 (`onPointerDown`, `onPointerMove`, `onPointerUp`)**: 拡大表示中にドラッグ操作を検知し、`transform()` を呼び出して表示位置 (`TranslateX`, `TranslateY`) をリアルタイムに更新。
-   **ユーティリティ連携 (`transformForUtilities`)**: メニュー表示/非表示時に、本全体を縮小して画面外に移動/元に戻すための `transform()` を実行。

### 1.3. EPUB ファイルアクセス (`O` モジュール)

#### `O` モジュール概要
-   "Operator" の略と思われ、ファイルアクセス、データ処理、環境情報取得、ユーティリティ関数などを提供。

#### `O.file(Source, Opt)`
-   **目的**: EPUB 内のリソース (XHTML, CSS, 画像など) の内容を取得します。ZIP 内からの抽出、前処理、URI 化を扱います。
-   **処理フロー**:
    1.  コンテンツ取得: `Source.Content` がなければ、展開ポリシー (`B.ExtractionPolicy`) に応じて処理。
        -   `on-the-fly`: `O.extract(Source)` で ZIP から動的抽出 (拡張機能)。
        -   `at-once`: ZIP 内データ (事前展開済み) から取得、なければエラー。
        -   ポリシーなし (展開済みフォルダなど): `O.download(Source)` で取得。
    2.  前処理 (`Opt.Preprocess`):
        -   `O.preprocess(Source)` を呼び出す。
        -   CSS/HTML/SVG 内のリソースパス (例: `@import`, `url()`, `src`, `href`) を正規表現で検出し、再帰的に `O.file({ URI: true })` を呼び出して Blob URL を取得し、パスを置換する。
        -   コメント除去やブラウザ互換性のための CSS プロパティ置換なども行う。
    3.  URI 化 (`Opt.URI`): `O.createBlobURL()` で内容から Blob URL を生成し `Source.URI` に格納。
    4.  最終的な `Source` オブジェクト (Content または URI を含む) を返す。

#### `O.openDocument(Source)`
-   **目的**: XML または HTML ファイルを読み込み、パースして Document オブジェクトを生成します。
-   **処理**: `O.file(Source)` でファイル内容を取得し、`DOMParser().parseFromString()` でパースします。

#### `O.loadZippedBookData(BookData)` (at-once 拡張)
-   **目的**: ZIP 形式の EPUB データ (File/Blob) を一括でメモリ上に展開します。
-   **処理 (推測)**: `JSZip` などを用いて ZIP を展開し、ファイルパスをキーとするファイル内容のマップ (`B.Unzipped`?) を生成。これが `at-once` ポリシー時の `O.file` のデータソースとなる。

#### `O.extract(Source)` (on-the-fly 拡張)
-   (コード未確認だが) Range Request を利用して ZIP ファイルから指定されたファイル (`Source`) のみを動的に抽出すると思われる (`bibi-zip-loader` と連携？)。 

## 2. 自作ビューワー開発に必要な技術要素の深掘り

### 2.3. クライアントサイド ZIP (`JSZip` 利用)

-   Bibi は ZIP 形式の EPUB をブラウザ上で展開するために **`JSZip`** ライブラリを利用しています (`at-once` 拡張機能内)。
-   ネットワーク上の ZIP ファイルは `JSZipUtils` を使って取得しています。
-   **`O.loadZippedBookData` (at-once 拡張) の処理**: 
    1.  ZIP ファイルを ArrayBuffer としてロード (`JSZipUtils` または `FileReader`)。
    2.  `JSZip.loadAsync(ArrayBuffer)` で ZIP を解析。
    3.  ZIP 内の各ファイルについて、内容を非同期に取得 (`BookDataArchive.file().async('blob'|'string')`)。
    4.  取得したファイルパスと内容 (Blob または Text) を `B.Package.Manifest` オブジェクトに格納。
-   この `B.Package.Manifest` が、`at-once` モードにおける `O.file` のデータソースとなります。
-   `on-the-fly` モードでは、別の拡張機能がおそらく `bibi-zip-loader` と連携し、Range Request を使って必要なファイルのみを動的に抽出していると推測されます。 

### 2.1. EPUB 構造解析 (OPF/NCX/NavDoc)

#### OPF Spine 解析 (`L.loadPackage.process` 内)
-   **目的**: OPF ファイルの `<spine>` 要素を解析し、コンテンツの表示順序と見開き構成を決定します。
-   **処理**:
    -   `<spine>` の `toc` 属性から NCX ファイルの ID を特定 (NavDoc が未定義の場合)。
    -   `page-progression-direction` 属性 (`ltr`/`rtl`) を `B.PPD` に格納。
    -   各 `<itemref>` をループ:
        -   `idref` から Manifest 内のソースを特定。
        -   `<iframe>` 要素 (`Item`) を生成し、ソース情報を紐付け。
        -   `properties` 属性 (rendition, page-spread など) を解析し `Item` に格納。
        -   `linear="no"` なら `R.NonLinearItems` に追加。
        -   `linear="yes"` なら `R.Items` に追加し、見開き (`Spread`) を決定/生成:
            -   固定レイアウトで `page-spread` が指定されている場合、前後のアイテムと結合して見開きを構成することがある (`Item.SpreadPair`)。
            -   それ以外は新しい見開き (`div.spread > div.spread-box`) を生成し `R.Spreads` に追加。
            -   アイテムのコンテナ (`div.item-box`) を生成し `Spread` に追加。
        -   固定レイアウトの場合、最初のページ要素 (`span.page`) を生成。
    -   生成した要素群 (`SpreadsDocumentFragment`) を `R.createSpine` に渡して DOM に追加。
    -   言語と PPD からデフォルトの書字方向 (`B.WritingMode`) を決定。

#### ナビゲーション解析 (`L.loadNavigation`)
-   **目的**: 目次情報 (NCX または Nav Doc) を読み込み、解析してパネル UI に表示するための HTML を生成します。
-   **処理**:
    -   `O.openDocument()` でナビゲーションファイル (XML/HTML) を取得。
    -   **NavDoc (EPUB3)**: `<nav epub:type="toc">` などの要素を取得し、クラスを付与してそのまま UI に追加。
    -   **NCX (EPUB2)**: `<navMap>` を再帰的に解析 (`makeNavULTree`) し、`<navPoint>` をネストされた `<ul><li><a href="...">...</a></li></ul>` 構造に変換して UI に追加。
    -   生成された各リンク (`<a>`) にイベントリスナーを追加し、クリック/タップで `R.focusOn()` を呼び出して該当箇所へジャンプするように設定。 