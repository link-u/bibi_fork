# Bibi 実装解析メモ

## 1.0 `__src/bibi/resources/scripts/bibi.js` の初期解析

このファイルは Webpack のエントリーポイントの一つであり、Bibi ビューワーの初期化と起動を担当していると考えられます。

### 1.1 依存関係

-   **`sml.js`**: プロジェクト固有のコアライブラリ。グローバルスコープ (`self.sML`) で利用可能。
-   **`./bibi.heart.js`**: Bibi の中心的な機能 (ヘルパー関数、定数、コアロジック `Bibi`, `O`, `I` など) を提供するモジュール。インポートされ、その内容がグローバルスコープ (`self`) に展開される。
-   **`./bibi.book.scss`**: EPUB コンテンツ自体に適用されるスタイルシート。

### 1.2 初期化処理 (`DOMContentLoaded` イベントリスナー内)

ページの DOM 構造が読み込まれた後に実行されます。

1.  **Polyfill の動的ロード**: `Promise`, `TextDecoder`, `IntersectionObserver` が未定義の場合、対応する Polyfill スクリプトを読み込みます。すべての Polyfill のロード完了を待ってから次の処理へ進みます。
2.  **ブック固有 CSS の抽出**: HTML 内に `/*! Bibi Book Style */` コメントを持つ `<style>` タグがあれば、その内容を抽出し Blob URL (`Bibi.BookStyleURL`) を生成します。元の `<style>` タグは削除されます。
3.  **IE (Trident) 向け対応**: スタイル適用タイミングの調整処理。
4.  **`Bibi.hello()` の呼び出し**: 上記の準備が完了した後、`Bibi` オブジェクトの `hello()` メソッドを呼び出します。これが Bibi の初期化プロセスの起点となると考えられます。

### 1.3 次のステップの候補

-   `bibi.js` 内での `Bibi.hello()` の実装詳細を確認する。
-   `bibi.heart.js` を確認し、`Bibi` オブジェクトやコア機能の定義を理解する。

## 2.0 `Bibi.hello()` と初期化処理 (`bibi.heart.js`)

`Bibi.hello()` は `bibi.js` から呼び出される初期化プロセスの起点です。Promise チェーンを用いて以下の処理を順次実行します。

```javascript
Bibi.hello()
  .then(Bibi.initialize)     // -> 2.1
  .then(Bibi.loadExtensions) // -> 2.2
  .then(Bibi.ready)          // -> 2.3
  .then(Bibi.getBookData)    // -> 2.4
  .then(Bibi.loadBook)       // -> 2.5 (主に 3.1)
  .then(Bibi.bindBook)       // -> 2.6 (主に 4.3 R.layOutBook)
  .then(Bibi.openBook)       // -> 2.7
  .catch(O.error);           // エラーハンドリング
```

### 2.1 `Bibi.initialize()`

-   **目的**: ブラウザ環境の判定、DOM要素の参照保持、主要モジュールの初期化など、基本的な準備を行います。
-   **主な処理**:
    -   オリジン、URLなどのパス情報を `O` オブジェクトに格納。
    -   `window`, `document`, `<html>`, `<head>`, `<body>` などの主要 DOM 要素への参照を `O` オブジェクトに格納。
    -   `<html>` 要素に環境 (OS, ブラウザ, touch有無, dev/debugモード, 言語) を示すクラスを追加。
    -   `E` (イベント), `O.Biscuits` (Cookie?), `P` (プリセット?), `U` (UI?), `D` (Dress?), `S` (設定?), `I` (UI/情報表示?) といった各モジュールの `initialize()` を呼び出す。
    -   埋め込み状況 (`O.Embedded`, `O.ParentBibi`) やフルスクリーン対応 (`O.FullscreenTarget`) を判定。
    -   IE 旧バージョンチェック。
    -   縦書き有効性、デフォルト/最小フォントサイズ、スライダー/メニューサイズなどを測定し `O` や `I` オブジェクトに格納。
    -   スクロールバーのサイズを測定し `O` オブジェクトに格納。
    -   `bibi:initialized` イベントを発行し、`Bibi.Status` を `Initialized` に更新。

### 2.2 `Bibi.loadExtensions()`

