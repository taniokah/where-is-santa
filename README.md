# Where is Santa?

This repo is a project for the advent calendar of information retrieval and search engine 2019.

このリポジトリでは、情報検索・検索エンジン Advent Calendar 2019 の 25日目、そう、クリスマスイブの夜からクリスマスの朝までにサンタクロースを捕捉し、その存在を証明するために、インターネットでサンタクロース情報を収集、インデックス、様々なクエリで検索するためのシステムを公開します。

# Crawler for Santa

まずは、サンタの情報を入手するためのクローラを用意します。クローラには、icrawler を使います。Google、Bing、Baidu などの画像検索サイトからダウンロードすることができます。

```
!pip install icrawler
```

各サイトで 1,000 件ずつダウンロードするなら、以下のようにします。

```
from icrawler.builtin import BaiduImageCrawler, BingImageCrawler, GoogleImageCrawler

crawler = GoogleImageCrawler(storage={"root_dir": "google_images"}, downloader_threads=4)
crawler.crawl(keyword="Santa", offset=0, max_num=1000)

bing_crawler = BingImageCrawler(storage={'root_dir': 'bing_images'}, downloader_threads=4)
bing_crawler.crawl(keyword='Santa', filters=None, offset=0, max_num=1000)

baidu_crawler = BaiduImageCrawler(storage={'root_dir': 'baidu_images'})
baidu_crawler.crawl(keyword='Santa', offset=0, max_num=1000)
```

# Indexer for Santa

# Searcher for Santa

