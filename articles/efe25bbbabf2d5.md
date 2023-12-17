---
title: "駆動表でWHERE句がなくてもExtraカラムにUsing whereが出る理由"
emoji: "🦤"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["MySQL"]
published: false
---

駆動表に`WHERE`句を使っていなくても実行計画のExtraカラムに`Using where`が出る場合について、例をあげて説明します。

## 結論
駆動表に`WHERE`句を使っていなくてもオプティマイザーが判断してWHERE句を追加することがあり、オプティマイザトレースで確認することができます。

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
内部表の結合キーは`c1`で固定した上で、駆動表の結合キーを`c1`（NOT NULL制約あり）、`c2`（NOT NULL制約なし）で内部結合を行います。

```sql
# 駆動表の結合キーにNULL制約が存在する
select * from t1 inner join t2 on t1.c1 = t2.c1

# 駆動表の結合キーにNULL制約が存在しない
select * from t1 inner join t2 on t1.c2 = t2.c1
```

## 実行計画
駆動表の結合キーにNULL制約が存在しない場合は、駆動表のExtraカラムに`Using where`が表示されています。

### 駆動表の結合キーにNULL制約が存在する場合

```
mysql> EXPLAIN select * from t1 inner join t2 on t1.c1 = t2.c1\g;
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
mysql> EXPLAIN select * from t1 inner join t2 on t1.c2 = t2.c1\g;
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

## オプティマイザトレースで確認する
駆動表に対してWHERE句を書いていないのに`Using where`が出ているのはなぜでしょうか。オプティマイザトレースを確認します。

```sql
SET optimizer_trace=`enabled=on`;

EXPLAIN select * from t1 inner join t2 on t1.c2 = t2.c1\g;

select * from information_schema.OPTIMIZER_TRACE\g;
```

```json
{
  "steps": [
    {
      "join_preparation": {
        "select#": 1,
        "steps": [
          {
            "expanded_query": "/* select#1 */ select `t1`.`c1` AS `c1`,`t1`.`c2` AS `c2`,`t2`.`c1` AS `c1`,`t2`.`c2` AS `c2` from (`t1` join `t2` on((`t1`.`c2` = `t2`.`c1`)))"
          },
          {
            "transformations_to_nested_joins": {
              "transformations": [
                "JOIN_condition_to_WHERE",
                "parenthesis_removal"
              ],
              "expanded_query": "/* select#1 */ select `t1`.`c1` AS `c1`,`t1`.`c2` AS `c2`,`t2`.`c1` AS `c1`,`t2`.`c2` AS `c2` from `t1` join `t2` where (`t1`.`c2` = `t2`.`c1`)"
            }
          }
        ]
      }
    },
    {
      "join_optimization": {
        "select#": 1,
        "steps": [
          {
            "condition_processing": {
              "condition": "WHERE",
              "original_condition": "(`t1`.`c2` = `t2`.`c1`)",
              "steps": [
                {
                  "transformation": "equality_propagation",
                  "resulting_condition": "multiple equal(`t1`.`c2`, `t2`.`c1`)"
                },
                {
                  "transformation": "constant_propagation",
                  "resulting_condition": "multiple equal(`t1`.`c2`, `t2`.`c1`)"
                },
                {
                  "transformation": "trivial_condition_removal",
                  "resulting_condition": "multiple equal(`t1`.`c2`, `t2`.`c1`)"
                }
              ]
            }
          },
          {
            "substitute_generated_columns": {
            }
          },
          {
            "table_dependencies": [
              {
                "table": "`t1`",
                "row_may_be_null": false,
                "map_bit": 0,
                "depends_on_map_bits": [
                ]
              },
              {
                "table": "`t2`",
                "row_may_be_null": false,
                "map_bit": 1,
                "depends_on_map_bits": [
                ]
              }
            ]
          },
          {
            "ref_optimizer_key_uses": [
              {
                "table": "`t2`",
                "field": "c1",
                "equals": "`t1`.`c2`",
                "null_rejecting": true
              }
            ]
          },
          {
            "rows_estimation": [
              {
                "table": "`t1`",
                "table_scan": {
                  "rows": 5,
                  "cost": 0.25
                }
              },
              {
                "table": "`t2`",
                "table_scan": {
                  "rows": 10,
                  "cost": 0.25
                }
              }
            ]
          },
          {
            "considered_execution_plans": [
              {
                "plan_prefix": [
                ],
                "table": "`t1`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "rows_to_scan": 5,
                      "filtering_effect": [
                      ],
                      "final_filtering_effect": 1,
                      "access_type": "scan",
                      "resulting_rows": 5,
                      "cost": 0.75,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 5,
                "cost_for_plan": 0.75,
                "rest_of_plan": [
                  {
                    "plan_prefix": [
                      "`t1`"
                    ],
                    "table": "`t2`",
                    "best_access_path": {
                      "considered_access_paths": [
                        {
                          "access_type": "ref",
                          "index": "c1",
                          "rows": 1,
                          "cost": 1.75,
                          "chosen": true
                        },
                        {
                          "rows_to_scan": 10,
                          "filtering_effect": [
                          ],
                          "final_filtering_effect": 1,
                          "access_type": "scan",
                          "using_join_cache": true,
                          "buffers_needed": 1,
                          "resulting_rows": 10,
                          "cost": 5.25004,
                          "chosen": false
                        }
                      ]
                    },
                    "condition_filtering_pct": 100,
                    "rows_for_plan": 5,
                    "cost_for_plan": 2.5,
                    "chosen": true
                  }
                ]
              },
              {
                "plan_prefix": [
                ],
                "table": "`t2`",
                "best_access_path": {
                  "considered_access_paths": [
                    {
                      "access_type": "ref",
                      "index": "c1",
                      "usable": false,
                      "chosen": false
                    },
                    {
                      "rows_to_scan": 10,
                      "filtering_effect": [
                      ],
                      "final_filtering_effect": 1,
                      "access_type": "scan",
                      "resulting_rows": 10,
                      "cost": 1.25,
                      "chosen": true
                    }
                  ]
                },
                "condition_filtering_pct": 100,
                "rows_for_plan": 10,
                "cost_for_plan": 1.25,
                "pruned_by_heuristic": true
              }
            ]
          },
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`t2`.`c1` = `t1`.`c2`)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`t1`",
                  "attached": "(`t1`.`c2` is not null)"
                },
                {
                  "table": "`t2`",
                  "attached": "(`t2`.`c1` = `t1`.`c2`)"
                }
              ]
            }
          },
          {
            "finalizing_table_conditions": [
              {
                "table": "`t1`",
                "original_table_condition": "(`t1`.`c2` is not null)",
                "final_table_condition   ": "(`t1`.`c2` is not null)"
              },
              {
                "table": "`t2`",
                "original_table_condition": "(`t2`.`c1` = `t1`.`c2`)",
                "final_table_condition   ": null
              }
            ]
          },
          {
            "refine_plan": [
              {
                "table": "`t1`"
              },
              {
                "table": "`t2`"
              }
            ]
          }
        ]
      }
    },
    {
      "join_explain": {
        "select#": 1,
        "steps": [
        ]
      }
    }
  ]
}
```

`attaching_conditions_to_tables`を見ると、オプティマイザが``t1`.`c2` is not null`という条件を追加していることがわかります。

```json
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`t2`.`c1` = `t1`.`c2`)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`t1`",
                  "attached": "(`t1`.`c2` is not null)"
                },
                {
                  "table": "`t2`",
                  "attached": "(`t2`.`c1` = `t1`.`c2`)"
                }
              ]
            }
          },
