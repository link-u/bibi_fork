# 開発状況レポート (フェーズ3完了時点)

本ドキュメントは `epub_vanillajs_viewer` プロジェクトのフェーズ3完了時点での実装状況と、Bibi L モジュールとの比較をまとめたものです。

---

## HEAD 情報
```bash
## コミットハッシュ
$ git rev-parse HEAD
ba9ea4414e2739acf0c6e57de232ac2174f3fdc8

## タグ
$ git tag --points-at $(git rev-parse HEAD)
2025-04-30_v1
```

---

## `epub_vanillajs_viewer` の現状 (フェーズ1～3)

### 完了フェーズ
- フェーズ1: プロジェクトセットアップ
- フェーズ2: EPUB 解析モジュール (Loader)
- フェーズ3: リソースロードと前処理

### 実装済みファイルと主要機能
- **`src/main.js`**: アプリケーションのエントリーポイント。ファイル選択から EPUB 解析、初期表示、リンクハンドリングまでの一連の流れを制御。
- **`src/utils/jszipWrapper.js`**: `loadZip()` - JSZip を用いた ZIP 展開。
- **`src/loader/containerParser.js`**: `parseContainer()` - `container.xml` 解析。
- **`src/loader/opfParser.js`**: `parseOpf()` - OPF (manifest/spine) 解析。
- **`src/loader/navParser.js`**: `parseNav()` - NavDoc/NCX (目次) 解析。
- **`src/utils/resourceLoader.js`**:
  - `loadHTMLAsBlobURL()`: HTML を Blob URL 化。
  - `loadCSSAsBlobURL()`: CSS を Blob URL 化 (内部 `url()` も置換)。
  - `loadImageAsBlobURL()`: 画像/SVG を Blob URL 化。
- **`src/utils/debug.js`**: `console_stringfy_log()` - 整形ログ出力ユーティリティ。

### 達成機能概要
EPUB ファイルを選択すると、その構造 (container, OPF, Nav) を解析し、最初に見つかった XHTML コンテンツを `<iframe>` に表示します。関連する CSS や画像も Blob URL を介して読み込まれ、スタイルが適用された状態で表示されます。`<iframe>` 内のリンクをクリックすると、対応する HTML コンテンツに表示が切り替わります。

---

## Bibi L モジュールとの詳細比較

| Bibi 機能           | `epub_vanillajs_viewer` 対応箇所                                                                   | 尊重度・再現度・差異                                                                                                                                                                                                                                                                                       | 再現度 |
| :------------------ | :------------------------------------------------------------------------------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |------:|
| **1. initializeBook** | `main.js` の `handleFile()` 内 (`loadZip` 呼び出し部分)                                             | **低い (20%)**。Bibi の複雑な初期化シーケンスや環境判定は未実装。単純な ZIP 展開のみ。                                                                                                                                                                                                                         |   20% |
| **2. loadContainer**  | `src/loader/containerParser.js` (`parseContainer`)                                                  | **非常に高い (100%)**。Bibi と同等のロジック (`DOMParser` 使用)。                                                                                                                                                                                                                                   |  100% |
| **3. loadPackage**    | `src/loader/opfParser.js` (`parseOpf`)                                                              | **高い (80%)**。Manifest/Spine の基本情報は取得。Bibi が扱う Metadata や Properties の詳細は省略。                                                                                                                                                                                           |   80% |
| **4. loadNavigation** | `src/loader/navParser.js` (`parseNav`)                                                              | **非常に高い (95%)**。EPUB3 NavDoc/EPUB2 NCX の標準形式に対応し、Bibi の主要ロジックを再現。フォールバックも追加。                                                                                                                                                                            |   95% |
| **5. loadItem**       | `src/utils/resourceLoader.js` (`loadHTMLAsBlobURL`) と `main.js` の `<iframe>` 生成・注入部分             | **高い (80%)**。Blob URL で `<iframe>` に注入する方式は Bibi の主要モードと同じ。スクリプト除去やサニタイズは省略。                                                                                                                                                                         |   80% |
| **6. postprocessItem**| (未実装)                                                                                           | **低い (0%)**。Bibi の `<iframe>` ロード後の調整 (リンク補正、スタイルパッチ等) は未実装。                                                                                                                                                                                                     |    0% |
| **7. preprocessResources**| `src/utils/resourceLoader.js` (`loadCSSAsBlobURL`, `loadImageAsBlobURL`)                        | **中程度 (70%)**。CSS/画像/SVG の Blob URL 化と CSS 内 `url()` 置換は実装。Bibi の再帰的な `@import` 処理やサニタイズは省略。                                                                                                                                                              |   70% |

---

## 総括
- EPUB のコア解析 (Container, OPF, Nav) と基本的なリソース表示 (HTML, CSS, 画像の Blob URL 化と `<iframe>` への注入) に関しては、Bibi の主要なロジックを尊重しつつ、シンプルに再実装できています。
- Bibi が持つ高度な機能 (複雑な初期化、詳細な Metadata 解析、`postprocessItem` での DOM 調整やスタイルパッチ、高度なリソース前処理、堅牢なエラーハンドリング、各種オプション等) は意図的に省略・簡略化しています。
- 現状は「Bibi L モジュールのコア機能を抽出・再実装した状態」であり、フェーズ4 (レンダリング基盤) へ進む準備が整っています。 