-   **目的**: 設定 (`S` オブジェクト) に基づいて必要な拡張機能をロードします。
-   **主な処理**:
    -   設定 (`S['allow-scripts-in-content']`, `S['zine']` など) や EPUB の指定状況 (`S['book']`, `S['accept-local-file']` など) から、追加でロードすべき拡張機能 (`sanitizer.js`, `zine.js`, `extractor/(on-the-fly|at-once).js`) を決定。
    -   設定 (`S['extensions']`) に定義された拡張機能と、上記で決定した追加拡張機能を `X.load()` を使って順番にロード。

### 2.3 `Bibi.ready()`

-   **目的**: 拡張機能ロード後、EPUB データ取得前の準備完了状態を示します。
-   **主な処理**:
    -   `<html>` 要素に一時的に `ready` クラスを付与。
    -   `bibi:readied` イベントを発行し、`Bibi.Status` を `Readied` に更新。

### 2.4 `Bibi.getBookData()`

-   **目的**: 読み込むべき EPUB のデータそのもの、またはその場所 (パス) を特定します。
-   **主な処理**:
    -   設定 (`S['book-data']` または `S['book']`) に EPUB 情報があればそれを解決。
    -   ローカルファイルを受け付ける設定 (`S['accept-local-file']`) であれば、ファイルがドラッグ＆ドロップされるのを待機 (Promise を返し、`<html>` に `waiting-file` クラスを付与)。
    -   上記以外の場合はエラーを発生させる。

### 2.5 `Bibi.loadBook(BookInfo)`

-   **目的**: `BookInfo` (EPUBデータ or パス) を元に EPUB を読み込み、構造を解析し、表示に必要な情報を準備します。
-   **主な処理**:
    -   ローディング表示を開始 (`Bibi.busyHerself()` など)。
    -   **`L.initializeBook(BookInfo)`**: EPUB の初期化 (展開、解析) を行う中心的な処理。書籍情報 (メタデータ、ファイルリスト、Spine など) を `B` オブジェクトに格納すると思われる。
    -   表示環境 (`S`, `R`) の更新。
    -   **`L.createCover()`**: カバー画像を生成・表示 (非同期)。
    -   **`L.loadNavigation()`**: ナビゲーション情報 (目次) をロード。
    -   `bibi:prepared` イベント発行、`Bibi.Status` を `Prepared` に更新。
    -   設定によりユーザー操作待機 (`L.wait()`)。
    -   **`L.preprocessResources()`**: EPUB 内リソース (CSS, JS) の前処理。
    -   **アイテム (コンテンツ文書) のロードと初期レイアウト**: 設定された開始位置 (`R.StartOn` または先頭) を決定。見開き (`R.Spreads`) ごとにアイテム (XHTML) を `L.loadSpread()` でロードし、`R.layOutSpreadAndItsItems()` で初期レイアウト。開始位置以外はプレースホルダーロードの可能性あり。
    -   最終的にレイアウト情報 (`LayoutOption`) を `Bibi.bindBook` へ渡す。

### 2.6 `Bibi.bindBook(LayoutOption)`

-   **目的**: ロードされた EPUB コンテンツをページ単位に分割し、表示領域に合わせて最終的なレイアウトを行います。
-   **主な処理**:
    -   **`R.organizePages()`**: コンテンツをページに分割・整理。
    -   `R.layOutStage()`: 表示領域全体のレイアウト。
    -   **`R.layOutBook(LayoutOption)`**: 指定された開始位置 (`LayoutOption`) に基づき、書籍全体の最終レイアウトとスクロール位置調整を行う。
    -   `bibi:laid-out-for-the-first-time` イベントを発行。

### 2.7 `Bibi.openBook(LayoutOption)`

-   **目的**: レイアウト済みの書籍を表示し、ユーザー操作を受け付けられる状態にします。
-   **主な処理**:
    -   ローディング表示を終了。
    -   `bibi:opened` イベント発行、`Bibi.Status` を `Opened` に更新。
    -   指定された開始位置 (`LayoutOption.Destination`) に対応するページを表示。
    -   表示履歴 (`I.History`) を初期化。
    -   スクロールに応じたアイテムの遅延ロード (`I.PageObserver.turnItems()`) や最終表示位置の保存 (`O.Biscuits.memorize()`) のためのイベントリスナーを設定。
    -   ページ移動 (`move-by`)、スクロール (`scroll-by`)、特定要素へのフォーカス (`focus-on`)、表示モード変更 (`change-view`) といったコマンドに対応するイベントリスナーを設定。
    -   開発モードの場合、開発版ノートを表示。

## 3.0 `L` (Loader) モジュール (`bibi.heart.js`)

EPUB の読み込み、展開、解析を担当するモジュール。

