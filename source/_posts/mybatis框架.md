---
title: mybatis框架
tags: [java]
categories: java
date: 2020-11-12 14:00:00
---

# 介绍

MyBatis 是一款优秀的持久层框架，它支持自定义 SQL、存储过程以及高级映射。MyBatis 免除了几乎所有的 JDBC 代码以及设置参数和获取结果集的工作。MyBatis 可以通过简单的 XML 或注解来配置和映射原始类型、接口和 Java POJO（Plain Old Java Objects，普通老式 Java 对象）为数据库中的记录。

简单来说就是增强jdbc。

官网地址：https://mybatis.org/mybatis-3/zh/index.html

# 配置

## 创建mysql表

在test库创建student表

```sql
CREATE TABLE `student` (
  `id` int(11) NOT NULL,
  `name` varchar(255) DEFAULT NULL,
  `email` varchar(255) DEFAULT NULL,
  `age` varchar(11) DEFAULT NULL,
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```



## pom.xml

```xml
<?xml version="1.0" encoding="UTF-8"?>

<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
  xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <!--项目信息-->
  <groupId>org.example</groupId>
  <artifactId>ch01-hello-mybatis</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <!--指定jdk版本-->
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
  </properties>

  <dependencies>
    <!--测试工具-->
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>4.11</version>
      <scope>test</scope>
    </dependency>
    <!--mybatis启动器-->
    <dependency>
      <groupId>org.mybatis.spring.boot</groupId>
      <artifactId>mybatis-spring-boot-starter</artifactId>
      <version>2.1.3</version>
    </dependency>
    <!--mysql驱动坐标-->
    <dependency>
      <groupId>mysql</groupId>
      <artifactId>mysql-connector-java</artifactId>
      <version>5.1.38</version>
    </dependency>
  </dependencies>

  <build>
    <resources>
      <resource>
        <!--mapper配置文件加载的配置-->
        <directory>src/main/java</directory>
        <includes>
          <include>**/*.properties</include>
          <include>**/*.xml</include>
        </includes>
      </resource>
    </resources>
  </build>
</project>
```

## 主配置

在项目根路径下创建`resources`文件夹，并更改为资源文件夹，在该文件夹下创建`mybatis.xml`文件：

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE configuration
        PUBLIC "-//mybatis.org//DTD Config 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-config.dtd">
<configuration>
    <environments default="development">
        <environment id="development">
            <transactionManager type="JDBC"/>
            <dataSource type="POOLED">
                <property name="driver" value="com.mysql.jdbc.Driver"/>
                <property name="url" value="jdbc:mysql://192.168.10.70:3306/test"/>
                <property name="username" value="root"/>
                <property name="password" value="mamahao"/>
            </dataSource>
        </environment>
    </environments>

    <!-- sql 映射文件位置,，编译后的类路径如果多个文件，可以继续增加mapper-->
    <mappers>
        <mapper resource="org/example/dao/StudentDao.xml"/>
    </mappers>
</configuration>
```

# 使用案例

创建接口文件：org.example.dao.StudentDao

```java
package org.example.dao;

import org.apache.ibatis.annotations.Param;
import org.example.entity.Student;

import java.util.List;

public interface StudentDao {

    //查询所有
    List<Student> selectStudent();

    //插入数据
    int insertStudent(Student student);

    //根据条件查询
    Student selectStudentById(Integer id);

    //多条件查询
    List<Student> selectStudentParam(@Param("colname") String name);
}
```

创建sql的mapper文件：org/example/dao/StudentDao.xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!--xml约束，校验语法标签-->

<!--
namespace：dao层接口的全限定名称
<select>：表示查询
id：sql的唯一标识，mybatis根据该值执行对应的sql，尽量使用接口中方法名命名
resultType：表示结果返回值类型，使用类的全限定名称
-->
<mapper namespace="org.example.dao.StudentDao">
    <select id="selectStudent" resultType="org.example.entity.Student">
        select id,name,email,age from Student order by id
    </select>

    <insert id="insertStudent">
        insert into student values(#{id},#{name},#{email},#{age})
    </insert>

    <!--  parameterType：指定入参的数据类型，可以使用全限定名称或mybatis的别名  -->
    <select id="selectStudentById" parameterType="java.lang.Integer" resultType="org.example.entity.Student">
        select * from student where id = #{id}
    </select>

    <select id="selectStudentParam" resultType="org.example.entity.Student">
        select * from student order by ${colname}
    </select>
</mapper>
```

