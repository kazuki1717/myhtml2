# myhtml2 - 高速なHTMLパーサー

C言語で書かれた軽量・高速なHTMLライブラリ、以前のバージョン (beta1.0 から 1.0) と比べてより高速、高機能、高安定性、読みやすいなどで全方位的にアップグレードされています

| バージョン   | 2.3.0       |
| :----------- | :---------- |
| プログラマー | かずき256   |
| 完成日時     | 24 Jun 2025 |

## 特徴

- 文字列、ファイル、カスタマイズリーダー (HtmlStream) や libcurl などからHTMLを解析できる
- 各HTMLバージョンの対応性、BeautifulSoup4 を参考したのエラードキュメント処理
- タグ名、クラス、ID や検索番目 [index] による要素検索
- 外部参考いらない、唯一参考 libcurl が導入していないと対応のメソッドを作成しない
- HtmlObject 書き込み可能
- 簡潔なコードができる
- デバックの表示、全メソッドに NULL 入力が処理が対処され、デバックメッセージをオフも可能

---

## インストールとコンパイル

1. `git clone https://www.github.com/Fuuki255/myhtml2` で myhtml2 をダウンロード
2. Cプログラムに `#include "myhtml2/myhtml.h"` ヘッダーでライブラリを導入すればいい
3. `gcc`、`g++` や `clang` でCファイルコンパイル、libcurl を使用する場合は `-lcurl` を追加する

---

## サンプル

```c
#include <curl/curl.h>        // libcurl の拡張機能を使用するには myhtml.h 前に定義する必要がある
#include "myhtml.h"

int main(int argc, char** argv) {
  /* libcurl 初期化 */
  CURL* curl = curl_easy_init();

  /* https://example.com からHTML を取得する */
  HtmlObject* doc = HtmlReadObjectFromCURL(curl, "https://example.com");

  /* 検索実例1: <title></title> */
  HtmlObject* tagTitle = HtmlFindObject(doc, "title");
  printf("Title %s\n", HtmlGetObjectInnerText(tagTitle));

  /* 検索実例2: body プリント */
  HtmlObject* tagBody = HtmlFindObject(doc, "body[-1]" /* インデックス -1 に設定することで逆方向の検索ができます */);

  // GetObjectInnerText() と違って、GetObjectText() は子オブジェクトのテキストも含めたゲットなので新しいバッファが必要
  printf("Text:\n%s\n", HtmlGetObjectText(tagBody));

  /* (オプション) HTML 出力 */
  HtmlWriteObjectToFile(doc, "output.html");

  /* クリーンアップ */
  HtmlDestroyObject(doc);

  curl_easy_cleanup(curl);
  return 0;
}
```

---

## スピードテスト

youtube.com からダウンロードした 3.39 MB の HTML を読み取り、`<img>` を検索するテストに、

mythml2 は 24.24ms で HTML を構造化し、2.17ms で 282 つの `<img>` を見つけました。

一方、BeautifulSoup4 は 428.74ms で HTML を構造化し、47.16ms で 282 つの `<img>` を見つけました。

myhtml2 は 17 倍以上の性能を持っている。

<img src="speedtestC.png">

<img src="speedtestPython.png">

<br>

こちらがスピードテストのコードです

コンパイルコマンド: `gcc -o speedtest speedtest.c -O3`