### 3.1 `L.initializeBook(BookInfo)`

-   **目的**: EPUB のソース (URI, File, Blob, Base64) を特定し、必要に応じて展開処理を行い、EPUB のコンテナファイル (`container.xml`) とパッケージファイル (OPF) を読み込む準備をします。
-   **処理フロー**:
    1.  入力形式 (`BookInfo`) を判定 (URI, File, Blob, Base64)。
    2.  **URI の場合**: 展開が必要か判定。Range Request の可否に応じて `O.extract()` (on-the-fly) または `O.loadZippedBookData()` (一括) を呼び出すか、展開済みフォルダとして `O.download(container.xml)` を試みる。
    3.  **File/Blob/Base64 の場合**: 設定に基づき受け入れ可否と MIME タイプをチェック後、`O.loadZippedBookData()` で一括展開。
    4.  成功すると、展開方法 (`B.ExtractionPolicy`) を記録し、`InitializedAs` (例: "EPUB file") を返す Promise を解決。
    5.  解決後、`.then()` で書籍タイプに応じて `X.Zine.loadZineData()` (Zineの場合) または **`L.loadContainer().then(L.loadPackage)`** (EPUBの場合) を呼び出す。
    6.  最後に `bibi:initialized-book` イベントを発行。

### 3.2 `L.loadContainer()`

-   **目的**: `META-INF/container.xml` を読み込み、パッケージファイル (OPF) のパスを取得します。
-   **処理**: `O.openDocument(container.xml)` でファイル内容 (XML) を取得し、`<rootfile>` 要素の `full-path` 属性から OPF ファイルのパスを特定して `B.Package.Source.Path` に設定します。

### 3.3 `L.loadPackage()`

-   **目的**: パッケージファイル (OPF) を読み込み、メタデータ、マニフェスト、スパイン情報を解析して `B.Package` オブジェクトに格納します。
-   **処理 (`L.loadPackage.process`)**:
    -   `O.openDocument(OPFパス)` でファイル内容 (XML) を取得。
    -   **Metadata**: `<metadata>` を解析し、タイトル、言語、作者、識別子、Rendition (`layout`, `spread`, `orientation`)、Viewport などの情報を `B.Package.Metadata` に格納。
    -   **Manifest**: `<manifest>` 内の `<item>` を解析し、各リソースの ID, パス (`href` から算出), メディアタイプ, プロパティ (`cover-image` など) を `B.Package.Manifest` に格納。カバー画像リソースは `B.CoverImage.Source` にも保持。
    -   **Spine**: (コード範囲外だが) `<spine>` 内の `<itemref>` を解析し、コンテンツの表示順序 (`linear` 属性含む) を `B.Package.Spine` に格納し、`toc` 属性から NCX ファイルの ID を特定すると思われる。 

### 3.4 `L.loadSpread(Spread, Opt)`

-   **目的**: 見開き (`Spread`) 内の全アイテム (XHTML) をロードします。
-   **処理**: `Spread.Items` をループし、各 `Item` について `L.loadItem()` を呼び出します。`Opt.AllowPlaceholderItems` が true の場合はプレースホルダーとしてロードするオプションを渡します。

### 3.5 `L.loadItem(Item, Opt)`

-   **目的**: 個々のアイテム (XHTML, 画像, SVG) のコンテンツを取得・処理し、`<iframe>` (`Item` オブジェクト自体) に読み込ませます。プレースホルダーの場合はスキップします。
-   **主な処理**:
    -   プレースホルダーとして扱うか (`IsPlaceholder`) を決定。
    -   **コンテンツ取得 (プレースホルダーでない場合)**:
        -   **(X)HTML**: `O.file()` で内容取得。必要に応じて前処理・サニタイズ (`O.sanitizeItemSource()`)。`<script>` 除去。
        -   **画像**: `O.file({ URI: true })` で URI 取得後、`<img>` を含む HTML を生成。
        -   **SVG**: `O.file()` で内容取得。`<?xml-stylesheet?>` を `<link>` に変換。必要に応じてサニタイズ。
    -   **`<iframe>` 設定**: 取得した内容から完全な HTML を生成し、デフォルトスタイル (`Bibi.BookStyleURL`) や `<base>` タグを追加。
        -   IE/Edge Legacy 以外: HTML から Blob URL (`Item.ContentURL`) を生成し `Item.src` に設定。
        -   IE/Edge Legacy: `document.write()` でコンテンツを書き込む。
    -   `<iframe>` を DOM に挿入。
    -   完了後、`L.postprocessItem(Item)` を呼び出す。
    -   `bibi:loaded-item` イベント発行。