```

駆動表のキーである`c1`カラムはNOT NULL制約が付いておらず`NULL`が入る可能性があります。
しかし、結合条件である等価比較に`NULL`が含まれるとTRUEもしくはFALSEを返すことがないため[^1]、`c1`が内部結合の結合キーとなった時点で`c1`に`NULL`が含まれるレコードが結果に含まれることはありません。そのため外部表をフェッチして内部表と結合を行う前に、`c1`に`NULL`が含まれるレコードを取り除いておくとパフォーマンスが上がるとオプティマイザが判断したと考えられます。
これで実行計画のExtraカラムに`Using where`が追加されている理由がわかりました。

[^1]:https://dev.mysql.com/doc/refman/8.0/ja/working-with-null.html

## 補足
当然ですが、比較した`t1.c1`を駆動表の結合キーにした内部結合のクエリのオプティマイザトレースをみると、`attaching_conditions_to_tables`を見ると条件は追加されていないことがわかります。
これは`t1.c1`にはすでに`NOT NULL`制約がついているため追加する必要がないからだと考えられます。

```json
          {
            "attaching_conditions_to_tables": {
              "original_condition": "(`t2`.`c1` = `t1`.`c1`)",
              "attached_conditions_computation": [
              ],
              "attached_conditions_summary": [
                {
                  "table": "`t1`",
                  "attached": null
                },
                {
                  "table": "`t2`",
                  "attached": "(`t2`.`c1` = `t1`.`c1`)"
                }
              ]
            }
          },
```

## 参考文献
- [SQL実践入門](https://gihyo.jp/book/2015/978-4-7741-7301-6)
- [詳解MySQL 5.7 止まらぬ進化に乗り遅れないためのテクニカルガイド](https://www.shoeisha.co.jp/book/detail/9784798147406)
