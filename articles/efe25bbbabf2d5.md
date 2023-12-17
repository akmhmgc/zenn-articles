---
title: "駆動表でWHERE句がなくてもExtraカラムに`Using where`が出る理由"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MySQL"]
published: false
---

駆動表に`WHERE`句を使っていなくても実行計画のExtraカラムに`Using where`が出る場合について解説します。

## 環境
- MySQL 8.0
- InnoDB

## サンプルレコード
```sql
CREATE TABLE `t1` (
  `c1` int NOT NULL,
  `c2` int DEFAULT NULL,
  PRIMARY KEY (`c1`),
  KEY `idx_on_c2` (`c2`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_0900_ai_ci

CREATE TABLE `t2` (
  `c1` int NOT NULL,
  `c2` int DEFAULT NULL,
  PRIMARY KEY (`c1`),
  KEY `idx_on_c2` (`c2`)
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

`c1`がPRIMARY,`c2`が`DEFAULT NULL`のカラムを持つ`t1`,`t2`テーブルを作成します。
駆動表を`t1`とするために`t1`のレコード数を少なくしています。