```c
#include "myhtml.h"
#include <time.h>

// a timeit method that getting function usetime in ms
double Timeit(void (*func)(void*), void* param) {
	struct timespec start, end;
	clock_gettime(CLOCK_MONOTONIC, &start);
	func(param);
	clock_gettime(CLOCK_MONOTONIC, &end);
	return end.tv_sec - start.tv_sec + (end.tv_nsec - start.tv_nsec) / 1.0e6;
}


// -- define test function --
HtmlObject* doc;
HtmlArray array;

void TestParse(void* param) {
	doc = HtmlReadObjectFromStream((HtmlStream*)param);
}

void TestSelect(void* param) {
	array = HtmlFindAllObjects(doc, "img", 0);
}

// -- main --
int main(int argc, char** argv) {
	// read html file
    // library have HtmlReadObjectFromFile, but file reading will slow down the speed
	size_t readed, fileSize = 0;

	HtmlStream stream = HtmlCreateStreamBuffer(4096);
	FILE* file = fopen("riskroll.html", "r");
	char buffer[4096];

	while ((readed = fread(buffer, 1, sizeof(buffer), file))) {
		stream.write(buffer, 1, readed, stream.data);
		fileSize += readed;
	}
	fclose(file);


	// getting usetime
	double parseUsetime = Timeit(TestParse, &stream);
	double selectUsetime = Timeit(TestSelect, NULL);


	// -- print <img> --
	for (int i = 0; i < array.length; i++) {
		printf("	%s\n", HtmlWriteObjectToString(array.values[i]));
	}
	putchar('\n');

	// -- print information --
    double fileSizeDouble = (double)fileSize / 1024 / 1024;

	printf("C myhtml2:\n");
	printf("  file size: %.2lf MB\n", fileSizeDouble);
	printf("  <img> count: %d\n", array.length);
	printf("  parse usetime: %.2lf ms (every %.2lf ms per MB)\n", parseUsetime, parseUsetime / fileSizeDouble);
	printf("  select <img> usetime: %.2lf ms (every %.2lf us per img)\n", selectUsetime, selectUsetime * 1000 / array.length);

	// -- cleanup --
	HtmlDestroyObject(doc);
	HtmlDestroyArray(&array);
	HtmlDestroyStream(&stream);
	return 0;
}

```

<br>

Python BeautifulSoup のスピードテスト

```python
from bs4 import BeautifulSoup
from time import time

### read file text ###
file = open("riskroll.html", "r", encoding="utf-8")
html = file.read()
file.close()

### test parse usetime ###
parseStart = time()
doc = BeautifulSoup(html)
parseEnd = time()


### test select usetime ###
selectStart = time()
select = doc.select("img");
selectEnd = time()


### print <img> ###
print("All <img>:")
for tag in select:
    print("  " + str(tag))
print()


### calculation ###
filesize = len(html) / 1024 / 1024
parseUsetime = (parseEnd - parseStart) * 1000
selectUsetime = (selectEnd - selectStart) * 1000

### print information ###
print("Python BeautifulSoup4:")
print("  file size: %.2f MB" % (filesize))
print("  <img> count: %d" % len(select))
print("  parse usetime: %.2f ms (every %.2f ms per MB)" % (parseUsetime, parseUsetime / filesize))
print("  select <img> usetime: %.2f ms (every %.2f us per img)" % (selectUsetime, selectUsetime * 1000 / len(select)))
```

---

## 使い方

### 1. HTMLの読み込み

```c
HtmlObject* obj;
FILE* fp;
HtmlStream htmlStream;

// 文字列から
obj = HtmlReadObjectFromString("<html>...</html>");

// ファイルから
obj = HtmlReadObjectFromFile("example.html");

// FILE* から
obj = HtmlReadObjectFromFileObject(fp);

// ストリームから（カスタム実装）
obj = HtmlReadObjectFromStream(&htmlStream);

// libcurl で取得（拡張）
obj = HtmlReadObjectFromCURL(curl, "https://example.com");
```

---

### 2. オブジェクト検索

"tagName.className#tagId[index] (nextPattern ...)" というバタンを使用した検索

