---
title: 线上慢sql优化
tag: 
- sql
categories: database
---



最近即将双11，收到slow sql的邮件，进行一波优化

<!--more-->

## 优化器对于where与order by索引列的选择

### 实验测试

1.1 ``chat_session``的索引如下：

```mysql
  PRIMARY KEY (`id`),
  UNIQUE KEY `uk_sub_session` (`sub_session_id`),
  KEY `idx_sso_user` (`sso_id`,`user_id`),
  KEY `idx_third_user` (`third_code`,`user_id`),
  KEY `idx_session_id` (`session_id`)
```



1.2 explain分析

```mysql
explain SELECT id, sub_session_id, session_id, user_id, third_code, start_time, end_time, channel_code, queue_id, queue_start_time, kefu_join_time, sso_id, score, min_msg_id, max_msg_id, create_time, update_time FROM chat_session WHERE sso_id = '4E5CE40451534858AE1A1C7993E75EF5' AND sso_id NOT IN ( 'v001@robot' , '' ) AND start_time >= '2020-09-09 11:38:56.949' AND start_time < '2020-10-09 11:38:57.949' AND end_time != '1970-01-01 00:00:00' AND id < 9223372036854775807 order by id desc LIMIT 30
```

| id   | select_type | table        | partitions | type  | possible_keys        | key     | key_len | ref  | rows     | filtered | Extra       |
| ---- | ----------- | ------------ | ---------- | ----- | -------------------- | ------- | ------- | ---- | -------- | -------- | ----------- |
| 1    | SIMPLE      | chat_session |            | range | PRIMARY,idx_sso_user | PRIMARY | 8       |      | 19779213 | 0.01     | Using where |

``possible_keys``:执行器可能走到的索引，PRIMARY,idx_sso_user  

`key ` :实际使用的索引，PRIMARY

``rows``:mysql认为必须要逐行去检查和判断的记录的条数，这里是19779213

``filtered``:此查询条件所过滤的数据的百分比，这里最终从将近两千万里查出了4行数据，4/19779213



Q：这里为什么走了``idx_sso_user``索引，而走了主键？为了解答这个问题，尝试了几个测试对比



1.3 order by id desc 去除

| id   | select_type | table        | partitions | type | possible_keys        | key          | key_len | ref   | rows  | filtered | Extra                               |
| ---- | ----------- | ------------ | ---------- | ---- | -------------------- | ------------ | ------- | ----- | ----- | -------- | ----------------------------------- |
| 1    | SIMPLE      | chat_session |            | ref  | PRIMARY,idx_sso_user | idx_sso_user | 152     | const | 47788 | 5.00     | Using index  condition; Using where |

结果：order by 的索引列被去掉了，此时只能走到idx_sso_user索引

1.4 改成order by id asc

| id   | select_type | table        | partitions | type  | possible_keys        | key     | key_len | ref  | rows     | filtered | Extra       |
| ---- | ----------- | ------------ | ---------- | ----- | -------------------- | ------- | ------- | ---- | -------- | -------- | ----------- |
| 1    | SIMPLE      | chat_session |            | range | PRIMARY,idx_sso_user | PRIMARY | 8       |      | 19784763 | 0.01     | Using where |

猜想：mysql8.0前构建的都是正向索引树，这里desc会进行反向扫描。

> 在理论上，反向检索与正向检索的速度一样的快。但是在某些操作系统上面，并不支持反向的read-ahead预读，所以反向检索会略慢。

结果：正向排序跟反向排序一样，所以问题不在正反向扫描



1.5 数据量少的测试环境运行同样的sql

| id   | select_type | table        | partitions | type | possible_keys        | key          | key_len | ref   | rows | filtered | Extra                                              |
| ---- | ----------- | ------------ | ---------- | ---- | -------------------- | ------------ | ------- | ----- | ---- | -------- | -------------------------------------------------- |
| 1    | SIMPLE      | chat_session |            | ref  | PRIMARY,idx_sso_user | idx_sso_user | 152     | const | 291  | 5.00     | Using index condition; Using where; Using filesort |

结果：同样的sql在数据量少的情况下走到了idx_sso_user索引

 

1.6 线上数据清理

| id   | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra |
| ---- | ----------- | ----- | ---------- | ---- | ------------- | ---- | ------- | ---- | ---- | -------- | ----- |
| SIMPLE | chat_session | ref  | PRIMARY,idx_sso_user | idx_sso_user | 152  | const | 691  | 5.00 | Using index  condition; Using where; Using filesort |

结果：因为这是历史会话数据表，之前有6千万的数据一直未清理，近期进行了数据归档，数据只有一千万后，索引如期走到了我们所希望的。



### 结论

order by 使用的是主键索引，where里使用的是 KEY `idx_sso_user` (`sso_id`,`user_id`)，虽然说order by id是在where之后执行，但也会影响执行计划，where中的联合索引只使用了部分，sql优化器觉得优化效率没有主键索引高，没有走我们期望的'idx_sso_user'索引，最终扫描了这个时间段的数据根据ssoId进行过滤，随后根据主键排序。

有的时候，sql优化器并非如我们所愿去选择，可以使用use('idx_sso_user')强制使用idx_sso_user索引



## type = range与 type = ref

### 实验测试

1.1 ``offline_message_dispatched``的索引如下：

```mysql
  PRIMARY KEY (`id`),
  KEY `idx_sso_id` (`sso_id`),
  KEY `idx_user_third` (`user_id`,`third_code`)
```



1.2 explain分析

```sql
explain SELECT id, user_id, third_code, queue_id, sso_id, db_state, create_time, update_time FROM offline_message_dispatched WHERE sso_id = '4D9289CEA8B44398AD955DBDFA4E4810' AND id > 0 AND db_state in ( 0 , 1 ) ORDER BY id ASC LIMIT 20
```

| id   | select_type | table                      | partitions | type  | possible_keys      | key     | key_len | ref  | rows   | filtered | Extra       |
| ---- | ----------- | -------------------------- | ---------- | ----- | ------------------ | ------- | ------- | ---- | ------ | -------- | ----------- |
| 1    | SIMPLE      | offline_message_dispatched |            | range | PRIMARY,idx_sso_id | PRIMARY | 8       |      | 402840 | 2.00     | Using where |

这里查询类型是range，sql优化器选择了``PRIMARY``，没有走到了我们期望的``idx_sso_id``索引



1.3 id > 0去掉

| id   | select_type | table                      | partitions | type | possible_keys | key        | key_len | ref   | rows  | filtered | Extra                               |
| ---- | ----------- | -------------------------- | ---------- | ---- | ------------- | ---------- | ------- | ----- | ----- | -------- | ----------------------------------- |
| 1    | SIMPLE      | offline_message_dispatched |            | ref  | idx_sso_id    | idx_sso_id | 202     | const | 10782 | 20.00    | Using index  condition; Using where |

id > 0是可以直接去掉不影响sql的查询结果，看到type又是range，这里是没必要的

> ALL < index < range ~ index_merge < ref < eq_ref < const < system

结果：type改为ref，走到了``idx_sso_id``索引

## 参考

[MySQL · 答疑解惑 · InnoDB 预读 VS Oracle 多块读](http://mysql.taobao.org/monthly/2015/05/04/)

[order by详解]( https://segmentfault.com/a/1190000015987895?utm_source=weekly&utm_medium=email&utm_campaign=email_weekly)

[order by 主键id导致全表扫描的问题](https://cloud.tencent.com/developer/article/1181275)

