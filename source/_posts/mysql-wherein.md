---
title: 记录一次WhereIn没有使用到索引的排查过程
date: 2022-02-13 22:11:41
tags: MySQL,索引
categories: MySQL
banner_img: /img/mysql_wherein.jpeg
index_img: /img/mysql_wherein.jpeg
---

### 背景

- 字段summary_id是表已经有一千多万数据之后新增的，历史数据默认值为0 ，加好summary_id字段后新增了索引：idx_summary_id

- sql查询没有用索引，猜测是和索引的基数有关，MySQL in查询能不能用到索引主要是看使用索引的成本和全表扫描的成本那个更低。下面是证实排查过程

- 没有用到索引的sql语句为 

```sql
select * from `btb_bid_user_apply` where summary_id in (108,156,166,167,168,170,172,174,176,178,179,193,196);
```

- 表结构，省略了无关字段

```sql
CREATE TABLE `btb_bid_user_apply` (
  `id` int(11) unsigned NOT NULL AUTO_INCREMENT,
  `summary_id` int(11) unsigned NOT NULL DEFAULT '0' COMMENT '出价汇总表id'
  KEY `idx_summary_id` (`summary_id`)
) ENGINE=InnoDB AUTO_INCREMENT=15526682 DEFAULT CHARSET=utf8 ROW_FORMAT=COMPRESSED KEY_BLOCK_SIZE=8;
```


### 排查过程

- 第一步 查询 eq_range_index_dive_limit（SHOW VARIABLES LIKE '%dive%';）线上结果为10，
    - 这个系统变量的意思是，当in语句中的参数个数大于或等于系统变量eq_range_index_dive_limit 时 不会使用index dive 计算单个区间节点对应的索引记录条数，而是使用索引统计数据
   
    - 查出 idx_summary_id的Cardinality=1，Rows= 13945557，这里计算单个区间节点对应的索引记录数 = 13945557 / 1, 总索引记录数为 13945557 * 13 = 181292241

​				 					

- 第二步 使用optimizer_trace 分析查询成本

  ```sql
  #开启查询分析
  SET optimizer_trace="enabled=on";
  
  select * from `btb_bid_user_apply` where summary_id in (108,156,166,167,168,170,172,174,176,178,179,193,196);
  
  #查询刚执行的语句的优化过程
  SELECT * FROM information_schema.OPTIMIZER_TRACE; 
  ```

  - 查询优化过程主要信息

    ```json
    # 预估不同单表访问方法的访问成本
    {
        "rows_estimation": [{
            "table": "`btb_bid_user_apply`", # 全表扫描的行数以及成本
            "range_analysis": {
                "table_scan": {
                    "rows": 13945557,
                    "cost": 2870000
                },
                # 分析可能使用的索引
                "potential_range_indexes": [{
                    "index": "PRIMARY",
                    "usable": false,
                    "cause": "not_applicable"
                },{
                    "index": "idx_summary_id",
                    "usable": true,#可能被使用
                    "key_parts": ["summary_id", "id"]
                }],
                "setup_range_conditions": [],
                "group_index_range": {
                    "chosen": false,
                    "cause": "not_group_by_or_distinct"
                },
    						# 分析各种可能使用的索引的成本
                "analyzing_range_alternatives": {
                    "range_scan_alternatives": [{
                        "index": "idx_summary_id",
                        "ranges": ["108 <= summary_id <= 108", "156 <= summary_id <= 156", "166 <= summary_id <= 166", "167 <= summary_id <= 167", "168 <= summary_id <= 168", "170 <= summary_id <= 170", "172 <= summary_id <= 172", "174 <= summary_id <= 174", "176 <= summary_id <= 176", "178 <= summary_id <= 178", "179 <= summary_id <= 179", "193 <= summary_id <= 193", "196 <= summary_id <= 196"],
                        "index_dives_for_eq_ranges": false,   # 是否使用index dive
                        "rowid_ordered": false,  # 使用该索引获取的记录是否按照主键排序
                        "using_mrr": true,# 是否使用mrr
                        "index_only": false, #是否是索引覆盖访问
                        "rows": 181292241,  #使用该索引获取的记录条数和第一步计算的扫描行数一样
                        "cost": 199000000, # 使用该索引的成本
                        "chosen": false, # 是否选择该索引
                        "cause": "cost"
                    }],
                    "analyzing_roworder_intersect": {
                        "usable": false,
                        "cause": "too_few_roworder_scans"
                    }
                }
            }
        }]
    }
    
    #最终使用的查询计划，可以看到使用全表扫描
    {
        "considered_execution_plans": [{
            "plan_prefix": [],
            "table": "`btb_bid_user_apply`",
            "best_access_path": {
                "considered_access_paths": [{
                    "rows_to_scan": 13945557,
                    "access_type": "scan",
                    "resulting_rows": 13900000,
                    "cost": 2870000, 
                    "chosen": true
                }]
            },
            "condition_filtering_pct": 100,
            "rows_for_plan": 13900000,
            "cost_for_plan": 2870000,
            "chosen": true
        }]
    }
    ```

   -	第三步 解释和验证查询成本
     -	MySQL 将 IO 成本设为1，cpu 判断成本设为0.2。IO成本是扫描一页的成本，对于回表操作的IO成本是每回表一次相当于一次磁盘IO
     -	扫描全表的计算方式 （ 聚簇索引叶子节点数 * 1  + 1） + （记录数*0.2+1）, 聚簇索引叶子节点数  通过show table status like "%btb_bid_user_apply%"  中的 Data_length / 16/1024 计算得出，本次查询的  聚簇索引叶子节点数大概为 646971392/16/1024=39488，成本为：39488 + 13945557*0.2 =2828599，接近于查询成本分析的 2870000
     -	通过查询索引 idx_summary_id 成本： 181292241*1 + ( 181292241*0.2 ) =217550689，与查询成本分析 199000000 多了一千七百多万，暂时还不清楚具体原因
       -	181292241*1 表示 回表的IO成本
       -	 ( 181292241*0.2 ) 表示回表之后 cpu 计算的成本
       
       
### 结论
- 关于in 能不能用到索引 主要还是看 使用索引的成本的全表扫描成本那个小
    - 全表扫描成本：（ 聚簇索引叶子节点数 * 1 + 1） + （记录数*0.2+1） 
        - 聚簇索引叶子节点数：通过show table status like "%btb_bid_user_apply%"  中的 Data_length / 16/1024 计算得出
    
    - 查询索引的成本：当in语句中的参数个数小于系统变量eq_range_index_dive_limit 使用index dive 否则使用索引统计数据
       - index dive：是计算单个区间节点的对应索引记录数的一种方式，其主要原理是先获取索引对应的`B+`树的`区间最左记录`和`区间最右记录`，然后再计算这两条记录之间有多少记录（记录条数少的时候可以做到精确计算，多的时候只能估算）。这种通过直接访问索引对应的`B+`树来计算某个范围区间对应的索引记录条数的方式称之为`index dive`
        - 单个区间节点是 in查询会转换的范围查
                
       - 索引统计数据 是 用估算的方式来计算出单个区间节点对应的索引记录数量，涉及到两个值，Cardinality（索引基数，通过show index from table_name） 和 Rows（总记录数 通过 show table status like "%table_name%";） 总记录数/索引基数*in语句中的参数个数=估算的扫描条数；
       - 查询成本 = 估算的扫描条数*1 + （估算的扫描条数 *0.2）