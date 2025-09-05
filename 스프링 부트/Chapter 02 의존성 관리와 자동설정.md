## Chapter 02: 의존성 관리와 자동설정

### 1. 스프링 부트의 의존성 관리

#### 1.1 스타터로 의존성 관리하기

**스타터(Starter)**는 필요한 도구들을 묶어놓은 패키지다. 이거는 회사에서 "회의실 예약 시스템"을 쓰려면 달력, 알림, 이메일 기능이 다 필요한데, 이걸 "회의실 패키지"로 한 번에 설치하는 것과 같다.

주요 스타터들:

XML

```
<!-- 웹 개발 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-web</artifactId>
</dependency>
<!-- 이거 하나로 다음이 모두 포함된다:
     - Spring MVC (웹 프레임워크)
     - Tomcat (웹 서버)
     - Jackson (JSON 변환)
     - Validation (입력값 검증)
-->

<!-- 데이터베이스 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
<!-- 이거 하나로 다음이 모두 포함된다:
     - Hibernate (ORM)
     - Spring Data JPA
     - Spring Transaction
-->

<!-- 보안 스타터 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

#### 1.2 의존성 재정의하기

**의존성 재정의**는 기본 패키지에서 특정 부분만 바꾸는 거다. 이거는 회사 노트북에 기본으로 깔린 오피스 2019를 2021로 업그레이드하는 것과 같다.

XML

```
<!-- 스프링 부트가 관리하는 버전 재정의 -->
<properties>
    <mysql.version>8.0.33</mysql.version>  <!-- MySQL 버전 변경 -->
</properties>

<!-- 특정 라이브러리 제외하고 다른 것 사용 -->
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
<!-- Tomcat 대신 Jetty 사용 -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-jetty</artifactId>
</dependency>
```

### 2. 스프링 부트의 자동설정

#### 2.1 자동설정 이해하기

**자동설정(Auto Configuration)**은 스프링 부트가 알아서 필요한 설정을 해주는 거다. 이거는 새 직원이 왔을 때 IT팀이 알아서 이메일 계정, 메신저, 사내 시스템 접속 권한을 다 설정해주는 것과 같다.

자동설정이 하는 일:

1. **클래스패스 스캔**: 어떤 라이브러리가 있는지 확인한다
2. **조건 확인**: 필요한 조건이 충족되는지 확인한다
3. **빈 생성**: 필요한 객체들을 자동으로 만든다

Java

```
// 자동설정 예시
@Configuration
@ConditionalOnClass(DataSource.class)  // DataSource 클래스가 있으면
@EnableConfigurationProperties(DataSourceProperties.class)
public class DataSourceAutoConfiguration {
    
    @Bean
    @ConditionalOnMissingBean  // 다른 DataSource가 없으면
    public DataSource dataSource(DataSourceProperties properties) {
        // 자동으로 DataSource를 만든다
        return properties.initializeDataSourceBuilder().build();
    }
}
```

#### 2.2 사용자 정의 스타터

**사용자 정의 스타터**는 우리 회사만의 특별한 패키지를 만드는 거다. 예를 들어 우리 회사만 쓰는 "전자결재 시스템 패키지"를 만드는 거다.

Java

```
// 1. 자동설정 클래스 만들기
@Configuration
@ConditionalOnProperty(name = "company.approval.enabled", havingValue = "true")
public class ApprovalSystemAutoConfiguration {
    
    @Bean
    public ApprovalService approvalService() {
        return new ApprovalService();
    }
    
    @Bean
    public ApprovalController approvalController(ApprovalService service) {
        return new ApprovalController(service);
    }
}

// 2. spring.factories 파일에 등록
// META-INF/spring.factories
org.springframework.boot.autoconfigure.EnableAutoConfiguration=\
com.mycompany.approval.ApprovalSystemAutoConfiguration
```

#### 2.3 자동설정 재정의하기

**자동설정 재정의**는 기본 설정을 우리 회사에 맞게 바꾸는 거다:

Java

```
@Configuration
public class CustomWebConfig {
    
    @Bean
    public WebMvcConfigurer webMvcConfigurer() {
        return new WebMvcConfigurer() {
            @Override
            public void addCorsMappings(CorsRegistry registry) {
                // CORS 설정을 우리 회사에 맞게 변경한다
                registry.addMapping("/api/**")
                        .allowedOrigins("https://mycompany.com")
                        .allowedMethods("GET", "POST", "PUT", "DELETE");
            }
        };
    }
}
```