### 3.6 `L.postprocessItem(Item)`

-   **目的**: `<iframe>` のロード完了後、アイテム内の DOM を調整します。
-   **主な処理**:
    -   `<iframe>` 内の `<html>`, `<head>`, `<body>` への参照を保持。
    -   `lang` 属性を調整。
    -   `<body>` 内の `<link>` を `<head>` へ移動。
    -   IE 向け SVG 内 `<image>` の `width`/`height` 調整。
    -   `L.coordinateLinkages()`: リンク調整？
    -   アイテム内容の特性 (単一画像/SVGなど) を判定しフラグ設定。
    -   リフロー型の場合 `L.patchItemStyles(Item)` を呼び出す。
    -   `bibi:postprocessed-item` イベント発行。

### 3.7 `L.patchItemStyles(Item)`

-   **目的**: リフロー型アイテムのスタイルシートを読み込み、ブラウザ互換性などのために調整 (パッチ) を行います。
-   **主な処理**:
    -   アイテム内の全スタイルシート (`<link>`, `<style>`) がロードされるのを待機。
    -   IE 向けに `column-count`, `writing-mode`, `text-combine-upright`, `text-justify` などを調整。
    -   その他ブラウザ向けに `column-count` を調整。
    -   `<html>` と `<body>` の `writing-mode` を同期させ、最終的な `writing-mode` を判定してクラス (`bibi-vertical-text` など) を `<html>` に付与。

## 4.0 `R` (Renderer/Reader) モジュール (`bibi.heart.js`)

EPUB コンテンツのレンダリング、ページネーション、レイアウト、ユーザーインタラクションを担当するモジュール。

### 4.1 レイアウトの準備とページ分割

#### 4.1.1 `R.createSpine()`, `R.resetStage()`
-   書籍表示のための主要な HTML 要素 (`#bibi-main`, `#bibi-main-book`) を生成し、現在のウィンドウサイズや設定に基づいて表示領域 (`R.Stage`) のサイズやスタイルを初期化します。

#### 4.1.2 `R.layOutSpreadAndItsItems()`, `R.layOutSpread()`, `R.layOutItem()`
-   見開き (`Spread`) とその中のアイテム (`Item`) のサイズと配置を計算し、スタイルを適用します。アイテムがリフロー型か固定レイアウト型か、ビューモード、画面の向きなどを考慮してレイアウトを決定します。
-   `R.layOutItem()` はアイテムの種類に応じて `R.renderReflowableItem()` または `R.renderPrePaginatedItem()` を呼び出します。

#### 4.1.3 `R.renderReflowableItem(Item)`
-   **目的**: リフロー型アイテム (`<iframe>`) のコンテンツをページ分割するための計算とスタイル適用を行います。
-   **主な処理**:
    -   表示領域と設定から1ページあたりのコンテンツ領域サイズ (`PageCB`, `PageCL`) を計算。
    -   見開き表示にするか (`Item.Spreaded`) を判定し、適用。
    -   アイテム (`<iframe>`) 自体のサイズを設定。
    -   アイテム内の画像サイズをページ内に収まるよう調整。
    -   コンテンツがページサイズを超えるか判定し、ページ分割方法 (`PaginateWith`) を決定 ('C': CSS Columns, 'S': Scroll/Shape)。
    -   決定した方法に基づき、CSS Columns スタイル (`bibi-columned` クラスと `column-*` プロパティ) または `shape-outside: polygon()` を用いたスクロール分割スタイル (`bibi-with-gutters` クラスと `Item.HTML` のサイズ調整) を適用。

#### 4.1.4 `R.renderPrePaginatedItem(Item)`
-   **目的**: 固定レイアウト型アイテムのサイズと表示倍率 (`Item.Scale`) を計算し、スタイルを適用します。
-   **主な処理**:
    -   アイテムのビューポート (`Item.Viewport`) を決定 (`<meta name="viewport">`, SVG `viewBox`, 画像サイズなどから)。
    -   見開き表示 (`Item.Spreaded`) かどうかを判定。
    -   見開き表示の場合はペアアイテムも考慮し、表示領域 (`R.Stage`) に収まる最大の表示倍率 (`Item.Scale`) を計算。
    -   見開きでない場合は、表示領域に収まるように `Item.Scale` を計算 (スクロールモードで幅いっぱいに表示する設定の場合は幅基準で計算)。
    -   アイテムのコンテナ (`Item.Box`) に最終的な表示サイズを設定。
    -   アイテム (`<iframe>`) には元のビューポートサイズと `transform: scale()` を設定。

