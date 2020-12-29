---
title: "Spring Boot + MyBatis (Configuration)"
date: 2020-10-21
categories:
  - Spring
tags:
  - Spring
  - Mybatis
---


## 구글링해서 나온 Spring Boot + MyBatis 연동 예제는 legacy 코드가 너무 많았다.
SpringBoot 와 mybatis를 연동해서 토이프로젝트를 진행하는 도중에 구글링을 여러번 하면서 다양한분들의 예제코드를 많이 봤다. 하지만 최근에 올라온 글임에도 불구하고 legacy 코드를 그대로 사용 하시는 분들도 있는걸 많이 보았고, 개개인의 주관이 없다고 느껴졌다. 그 판단이 틀리던 맞던 자기 주관이 있어야 고칠수있다고 생각한다. 나는 연동부분을 간단하게 테스트해보며 어떻게 사용하는게 맞는지 적어보려고 한다.  

## - Spring boot와 Mybatis 연동하기(MySQL,Maven)

### 1. Maven을 이용해 mabatis 와 MySQL과 연결해줄 connector 의존성을 주입한다.

```xml
<!-- https://mvnrepository.com/artifact/org.mybatis.spring.boot/mybatis-spring-boot-starter -->
<dependency>
    <groupId>org.mybatis.spring.boot</groupId>
	    <artifactId>mybatis-spring-boot-starter</artifactId>
		<version>2.1.3</version>
	</dependency>
	<dependency>
		<groupId>mysql</groupId>
		<artifactId>mysql-connector-java</artifactId>
	</dependency>
```
  
### 2. DB 연결을 위한 프로퍼티를 만들기위해 application.properties 파일 수정하기  
  파일경로 : src/main/resources/application.properties 
```
spring.datasource.hikari.driver-class-name=com.mysql.cj.jdbc.Driver
spring.datasource.hikari.jdbc-url=jdbc:mysql://포스트주소:포트/데이터베이스명?characterEncoding=DB캐릭터셋&serverTimezone=DB서버타임존
spring.datasource.hikari.username=DB접속시 사용할 아이디 
spring.datasource.hikari.password=DB접속 사용할비밀번호
```  

### 3. configuration class를 이용하여 DataSource,SqlFactory Bean 등록하기  
mvc에서는 application-Context.xml을 이용해서 빈을 등록합니다. 하지만 스프링진영에서 스프링부트를 메인으로 내세우고있고, mvc로 개발하던 자바 개발자들이 xml 지옥에 빠진것을 알고 최신버전으로 올라갈수록 xml을 지양하는 움직임을 보이고 있는것 같습니다. 그래서 저는 @Configuration 어노테이션을 이용하여 설정합니다.  

**1. configuration class는 어떻게 로딩될까?**  

스프링부트에서 configuration 어노테이션이 붙은 클래스는 스프링 부트 어플리케이션이 로드될때 Component-scan 을 통해 해석되게 됩니다.
Spring boot 기준으로, main 메서드에서 다음과 같은 메서드를 부릅니다.
```java
@SpringBootApplication
public class AuthappApplication {
	public static void main(String[] args) {
		SpringApplication.run(AuthappApplication.class, args);
	}
}
```
이 위에 선언된 @SpringBootApplication 어노테이션의 구현부를 간단히 살펴보면 아래와같습니다.  

```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Inherited
@SpringBootConfiguration
@EnableAutoConfiguration
@ComponentScan(excludeFilters = { @Filter(type = FilterType.CUSTOM, classes = TypeExcludeFilter.class),
		@Filter(type = FilterType.CUSTOM, classes = AutoConfigurationExcludeFilter.class) })
public @interface SpringBootApplication {
    /* 생략 */
}
```  
보시다시피 ConponentScan 어노테이션을 통해 컴포넌트들을 스캔하고 있습니다.

