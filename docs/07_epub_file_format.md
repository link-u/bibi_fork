ZIPファイルの中に指定のファイル形式で目次データやコンテンツが配置される。

パスが固定なのは以下の２ファイル（本当はもっとあるけど使ってるのを見たことない）

```
mimetype
META-INF/
  container.xml
```

mimetype

テキストファイルで中身は`application/epub+zip`と書いてあるだけ。このファイルがEPUBであることを示している。

ZIPの中のファイルには順番があり、EPUBではこのmimetypeが１番目のファイルになる必要がある。また圧縮してはならない。(zipはファイルごとに圧縮/無圧縮を選べる)

META-INF/container.xml

パッケージドキュメント(OPFファイル)の場所を示すファイル

rootfileエレメントにOPFファイルのパスを指定する。パスはzipのルートからの相対パス（先頭に/をつけてはいけない）

```xml
<?xml version="1.0"?>
<container version="1.0" xmlns="urn:oasis:names:tc:opendocument:xmlns:container">
   <rootfiles>
      <!--rootfileは複数書くことができるが、リーディングシステムで読む必要があるのは最初の１つだけ-->
      <rootfile
          full-path="item/standard.opf"
          media-type="application/oebps-package+xml" />
   </rootfiles>
</container>
```

[[EPUB] Package Document](https://www.notion.so/EPUB-Package-Document-89ff7cfc30134692ba66c8ed7f7d3e08?pvs=21) 

### 電書協のガイドラインで指定されたフォルダ構成

フォルダ構成は本来は任意だが、ガイドラインではすべてitemフォルダの下に置くことになっている。

```
item/
  standard.opf
  navigation-document.xhtml
  image/
  style/
  xhtml/
```

参考

https://www.w3.org/TR/epub-33/#sec-ocf
