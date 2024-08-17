MyBatis

核心组件

- SqlSessionFactory:用于创建SqlSession的工厂类。

- SqlSession:MyBatis的核心组件,用于向数据库执行SQL。(spring整合了mybatis，ioc容器自动注入该bean，我们直接用就好)

- 主配置文件:XML配置文件,可以对MyBatis的底层行为做出详细的配置。（springboot整合了mybatis配置，在application.properties配置就行）

  实际我们程序中要写的就是下面两个

- Mapper接口:就是DAO接口,在MyBatis中习惯性的称之为Mapper。

- Mapper映射器:用于编写SQL,并将SQL和实体类映射的组件,采用XML、注解均可实现。

使用

http://www.mybatis.org/mybatis-3
http://www.mybatis.org/spring

导入依赖

```
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.1.2</version>
		</dependency>
```

配置

```
# DataSourceProperties  数据库连接池配置
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver  # mysql 数据库驱动
spring.datasource.url=jdbc:mysql://localhost:3306/community?characterEncoding=utf-8&useSSL=false&serverTimezone=Hongkong
spring.datasource.username=zhong
spring.datasource.password=zhong
spring.datasource.type=com.zaxxer.hikari.HikariDataSource # 使用springboot内置连接池中性能最好的
spring.datasource.hikari.maximum-pool-size=15 # 连接池最大连接数
spring.datasource.hikari.minimum-idle=5
spring.datasource.hikari.idle-timeout=30000

# MybatisProperties
# classpath为编译后的类路径，就是classes文件夹所在的位置，与接口方法与sql语句做映射
mybatis.mapper-locations=classpath:mapper/*.xml       
mybatis.type-aliases-package=com.nowcoder.community.entity # 在xml配置sql时用到实体类时就不用写包名了
mybatis.configuration.useGeneratedKeys=true							# 启用自动增长主键
# java中变量名与数据库中的变量名格式不一样，配置这个后headUrl能匹配上head_url
mybatis.configuration.mapUnderscoreToCamelCase=true  
```

注解

```java
@Mapper // 数据库访问层接口上使用，同@Repository
```

xml

```xml
<?xml version="1.0" encoding="UTF-8" ?>
<!DOCTYPE mapper
        PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN"
        "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<!-- 接口的命名空间 -->
<mapper namespace="com.nowcoder.community.dao.UserMapper">

    <!-- 定义可复用字段，在下面可以用<include>使用   -->
    <sql id="insertFields">
        username, password, salt, email, type, status, activation_code, header_url, create_time
    </sql>

    <!-- id对应接口中的方法，#{id}为获取接口中的id参数  resultType为返回值类型，parameterType为接口中参数列表的类型，
    如果为基本数据类型可以省略（但是查询方法的话resultType一定要写）， -->
    <select id="selectById" resultType="User">
        select <include refid="selectFields"></include>
        from user
        where id = #{id}
    </select>

    <!--  keyProperty为主键的变量名，插入用户时不带id，加了该属性后mybatis在将用户插入到数据库之后将它的id自动回写到user实体类中  
	如何做到的？
mybatis的底层是JDBC，通过JDBC向数据库插入数据后，可以通过结果集获得刚刚生成的ID。而将获取的值，设置给对象，利用java的类反射技术来实现。-->
    <insert id="insertUser" parameterType="User" keyProperty="id">
        insert into user (<include refid="insertFields"></include>)
        values(#{username}, #{password}, #{salt}, #{email}, #{type}, #{status}, #{activationCode}, #{headerUrl}, #{createTime})
    </insert>

</mapper>
```