그래서 우리가 클래스를 만들어 @configuration 어노테이션을 등록했을때 이것을 컴포넌트라고 인식하고 스캔하게됩니다. 하지만, component라고 선언한것도 아닌데 어떻게 컴포넌트로 인식하고 스캔할까요? 답은 @Configuration 어노테이션의 구현부를 살펴보면 됩니다.  
  
```java
@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
@Documented
@Component
public @interface Configuration {
    /*생략 */
}
```  
@Configuration 어노테이션의 구현에서 이미 @Component 어노테이션을 물고있습니다. 그래서 우리가 @Configuration 어노테이션만 적어도 이미 컴포넌트로서 인식되고 있습니다.

이제 이해했으니 Configuration 클래스를 작성해보겠습니다.  
  
**2. configuration class 구현**  
```java
/**
 * @packageName : com.jaehyun.authapp.configuration
 * @fileName : DatabaseConfiguration.java
 * @author : parkjaehyun
 * @date : 2020.10.20
 * @description : 데이터베이스 설정 클래스
 *  ============================================================================
 *     DATE 	 AUTHOR      NOTE 
 *  ---------------------------------------------------------------------------- 
 *  2020.10.20 parkjaehyun   최초생성 
 *  2020.10.20 parkjaehyun   커넥션풀은 hikariCP를 이용, Mybatis 설정
 */

@Configuration
@PropertySource("classpath:/application.properties")
public class DatabaseConfiguration {
	@Autowired
	ApplicationContext applicationContext;

	/**
	 * @methodName : hikariConfig
	 * @author : jaehyun Park
	 * @date : 2020.10.20
	 * @description : hikari Connection pool option을 properties에서 가져와 등록.
	 * @return
	 */
	@Bean
	@ConfigurationProperties(prefix = "spring.datasource.hikari")
	public HikariConfig hikariConfig() {
		return new HikariConfig();
	}

	/**
	 * @methodName : dataSource
	 * @author : jaehyun Park
	 * @date : 2020.10.20
	 * @description : hikari Conntion poll을 이용한 데이터베이스 리소스 객체생성
	 * @return
	 * @throws Exception
	 */
	@Bean
	public DataSource dataSource() throws Exception {
		DataSource dataSource = new HikariDataSource(hikariConfig());
		return dataSource;
	}

	/**
	 * @methodName : sqlSessionFactory
	 * @author : jaehyun Park
	 * @date : 2020.10.20
	 * @description : Mybatis가 사용할 sqlSessionFactory을 생성한다.
	 * @param dataSource
	 * @return
	 * @throws Exception
	 */
	@Bean
	public SqlSessionFactory sqlSessionFactory(DataSource dataSource) throws Exception {
		SqlSessionFactoryBean sqlSessionFactoryBean = new SqlSessionFactoryBean();
		sqlSessionFactoryBean.setDataSource(dataSource);
		sqlSessionFactoryBean.setMapperLocations(applicationContext.getResources("classpath:/mappers/**/*.xml"));
		return sqlSessionFactoryBean.getObject();
	}

	/**
	 * @methodName : sqlSessionTemplate
	 * @author : jaehyun Park
	 * @date : 2020.10.20
	 * @description : mybatis가 사용할 sessionTemplate의 factory에 sessionFactory를 적용한다.
	 * @param sqlSessionFactory
	 * @return
	 */
	@Bean
	public SqlSessionTemplate sqlSessionTemplate(SqlSessionFactory sqlSessionFactory) {
		return new SqlSessionTemplate(sqlSessionFactory);
	}
}

```  

이 클래스가 하는 일은 아래와 같습니다.  
- hikari connection pool을 이용하여 DataSource 설정하기
- SqlSessionFacotry 빈 등록 (datasource와 매퍼xml 파일의 위치를 설정)
- SqlSessionTemplate 빈 등록 

이제 스프링부트가 실행될때 이 configuration 클래스를 스캔하고, 우리가 @Bean 어노테이션을 통해 선언해준 빈들이 spring bean container에 등록되어 사용됩니다.