#### 4.1.5 `R.organizePages()`
-   **目的**: 全アイテムのページ分割完了後、書籍全体のページ情報を整理します。
-   **処理**: 全てのページオブジェクトを `R.Pages` 配列に格納し、各ページに書籍全体でのインデックス (`Page.Index`) と、所属アイテム/見開き内でのインデックス (`Page.IndexInItem`, `Page.IndexInSpread`) を付与します。`linear="no"` のアイテムのページは `Page.Index` を `undefined` にします。

### 4.2 書籍全体のレイアウトと表示

#### 4.2.1 `R.layOutStage()`
-   (organizePages 後に呼ばれる) 全見開きのレイアウト計算結果に基づき、書籍全体の長さ (`#bibi-main-book` の `width` または `height`) を計算して設定します。

#### 4.2.2 `R.layOutBook(Opt)`
-   **目的**: 書籍全体の最終的なレイアウトを行い、指定された位置が表示されるようにスクロールを調整します。ビューモード変更時などにも使用されます。
-   **主な処理**:
    -   `Opt.Reset` が true の場合、ステージリセット (`R.resetStage`)、全見開き・アイテムの再レイアウト (`R.layOutSpreadAndItsItems`)、ページ再整理 (`R.organizePages`)、ステージ再レイアウト (`R.layOutStage`) を実行。
    -   `R.focusOn({ Destination: Opt.Destination, Duration: 0 })` を呼び出し、指定された位置 (`Opt.Destination`) へ即座にスクロール。
    -   レイアウト完了後、`bibi:laid-out` イベントを発行。

## 5.0 `I` (UserInterfaces) モジュール (`bibi.heart.js`)

ユーザーインターフェース要素の管理とユーザー操作の監視・処理を担当するモジュール群。

### 5.1 ページめくり操作

#### 5.1.1 `I.FlickObserver`
-   **目的**: タッチ (およびマウスドラッグ) によるフリック、スワイプ、パン操作を検知し、ページ移動や関連アクションを実行します。
-   **処理**: `pointerdown`, `pointermove`, `pointerup` イベントを監視。
    -   短い時間での一定距離以上の移動をフリック/スワイプと判定。
    -   移動方向とビューモードに応じて `R.focusOn()` (ページ単位移動) または `R.scrollBy()` (スクロール移動) を呼び出し。
    -   スクロール方向と直交する方向へのフリックには、設定に応じてビューモード切り替え (`I.AxisSwitcher.switchAxis`) やメニュー表示 (`I.Utilities.toggleGracefuly`) を割り当て可能。
    -   Paged ビューでのドラッグスクロール (`Pan`) や、操作終了時のページへのスナップバックも処理。

#### 5.1.2 `I.KeyObserver`
-   **目的**: キーボード入力を監視し、ページ移動などのアクションを実行します (`S['use-keys']` が true の場合のみ有効)。
-   **処理**: `keydown` イベントを監視。
    -   **矢印キー (←, →)**: `R.moveBy({ Distance: +/-1 })` でページ移動。
    -   **矢印キー (↑, ↓)**: 設定に応じて `R.moveBy()` または `R.scrollBy()`。
    -   **Space**: `R.moveBy({ Distance: +1 })` (Shift+Space で -1)。
    -   **PageUp/PageDown**: `R.moveBy({ Distance: +/-1 })`。
    -   **Home/End**: `R.focusOn()` で先頭/末尾ページへ。
    -   **Esc**: フルスクリーン解除など。
    -   **?**: ヘルプ表示。

#### 5.1.3 `R.moveBy(Par)`
-   ページ移動のコア処理。ビューモードやコンテンツの種類に応じて、ページ単位でフォーカスを移動する `R.focusOn()` か、画面単位でスクロールする `R.scrollBy()` を呼び分けます。

#### 5.1.4 `R.scrollBy(Par)`
-   指定距離をアニメーション付きでスクロールする処理。主に Scroll ビューモードで使用されます。

#### 5.1.5 `R.focusOn(Par)`
-   (コード未確認だが) 指定された要素やページ位置が表示されるようにスクロール位置を調整する処理。ページ単位移動や初期表示位置への移動で使用されます。 