创建实体类：org.example.entity.Student

```java
package org.example.entity;

public class Student {
    private Integer id;
    private String name;
    private String email;
    private Integer age;

    public Student() {
    }

    public Student(Integer id, String name, String email, Integer age) {
        this.id = id;
        this.name = name;
        this.email = email;
        this.age = age;
    }

    public Integer getId() {
        return id;
    }

    public void setId(Integer id) {
        this.id = id;
    }

    public String getName() {
        return name;
    }

    public void setName(String name) {
        this.name = name;
    }

    public String getEmail() {
        return email;
    }

    public void setEmail(String email) {
        this.email = email;
    }

    public Integer getAge() {
        return age;
    }

    public void setAge(Integer age) {
        this.age = age;
    }

    @Override
    public String toString() {
        return "Student{" +
                "id=" + id +
                ", name='" + name + '\'' +
                ", email='" + email + '\'' +
                ", age=" + age +
                '}';
    }
}

```

创建mybatis工具类，去除重复代码

```java
package org.example.utils;

import org.apache.ibatis.io.Resources;
import org.apache.ibatis.session.SqlSession;
import org.apache.ibatis.session.SqlSessionFactory;
import org.apache.ibatis.session.SqlSessionFactoryBuilder;

import java.io.IOException;
import java.io.InputStream;

public class MyBatisUtils {

    //定义SqlSessionFactory对象类型
    private static SqlSessionFactory factory = null;

    //静态代码块，内部只执行一次
    static {
        String config = "mybatis.xml";
        try {
            InputStream in = Resources.getResourceAsStream(config);
            factory = new SqlSessionFactoryBuilder().build(in);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    //返回sqlSession对象，mybatis通过操作该对象执行sql
    public static SqlSession getSqlSession() {
        SqlSession sqlSession = null;
        if ( factory != null){
            sqlSession = factory.openSession();
        }
        return  sqlSession;
    }
}

```

创建启动程序文件

```java
package org.example;

import org.apache.ibatis.session.SqlSession;
import org.example.dao.StudentDao;
import org.example.entity.Student;
import org.example.utils.MyBatisUtils;

import java.util.List;

public class App 
{
    public static void main( String[] args ) {

//        //select
//        SqlSession sqlSession = new MyBatisUtils().getSqlSession();
//        //mybatis动态代理，自动创建接口实现类，实现对应的sql语句，原理（反射）
//        StudentDao mapper = sqlSession.getMapper(StudentDao.class);
//        List<Student> studentList1 = mapper.selectStudent();
//        for (Student student : studentList1) {
//            System.out.println(student);
//        }
//
//        //insert
//        SqlSession sqlSession1 = new MyBatisUtils().getSqlSession();
//        StudentDao mapper1 = sqlSession1.getMapper(StudentDao.class);
//        Student insertSql = new Student(5, "六百", "444@444.com", 90);
//        int nums = mapper1.insertStudent(insertSql);
//        sqlSession1.commit();
//        System.out.println("结果返回数：" + nums);

//        SqlSession sqlSession1 = new MyBatisUtils().getSqlSession();
//        StudentDao mapper1 = sqlSession1.getMapper(StudentDao.class);
//        Student student = mapper1.selectStudentById(1);
//        System.out.println(student);

        //多参数使用
        SqlSession sqlSession2 = new MyBatisUtils().getSqlSession();
        //mybatis动态代理，自动创建接口实现类，实现对应的sql语句，原理（反射）
        StudentDao mapper2 = sqlSession2.getMapper(StudentDao.class);
        List<Student> studentList = mapper2.selectStudentParam("age");
        for (Student student1 : studentList) {
            System.out.println(student1);
        }
        sqlSession2.close();

    }
}
```