**3. sqlSessionTemplate 이 뭘까요?**  
SqlSessionTemplate은 마이바티스 스프링 연동모듈의 핵심입니다.
SqlSessionTemplate은 SqlSession을 구현하고 코드에서 SqlSession를 대체하는 역할을 한다. SqlSessionTemplate 은 쓰레드에 안전하고 여러개의 DAO나 매퍼에서 공유할수 있습니다.  

SQL을 처리하는 마이바티스 메서드를 호출할때 SqlSessionTemplate은 SqlSession이 현재의 스프링 트랜잭션에서 사용될수 있도록 보장합니다. 추가적으로 SqlSessionTemplate은 필요한 시점에 세션을 닫고, 커밋하거나 롤백하는 것을 포함한 세션의 생명주기를 관리합니다. 또한 마이바티스 예외를 스프링의 DataAccessException로 변환하는 작업또한 처리합니다.  

SqlSessionTemplate은 생성자 인자로 SqlSessionFactory를 사용해서 생성될 수 있습니다.  

### 4. Database,Mappers.xml,DTO,Mapper interface 만들기  
-**1.Database table 만들기**  
테스트에 사용한 테이블의 형태는 다음과같습니다.  

<img src="/img/2020_10_21_data.png" width="700" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">  

<br>  

-**2. Mapper.xml 구현하기**  
앞전에 Configuration을 통해 resources/mappers/**/*.xml 경로의 파일들을 mapper 파일로 볼 수 있게 설정해두었으니 동일한 위치에 디렉토리를 생성해주고 xml 파일을 만들어 작성해줍니다.

<img src="/img/2020_10_21_explorer.png" width="400" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;"> <br>  

저는 MemberMapper.xml 이라는 파일명으로 맵퍼 xml을 구성했습니다.  

- MemberMapper.xml  

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.jaehyun.authapp.dao.MemberMapper">
	<!-- 아이디 기반 회원정보검색 -->
	<select id="findByUserName" parameterType="String"
		resultType="com.jaehyun.authapp.dto.Member">
    	 <![CDATA[
    	select * 
    	from member
    	where username=#{username}
    ]]>
	</select>
	<select id="getAuthorities" parameterType="String"
		resultType="String">
		<![CDATA[
			SELECT memberAuthorities 
			FROM authority WHERE 
			username = #{username}
		]]>
	</select>
</mapper>
```

-**3. Member DTO 구현하기**
Mybatis를 이용해 결과를 받을 DTO를 생성합니다. lombok을 이용하였습니다.
spring security를 이용하지 않았다면 용도에맞게 일반적인 DTO를 만들어 사용하시면 됩니다.
```java
/**  
* @packageName : com.jaehyun.authapp.dto
* @fileName	   : Member.java
* @author 	   : parkjaehyun
* @date 	   : 2020.10.21
* @description : Spring Security UserDetails interface를 구현한 회원 UserDetails 클래스
* ============================================================================
* DATE       	   AUTHOR  	       NOTE
* ----------------------------------------------------------------------------
* 2020.10.21       parkjaehyun     최초생성
*/ 
@ToString
@Setter
@Getter
public class Member implements UserDetails{
	private static final long serialVersionUID = 1L;
	private String username;
	private String password;
	private String nickname;
	private boolean isAccountNonExpired;
	private boolean isAccountNonLocked;
	private boolean isCredentialsNonExpried;
	private boolean isEnabled;
	private Collection<? extends GrantedAuthority> authorities;
	@Override
	public Collection<? extends GrantedAuthority> getAuthorities() {
		return authorities;
	}
	@Override
	public String getPassword() {
		return password;
	}
	@Override
	public String getUsername() {
		return username;
	}
	@Override
	public boolean isAccountNonExpired() {
		return isAccountNonExpired;
	}
	@Override
	public boolean isAccountNonLocked() {
		return isAccountNonLocked;
	}
	@Override
	public boolean isCredentialsNonExpired() {
		return isCredentialsNonExpried;
	}
	@Override
	public boolean isEnabled() {
		return isEnabled;
	}
}

