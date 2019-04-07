---
title: SpringBoot2.0应用（四）：SpringBoot2.0之spring-data-jpa
categories: SpringBoot
tags:
  - SpringBoot
  - 应用
comments: true
abbrlink: 20006
date: 2018-04-17 15:51:30
---

# 如何整合spring data jpa
## 1、pom依赖
```
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-jpa</artifactId>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
		</dependency>
```
## 2、添加配置
```
spring.datasource.url=jdbc:mysql://localhost:3306/test?characterEncoding=utf8&useSSL=true
spring.datasource.username=root
spring.datasource.password=1234
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```
## 3、创建dto对象
```
@Entity
public class City implements Serializable {

	private static final long serialVersionUID = 1L;

	@Id
	@SequenceGenerator(name = "city_generator", sequenceName = "city_sequence", initialValue = 23)
	@GeneratedValue(generator = "city_generator")
	private Long id;

	@Column(nullable = false)
	private String name;

	@Column(nullable = false)
	private String state;

	@Column(nullable = false)
	private String country;

	@Column(nullable = false)
	private String map;
    ......
}
```
## 4、创建操作数据的Repository对象
```
public interface CityRepository extends Repository<City, Long> {

	Page<City> findAll(Pageable pageable);

	Page<City> findByNameContainingAndCountryContainingAllIgnoringCase(String name,
			String country, Pageable pageable);

	City findByNameAndCountryAllIgnoringCase(String name, String country);

}
```
## 5、写个简单的Controller触发调用
```
@Controller
public class CityController {

    @Autowired
    private CityRepository cityRepository;

    @GetMapping("/")
    @ResponseBody
    @Transactional(readOnly = true)
    public void helloWorld() {
        City city = cityRepository.findByNameAndCountryAllIgnoringCase("Bath", "UK");
        System.out.println(city);

        Page<City> cityPage = cityRepository.findAll(new PageRequest(0,
                10,
                new Sort(Sort.Direction.DESC, "name")));
        System.out.println(Arrays.toString(cityPage.getContent().toArray()));
    }
}
```
启动项目后访问http://localhost:8080/，控制台输出：
```
Bath,Somerset,UK
[Washington,DC,USA, Tokyo,,Japan, Tel Aviv,,Israel, Sydney,New South Wales,Australia, Southampton,Hampshire,UK, San Francisco,CA,USA, Palm Bay,FL,USA, New York,NY,USA, Neuchatel,,Switzerland, Montreal,Quebec,Canada]
```
到此，一个简单的`SpringBoot2.0`集成`spring-data-jpa`就完成了。
spring-data-jpa对一些简单的数据库操作进行了支持。具体的关键字如下：And，Or，Is,Equals，Between，LessThan，LessThanEqual，GreaterThan，GreaterThanEqual，After，Before，IsNull，IsNotNull,NotNull，Like，NotLike，StartingWith，EndingWith，Containing，OrderBy，Not，In，NotIn，TRUE，FALSE，IgnoreCase。spring-data-jpa对这些关键字的支持原理将在源码分析篇讲解，欢迎关注。

如果有复杂一些的sql语句，依靠上面的关键字是肯定不行的，所以`spring-data-jpa`还提供了注解用来支持自定义sql。在SQL的查询方法上面使用@Query注解，如涉及到删除和修改在需要加上@Modifying。
例：
```
    @Query("select c from City c where c.id = ?1")
    City queryById(long id);

    @Modifying
    @Query("update City c set c.name = ?2 where c.id = ?1")
    int updateNameById(long id, String name);
```
注意：自定义sql的语句是对对象进行操作，风格和hql相似。


SQL数据文件在源码中,源码地址：[GitHub](https://github.com/KAMIJYOUDOUMA/spring-boot-samples)



 

