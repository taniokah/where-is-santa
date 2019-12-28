# Where is Santa?

This repo is a project for the advent calendar of information retrieval and search engine 2019.

このリポジトリでは、情報検索・検索エンジン Advent Calendar 2019 の 25日目、そう、クリスマスイブの夜からクリスマスの朝までにサンタクロースを捕捉し、その存在を証明するために、インターネットでサンタクロース情報を収集、インデックス、様々なクエリで検索するためのシステムを公開します。すべてのプログラムは、Google colab 上で試しているので、みなさんが実行することは難しくないと思います。

# Crawler for Santa

まずは、サンタの情報を入手するためのクローラを用意します。クローラには、icrawler を使います。Google、Bing、Baidu などの画像検索サイトからダウンロードすることができます。

``` 
!pip install icrawler
```

各サイトで 1,000 件ずつダウンロードするなら、以下のようにします。

```Python
from icrawler.builtin import BaiduImageCrawler, BingImageCrawler, GoogleImageCrawler

crawler = GoogleImageCrawler(storage={"root_dir": "google_images"}, downloader_threads=4)
crawler.crawl(keyword="Santa", offset=0, max_num=1000)

bing_crawler = BingImageCrawler(storage={'root_dir': 'bing_images'}, downloader_threads=4)
bing_crawler.crawl(keyword='Santa', filters=None, offset=0, max_num=1000)

baidu_crawler = BaiduImageCrawler(storage={'root_dir': 'baidu_images'})
baidu_crawler.crawl(keyword='Santa', offset=0, max_num=1000)
```

# Indexer for Santa

次に、elasticsearch の cosineSimilaritySparse を使ってインデックスを作ります。今回は、ベクトル検索をしてみます。ベクトル検索といっても、VGG-16 の出力ラベル 1,000 種類をベクトルとして使うので、embedded とは少し違いますが、やろうと思えば同じやり方でできます。今回は、7.5.1 を使いました。

```
!wget https://artifacts.elastic.co/downloads/elasticsearch/elasticsearch-7.5.1-linux-x86_64.tar.gz -q
!tar -xzf elasticsearch-7.5.1-linux-x86_64.tar.gz

!chown -R daemon:daemon elasticsearch-7.5.1/
```

elasticsearch がインストールできたら、起動してみます。Google colab では、サーバ起動は簡単にはできないのですが、daemon として実行することは可能です。

```Python
import os
from subprocess import Popen, PIPE, STDOUT
es_server = Popen(['elasticsearch-7.5.1/bin/elasticsearch'], 
                  stdout=PIPE, stderr=STDOUT,
                  preexec_fn=lambda: os.setuid(1)  # as daemon
                 )

!ps aux | grep elastic
!sleep 30
!curl -X GET "localhost:9200/"
```

30秒待ってからサーバの起動状態をチェックしています。クライアント側の環境も整えましょう。

```Python
!pip install elasticsearch

from datetime import datetime
from elasticsearch import Elasticsearch
es = Elasticsearch(timeout=60)

doc = {
    'author': 'Santa Claus',
    'text': 'Where is Santa Claus?',
    'timestamp': datetime.now(),
}
res = es.index(index="test-index", doc_type='tweet', id=1, body=doc)
print(res['result'])

res = es.get(index="test-index", doc_type='tweet', id=1)
print(res['_source'])

es.indices.refresh(index="test-index")

res = es.search(index="test-index", body={"query": {"match_all": {}}})
print("Got %d Hits:" % res['hits']['total']['value'])
for hit in res['hits']['hits']:
    print("%(timestamp)s %(author)s: %(text)s" % hit["_source"])
```

ここまでうまくいけば、次は画像から特徴ベクトルを取り出すことを考えます。今回は、おなじみ VGG-16 を利用しています。

```Python
# Load libraries
from keras.applications.vgg16 import VGG16, preprocess_input, decode_predictions
from keras.preprocessing import image
from PIL import Image
import matplotlib.pyplot as plt
import numpy as np
import sys

model = VGG16(weights='imagenet')
```

さらに、画像と特徴ベクトルの情報を表示するためのプログラムを用意しまして、、

