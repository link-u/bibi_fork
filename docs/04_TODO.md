## 開発方針

- Bibi の既存コードは直接流用せず、EPUB の仕様や実装の詳細を理解するための「参考資料」「設計図」として活用する。
- 必要な機能は、Bibi のロジックを参考にしながら、Vanilla JavaScript を用いて段階的に再実装する。
- まずは EPUB のコア構造（container.xml, OPF）の解析部分から実装を開始する。

---

# TODOリスト: Bibi 軽量化バージョン開発

## 1. コア機能の維持・選定

-   [ ] `Bibi` コア制御: 初期化シーケンス (`hello` -> `initialize` -> ...) の基本構造を確認・維持。
-   [ ] `L` (Loader) モジュール:
    -   [ ] EPUB 解析 (`container.xml`, OPF, NavDoc/NCX) 機能の維持 (`L.initializeBook`, `L.loadContainer`, `L.loadPackage`, `L.loadNavigation`)。
    -   [ ] アイテムロード (`<iframe>`) と前処理・スタイル適用機能の維持 (`L.loadItem`, `L.postprocessItem`, `L.patchItemStyles`)。特に縦書き関連の調整。
    -   [ ] **画像表示の実現:** XHTML 内の `<img>` タグの `src` パス解決と表示。 (新規追加)
    -   [ ] **CSS内リソースの解決:** CSS 内の `url()` で参照される画像パス等を解決し、背景画像等を表示可能にする。(新規追加)
    -   [ ] **Blob URL の活用:** ZIP 内リソース等へのアクセスを可能にするため、画像や CSS 等に対する Blob URL の生成・利用ロジックを検討・実装。 (新規追加)
-   [ ] `R` (Renderer/Reader) モジュール:
    -   [ ] レンダリング基盤 (`Stage` 作成/リセット) 機能の維持 (`R.createSpine`, `R.resetStage`)。
    -   [ ] リフロー型コンテンツのページ分割機能の維持 (`R.renderReflowableItem`)。
    -   [ ] (検討) 固定レイアウト (`R.renderPrePaginatedItem`) 機能の削除または簡略化。
    -   [ ] 見開き・アイテムレイアウト計算機能の維持 (`R.layOutSpreadAndItsItems`)。
    -   [ ] ページ管理機能の維持 (`R.organizePages`)。
    -   [ ] 最終レイアウト・スクロール調整機能の維持 (`R.layOutStage`, `R.layOutBook`)。
    -   [ ] 指定位置へのジャンプ機能の維持 (`R.focusOn`)。
    -   [ ] ページ移動処理 (`R.moveBy`) 呼び出し元の維持。
-   [ ] `O` (Operator) モジュール:
    -   [ ] EPUB 内リソースアクセス機能の維持 (`O.file`, `O.openDocument`)。 **画像や CSS ファイルを正しく取得できること。** (追記)
    -   [ ] 基本的なユーティリティ関数の選定・維持。
    -   [ ] ZIP 展開を `JSZip` ベースのシンプルな方式 (`at-once` 参考) に限定。
    -   [ ] **Blob URL 生成機能:** 取得した画像や CSS データから Blob URL を生成する機能の確認・実装。(新規追加)
-   [ ] `I` (UserInterfaces) モジュール (一部):
    -   [ ] スワイプ検出によるページ移動 (`R.moveBy`) の実装 (FlickObserver参考)。
    -   [ ] クリック (画面左右/リンク) によるページ移動/ジャンプ (`R.moveBy`/`R.focusOn`) の実装。

## 2. 削除・大幅削減する機能

-   [ ] 拡張機能 (`X` モジュール、`Bibi.loadExtensions`, `S['extensions']`) の完全削除。
-   [ ] 複雑な設定管理 (`S` モジュール, `bibi-preset.js`) の廃止、必要最低限の設定に。
-   [ ] 高度な UI (`I` モジュール) の削除:
    -   [ ] メニューバー、設定パネル、スライダー、インフォメーション表示、ヘルプ。
    -   [ ] ピンチイン/アウトによる拡大縮小 (`I.Loupe`, `I.PinchObserver`)。
    -   [ ] キーボード操作 (`I.KeyObserver`)。
    -   [ ] ビューモード切り替え (`I.AxisSwitcher`)。
-   [ ] Zine 対応コードの削除。
-   [ ] `on-the-fly` Extractor (Range Request) の削除。
-   [ ] `O.Biscuits` (Cookie/状態保存) の削除 (必要なら localStorage で代替)。
-   [ ] レガシー IE 対応コード (Polyfill, CSS ハック) の削除。
-   [ ] カバー画像生成 (`L.createCover`) の削除。
-   [ ] 開発モード関連コードの削除。

## 3. 新規・改造で追加する機能 (要求定義対応)

-   [ ] 文字サイズ変更: UI (ボタン等) と全 `<iframe>` の `font-size` を変更するロジックの実装。
-   [ ] テーマ変更: UI (ボタン等) と `<body>` へのクラス付与 + CSS による背景色・文字色変更ロジックの実装。
-   [ ] 目次表示: `L.loadNavigation` のデータからシンプルなリスト UI を生成し、項目クリックで `R.focusOn` を呼び出す機能の実装。
-   [ ] 進捗表示: 現在ページ/総ページ数からパーセンテージを計算し表示する UI の実装。
-   [ ] クリックによるページめくり: 画面左右領域へのクリックイベントリスナーと `R.moveBy` 呼び出し処理の実装 (1. のタスクと重複する可能性あり、要確認)。

## 4. 開発ステップ

-   [ ] ステップ 0: 初期ディレクトリ構成の策定
-   [ ] ステップ 1: 不要機能の削除・コメントアウト。
-   [ ] ステップ 2: `JSZip` ベースの ZIP 展開・ファイル読み込み実装。
-   [ ] ステップ 3: 基本的な EPUB 表示とページめくり (スワイプ/クリック) の実現。
-   [ ] ステップ 4: 目次表示・ジャンプ、ページ内リンクの動作確認・実装。
-   [ ] ステップ 5: 縦書き表示の確認・調整。
-   [ ] ステップ 6: 文字サイズ変更、テーマ変更、進捗表示の UI・ロジック実装。 