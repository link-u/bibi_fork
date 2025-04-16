# [EPUB] Navigation Document

目次を書くファイル。リーダーが内部的に使う目次ファイルを論理目次、ユーザーが見るための目次ファイルを本文目次と呼んで区別することがある。

EPUB2まではncxファイルで目次を表したが、EPUB3ではxhtmlに変更された。

```xml
<nav epub:type="toc">
<h1>title</h1><!--任意(h1-h6) 0 or 1個-->
<ol><!--必須１個-->
<li><!--ol内に１個以上必要-->
<!--spanかaが１個-->
	<a href="path/to/xhtml">Chapter 1</a>
</li>
<li><!--複数の目次をグループ化できる-->
	<span>Tables in Chapter 2</span>
	<ol>
		<li><a href="...">Table 2.1</a>
		<li><a href="...">Table 2.2</a>
	</ol>
</li>
<li>
	<a href="...">Tables in Chapter 3</span>
	<ol>
		<li><a href="...">Table 3.1</a>
		<li><a href="..."><img src="a.jpg" alt="画像タイトル" /></a>
	</ol>
</li>
</ol>
</nav>
```

### ncxファイル

電子書籍協会のガイドラインではncxとnavigation docutmentが両方あったらncxは無視するべきとある。つまりEPUB3ではnavigation documentだけ読めばよい。

参考

https://techracho.bpsinc.jp/baba/2017_07_04/42617

https://idpf.org/epub/30/spec/epub30-contentdocs.html#sec-xhtml-nav

https://www.w3.org/TR/epub-33/#sec-nav