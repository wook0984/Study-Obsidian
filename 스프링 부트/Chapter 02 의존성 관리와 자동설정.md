## 1. 스프링 부트의 의존성 관리

### 1.1 스타터로 의존성 관리하기

스프링 부트 스타터는 관련된 라이브러리를 묶어놓은 패키지다. 이거는 회사에서 업무별로 필요한 사무용품을 세트로 제공하는 것과 같다.

주요 스타터 패키지:

XML

```
<!-- 웹 개발용 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>

<!-- 데이터베이스 연동용 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>

<!-- 보안 기능용 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

이렇게 한 줄만 추가하면 관련된 모든 라이브러리가 자동으로 포함된다. 예를 들어 `spring-boot-starter-web`은 웹 서버, MVC 프레임워크, JSON 처리 등 웹 개발에 필요한 모든 것을 포함한다.

### 1.2 의존성 재정의하기

스프링 부트가 관리하는 기본 의존성을 변경할 수 있다. 이거는 회사에서 제공하는 기본 장비 대신 내가 원하는 다른 장비로 바꾸는 것과 같다.

XML

```
<properties>
    <java.version>17</java.version>
    <mysql.version>8.0.30</mysql.version>  <!-- MySQL 버전 지정 -->
</properties>

<!-- 특정 라이브러리 제외하기 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
    <exclusions>
        <exclusion>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-tomcat</artifactId>
        </exclusion>
    </exclusions>
</dependency>

<!-- 대체 라이브러리 추가 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

이 예시는 기본 웹 서버인 Tomcat 대신 Jetty를 사용하도록 설정한 것이다.

## 2. 스프링 부트의 자동설정

### 2.1 자동설정 이해하기

스프링 부트의 자동설정은 클래스패스에 있는 라이브러리를 검사하여 필요한 설정을 자동으로 적용한다. 이거는 회사 컴퓨터에 필요한 프로그램이 자동으로 설치되고 설정되는 것과 같다.

예를 들어:

- H2 데이터베이스 라이브러리가 있으면 자동으로 인메모리 데이터베이스를 설정한다
- Thymeleaf 라이브러리가 있으면 자동으로 템플릿 엔진을 설정한다
- Spring Security 라이브러리가 있으면 자동으로 기본 보안 설정을 적용한다

자동설정의 작동 원리:

1. `@SpringBootApplication`에 포함된 `@EnableAutoConfiguration` 어노테이션이 자동설정을 활성화한다
2. 스프링 부트는 META-INF/spring.factories 파일에 나열된 자동설정 클래스를 로드한다
3. 각 자동설정 클래스는 특정 조건을 확인하고, 조건이 맞으면 빈을 등록한다

### 2.2 사용자 정의 스타터

회사에서 자주 사용하는 기능을 모아 자체 스타터를 만들 수 있다. 이거는 회사 전용 업무 도구 세트를 만드는 것과 같다.

사용자 정의 스타터 만들기:

1. 자동설정 클래스 작성:

Java

```
@Configuration
@ConditionalOnClass(EmployeeService.class)
public class EmployeeAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean
    public EmployeeService employeeService() {
        return new DefaultEmployeeService();
    }
    
    @Bean
    @ConditionalOnProperty(name = "company.reports.enabled", havingValue = "true")
    public ReportGenerator reportGenerator() {
        return new ExcelReportGenerator();
    }
}
```

2. META-INF/spring.factories 파일 작성:

text

```
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mycompany.EmployeeAutoConfiguration
```

이렇게 하면 이 스타터를 사용하는 모든 프로젝트에서 직원 관리와 보고서 생성 기능을 자동으로 사용할 수 있다.

### 2.3 자동설정 재정의하기

스프링 부트의 자동설정을 필요에 맞게 재정의할 수 있다. 이거는 회사의 기본 업무 방식을 프로젝트에 맞게 수정하는 것과 같다.

Java

```
@Configuration
public class CustomWebConfiguration {
    
    @Bean
    public WebMvcConfigurer webMvcConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addFormatters(FormatterRegistry registry) {
                // 날짜 형식 변환기 추가
                registry.addConverter(new StringToDateConverter());
            }
            
            @Override
            public void addInterceptors(InterceptorRegistry registry) {
                // 로그인 체크 인터셉터 추가
                registry.addInterceptor(new LoginCheckInterceptor())
                       .addPathPatterns("/**")
                       .excludePathPatterns("/login", "/static/**");
            }
        };
    }
}
```

이 예시는 웹 MVC 설정을 커스터마이징하여 날짜 변환기와 로그인 체크 인터셉터를 추가한 것이다.