---
title: SpringBoot2.0应用（五）：SpringBoot2.0整合MyBatis
categories: SpringBoot
tags:
  - SpringBoot
  - 应用
comments: true
abbrlink: 20008
date: 2018-05-01 15:51:30
---

# 如何整合MyBatis
## 1、pom依赖
```
        <dependency>
            <groupId>org.mybatis.spring.boot</groupId>
            <artifactId>mybatis-spring-boot-starter</artifactId>
            <version>1.3.2</version>
        </dependency>
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <scope>runtime</scope>
        </dependency>
        <!-- 分页插件 -->
        <dependency>
            <groupId>com.github.pagehelper</groupId>
            <artifactId>pagehelper-spring-boot-starter</artifactId>
            <version>1.2.5</version>
        </dependency>
        <!--mapper -->
        <dependency>
            <groupId>tk.mybatis</groupId>
            <artifactId>mapper-spring-boot-starter</artifactId>
            <version>1.1.3</version>
        </dependency>
        <!-- alibaba的druid数据库连接池 -->
        <dependency>
            <groupId>com.alibaba</groupId>
            <artifactId>druid-spring-boot-starter</artifactId>
            <version>1.1.9</version>
        </dependency>
```
## 2、添加配置
```
spring.datasource.name=mysql_test
spring.datasource.type=com.alibaba.druid.pool.DruidDataSource
#druid相关配置
spring.datasource.druid.filters=stat
spring.datasource.druid.driver-class-name=com.mysql.jdbc.Driver
#基本属性
spring.datasource.druid.url=jdbc:mysql://localhost:3306/test?characterEncoding=utf8&useSSL=false
spring.datasource.druid.username=root
spring.datasource.druid.password=1234
#配置初始化大小/最小/最大
spring.datasource.druid.initial-size=1
spring.datasource.druid.min-idle=1
spring.datasource.druid.max-active=20
#获取连接等待超时时间
spring.datasource.druid.max-wait=60000
#间隔多久进行一次检测，检测需要关闭的空闲连接
spring.datasource.druid.time-between-eviction-runs-millis=60000
#一个连接在池中最小生存的时间
spring.datasource.druid.min-evictable-idle-time-millis=300000
spring.datasource.druid.validation-query=SELECT 'x'
spring.datasource.druid.test-while-idle=true
spring.datasource.druid.test-on-borrow=false
spring.datasource.druid.test-on-return=false
#打开PSCache，并指定每个连接上PSCache的大小。oracle设为true，mysql设为false。分库分表较多推荐设置为false
spring.datasource.druid.pool-prepared-statements=false
spring.datasource.druid.max-pool-prepared-statement-per-connection-size=20
#Mapper路径
mybatis.mapper-locations=classpath=mapper/*.xml
#pagehelper
pagehelper.helperDialect=mysql
pagehelper.reasonable=true
pagehelper.supportMethodsArguments=true
pagehelper.params=count=countSql
```
## 3、自动生成Mapper
### 添加Mybatis的自动生成插件
```
            <plugin>
                <groupId>org.mybatis.generator</groupId>
                <artifactId>mybatis-generator-maven-plugin</artifactId>
                <version>1.3.2</version>
                <configuration>
                    <configurationFile>${basedir}/src/main/resources/generator/generatorConfig.xml</configurationFile>
                    <overwrite>true</overwrite>
                    <verbose>true</verbose>
                </configuration>
                <dependencies>
                    <dependency>
                        <groupId>mysql</groupId>
                        <artifactId>mysql-connector-java</artifactId>
                        <version>${mysql.version}</version>
                    </dependency>
                    <dependency>
                        <groupId>tk.mybatis</groupId>
                        <artifactId>mapper</artifactId>
                        <version>3.4.0</version>
                    </dependency>
                </dependencies>
            </plugin>
```
### 配置Mybatis的generatorConfig
```
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE generatorConfiguration
        PUBLIC "-//mybatis.org//DTD MyBatis Generator Configuration 1.0//EN"
        "http://mybatis.org/dtd/mybatis-generator-config_1_0.dtd">
<generatorConfiguration>
    <properties resource="application.properties"/>
    <context id="Mysql" targetRuntime="MyBatis3Simple" defaultModelType="flat">
        <property name="beginningDelimiter" value="`"/>
        <property name="endingDelimiter" value="`"/>
        <plugin type="tk.mybatis.mapper.generator.MapperPlugin">
            <property name="mappers" value="sample.mybatis.mapper.MyMapper"/>
        </plugin>
        <jdbcConnection connectionURL="jdbc:mysql://localhost:3306/test?useUnicode=true"
                        driverClass="com.mysql.jdbc.Driver"
                        userId="root"
                        password="1234">
        </jdbcConnection>
        <javaModelGenerator targetPackage="sample.mybatis.entity" targetProject="src/main/java">
        </javaModelGenerator>
        <sqlMapGenerator targetPackage="mapper" targetProject="src/main/resources"/>
        <javaClientGenerator targetPackage="sample.mybatis.dao" targetProject="src/main/java"
                             type="XMLMAPPER"/>
        <table tableName="city">
            <!--mysql 配置-->
            <generatedKey column="id" sqlStatement="Mysql" identity="true"/>
        </table>
    </context>
</generatorConfiguration>
```
执行插件会自动生成实体Bean，Mapper接口和对应的xml文件。
## 5、写个简单的Controller触发调用
```
@RestController
public class CityController {

    @Autowired
    private CityMapper cityMapper;

    @GetMapping("/")
    public List<City> index() {
        PageHelper.startPage(0, 3);
        List<City> cities = this.cityMapper.selectAll();
        return cities;
    }

}
```
启动项目后通过PostMan访问：http://localhost:8080/
![](https://user-gold-cdn.xitu.io/2018/11/25/1674b86a98ad8f83?w=359&h=332&f=png&s=28125)



源码地址：[GitHub](https://github.com/KAMIJYOUDOUMA/spring-boot-samples)






 

