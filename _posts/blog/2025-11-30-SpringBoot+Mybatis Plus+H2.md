---

layout: post
title: SpringBoot+Mybatis Plus+H2
categories: [SpringBoot,Mybatis Plus,H2]
tags: [博客, SpringBoot,Mybatis Plus,H2]

---

# 一、SpringBoot项目初始化
## 1、Spring initailizr初始化项目
Spring Boot 是一个基于 Spring 的框架，旨在简化 Spring 应用的配置和开发过程，通过自动配置和约定大于配置的原则，使开发者能够快速搭建独立、生产级别的应用程序。

Spring官方提供了 [Spring initializr](https://start.springboot.io/)工具初始化SpringBoot项目，初始化后下载下来就可以了。

## 2、依赖引入，启动项目
```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
```


# 二、H2内存数据库
H2 数据库是一个轻量级的嵌入式关系型数据库，特别适合 Spring Boot 项目的开发、测试和演示环境。以下是 H2 数据库的详细介绍和使用指南。
## 1、H2 数据库特点

**嵌入式模式**：数据库引擎与应用运行在同一个 JVM 中

**内存模式**：数据存储在内存中，应用重启后数据清空

**文件模式**：数据持久化到磁盘文件

**纯 Java 编写**：跨平台，无需额外安装

**兼容性**：支持大部分 SQL 标准和 ODBC、JDBC API

## 2、引入maven依赖
```
<dependency>
    <groupId>com.h2database</groupId>
    <artifactId>h2</artifactId>
</dependency>
```

## 3、应用配置
```
spring:
  application:
    name: springboot
  datasource:
    driver-class-name: org.h2.Driver
    username: root
    password: root
    url: jdbc:h2:mem:test
  h2:
    console:
      enabled: true
      path: /h2-console
  sql:
    init:
      schema-locations: classpath:h2/schema.sql
      data-locations: classpath:h2/init.sql
      
logging:
  level:
    com.boyz.space.springboot.dao.mapper: debug
    org.apache.ibatis: debug
```

## 4、初始化SQL
```
schema.sql
create table t_user (
    id int not null auto_increment,
    username varchar(64) not null default '',
    password varchar(128) not null default '',
    valid int not null default 0,
    create_time timestamp not null default CURRENT_TIMESTAMP,
    update_time timestamp not null default CURRENT_TIMESTAMP,
    version int not null default 0,
    primary key (id)
);

init.sql
insert into t_user (username, password) values('zhangsan', '123456');
insert into t_user (username, password) values('lisi', '123456');
```

# 三、Mybatis Plus
MyBatis-Plus（简称 MP）是一个强大的 MyBatis 增强工具，在 MyBatis 的基础上只做增强不做改变，简化开发、提高效率。以下是 MyBatis-Plus 的完整指南。

## 1、核心特性

**无侵入**：只做增强不做改变，引入它不会对现有工程产生影响

**损耗小**：启动即会自动注入基本 CURD，性能基本无损耗

**强大的 CRUD 操作**：内置通用 Mapper、通用 Service，仅通过少量配置即可实现单表大部分 CRUD 操作

**支持 Lambda 形式调用**：通过 Lambda 表达式，方便的编写各类查询条件

**支持主键自动生成**：支持多达 4 种主键策略

**内置分页插件**：基于 MyBatis 物理分页，开发者无需关心具体操作

**内置性能分析插件**：可输出 SQL 语句以及其执行时间

**内置全局拦截插件**：提供全表 delete、update 操作智能分析阻断

**内置 SQL 注入剥离器**：支持数据库关键字自动转义

## 2、引入maven配置
```
<dependency>
    <groupId>com.baomidou</groupId>
    <artifactId>mybatis-plus-boot-starter</artifactId>
    <version>3.5.3</version>
</dependency>

```

## 3、应用配置
```
mybatis-plus:
  mapper-locations: classpath*:mapper/*.xml
  configuration:
    map-underscore-to-camel-case: true
```

# 四、项目整合应用

## 1、创建Entity类
```
package com.boyz.space.springboot.dao.entity;

import com.baomidou.mybatisplus.annotation.TableName;
import lombok.Data;

import java.util.Date;

@TableName("t_user")
@Data
public class UserEntity {

    private Integer id;

    private String username;

    private String password;

    private Integer valid;

    private Date createTime;

    private Date updateTime;

    private Integer version;
}
```

## 2、创建Mapper文件
```
package com.boyz.space.springboot.dao.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.boyz.space.springboot.dao.entity.UserEntity;
import org.apache.ibatis.annotations.Mapper;
import org.apache.ibatis.annotations.Param;
import org.apache.ibatis.annotations.Select;

import java.util.List;

@Mapper
public interface UserMapper extends BaseMapper<UserEntity> {

    @Select("select * from t_user where username = #{username}")
    List<UserEntity> selectByUserName(@Param("username") String username);

    UserEntity selectCustomUsers(@Param("username") String username);
}
```

```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.boyz.space.springboot.dao.mapper.UserMapper">

    <select id="selectCustomUsers" parameterType="string" resultType="com.boyz.space.springboot.dao.entity.UserEntity">
        SELECT * FROM t_user
        WHERE username = #{username}
        AND valid = 1
        limit 1
    </select>

</mapper>

```

Mapper方法有两种类型，一种申明方法定义，通过@Select、@Update等注解直接编写SQL；第二种是在Mapper类中申明方法定义，通过在xml文件中定义SQL，通过namespace与Mapper类关联

## 3、业务应用
```
package com.boyz.space.springboot.springboot;

import com.baomidou.mybatisplus.core.conditions.Wrapper;
import com.baomidou.mybatisplus.core.conditions.query.QueryWrapper;
import com.boyz.space.springboot.dao.entity.UserEntity;
import com.boyz.space.springboot.dao.mapper.UserMapper;
import org.mybatis.spring.annotation.MapperScan;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.util.StringUtils;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@SpringBootApplication
@RestController
@MapperScan("com.boyz.space.springboot.dao.mapper")
public class SpringbootApplication {

    @Value("${spring.application.name}")
    public String applicationName;

    @Autowired
    private UserMapper userMapper;

    public static void main(String[] args) {
        SpringApplication.run(SpringbootApplication.class, args);
    }

    @GetMapping("/hello")
    public String hello(@RequestParam(name = "name", defaultValue = "world") String name) {

        QueryWrapper<UserEntity> wrapper = new QueryWrapper<>();
        if(StringUtils.hasLength(name)) {
            wrapper.eq("username", name);
        }

        UserEntity userEntities = userMapper.selectCustomUsers(name);
//        List<UserEntity> userEntities = userMapper.selectByUserName(name);
//
//        List<UserEntity> userEntities = userMapper.selectList(wrapper);
//
//        for (UserEntity userEntity : userEntities) {
//            System.out.println(userEntity.getUsername() + " " + userEntity.getPassword());
//        }

        return "hello " + (userEntities != null ? userEntities.getId() : name);
    }
}

```