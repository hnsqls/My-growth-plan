## Day 1

复习了项目的整体架构，前端太薄弱。

### 评论模块的设计

表的设计

```sql
create table comments
(
    commentId  bigint auto_increment comment 'id'
        primary key,
    userId     bigint                             not null comment '用户id',
    questionId bigint                             not null comment '题目id',
    content    text                               not null comment '评论内容',
    createTime datetime default CURRENT_TIMESTAMP not null comment '创建时间',
    updateTime datetime default CURRENT_TIMESTAMP not null on update CURRENT_TIMESTAMP comment '更新时间',
    isDelete   tinyint  default 0                 not null comment '是否删除'
)
    comment '评论' collate = utf8mb4_unicode_ci;
```

#### 功能梳理

1. 添加评论

2. 删除评论（用户自己的评论，或者管理员删除任意的评论）

3. 修改评论 (修改自己的评论)

4. 查询某一题目下自己的评论

5. 分页查询某一题目下关联的评论并按照时间降序



#### 1.添加评论

接口

```java
    /**
     * 新增评论
     * @param commentsAddRequest
     */
    @PutMapping("/add")
    public BaseResponse<Long> addComment(@RequestBody CommentsAddRequest commentsAddRequest){

        return commentsService.addComments(commentsAddRequest);
    }
```

请求体

```java
/**
 * 创建评论请求
 */
@Data
public class CommentsAddRequest implements Serializable {



    /**
     * 题目id
     */
    private Long questionId;

    /**
     * 评论内容
     */
    private String content;


    private static final long serialVersionUID = 1L;
}
```

tips: 用户id可以在httpServletRequest中获取。



实现接口以及实现类

```java
   /**
     * 用户添加评论
     * @param comments
     * @return
     */
    BaseResponse<Long> addComments(CommentsAddRequest comments);


/**
     * 发表评论
     *
     * @param commentsAddRequest
     * @return
     */
    public BaseResponse<Long> addComments(CommentsAddRequest commentsAddRequest) {


        ThrowUtils.throwIf(commentsAddRequest == null, ErrorCode.PARAMS_ERROR);
        // 在此处将实体类和 DTO 进行转换
        Comments comments = new Comments();
        BeanUtils.copyProperties(commentsAddRequest, comments);

        // 数据填充 userId ,

        // todo 填充默认值
        User loginUser = userService.getLoginUser(request);
        comments.setUserId(loginUser.getId());
        // 写入数据库
        boolean result = this.save(comments);
        ThrowUtils.throwIf(!result, ErrorCode.OPERATION_ERROR);
        // 返回新写入的数据 id
        long newCommentId = comments.getCommentId();
        return ResultUtils.success(newCommentId);
    }
```

#### 2.删除评论

用户只能删除自己的评论，若是管理员的话也可以删除任意评论

接口

```java
  /**
     * 删除评论
     * @param commentsDeleteRequest
     */
    @DeleteMapping("/delete")
    public BaseResponse<Long> deleteComment(@RequestBody CommentsDeleteRequest commentsDeleteRequest){
        return commentsService.deleteComments(commentsDeleteRequest);
    }
```

请求体

```java
/**
 * 删除评论请求
 */
@Data
public class CommentsDeleteRequest implements  Serializable{
        
        /**
         * 评论id
         */
        private Long commentId;
        

        private static final long serialVersionUID = 1L;
}
```

多余，删除操作的请求，核心参数都是其id,感觉之后再写删除就不要单独写了。



实现接口以及实现类

