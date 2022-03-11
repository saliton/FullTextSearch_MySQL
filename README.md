[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/Soliton-Analytics-Team/FullTextSearch_MySQL/blob/main/FullTextSearch_MySQL.ipynb)

# Colabで全文検索（その１：MySQL編）

各種全文検索ツールをColabで動かしてみるシリーズです。全7回の予定です。今回はMySQLです。

処理時間の計測はストレージのキャッシュとの兼ね合いがあるので、2回測ります。2回目は全てがメモリに載った状態での性能評価になります。ただ1回目もデータを投入した直後なので、メモリに載ってしまっている可能性があります。

## 準備

まずは検索対象のテキストを日本語wikiから取得して、Google Driveに保存します。（※ Google Driveに約１GBの空き容量が必要です。）

Google Driveのマウント


```python
from google.colab import drive
drive.mount('/content/drive')
```

    Mounted at /content/drive


jawikiの取得とjson形式への変換。90分ほど時間がかかります。他の全文検索シリーズでも同じデータを使うので、他の記事も試したい方は wiki.json.bz2 を捨てずに残しておくことをおすすめします。


```shell
%%time
%cd /content/
import os
if not os.path.exists('/content/drive/MyDrive/wiki.json.bz2'):
    !wget https://dumps.wikimedia.org/jawiki/latest/jawiki-latest-pages-articles.xml.bz2
    !pip install wikiextractor
    !python -m wikiextractor.WikiExtractor --no-templates --processes 4 --json -b 10G -o - jawiki-latest-pages-articles.xml.bz2 | bzip2 -c > /content/drive/MyDrive/wiki.json.bz2
```

    /content
    Requirement already satisfied: wikiextractor in /usr/local/lib/python3.7/dist-packages (3.0.6)
    INFO: Starting page extraction from jawiki-latest-pages-articles.xml.bz2.
    INFO: Using 4 extract processes.
    INFO: Extracted 100000 articles (266.4 art/s)
    INFO: Extracted 200000 articles (409.4 art/s)
    INFO: Extracted 300000 articles (504.7 art/s)
    INFO: Extracted 400000 articles (556.6 art/s)
    INFO: Extracted 500000 articles (610.1 art/s)
    INFO: Extracted 600000 articles (650.8 art/s)
    INFO: Extracted 700000 articles (659.1 art/s)
    INFO: Extracted 800000 articles (679.1 art/s)
    INFO: Extracted 900000 articles (614.8 art/s)
    INFO: Extracted 1000000 articles (645.4 art/s)
    INFO: Extracted 1100000 articles (654.3 art/s)
    INFO: Extracted 1200000 articles (637.7 art/s)
    INFO: Extracted 1300000 articles (610.6 art/s)
    INFO: Extracted 1400000 articles (593.2 art/s)
    INFO: Extracted 1500000 articles (641.7 art/s)
    INFO: Extracted 1600000 articles (634.1 art/s)
    INFO: Extracted 1700000 articles (602.7 art/s)
    INFO: Extracted 1800000 articles (629.0 art/s)
    INFO: Extracted 1900000 articles (621.6 art/s)
    INFO: Extracted 2000000 articles (683.6 art/s)
    INFO: Extracted 2100000 articles (688.4 art/s)
    INFO: Finished 4-process extraction of 2119081 articles in 3690.6s (574.2 art/s)
    CPU times: user 26.7 s, sys: 3.28 s, total: 30 s
    Wall time: 1h 1min 34s


json形式に変換されたデータを確認


```python
import json
import bz2

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    for n, line in enumerate(fin):
        data = json.loads(line)
        print(data['title'].strip(), data['text'].replace('\n', '')[:40], sep='\t')
        if n == 5:
            break
```

    アンパサンド	アンパサンド（&amp;, ）は、並立助詞「…と…」を意味する記号である。ラテン
    言語	言語（げんご）は、広辞苑や大辞泉には次のように解説されている。『日本大百科事典』
    日本語	 日本語（にほんご、にっぽんご）は、日本国内や、かつての日本領だった国、そして日
    地理学	地理学（ちりがく、、、伊：geografia、）は、。地域や空間、場所、自然環境
    EU (曖昧さ回避)	EU
    国の一覧	国の一覧（くにのいちらん）は、世界の独立国の一覧。対象.国際法上国家と言えるか否


## MySQLのインストール