```


-**4. Member Mapper interface 구현하기**
Mybatis는 @Mapper 어노테이션을통해 interface 형태로 맵퍼를 이용할 수 있으며, mybatis에서 이를 해석하고 자연스럽게 DTO에 매칭해주게 됩니다.

```java
/**
* @packageName : com.jaehyun.authapp.mappers
* @fileName	   : MemberMapper.java
* @author 	   : parkjaehyun
* @date 	   : 2020.10.21
* @description : Mybatis를 이용한 회원 Mapper 클래스
* ============================================================================
* DATE       	   AUTHOR  	       NOTE
* ----------------------------------------------------------------------------
* 2020.10.21       parkjaehyun     최초생성
*/ 
@Mapper
public interface MemberMapper {
	/**
	 * @methodName  : findByUserName
	 * @author      : jaehyun Park
	 * @date        : 2020.10.21
	 * @description : 회원 조회(아이디기반)
	 * @param username
	 * @return
	 */
	public Member findByUserName(String username);
	/**
	 * @methodName  : joinMember
	 * @author      : jaehyun Park
	 * @date        : 2020.10.21
	 * @description : 회원가입 
	 * @param member
	 * @return
	 */
	public int joinMember(Member member);
	
    /**
     * @methodName  : readAuthority
     * @author      : jaehyun Park
     * @date        : 2020.10.21
     * @description : 해당 유저의 권한을 모두 가져온다.
     * @param username
     * @return
     */
    public List<String> getAuthorities(String username);

}
```

#### MyBatis의 Mapper와 DAO는 같은 행위를 하고있는데요?

제가 구글링하면서 불편함을 많이 느꼇던 부분이 바로 이부분입니다.
많은 예제들이 DAO와 Mapper interface를 별개로 분류하고 **DAO에서 Mapper 인터페이스를 물고 이용하는 방식**으로 구성되어 있었습니다.

하지만 자세히 생각해보면 DAO 계층과 Mapper interface의 행위와 책임은 거의 같습니다. 
Mybatis가 SqlSession을 이용해 DataBase에 Access하는 행위를 하고있고, 기존에 생성되어있는 DAO에서 또한 그런 행위에 대한 책임만을 가지고있습니다. Mybatis는 DAO에서 하는 작업을 XML을 이용하여 한번더 나누었고, 인터페이스를 통해 전반적인 구현, 실행의 책임을 Mybatis core에 맡겼을 뿐입니다. 저는 그래서 DAO와 Mapper계층의 공존이 정상적이지 않다고 생각합니다.

### 5. JUnit을 이용해 정상적인 연동이 되었는지 확인하기  
우선 연동을 확인하기 위해 만들어볼 코드가 있습니다.
저는 Member라는 DTO 를 만들어 사용할 예정이며, Spring security를 함께 사용하고 있기 때문에 DTO 클래스가 복잡하네요 :(
<img src="/img/2020_10_21_member.png" width="500" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">
<br>


-**1. Mapper interface 이용해 정상적인 값을 가지고오는지 확인하기**  

JUnit5를 이용하여 테스트해봅니다.
```java
@SpringBootTest
class MemberMapperTest {
    private static final Logger logger = LoggerFactory.getLogger(Assert.class);
	@Autowired
	MemberMapper mapper;
	@Test
	@DisplayName("맵퍼 회원이름 조회 테스트케이스")
	void 회원이름으로_조회_잘되는지() {
		 mapper.findByUserName("admin");
		 Member member = mapper.findByUserName("admin");
		 logger.info(member.toString());
		 assertTrue("admin".equals(member.getUsername()));
		 assertTrue("admin".equals(member.getNickname()));
		 assertTrue("admin".equals(member.getPassword()));
         }

}
```
-**2. 결과**

<img src="/img/2020_10_21_junit.png" width="700" style="display:block;margin-left:auto; margin-right:auto;margin-top:20px;">
<br>

테스트가 정상 통과되었고 원하는대로 잘 나와주었습니다.