```java
    /**
     * 用户删除自己的评论
     * 管理员删除任何评论
     * @param commentsDeleteRequest
     * @return
     */
    BaseResponse<Long> deleteComments(CommentsDeleteRequest commentsDeleteRequest);



/**
     * 用户删除自己的评论
     * 管理员删除评论
     *
     * @param commentsDeleteRequest
     * @return
     */
    @Override
    public BaseResponse<Long> deleteComments(CommentsDeleteRequest commentsDeleteRequest) {
        // 参数校验
        ThrowUtils.throwIf(commentsDeleteRequest == null, ErrorCode.PARAMS_ERROR);
        long commentId = commentsDeleteRequest.getCommentId();
        ThrowUtils.throwIf(commentId <= 0, ErrorCode.PARAMS_ERROR);

        // 该comment 是否存在
        Comments oldComments = this.getById(commentId);
        ThrowUtils.throwIf(oldComments == null, ErrorCode.NOT_FOUND_ERROR);

        // 判断是否是该用户的评论
        User loginUser = userService.getLoginUser(request);
        Long userId = loginUser.getId();

        // 不是自己并且不是管理员身份，不能删除
        ThrowUtils.throwIf(!oldComments.getUserId().equals(userId) && !UserRoleEnum.ADMIN.getValue().equals(loginUser.getUserRole()), ErrorCode.NO_AUTH_ERROR);

        // 删除评论
        boolean result = this.removeById(commentId);
        ThrowUtils.throwIf(!result, ErrorCode.OPERATION_ERROR);
        return ResultUtils.success(commentId);
    }
```



#### 3.修改评论

只能修改自己的评论



接口

```java
   /**
     * 更新自己的评论
     * @param commentsUpdateRequest
     * @return
     */
    @PutMapping("/update")
    public BaseResponse<Long> updateComment(@RequestBody CommentsUpdateRequest commentsUpdateRequest){


        return commentsService.updateComments(commentsUpdateRequest);
    }
```

请求体

```java
/**
 * 更新评论请求
 */
@Data
public class CommentsUpdateRequest implements Serializable {

    /**
     * 评论id
     */
    private Long commentId;


    /**
     * 评论内容
     */
    private String content;



    private static final long serialVersionUID = 1L;
}

```



接口以及实现类

```java
 /**
     * 用户更新自己的评论
     * @param commentsUpdateRequest
     * @return
     */
    BaseResponse<Long> updateComments(CommentsUpdateRequest commentsUpdateRequest);




/**
     * 更新自己的评论
     * @param commentsUpdateRequest
     * @return
     */
    @Override
    public BaseResponse<Long> updateComments(CommentsUpdateRequest commentsUpdateRequest) {
        // 参数校验
        ThrowUtils.throwIf(commentsUpdateRequest == null, ErrorCode.PARAMS_ERROR);

        long commentId = commentsUpdateRequest.getCommentId();
        ThrowUtils.throwIf(commentId <= 0, ErrorCode.PARAMS_ERROR);

        // 该comment 是否存在
        Comments oldComments = this.getById(commentId);
        ThrowUtils.throwIf(oldComments == null, ErrorCode.NOT_FOUND_ERROR);

        // 判断是否是该用户的评论
        User loginUser = userService.getLoginUser(request);
        Long userId = loginUser.getId();
        ThrowUtils.throwIf(!oldComments.getUserId().equals(userId), ErrorCode.NO_AUTH_ERROR);

        // 更新评论 老的评论属性复制
        Comments comments = new Comments();
        BeanUtil.copyProperties(oldComments, comments);
        // 补充字段 --> 评论内容
        comments.setContent(commentsUpdateRequest.getContent());

        //更新评论
        boolean result = updateById(comments);
        ThrowUtils.throwIf(!result, ErrorCode.OPERATION_ERROR);

        return ResultUtils.success(commentId);

    }
```



#### 4.查询某一题目下自己的评论

接口

```java
/**
     * 用户查看自己的某一题目下自己的评论
     * @param questionId
     * @return
     */
    @GetMapping("/{questionId}")
    public BaseResponse<List<Comments>> getCommentsByIdAndQuestionId(@PathVariable long questionId){

        return commentsService.getCommentsById(questionId);
    }

```



请求体 : 直接id 查了。



接口实现，实现类