```shell
!sudo apt update
!sudo apt install mysql-server mysql-client
```

    Get:1 http://security.ubuntu.com/ubuntu bionic-security InRelease [88.7 kB]
    Get:2 https://cloud.r-project.org/bin/linux/ubuntu bionic-cran40/ InRelease [3,626 B]
    Ign:3 https://developer.download.nvidia.com/compute/cuda/repos/ubuntu1804/x86_64  InRelease
    ・
    ・
    Setting up mysql-server (5.7.37-0ubuntu0.18.04.1) ...
    Processing triggers for systemd (237-3ubuntu10.53) ...
    Processing triggers for man-db (2.8.3-2ubuntu0.1) ...
    Processing triggers for libc-bin (2.27-3ubuntu1.3) ...
    /sbin/ldconfig.real: /usr/local/lib/python3.7/dist-packages/ideep4py/lib/libmkldnn.so.0 is not a symbolic link
    



```shell
!mysql --version
```

    mysql  Ver 14.14 Distrib 5.7.37, for Linux (x86_64) using  EditLine wrapper


## MySQLの立ち上げ


```shell
!service mysql start
```

     * Starting MySQL database server mysqld
    No directory, logging in with HOME=/
       ...done.



```shell
!service mysql status
```

     * /usr/bin/mysqladmin  Ver 8.42 Distrib 5.7.37, for Linux on x86_64
    Copyright (c) 2000, 2022, Oracle and/or its affiliates.
    
    Oracle is a registered trademark of Oracle Corporation and/or its
    affiliates. Other names may be trademarks of their respective
    owners.
    
    Server version		5.7.37-0ubuntu0.18.04.1
    Protocol version	10
    Connection		Localhost via UNIX socket
    UNIX socket		/var/run/mysqld/mysqld.sock
    Uptime:			1 sec
    
    Threads: 1  Questions: 8  Slow queries: 0  Opens: 105  Flush tables: 1  Open tables: 98  Queries per second avg: 8.000


## DB作成


```shell
!echo "create database db" | mysql
```

## Pythonクライアントのインストール


