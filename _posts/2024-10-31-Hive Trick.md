---
title: "Hive Trick"
categories:
  - blog
tags:
  - Debug
---

# AWS Hive

记录一次比较有意思的 AWS Hive 问题排查记录。

## 问题复现

给公司用metabase搭了个bi平台，后面配了两个数据库引擎，一个 ClickHouse， 一个 Hive，嗯然后嘞执行 Query 偶尔返回“您的问题花费了太长时间，我们无法得到答案”这样的报错。但是这个Query在Azure+Redash这样的bi就执行正常，于是开始调查。

这个sql大概长这样:

```sql
with t_user_msg as (
  select
    t_events.device_id as device_id,
    count(1) as message_count,
    t_events.received_date as dt
  from
    v2_ods_app_user_action_v2_events t_events
  where
    t_events.received_date >= timestamp '2024-10-01 00:00:00.000'
    and (
      timestamp '2024-10-20 00:00:00.000' = '-'
      or t_events.received_date <= timestamp '2024-10-20 00:00:00.000'
    )
    and t_events.event_name in ('newRP_click_sendmsg')
    and t_events.platform in ('android', 'ios')
    and (
      decode(unhex('e698af'), 'utf-8') = '-'
      or map(
        '是',
        get_json_object(t_events.parameters, '$.current_page') in ('/main#Home', '/main', '/main#Museland'),
        '否',
        get_json_object(t_events.parameters, '$.current_page') not in ('/main#Home', '/main', '/main#Museland')
      ) [ decode(unhex('e698af'), 'utf-8') ]
    )
    and t_events.user_id > 0
    and get_json_object(t_events.parameters, '$.item_id') > 0
  group by
    get_json_object(t_events.parameters, '$.item_id'),
    t_events.device_id,
    t_events.received_date
)
SELECT
  SUM(
    if(
      int(t_user_msg.device_id / 1000 % 100) in (0, 2, 3, 6, 7),
      t_user_msg.message_count,
      0
    )
  ) AS `实验组1(02367控制组)聊天数`,
  SUM(
    if(
      int(t_user_msg.device_id / 1000 % 100) in (1, 4, 5, 8, 9),
      t_user_msg.message_count,
      0
    )
  ) AS `实验组2(14589变量组)聊天数`,
  t_user_msg.dt as dt
FROM
  t_user_msg
GROUP BY
  t_user_msg.dt%
```

## 问题调查

### Clickhouse

一开始呢是ClickHouse发现了这样的问题，我们在jdbc字符串上面加上 `connection_timeout=3600000&socket_timeout=30000000` 就解决了问题。

### Hive

现在的Hive呢托管在AWS上，Hive这边的问题，查了一个星期，大概流程如下：

1. 看浏览器打开页面的速度，排查网络问题，发现它总是1min后超时
2. 看接到AWS的负载均衡器，有一个1min的超时时间，改成30min后，原本的浏览器超时从1min提到了5.1min。
3. 然后google排查metabase侧，发现metabase有一个写死的数据库超时时间长10min，目录在 `src/metabase/driver/mysql/ddl.clj`，排查metabase机器上的日志发现并没有走到这个逻辑，超时时间也不是上面所写的十分钟，根据目录结构看，这个应该是metabase实例上存储查询结果的数据库，所以问题也可以排除在这里：

```clojure
(defmethod ddl.i/refresh! :mysql
  [driver database definition dataset-query]
  (let [{:keys [query params]} (qp.compile/compile dataset-query)
        db-spec (sql-jdbc.conn/db->pooled-connection-spec database)]
    (sql-jdbc.execute/do-with-connection-with-options
     driver
     database
     {:write? true}
     (fn [conn]
       (sql.ddl/execute! conn [(sql.ddl/drop-table-sql database (:table-name definition))])
       ;; It is possible that this fails and rollback would not restore the table.
       ;; That is ok, the persisted-info will be marked inactive and the next refresh will try again.
       (execute-with-timeout! driver
                              conn
                              db-spec
                              (.toMillis (t/minutes 10))
                              (into [(sql.ddl/create-table-sql database definition query)] params))
       {:state :success}))))
```
4. 怀疑是jdbc配置项的问题，在aws我们开的case内，有一位技术支持猜测是因为超时问题，我根据提示修改了jdbc连接配置项后，通过观察network发现的确生效，改成600s后就是10分钟超时，改成900s后就是15分钟超时。。。将引擎改为mr和spark也不起作用
> 透过内部工具我了解到您的SQL是通过tez引擎运行，并且日志中并没有报错信息。并且您可以透过其他方法成功运行SQL，因此我认为导致失败的原因确实是由于timeout 参数导致。我查询到tez.session.am.dag.submit.timeout.secs参数默认为300秒[1]，会在没有接受新dag5分钟后关闭container。因此我推测可能是由于此参数导致任务5分钟报错。

5. 所以问题应该另有原因，于是来了另一位support帮忙调查，查看了mgr节点上的日志，并没得出什么结论，然后我们打开后台，看tez监控，发现了问题，我们查看了多个稳定失败的applicatuion_id，发现失败的Tasks都集中在一个node上，并且这个node没有成功过，
```log
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.internal:8041
ip-10-200-18-87.ec2.
```

6. 我们进去了这个节点实例内，将hadoop的任务分发服务关闭，重新执行sql，发现任务分散到了剩下的11个实例中，并且在6m22s后得到结果。


## 复盘

还有一些未知的问题亟待解决：

1. 当一个application里的一堆task因为某些原因失败后，为什么程序会pending？
2. 当我打开了异常节点更换（？应该是叫这个，为什么它并没有给我更换剩余节点呢？

留给aws的支持去找答案。