```java
  /**
     * 用户查看自己的某一题目下的评论
     * @param questionId
     * @return
     */
    BaseResponse<List<Comments>> getCommentsById(long questionId);



/**
     * 用户查看某一题目下自己的评论
     * @param questionId
     * @return
     */
    @Override
    public BaseResponse<List<Comments>> getCommentsById(long questionId) {
        // 参数校验
        ThrowUtils.throwIf(questionId <= 0, ErrorCode.PARAMS_ERROR);

        //获取登录用户
        User loginUser = userService.getLoginUser(request);

        //校验用户
        ThrowUtils.throwIf(loginUser == null, ErrorCode.NOT_LOGIN_ERROR);

        Long userId = loginUser.getId();
        // 查询评论列表
        LambdaQueryWrapper<Comments> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        lambdaQueryWrapper.eq(Comments::getQuestionId, questionId)
                .eq(Comments::getUserId, userId);
        List<Comments> list = this.list(lambdaQueryWrapper);
        return ResultUtils.success(list);
    }
```

#### 5.分页查询某一题目下关联的评论并按照时间降序.

接口

```java
 /**
     * 查询题目关联的评论列表（分页）
     * @param commentsQueryPageRequest
     * @return
     */
    @PutMapping("/list/page")
    public BaseResponse<Page<Comments>> listCommentByPage(@RequestBody CommentsQueryPageRequest commentsQueryPageRequest){

        return commentsService.queryCommentsByPage(commentsQueryPageRequest);
    }

```

请求体

```java
/**
 * 查询题库下的评论列表请求
 */

@Data
public class CommentsQueryPageRequest  implements Serializable {


    /**
     * 题库id
     */
    private Long questionId;


    /**
     * 当前页号
     */
    private int current = 1;

    /**
     * 页面大小
     */
    private int pageSize = 10;


    private static final long serialVersionUID = 1L;
}
```

实现类

```java
/**
     * 分页获取题库下相关联的评论并按照时间降序
     * @param commentsQueryPageRequest
     * @return
     */
    //分页获取
    BaseResponse<Page<Comments>> queryCommentsByPage(CommentsQueryPageRequest commentsQueryPageRequest);


 /**
     * 分页获取题库相关联的评论
     * @param commentsQueryPageRequest
     * @return
     */
    @Override
    public BaseResponse<Page<Comments>> queryCommentsByPage(CommentsQueryPageRequest commentsQueryPageRequest ) {

        // 参数校验
        ThrowUtils.throwIf(commentsQueryPageRequest == null, ErrorCode.PARAMS_ERROR);

        //获得参数
        Long questionId = commentsQueryPageRequest.getQuestionId();
        int current = commentsQueryPageRequest.getCurrent();
        int pageSize = commentsQueryPageRequest.getPageSize();



        //校验 题目id 是否存在
        ThrowUtils.throwIf(questionId <= 0, ErrorCode.PARAMS_ERROR);


        // 构造查询条件
        LambdaQueryWrapper<Comments> lambdaQueryWrapper = new LambdaQueryWrapper<>();
        lambdaQueryWrapper.eq(Comments::getQuestionId, questionId);

        // 构造分页
        Page<Comments> page = Page.of(current, pageSize);

        //分页排序
        page.addOrder(OrderItem.desc("updateTime"));


        // 分页查询
        page = this.page(page, lambdaQueryWrapper);
        // 获取分页结果
        List<Comments> records1 = page.getRecords();
        return ResultUtils.success(page);
    }

```

tips： 使用了Mybatis-Plus的分页插件

```java
**
 * MyBatis Plus 配置
 *
 */
@Configuration
@MapperScan("com.ls.mianshidog.mapper")
public class MyBatisPlusConfig {

    /**
     * 拦截器配置
     *
     * @return
     */
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 分页插件
        interceptor.addInnerInterceptor(new PaginationInnerInterceptor(DbType.MYSQL));
        return interceptor;
    }
}
```



### Sql

