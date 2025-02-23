## Day 19



### SQL

20：13  - 20: 44

1. 对 NULL值的判断

不能  colume == null  或者 colume != null   而是 colume  IS NULL  或者  colume  IS NOT  NULL

```sql
select 
    name
        from Customer
              where referee_id != 2 or referee_id IS null;
```

2. sql 执行的顺序

以去重为例，复习一下Sql的执行顺序。

```sql
select 
    Distinct author_id as id
        from Views
            where author_id = Viewer_id
              order by author_id;
```

| 步骤 | 子句               | 作用                                                |
| :--- | :----------------- | :-------------------------------------------------- |
| 1    | **FROM** **JOIN**  | 确定数据来源（表、子查询、连接等）。                |
| 2    | **WHERE**          | 过滤行（基于条件，在分组前过滤）。                  |
| 3    | **GROUP BY**       | 将数据按指定列分组。                                |
| 4    | **HAVING**         | 过滤分组后的结果（基于聚合值，如 `SUM`, `COUNT`）。 |
| 5    | **SELECT**         | 选择最终输出的列，并计算表达式或聚合函数。          |
| 6    | **DISTINCT**       | 去重（移除重复行）。                                |
| 7    | **ORDER BY**       | 对结果排序。                                        |
| 8    | **LIMIT / OFFSET** | 限制返回的行数或跳过指定行（如分页）。              |

### BOSS 自动投递简历

20: 44 -21:42

[get_jobs: 💼【找工作最强助手】全平台自动投简历脚本：(boss、前程无忧、猎聘、拉勾、智联招聘)](https://gitee.com/lok666/get_jobs)

之后在搞一下AI打招呼。

### VUE

了解了一下Axios。



### 总结

做一天火车，很晚才学习，找到个自动投简历的工具，sql有点弱，每天写几道sql。看会八股睡觉。



> 明日任务

Sql，项目，Vue, 优化简历，八股，面试。