```Python
def predict(filename, featuresize, scale=1.0):
    img = image.load_img(filename, target_size=(224, 224))
    return predictimg(img, featuresize, scale=1.0)

def predictpart(filename, featuresize, scale=1.0, size=1):
    im = Image.open(filename)
    width, height = im.size
    im = im.resize((width * size, height * size))
    im_list = np.asarray(im)
    # partition
    out_img = []
    if size > 1: 
        v_split = size
        h_split = size
        [out_img.extend(np.hsplit(h_img, h_split)) for h_img in np.vsplit(im_list, v_split)]
    else:
        out_img.append(im_list)
    reslist = []
    for offset in range(size * size):
        img = Image.fromarray(out_img[offset])
        reslist.append(predictimg(img, featuresize, scale))
    return reslist

def predictimg(img, featuresize, scale=1.0):
    width, height = img.size
    img = img.resize((int(width * scale), int(height * scale)))
    img = img.resize((224, 224))
    x = image.img_to_array(img)
    x = np.expand_dims(x, axis=0)
    preds = model.predict(preprocess_input(x))
    results = decode_predictions(preds, top=featuresize)[0]
    return results

def showimg(filename, title, i, scale=1.0, col=2, row=5):
    im = Image.open(filename)
    width, height = im.size
    im = im.resize((int(width * scale), int(height * scale)))
    im = im.resize((width, height))
    im_list = np.asarray(im)
    plt.subplot(col, row, i)
    plt.title(title)
    plt.axis("off")
    plt.imshow(im_list)
    
def showpartimg(filename, title, i, size, scale=1.0, col=2, row=5):
    im = Image.open(filename)
    width, height = im.size
    im = im.resize((int(width * scale), int(height * scale)))
    #im = im.resize((width, height))
    im = im.resize((width * size, height * size))
    im_list = np.asarray(im)
    # partition
    out_img = []
    if size > 1: 
        v_split = size
        h_split = size
        [out_img.extend(np.hsplit(h_img, h_split)) for h_img in np.vsplit(im_list, v_split)]
    else:
        out_img.append(im_list)
    # draw image
    for offset in range(size * size):
        im_list = out_img[offset]
        pos = i + offset
        print(str(col) + ' ' + str(row) + ' ' + str(pos))
        plt.subplot(col, row, pos)
        plt.title(title)
        plt.axis("off")
        plt.imshow(im_list)
        out_img[offset] = Image.fromarray(im_list)
    return out_img
````

1枚目1枚目の画像を表示してみましょう。

```Python
# Predict an image
scale = 1.0
filename = "google_images/000046.jpg"
plt.figure(figsize=(20, 10))

#showimg(filename, "query", i+1, scale)
imgs = showpartimg(filename, "query", 1, 1, scale)
plt.show()

for img in imgs:
    reslist = predictpart(filename, 10, scale)
    for results in reslist:
        for result in results:
            print(result)
    print()
````

さらに、elasticsearch へ画像ベクトルのインデックスを作る準備です。

```Python
import math
import binascii
from datetime import datetime

def createindex(indexname):
    if es.indices.exists(index=indexname):
        es.indices.delete(index=indexname)
    es.indices.create(index=indexname,  body={
        "settings": {
            "index.mapping.total_fields.limit": 10000,
        }
    })

    mapping = {
        "image": {
            "properties": {
                "f": {
                    "type": "text"
                },
                's': {
                    "type": "sparse_vector"
                }
            }
        }
    }
    es.indices.put_mapping(index=indexname, doc_type='image', body=mapping, include_type_name=True)

wnidmap = {}

def loadimages(directory):
    imagefiles = []
    for file in os.listdir(directory):
        if file.rfind('.jpg') < 0:
            continue
        filepath = os.path.join(directory, file)
        imagefiles.append(filepath)
    return imagefiles

def indexfiles(indexname, directory, featuresize=10, docsize=1000):
    imagefiles = loadimages(directory)
    for i in range(len(imagefiles)):
        if i >= docsize:
            return
        filename = imagefiles[i]
        indexfile(indexname, filename, i, featuresize)
        sys.stdout.write("\r%d" % (i + 1))
        sys.stdout.flush()
    es.indices.refresh(index=indexname)    

def indexfile(indexname, filename, i, featuresize):
    global wnidmap

    rounddown = 16
    doc = {'f': filename, 's':{}}
    results = predict(filename, featuresize) 

    #print(len(results))
    synset = doc['s']
    for result in results:
        score = float(str(result[2]))
        wnid = result[0]
        id = 0
        if wnid in wnidmap.keys():
            id = wnidmap[wnid]
        else:
            id = len(wnidmap)
            wnidmap[wnid] = id
        synset[str(id)] = score

    #print(doc)
    #count = es.count(index=indexname, doc_type='image')['count']
    count = i
    res = es.index(index=indexname, doc_type='image', id=count, body=doc)
```

いよいよ、画像の特徴ベクトルを elasticsearch へ追加していきましょう。

```Python
createindex("santa-search")

directory = "google_images/"
indexfiles("santa-search", directory, 100, 1000)
#directory = "bing_images/"
#indexfiles("santa-search", directory, 100, 1000)
#directory = "baidu_images/"
#indexfiles("santa-search", directory, 100, 1000)
```

# Searcher for Santa

サンタの画像の特徴ベクトルを用いて elasticsearch のインデックスを作りました。いよいよ、この画像の中から真のサンタ "My Santa Claus" を見つけましょう。今回の検索クエリは、__画像__です。試しに、ダウンロードした中から1つ画像を選んで、クエリにしてみましょう。

```Python
disp = True
filename = "google_images/000001.jpg"
_ = searchimg('santa-search', filename, 10, 10, 'vcos', 1.0, 1)
```

どうですか？うまく検索できましたか？うまくいけば、一番近い画像から順に表示されるはずです。インデックスした画像とは別に、新たな画像を用意して、検索してもよいかもしれません。検索エンジンをつかって、世界中のサンタのなかから、あなただけのサンタを探してみましょう。
