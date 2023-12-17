---
title: "駆動表でWHERE句がなくてもExtraカラムに`Using where`が出る理由"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MySQL"]
published: false
---

駆動表に`WHERE`句を使っていなくても実行計画のExtraカラムに`Using where`が出る場合について、例をあげて説明します。

## 結論
駆動表に`WHERE`句を使っていなくてもオプティマイザーが判断してWHERE句を追加することがあり、オプティマイザとレースで確認することができます。

## 環境
- MySQL 8.0
- InnoDB

## サンプルレコード
```sql
CREATE TABLE `t1` (
  `c1` int NOT NULL,
  `c2` int DEFAULT NULL
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

CREATE TABLE `t2` (
  `c1` int NOT NULL,
  `c2` int DEFAULT NULL,
  KEY (`c1`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci


insert into t1 (c1,c2) values
(1,1),
(2,2),
(3,3),
(4,4),
(5,5);

insert into t2 (c1,c2) values
(1,1),
(2,2),
(3,3),
(4,4),
(5,5),
(6,6),
(7,7),
(8,8),
(9,9),
(10,10);
```

オプティマイザが駆動表を`t1`として利用する(`t2`を内部表とする)ために、`t2.c1`にインデックスを張っています。

## 実行するクエリ
内部表の結合キーは`c1`で固定した上で、駆動表の結合キーを`c1`（NOT NULL制約あり）、`c2`（NOT NULL制約なし）で内部結合を行なってみます。

```sql
# 駆動表の結合キーにNULL制約が存在する
select * from t1 inner join t2 on t1.c1 = t2.c1

# 駆動表の結合キーにNULL制約が存在しない
select * from t1 inner join t2 on t1.c2 = t2.c1
```

## 実行計画

### 駆動表の結合キーにNULL制約が存在する場合

```
mysql> EXPLAIN select * from t1 inner join t2 on t1.c1 = t2.c1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 100.00
        Extra: NULL
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: t2
   partitions: NULL
         type: ref
possible_keys: c1
          key: c1
      key_len: 4
          ref: world.t1.c1
         rows: 1
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.01 sec)
```

### 駆動表の結合キーにNULL制約が存在しない場合
```
mysql> EXPLAIN select * from t1 inner join t2 on t1.c2 = t2.c1\G;
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: t1
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 5
     filtered: 100.00
        Extra: Using where
*************************** 2. row ***************************
           id: 1
  select_type: SIMPLE
        table: t2
   partitions: NULL
         type: ref
possible_keys: c1
          key: c1
      key_len: 4
          ref: world.t1.c2
         rows: 1
     filtered: 100.00
        Extra: NULL
2 rows in set, 1 warning (0.02 sec)
```

実行計画を確認すると`t1`が駆動表になっています。
また、Extraカラムを確認すると`Using Where`が
