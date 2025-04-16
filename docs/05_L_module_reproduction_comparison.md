# Lモジュール再現度比較

本ドキュメントでは、Bibi の L モジュール（EPUB の展開・解析部分）と当プロジェクトでの再現実装状況を比較・記録します。

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

## Bibi L モジュールの主要機能
1. **initializeBook**
   - 入力形式判定 (URI/File/Blob/Base64)
   - 展開ポリシー (on-the-fly / at-once) の設定
2. **loadContainer**
   - `META-INF/container.xml` 解析 → OPF ファイルパス取得
3. **loadPackage**
   - OPF 解析
   - Metadata、Manifest、Spine 情報格納
4. **loadNavigation**
   - NavDoc(NCX) 解析 → 目次情報取得
5. **loadItem**
   - XHTML/CSS/画像/SVG の前処理
   - Blob URL 生成 or document.write
   - `<iframe>` に注入
6. **postprocessItem**
   - リンク補正、縦書きパッチなど
7. **preprocessResources**
   - CSS/HTML の再帰的パス置換・サニタイズ

---

## 本プロジェクトでの再現状況
以下は主要機能ごとの再現度をまとめたものです。

| 機能                | Bibi 実装                                                                                       | 本実装                                                     | 再現度 |
|--------------------|------------------------------------------------------------------------------------------------|------------------------------------------------------------|------:|
| ZIP 展開           | JSZip による on-the-fly / at-once の両対応                                                       | **at-once** のみ (JSZip loadAsync)                         |    40%|
| loadContainer      | `container.xml` → OPF パス取得                                                                 | DOMParser で同等に実装 (parseContainer)                    |   100%|
| loadPackage        | OPF の Metadata / Manifest / Spine / Rendition 等多彩に格納                                    | manifest/spine のみ取得 (parseOpf)                         |    80%|
| loadNavigation     | NavDoc(NCX) と NCX 両対応、高度な NavDoc 拡張                                                 | NavDoc(NCX) 両対応 + フォールバック実装 (parseNav)        |    90%|
| loadItem           | XHTML/CSS/画像/SVG の前処理・Blob URL / document.write                                          | HTML アイテムの Blob URL 化 + `<iframe>` 注入              |    80%|
| postprocessItem    | CSS パッチ、縦書き補正、リンク補正など                                                          | 未実装                                                    |     0%|
| preprocessResources| CSS 内 `@import`/`url()` の再帰的な Blob URL 変換とサニタイズ                                   | CSS/画像/SVG の Blob URL 変換を実装                         |    70%|

---

### 総括
- Container→OPF→Nav 解析およびリソース取得系機能はほぼ Bibi レベルで再現済み。
- _postprocessItem_ 以降のブラウザ依存パッチやページネーション、縦書き処理は未着手。
- 今後はこれらの機能を段階的に追加し、Bibi と同等の完成度を目指します。 