- tagName はタグ名で検索 （大小文字気にせず）
- className はクラス名で検索（大小文字区別あり）
- tagId は ID で検索（大小文字区別あり）
- [index] はその検索結果から index 番目の結果を戻す (、整数は正順から、-1 は逆番から

* tagName, className や tagId 任意の一つが必要
* [index] はオプション
* CreateSearch(), FindObject(), FindAllObjects() の検索結果は同じで結果方式が違う

```c
HtmlObject* object;

/* イテレータ検索 */
HtmlSelect* select = HtmlCreateSelect(doc, "div.main" /* クラスが main のタグを選択*/);

while ((object = HtmlNextSelect(&select)) {
	// do something ...
}

HtmlDestroySelect(&select);


/* 最初を検索 (HtmlSearch ベース) */
HtmlObject* found = HtmlFindObject(doc, "form#searchbox" /* ID searchbox のformを検査 */);

// do something ...


/* すべてを検索し、配列に保存する (HtmlSearch ベース)

HtmlArray HtmlFindAllObjects(HtmlObject* object, const char* patterns, int limit);

パラメーター:
- object HTMLオブジェクト
- patterns 検索パタン
- limit 最大オブジェクト検索数、-1 で制限なし

return オブジェクト配列
*/
HtmlArray array = HtmlFindAllObjects(doc, "a.bottom", 7);

for (int i = 0; i < array.length; i++) {
	// do something ...
}

HtmlDestroyArray(&array);
```

---

### 3. データ取得

```c
// -- 属性データを取得 --
const char* value = HtmlGetObjectAttributeValue(found, "href");


// -- 内部テキストを取得 --
const char* innerText = HtmlGetObjectInnerText(object);


// -- 内部すべてのタグのテキストを取得 --
// 新メモリを作るため、通常ではストリームを作成してそこに書くだが、
// それは EX 版に移し、メモリの余剰空間を使用する HtmlGetObjectText となった
// その空間の具体的な場所は `object->name + strlen(object->name) + 1`
const char* text = HtmlGetObjectText(object);


// -- 内部すべてのタグのテキストを取得 EX --
HtmlStream stream = HtmlCreateStreamBuffer(256);
HtmlGetObjectTextEx(object, &stream);
const char* textEx = HtmlGetStreamString(&stream);

HtmlDestroyStream(&stream);


// -- 子オブジェクトをカウント --
int childCount = HtmlCountObjectChildren(object);


// -- 属性をカウント --
int attrCount = HtmlCountObjectAttributes(object);
```

---

### 4. (オプション) オブジェクト作成

```c
// ドキュメント作成
// フルネーム HtmlCreateObjectDocument() 両方使用可能
// <html> ではなく、配列として機能している
HtmlObject* doc = HtmlCreateDocument();


// <!DOCTYPE html> 作成
HtmlObject* doctype = HtmlCreateObjectDoctype(doc, "html");


/* HtmlCreateObjectTag() */

// ドキュメントに普通のタグを作る <html></html>
HtmlObject* tagHtml = HtmlCreateObjectTag(doc, "html");

// <html></html> に普通のタグを作る <head></head>
HtmlObject* tagHead = HtmlCreateObjectTag(tagHtml, "head");


/* EX版タグ作成 <title>Hello, World</title>

HtmlObject* CreateObjectTagEx(
    HtmlObject* parent,
    const char* tagName,
    const char* innerText,
    const char* afterText);

パラメーター:
- parent 親タグ
- tagName タグ名
- innerText 内部テキスト
- afterText 次のテキストまでの隙間

return タグオブジェクト
*/
HtmlCreateObjectTagEx(tagHead, "title", "Hello, World", NULL);


// シングルタグを作成 <meta>
// EX版作成予定あり
HtmlObject* tagMeta = HtmlCreateObjectSingle(tagHead, "meta");
// 属性変更
HtmlSetObjectAtterValue(tagMeta, "charset", "utf-8");


// スクリプト作成 Script, Style
HtmlCreateObjectScript(tagHead,
    "console.log(\"Hello, World\");");

HtmlCreateObjectStyle(tagHead,
    "p { text-align: center; }");


// コメント作成 <!-- This is a comment -->
HtmlCreateObjectComment(tagHead, " This is a comment ");

```

---

### 5. (オプション) オブジェクト変更

```c
// テキスト設定
HtmlObjectSetInnerText(tagH1, "Hello, World");

// 属性設定
HtmlSetObjectAtterValue(tagMeta, "charset", "utf-8");


// オブジェクト追加
HtmlAddObjectChild(object, child);

// object を target の前に移動
HtmlInsertObjectChildBefore(parent, target, object);

// object を target の後に移動
HtmlInsertObjectChildAfter(parent, target, object);


// html->head の <meta> を削除
// tagMeta がその <meta> だった場合、
HtmlDestroyObject(tagMeta);

// 属性削除
HtmlRemoveObjectAttribute(object, "attrName");


// オブジェクトクリア
HtmlClearObjectChildren(object);

// 属性クリア
HtmlClearObjectAttributes(object)
```

---

### 6. (オプション) HTMLの出力

```c
// ファイルに書き込む
HtmlWriteObjectToFile(doc, "output.html");

// 既存の FILE に書き込む
HtmlWriteObjectToFileObject(doc, file);

// 文字列に変換、戻り値は自動的に解放されるのでfree不要
const char* str = HtmlWriteObjectToString(doc);

// カスタマイズオブジェクトに書き込む
HtmlWriteObjectToStream(doc, &htmlStream);
```

---

### 7. メモリ解放

```c
// HtmlObject 削除
HtmlDestroyObject(obj);

// HtmlStream 削除
HtmlDestroyStream(&stream);

// HtmlArray 削除
HtmlDestroyArray(array);

// HtmlSelect 削除
HtmlDestroySelect(select);
```

---

## アップデート予定

- マルチオブジェクト検索 ("patterns, ...")
- 属性でオブジェクト検索 ("t.c#i[attrName=attrValue, index]")
- HTML記号 (e.g. &amp) 処理
- tidy機能追加、未実装だが beta1.0 で既にあったが。。。
- libcurl ストリーミングパースの追加 (スレッド)

**myhtml 3.0**

C++ にアップグレードするつもりで、新しいバージョンはこのように

```cpp
#include <curl/curl.h>        // libcurl の拡張機能を使用するには myhtml.h 前に定義する必要がある
#include "myhtml.h"

int main(int argc, char** argv) {
  // == libcurl 初期化 ==
  CURL* curl = curl_easy_init();

  // == https://example.com からHTML を取得する ==
  html_t doc(curl, "http://example.com");

  // html_t doc("<p>hello world</p>")
  // または html_t doc = html_read("index.html")

  // == 検索実例1: <title></title> ==
  html_t tag_title = doc.find(doc, "title");
  std::cout << "title: " << tag_title.get_inner_text() << "\n";

  // == 検索実例2: body プリント ==
  html_t tag_body = doc.find("body[-1]" /* インデックス -1 に設定することで逆方向の検索ができます */);

  // GetObjectInnerText() と違って、GetObjectText() は子オブジェクトのテキストも含めたゲットなので新しいバッファが必要
  std::cout << "text:\n"
    << tag_body.get_text() << "\n";

  // == (オプション) HTML 出力 ==
  doc.write("output.html");

  // libcurl クリーンアップ ==
  curl_easy_cleanup(curl);
  return 0;
}
```

---

## ライブラリ歴史

- **myhtml（beta1.0.0, 2023年）**

  - フォルダサイズ: 63.5KB
  - 最初のパーサーとして完全な機能を持っているが、サンプルドキュメントが残れず、記憶では動けるが実際は動けない可能性がある、まだコードが残っていてるとは言え、開発後に使用したことがない
  - 使用しづらい、安定性にかけている

- **MyHtml (1.0.0, 2024年)**

  - フォルダサイズ: 29.1KB
  - コードを大幅に簡潔化し、無駄なファイルを減らして再設計。
  - 検索機能が未開発で、パースや再出力ができても必要なデータが得られない、中途半端なものであるため。

- **myhtml2（2.0.0, 2025年）**

  - フォルダサイズ: 58.3KB
  - より進化した設計で、ファイル分割を最小限に抑えつつ、安定性・パフォーマンス・機能性のすべてが最高レベル。
  - 初代のドキュメント整理機能が付いていないけど、使いやすさ、安定性やパフォーマンスが非常に高く、自慢の作品です

---

## ライセンス

MIT License