[1251. 平均售价 - 力扣（LeetCode）](https://leetcode.cn/problems/average-selling-price/description/?envType=study-plan-v2&envId=sql-free-50)

```sql
select P.product_id,ifnull(round(sum(price * units) / sum(units),2),0) as average_price
     from Prices P
     	left join UnitsSold U
        	on P.product_id = U.product_id
       			 where purchase_date between start_date and end_date or purchase_date IS Null
          			  group by P.product_id;
```



```sql
select p.product_id, ifnull(round((sum(price * units) / sum(units)), 2), 0) average_price 
	from prices p 
		left join unitssold u 
			on p.product_id = u.product_id and purchase_date between start_date and end_date 
group by product_id;
```

上述两者代码都可以通过。

本来确实没什么难的，用round(x,2)保留小数

ifnull(X1，X2) 的意思是：if(x1 == null) 值为x2,if(x1 != null)  值为 x1。

要考虑 purchase_date 为null 的情况，在过滤条件添加 or purchase_date IS Null,保证能查到所有的id。

过滤条件放在on 后和 where后的区别。

[SQL语句连接筛选条件放在on和where后的区别（一篇足矣）_sql on后面跟两个等于条件-CSDN博客](https://blog.csdn.net/qq_42434318/article/details/112819267)

总的来说，on就是连接，遵循连接的特性，如左连接，就保留左表的所有字段，即使为null。where 就是正常的过滤。





[1633. 各赛事的用户注册率 - 力扣（LeetCode）](https://leetcode.cn/problems/percentage-of-users-attended-a-contest/submissions/604075464/?envType=study-plan-v2&envId=sql-free-50)

```sql
select contest_id,round(count(*) * 100 / (
    select count(*) from Users),2) as percentage
    from Users U
     join Register R
            on U.user_id = R.user_id
                group by contest_id
                    order by percentage desc,contest_id asc
```

子查询 + 多字段排序。





### 每日一题

#### 1. InnoDB和MyISAM的区别

主要的区别： 

1. **数据的存储结构不一样**

+ **InnoDB**：
  + 数据和索引存储在同一个文件中（`.ibd` 文件）。支持聚簇索引（Clustered Index），索引和值放在一起（二级索引是聚簇索引吗）。
+ **MyISAM**：
  + 数据（`.MYD` 文件）和索引（`.MYI` 文件）分开存储。不支持聚簇索引。



tips：数据的存储结构不一样，导致查询效率也不一样，**首先**，innoDB数据和索引在一个文件中，文件更大查询的较慢，MyISAM只存储索引，所以相同索引下，MyISAM使用的存储空间更少（页更少）,查的页数就少，所以MyISAM查的比InnoDB快；**其次** 聚簇索引的存储方式，可能会发生回表（使用二级索引），而MyISAM不会发生回表，因为他的索引（叶子节点存储的是指向这个数据的物理地址，所以可以一下获得全部的信息，而不用回表）

**2. 锁的级别不一样**

* InnoDB

  * 支持行锁，锁的粒度较低，所以并发能力高，同时行锁的性能开销更大。

* MyISAM

  * 支持表锁，锁的粒度较高，所以锁发能力低。

  

tips：对一行数据修改，InnoDB会使用行锁，MyISAM会使用表锁，表锁的管理简单，只需要维护一个锁对象（整个表）行锁的管理复杂，需要为每一行维护锁信息。

**3. 事务支持**

* InnoDB支持事务
  * redo log (持久性)， MVCC 和 锁（隔离性）， undo log(原子性）， 以上三种特性加上外键构成一致性。
* MyISAM 不支持事务
  * 不支持事务，反而没有那么多性能消耗，所以时候读多写少的情况。

总结: 

​	以上特性总结，InnoDB 支持行锁，事务，（外键），索引和数据存放在一起（.idb文件）（可能会回表），能够承受更多的并发，并且发生错误有回顾机制，更加安全。

​	MyISAM,支持表锁，不支持事务，索引和数据单独存放（.myd和.myi）,查询效率更高，但是不够安全。

#### 2. Spring的定时任务同时执行时穿行还是并行



在 **Spring** 中，定时任务的执行方式（串行或并行）取决于 **任务调度器（TaskScheduler）** 的配置以及任务本身的实现方式。

------
 1. **默认行为：串行执行**

+ **单线程任务调度器**：

  + 如果使用 Spring 默认的任务调度器（如 `ThreadPoolTaskScheduler` 未配置线程池），定时任务会在 **单线程** 中执行。
  + 这意味着多个定时任务会 **串行执行**，即使它们的触发时间重叠，也会依次排队执行。

+ **示例**：

  ```java
  @Scheduled(fixedRate = 1000)
  public void task1() {
      System.out.println("Task 1 - " + Thread.currentThread().getName());
  }
  
  @Scheduled(fixedRate = 1000)
  public void task2() {
      System.out.println("Task 2 - " + Thread.currentThread().getName());
  }
  ```

  + 如果未配置线程池，`task1` 和 `task2` 会在同一个线程中串行执行。

------

 2. **配置并行执行**

+ **多线程任务调度器**：

  + 通过配置 `ThreadPoolTaskScheduler` 或 `TaskExecutor`，可以为定时任务提供线程池支持，从而实现 **并行执行**。
  + 每个定时任务会在独立的线程中运行，互不干扰。

+ **配置示例**：

  ```java
@Configuration
  @EnableScheduling
  public class SchedulerConfig implements SchedulingConfigurer {
  
      @Override
      public void configureTasks(ScheduledTaskRegistrar taskRegistrar) {
          ThreadPoolTaskScheduler taskScheduler = new ThreadPoolTaskScheduler();
          taskScheduler.setPoolSize(5); // 设置线程池大小
          taskScheduler.setThreadNamePrefix("ScheduledTask-");
          taskScheduler.initialize();
          taskRegistrar.setTaskScheduler(taskScheduler);
      }
  }
  ```
  
  + 配置后，`task1` 和 `task2` 会在不同的线程中并行执行。

------

 3. **异步任务支持**

+ **使用 `@Async`**：

  + 如果定时任务方法标记了 `@Async`，Spring 会使用异步任务执行器来运行任务，从而实现并行执行。
  + 需要确保在配置类中启用异步支持（`@EnableAsync`）。

+ **示例**：

  

  ```java
@Configuration
  @EnableAsync
  @EnableScheduling
  public class AsyncConfig {
  }
  
  @Component
  public class ScheduledTasks {
  
      @Async
      @Scheduled(fixedRate = 1000)
      public void task1() {
          System.out.println("Task 1 - " + Thread.currentThread().getName());
      }
  
      @Async
      @Scheduled(fixedRate = 1000)
      public void task2() {
          System.out.println("Task 2 - " + Thread.currentThread().getName());
      }
  }
  ```
  
  + `task1` 和 `task2` 会在不同的线程中并行执行。

------
 4. **任务阻塞问题**

+ **单线程调度器的风险**：
  + 如果定时任务是阻塞的（如执行时间较长），且使用单线程调度器，会导致后续任务延迟执行。
+ **解决方案**：
  + 使用多线程调度器或异步任务支持，避免任务阻塞。

------

 5. **总结**

| 执行方式     | 配置方式                       | 特点                             |
| :----------- | :----------------------------- | :------------------------------- |
| **串行执行** | 默认行为，无需额外配置         | 任务依次执行，适合简单、短时任务 |
| **并行执行** | 配置 `ThreadPoolTaskScheduler` | 任务并行执行，适合复杂、长时任务 |
| **异步执行** | 使用 `@Async` + `@EnableAsync` | 任务异步执行，适合高并发场景     |

------

6. **最佳实践**

+ 如果定时任务执行时间较短且不频繁，可以使用默认的串行执行。
+ 如果任务执行时间较长或需要高并发，建议配置多线程调度器或使用异步任务支持。
+ 避免在单线程调度器中执行阻塞任务，以免影响其他任务的执行。





### 总结

​	实习第一天，公司常驻一个开发（明天出差），两个实习（一个我）开发，近一个月没需求，让自己在自己的项目上加功能。后端增删改查可以，前端不会，还要快速学前端。

​	带薪学习好还是不好，没人教，和自己学一样，只是有个人push着学。想着在这个公司边实习边面试呢？在公司也没投简历（虽然也没人管）。回家后用软件自动投递了会发现上海又有新的活跃招聘了，还不少，奇迹的22：00说明天线上面试，答应了，还不知道明天怎么面试。或许就拿着手机去楼下聊聊。

​	最好找工作的两月，最好找工作的俩月，八股看的少了，坚持投递，我只是小小的实习生，大不了不干也不能耽误面试。