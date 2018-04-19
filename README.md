# `PostgreSQL 10.2`によるパーティショニング（PARTITION BY）の動作検証

## 環境
|＃|環境|概要|
|:--|:--|:--|
|0|IPアドレス|192.168.33.50|
|1|Vagrant|2.0.3|
|2|CentOS|7(config.vm.box = "centos/7")|
|3|PostgreSQL|10.2|
|4|PostgreSQLのユーザ／パスワード|postgres/postgres|

## 事前準備

以下の記事を参考に、CentOS7の環境が構築済みの状態であること。
[Vagrant2.0.3を使ったCentOS7.4の環境構築](https://qiita.com/You_name_is_YU/items/1bdf36270e9f594d1c7e)

## やりたいこと

1つのDB内で、複数のユーザを管理するシステムは数多く存在すると思います。
やりたいこととして、他のユーザが登録したデータを相互に参照することはしないため、
他ユーザの登録件数に依存して検索速度が低下しないように、パーティショニングの機能を使って実現できるのか。ということを検証してみたいと思います。

## 環境準備

- CentOS7の標準yumリポジトリには、PostgreSQL9.2のバージョンが存在し、最新版の10.xは別途リポジトリを追加してインストールする必要がある。
まずは最新版のリポジトリを追加する。

```shell
yum -y localinstall https://download.postgresql.org/pub/repos/yum/10/redhat/rhel-7-x86_64/pgdg-centos10-10-2.noarch.rpm
```

- リポジトリの追加を確認

```shell
yum list postgresql*

#このコマンド実行で以下のpostgresqlパッケージが核にできればOK
postgresql10-server.x86_64                                                        10.2-1PGDG.rhel7
```

- PostgreSQLのDBサーバーをyumでインストールする。

```shell
yum -y install postgresql10-server
```

- インストールできたか確認

```shell
/usr/pgsql-10/bin/postgres --version
postgres (PostgreSQL) 10.2
```

- データベースを初期化<br>
initdbで実施していることは、postgresのOSユーザを追加し、データベースをまっさらにする。

```shell
/usr/pgsql-10/bin/postgresql-10-setup initdb
```

- PostgreSQLを自動起動設定する。
```shell
systemctl enable postgresql-10
```

- PostgreSQLを起動する。

```shell
systemctl start postgresql-10
```

- PostgreSQLへログインし、テーブルを用意する。

```shell
sudo -u postgres psql -U postgres
postgres=# create table users(user_id serial primary key, user_name text not null, create_dt date not null);
postgres=# create table emotion_dic(id serial primary key
 , user_id integer not null
 , entry_word text not null
 , reading text not null
 , part text not null
 , emotion_real_number double precision not null);

insert into users(user_name, create_dt) values ('A', now());
insert into users(user_name, create_dt) values ('B', now());
insert into users(user_name, create_dt) values ('C', now());
insert into users(user_name, create_dt) values ('D', now());
```

- 大量データをロード（emotion_dicテーブルのデータ）  
emotion_dicテーブルのデータは下記データを利用している。  
https://github.com/Yu-Yamaguchi/mariadb-partitioning/blob/master/emotion_dic_data.csv

```shell
postgres=# COPY emotion_dic FROM '/home/vagrant/emotion_dic_data.csv' WITH CSV;

※実際には、emotion_dic2というテーブルを別途作って、全てのカラムをtext型にし、インポートしたあとで、SELECT INSERTによってデータをemotion_dicへ移動した。
　なぜか以下のようなエラーが発生してしまい、正常にインポートできなかったため。

ERROR:  invalid input syntax for integer: "1"
CONTEXT:  COPY emotion_dic2, line 1, column user_id: "1"
```

## パーティショニング前の性能検証

```shell
cd /home/vagrant
vim test.sql

SELECT * FROM emotion_dic WHERE user_id = 3 AND entry_word LIKE '%負%';

su - postgres
cd /home/vagrant
/usr/pgsql-10/bin/pgbench -f test.sql -c 1 -t 1000 postgres

starting vacuum...ERROR:  relation "pgbench_branches" does not exist
(ignoring this error and continuing anyway)
ERROR:  relation "pgbench_tellers" does not exist
(ignoring this error and continuing anyway)
ERROR:  relation "pgbench_history" does not exist
(ignoring this error and continuing anyway)
end.
transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
number of transactions per client: 1000
number of transactions actually processed: 1000/1000
latency average = 84.343 ms
tps = 11.856389 (including connections establishing)
tps = 11.856866 (excluding connections establishing)
```

## パーティショニング後の性能検証

パーティショニング用の子テーブルを作成する。  
既に作成済みのテーブルに対して、alter tableによるパーティショニング設定ができない（かもしれない）ようなので、  
一旦テーブルをDropして作り直し、パーティショニング用の設定を行う。

```shell
postgres=# drop table emotion_dic;
postgres=# create table emotion_dic(id serial primary key
 , user_id integer not null
 , entry_word text not null
 , reading text not null
 , part text not null
 , emotion_real_number double precision not null)
 partition by list(user_id);

 ERROR:  primary key constraints are not supported on partitioned tables
 LINE 1: create table emotion_dic(id serial primary key
```

エラーが発生しているが、これはパーティショニングの親テーブルにPK（正確には、PKやインデックスなど）が存在してはいけない。  
というPostgreSQLの制限があるため発生している。  
解決策としては、パーティショニングの親テーブルにはPKなど一切指定せず、以下のように構築する。

```sql
 create table emotion_dic(id serial
 , user_id integer not null
 , entry_word text not null
 , reading text not null
 , part text not null
 , emotion_real_number double precision not null)
 partition by list(user_id);

 create table emotion_dic_0 partition of emotion_dic FOR VALUES IN (0);
 create table emotion_dic_1 partition of emotion_dic FOR VALUES IN (1);
 create table emotion_dic_2 partition of emotion_dic FOR VALUES IN (2);
 create table emotion_dic_3 partition of emotion_dic FOR VALUES IN (3);
 create table emotion_dic_4 partition of emotion_dic FOR VALUES IN (4);
 create table emotion_dic_5 partition of emotion_dic FOR VALUES IN (5);
 create table emotion_dic_6 partition of emotion_dic FOR VALUES IN (6);
 create table emotion_dic_7 partition of emotion_dic FOR VALUES IN (7);

 -- 子テーブルの作成が完了しているか確認する。
 \d+ emotion_dic

-- 別テーブルに保管していたデータをemotion_dicに移動（66万件）
-- なぜかWHERE句を指定してあげないと「ERROR:  invalid input syntax for integer: "1"」というエラーが発生する。。。
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '1';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '2';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '3';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '4';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '5';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '6';
insert into emotion_dic(user_id, entry_word, reading, part, emotion_real_number) select user_id::int, entry_word, reading, part, emotion_real_number from emotion_dic2 where user_id = '7';

-- 子テーブルは物理的にテーブルが存在している状態なので、件数は以下のように確認できる。
select count(*) from emotion_dic_1;
```

```shell
su - postgres
cd /home/vagrant
/usr/pgsql-10/bin/pgbench -f test.sql -c 1 -t 1000 postgres

transaction type: test.sql
scaling factor: 1
query mode: simple
number of clients: 1
number of threads: 1
number of transactions per client: 1000
number of transactions actually processed: 1000/1000
latency average = 12.090 ms
tps = 82.712281 (including connections establishing)
tps = 82.740146 (excluding connections establishing)
```

### 負荷試験の条件

|＃|クライアント数（-c）|繰り返し数（-t）|
|:--|:--|:--|
|1|1|1000|
|2|10|100|

#### １の実行結果

```
    -- パーティショニング前
    latency average = 83.128 ms
    tps = 12.029653 (including connections establishing)
    tps = 12.030354 (excluding connections establishing)

    -- パーティショニング後
    latency average = 12.090 ms
    tps = 82.712281 (including connections establishing)
    tps = 82.740146 (excluding connections establishing)
```

#### ２の実行結果

```
    -- パーティショニング前
    latency average = 793.207 ms
    tps = 12.607043 (including connections establishing)
    tps = 12.607669 (excluding connections establishing)

    -- パーティショニング後
    latency average = 119.669 ms
    tps = 83.563569 (including connections establishing)
    tps = 83.585851 (excluding connections establishing)
```

## まとめ

MariaDBと比べて、PostgreSQLが以下の点で優れていると考えられる。
* SELECTの速度がパーティショニング前／後に関わらず早い。
* パーティショニング用の子テーブル1つ1つに対して表領域を割り当てられる。
* サロゲートキーである「id」列を削除するなどの必要がなく機能を実現できている。

また、PostgreSQLの9.xバージョンでは、標準でパーティショニング機能は存在せず、  
チェック制約＋トリガー＋物理テーブル（子テーブル）を組み合わせることで、  
パーティショニングと同等の機能を実現する仕組みだったが、10.xバージョンでは、  
標準機能として搭載されたため、特に性能面で劣るようなものではなくなった。  
（機能名：宣言的パーティショニング）


## パーティショニング用の子テーブルに対してPKが付けられるか？

以下の通り、PK制約をALTER TABLEによって追加することが可能。
※親テーブルにはPKやINDEXを追加することができない。

```sql
ALTER TABLE emotion_dic_1 ADD PRIMARY KEY(id);

postgres=# \d+ emotion_dic_1
                                                          Table "public.emotion_dic_1"
       Column        |       Type       | Collation | Nullable |                 Default                 | Storage  | Stats target | Description
---------------------+------------------+-----------+----------+-----------------------------------------+----------+--------------+-------------
 id                  | integer          |           | not null | nextval('emotion_dic_id_seq'::regclass) | plain    |              |
 user_id             | integer          |           | not null |                                         | plain    |              |
 entry_word          | text             |           | not null |                                         | extended |              |
 reading             | text             |           | not null |                                         | extended |              |
 part                | text             |           | not null |                                         | extended |              |
 emotion_real_number | double precision |           | not null |                                         | plain    |              |
Partition of: emotion_dic FOR VALUES IN (1)
Partition constraint: ((user_id IS NOT NULL) AND (user_id = 1))
Indexes:
    "emotion_dic_1_pkey" PRIMARY KEY, btree (id)

    postgres=# \d+ emotion_dic_8
                                                              Table "public.emotion_dic_8"
           Column        |       Type       | Collation | Nullable |                 Default                 | Storage  | Stats target | Des
    cription
    ---------------------+------------------+-----------+----------+-----------------------------------------+----------+--------------+----
    ---------
     id                  | integer          |           | not null | nextval('emotion_dic_id_seq'::regclass) | plain    |              |
     user_id             | integer          |           | not null |                                         | plain    |              |
     entry_word          | text             |           | not null |                                         | extended |              |
     reading             | text             |           | not null |                                         | extended |              |
     part                | text             |           | not null |                                         | extended |              |
     emotion_real_number | double precision |           | not null |                                         | plain    |              |
    Partition of: emotion_dic FOR VALUES IN (8)
    Partition constraint: ((user_id IS NOT NULL) AND (user_id = 8))
```

## 子テーブル別々にserialでオートナンバーされるか。

```sql
postgres=# select count(*) cnt from emotion_dic_7;
 cnt
-----
   0
(1 row)

postgres=#
postgres=#
postgres=# insert into emotion_dic_7(user_id, entry_word, reading, part, emotion_real_number)
postgres-# values(7, 'test', 'test', 'test', 10);
INSERT 0 1
postgres=# select * from emotion_dic_7;
 id | user_id | entry_word | reading | part | emotion_real_number
----+---------+------------+---------+------+---------------------
  1 |       7 | test       | test    | test |                  10
(1 row)

postgres=# insert into emotion_dic_7(user_id, entry_word, reading, part, emotion_real_number)
postgres-# values(7, 'test2', 'test2', 'test2', 100);
INSERT 0 1
postgres=# select * from emotion_dic_7;
 id | user_id | entry_word | reading | part  | emotion_real_number
----+---------+------------+---------+-------+---------------------
  1 |       7 | test       | test    | test  |                  10
  2 |       7 | test2      | test2   | test2 |                 100
(2 rows)

postgres=# create table emotion_dic_8 partition of emotion_dic FOR VALUES IN (8);
CREATE TABLE
postgres=#
postgres=#
postgres=# insert into emotion_dic_8(user_id, entry_word, reading, part, emotion_real_number)
postgres-# values(8, 'test', 'test', 'test', 10);
INSERT 0 1
postgres=# insert into emotion_dic_8(user_id, entry_word, reading, part, emotion_real_number)
postgres-# values(8, 'test2', 'test2', 'test2', 10);
INSERT 0 1
postgres=#
postgres=# select * from emotion_dic_8;
 id | user_id | entry_word | reading | part  | emotion_real_number
----+---------+------------+---------+-------+---------------------
  3 |       8 | test       | test    | test  |                  10
  4 |       8 | test2      | test2   | test2 |                  10
(2 rows)
```
