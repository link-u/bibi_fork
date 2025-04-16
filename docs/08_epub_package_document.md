# [EPUB] Package Document

EPUBの作者などのメタデータやコンテンツファイルのパスや表示順を記述するファイル。

ファイル名は任意だが、`item/standard.opf`となっていることが多い。

- 名前空間
    
    xmlにはプログラム言語と似た名前空間という仕組みがある。PackageDocumentで多く使用する名前空間は`http://www.idpf.org/2007/opf` なのでこれをデフォルトの名前空間に指定することが多い。
    

### [package](https://www.w3.org/TR/epub/#sec-package-elem)要素

version: EPUB3では”3.0”を指定する

unique-identifier: 後述するmetadataのdc:identifier要素のidを指定する

```xml
<?xml version="1.0" encoding="utf-8"?>
<package version="3.0" unique-identifier="unique-id" xmlns="http://www.idpf.org/2007/opf">
<!--versionとunique-identifierは必須-->
  <metadata><!--必須-->
  </metadata>
  <manifest><!--必須-->
  </manifest>
  <spine><!--必須-->
  </spine>
</package>
```

### [metadata](https://www.w3.org/TR/epub-33/#sec-metadata-elem)要素

metadataの要素の一部にはdc名前空間を使う。dcとは[Dublin Core](https://ja.wikipedia.org/wiki/Dublin_Core)の略で、メタデータの規格のようなものらしい。

```xml
<metadata xmlns:dc="http://purl.org/dc/elements/1.1/">
  <dc:identifier id="unique-id">UUIDやISBNなどを書く</dc:identifier><!--必須1個以上-->
  <dc:title>タイトル</dc:title><!--必須1個以上-->
  <dc:title xml:lang="en">Title</dc:title><!--xml:langで言語を指定できる-->
  <dc:creator id="#creator01">作者</dc:creator><!--必須1個以上-->
  <meta property="file-as" refines="#creator01">さくしゃ</meta><!--任意：ソート順などに使う-->

  <meta property="dcterms:modified">2016-01-01T00:00:01Z</meta><!--必須：最終更新日時-->

  <!--小説ではなくマンガのようにFixed layoutとして表示する-->
  <meta property="rendition:layout">pre-paginated</meta>
</metadata>
```

- meta要素について
    
    property属性[必須]で指定できる値は次を参照。
    
    [property一覧](https://www.w3.org/TR/epub/#app-meta-property-vocab)
    
    [予約prefix一覧](https://www.w3.org/TR/epub/#sec-reserved-prefixes)
    
    また、package要素のprefix属性で標準以外のprefixを指定すると利用できるようになる。
    
    例えば電書協のガイドラインでは以下のようにprefixを追加してmeta要素で利用する必要がある。
    
    ```xml
    <package prefix="ebpaj: http://www.ebpaj.jp/">
    	<metadata>
    		<meta property="ebpaj:guide-version">1.1.3</meta>
    	</metadata>
    </package>
    ```
    

### [manifest](https://www.w3.org/TR/epub/#sec-pkg-manifest)要素

EPUB内のコンテンツファイルをリストする。(表示順序はここではなくspine要素で指定)

hrefに指定するパスはopfファイルからの相対パス。

```xml
<manifest>
  <!--目次ファイルにはproperties="nav"をつける-->
  <item id="nav" href="navigation-documents.html" properties="nav" media-type="application/xhtml+xml" />
	<!--表紙にはproperties="cover-image"をつける-->
	<item id="cover" href="image/cover.jpg" properties="cover-image" media-type="image/jpeg" />

</manifest>
```

### [spine](https://www.w3.org/TR/epub/#sec-pkg-spine)要素

itemrefによって本のページ順序を指定する

page-progression-direction: 読む方向 ”rtl”は右から左, “ltr”は左から右

```xml
<spine page-progression-direction="rtl"><!--マンガは右から左-->
	<!--page-spread-center:見開きではないページ-->
	<itemref idref="p-cover" properties="page-spread-center"/>
	<!--page-spread-right:見開きの右ページ-->
  <itemref idref="p-001" properties="page-spread-right"/>
	<!--page-spread-left:見開きの左ページ-->
  <itemref idref="p-002" properties="page-spread-left"/>
</spine>
```

itemref要素

idrefはmanifest.item.idで定義した値を指定する。

参考

https://www.w3.org/TR/epub-33/#sec-package-doc