```shell
!pip install mysqlclient
```

    Collecting mysqlclient
      Downloading mysqlclient-2.1.0.tar.gz (87 kB)
    [K     |████████████████████████████████| 87 kB 3.1 MB/s 
    [?25hBuilding wheels for collected packages: mysqlclient
      Building wheel for mysqlclient (setup.py) ... [?25l[?25hdone
      Created wheel for mysqlclient: filename=mysqlclient-2.1.0-cp37-cp37m-linux_x86_64.whl size=99970 sha256=b02277159390cd4b15d6de2a531b35a56ec5150e53609641eddc9ffd4b8d7ce1
      Stored in directory: /root/.cache/pip/wheels/97/d4/df/08cd6e1fa4a8691b268ab254bd0fa589827ab5b65638c010b4
    Successfully built mysqlclient
    Installing collected packages: mysqlclient
    Successfully installed mysqlclient-2.1.0


 ## データのインポート

テーブルを作成して、データを50万件登録します。10分ほど時間がかかります。


```python
import MySQLdb
import json
import bz2
from tqdm.notebook import tqdm

db = MySQLdb.connect(host='localhost', user='root', db='db', charset='utf8mb4')
cursor = db.cursor()

cursor.execute('drop table if exists wiki_jp')
cursor.execute('create table wiki_jp('
 'id bigint unsigned not null auto_increment primary key,'
 'title tinytext collate utf8mb4_unicode_ci storage memory,'
 'body mediumtext collate utf8mb4_unicode_ci storage memory)')

limit = 500000
insert_wiki = 'insert into wiki_jp (title, body) values (%s, %s);'

with bz2.open('/content/drive/MyDrive/wiki.json.bz2', 'rt', encoding='utf-8') as fin:
    n = 0
    for line in tqdm(fin, total=limit*1.5):
        data = json.loads(line)
        title = data['title'].strip()
        body = data['text'].replace('\n', '')
        if len(title) > 0 and len(body) > 0:
            cursor.execute(insert_wiki, (title, body))
            n += 1
        if n == limit:
            break
db.commit()
db.close()
```


      0%|          | 0/750000.0 [00:00<?, ?it/s]


テーブル定義を確認します。


```shell
!echo "show columns from db.wiki_jp" | mysql
```

    Field	Type	Null	Key	Default	Extra
    id	bigint(20) unsigned	NO	PRI	NULL	auto_increment
    title	tinytext	YES		NULL	
    body	mediumtext	YES		NULL	


登録件数を確認します。


```shell
!echo "select count(*) from db.wiki_jp" | mysql
```

    count(*)
    500000


## インデックスを使わない検索

like検索でシーケンシャルに検索した場合を測定します。mysqlコマンドに-vvvオプションを付けると、出力の最後から3行目にマッチ数と処理時間が出力されるので、その部分だけをtailコマンドとheadコマンドで切り出しています。


まずはインデックスを使っていないことを確認します。

```shell
!echo "explain select sql_no_cache * from db.wiki_jp where body like '%日本語%'" | mysql
```

    id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
    1	SIMPLE	wiki_jp	NULL	ALL	NULL	NULL	NULL	NULL	271545	11.11	Using where

bodyに「日本語」を含むレコードの数を取得します。


```python
%%time
!echo "select sql_no_cache count(*) from db.wiki_jp where body like '%日本語%'" | mysql -vvv
```

    --------------
    select sql_no_cache count(*) from db.wiki_jp where body like '%日本語%'
    --------------
    
    +----------+
    | count(*) |
    +----------+
    |    17006 |
    +----------+
    1 row in set, 1 warning (11.47 sec)
    
    Bye
    CPU times: user 107 ms, sys: 19.5 ms, total: 127 ms
    Wall time: 11.6 s


bodyに「日本語」を含むレコードを取得します。

```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where body like '%日本語%'" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (10.51 sec)
    CPU times: user 126 ms, sys: 19.5 ms, total: 146 ms
    Wall time: 13.6 s

2回目

```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where body like '%日本語%'" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (10.25 sec)
    CPU times: user 121 ms, sys: 16.8 ms, total: 138 ms
    Wall time: 13.8 s

インデックスを使わない場合、単に数を集計するだけでもbodyにアクセスしなければならないので、処理時間はあまり変わりません。
## 全文検索用インデックスの作成

インデックスの作成には60分ほどかかります。


```shell
!echo "alter table db.wiki_jp add fulltext index ngram_idx (body) with parser ngram" | mysql -vvv
```

    --------------
    alter table db.wiki_jp add fulltext index ngram_idx (body) with parser ngram
    --------------
    
    Query OK, 0 rows affected, 1 warning (1 hour 33.16 sec)
    Records: 0  Duplicates: 0  Warnings: 1
    
    Bye


## インデックスを使った検索

まずはインデックスを使っていることを確認します。

```shell
!echo "explain select sql_no_cache * from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql
```

    id	select_type	table	partitions	type	possible_keys	key	key_len	ref	rows	filtered	Extra
    1	SIMPLE	wiki_jp	NULL	fulltext	ngram_idx	ngram_idx	0	const	1	100.00	Using where; Ft_hints: no_ranking


bodyに「日本語」を含むレコードの数を取得します。


```python
%%time
!echo "select sql_no_cache count(*) from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv
```

    --------------
    select sql_no_cache count(*) from db.wiki_jp where match (body) against ('日本語' in boolean mode)
    --------------
    
    +----------+
    | count(*) |
    +----------+
    |    17006 |
    +----------+
    1 row in set, 1 warning (2.45 sec)
    
    Bye
    CPU times: user 22.2 ms, sys: 12 ms, total: 34.2 ms
    Wall time: 2.53 s


bodyに「日本語」を含むレコードを取得します。

```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (4.31 sec)
    CPU times: user 73.3 ms, sys: 14.2 ms, total: 87.5 ms
    Wall time: 8.05 s


2回目

```shell
%%time
!echo "select sql_no_cache * from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (3.97 sec)
    CPU times: user 60.8 ms, sys: 14.2 ms, total: 75 ms
    Wall time: 7.65 s


数の集計はまあまあ速いですが、他と比べると断然遅い。数の集計の場合でもインデックスだけで処理が済まず、bodyにアクセスしているのかもしれません。インデックスを使っても、使わない場合の半分にもならないのはいかがなものかと。



参考までにidとtitleのみを取得してみます。


```python
%%time
!echo "select sql_no_cache id title from db.wiki_jp where match (body) against ('日本語' in boolean mode)" | mysql -vvv | tail -3 | head -1
```

    17006 rows in set, 1 warning (3.03 sec)
    CPU times: user 33.8 ms, sys: 6.21 ms, total: 40 ms
    Wall time: 3.13 s


大して速くなりません。これはMySQLの欠点と言っても過言ではありませんね。

## DBの停止


```shell
!service mysql stop
```

     * Stopping MySQL database server mysqld
       ...done.
