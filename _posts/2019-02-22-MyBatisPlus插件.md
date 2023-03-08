---
layout: post
categories: [Mybatis,Mybatis-Plus]
description: none
keywords: Mybatis-Plus
---
# MyBatis-Plus插件


## MybatisPlusInterceptor
该插件是核心插件,目前代理了 Executor#query 和 Executor#update 和 StatementHandler#prepare 方法
InnerInterceptor
我们提供的插件都将基于此接口来实现功能

目前已有的功能:
- 自动分页: PaginationInnerInterceptor
- 多租户: TenantLineInnerInterceptor
- 动态表名: DynamicTableNameInnerInterceptor
- 乐观锁: OptimisticLockerInnerInterceptor
- sql 性能规范: IllegalSQLInnerInterceptor
- 防止全表更新与删除: BlockAttackInnerInterceptor
注意:
使用多个功能需要注意顺序关系,建议使用如下顺序
- 多租户,动态表名
- 分页,乐观锁
- sql 性能规范,防止全表更新与删除
总结: 对 sql 进行单次改造的优先放入,不对 sql 进行改造的最后放入


## PaginationInnerInterceptor

## OptimisticLockerInnerInterceptor
当要更新一条记录的时候，希望这条记录没有被别人更新
乐观锁实现方式：
- 取出记录时，获取当前 version
- 更新时，带上这个 version
- 执行更新时， set version = newVersion where version = oldVersion
- 如果 version 不对，就更新失败

spring boot 注解方式:
```java
@Bean
public MybatisPlusInterceptor mybatisPlusInterceptor() {
    MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
    interceptor.addInnerInterceptor(new OptimisticLockerInnerInterceptor());
    return interceptor;
}
```
在实体类的字段上加上@Version注解
```java
@Version
private Integer version;
```

## mybatis-plus-join
mybatis-plus-join是mybatis plus的一个多表插件，上手简单，十分钟不到就能学会全部使用方式，只要会用mp就会用这个插件，仅仅依赖了lombok，而且是扩展mp的构造器并非更改原本的构造器，不会对原有项目产生一点点影响。
maven依赖
```java
 <dependency>
    <groupId>icu.mhb</groupId>
    <artifactId>mybatis-plus-join</artifactId>
    <version>1.2.0</version>
 </dependency>
```

```java
// 第一步new 一个JoinLambdaWrapper构造参数是主表的实体对象（如果在service中直接使用joinLambdaWrapper()方法即可获得）
JoinLambdaWrapper<Users> wrapper = new JoinLambdaWrapper<>(Users.class);

// 第二步 使用leftJoin方法创建一个左连接
/*
	有三个方法可以使用 
	leftJoin 左联
	rightJoin 右联
	innerJoin 内联
*/

// 这一部分一个参数是join中定义的连接的表，第二个参数是随意的表，但是是要出现构造器中的
wrapper.leftJoin(UsersAge.class,UsersAge::getId,Users::getAgeId);
// 然后可以设置多表中的查询条件，这一步和mp一致
wrapper.eq(UserAge::getAgeName,"95")
  		.select(UserAge::getAgeName)
      // 最后一步 需要使用end方法结束
      .end();


// 完整的就是
JoinLambdaWrapper<Users> wrapper = new JoinLambdaWrapper<>(Users.class);
wrapper.leftJoin(UsersAge.class,UsersAge::getId,Users::getAgeId)
  	.eq(UserAge::getAgeName,"95")
  	.select(UserAge::getAgeName)
  	.end();

usersService.joinList(wrapper,UsersVo.class);

// 执行SQL 
select 
  users.user_id,
  users.user_name,
  users_age.age_name
from users users
  left join users_age users_age on users_age.id = users.age_id
where (
	users_age.age_name = '95'
)
```








