## Chapter 01: 스프링 부트 시작하기

### 1. 스프링 부트의 등장

#### 1.1 스프링 프레임워크

**스프링 프레임워크**는 자바로 웹 애플리케이션을 만들 때 쓰는 도구다. 이거는 마치 회사에서 엑셀 대신 더 강력한 ERP 시스템을 쓰는 것과 비슷하다.

하지만 문제가 있었다:

- **XML 설정 지옥**: 설정 파일만 수십 개가 필요했다
- **복잡한 의존성**: 필요한 라이브러리를 하나하나 찾아서 버전까지 맞춰야 했다
- **긴 개발 시간**: 간단한 웹사이트 하나 만드는데도 설정만 며칠이 걸렸다

XML

```
<!-- 예전 스프링 설정 예시 -->
<bean id="dataSource" class="org.apache.commons.dbcp.BasicDataSource">
    <property name="driverClassName" value="com.mysql.jdbc.Driver"/>
    <property name="url" value="jdbc:mysql://localhost:3306/company"/>
    <!-- 이런 설정이 수백 줄... -->
</bean>
```

#### 1.2 스프링 부트의 등장

**스프링 부트**는 이런 복잡한 설정을 자동으로 해주는 거다. 이거는 회사에서 신입사원이 들어왔을 때 컴퓨터, 책상, 사원증을 일일이 신청하는 대신 "신입사원 패키지"로 한 번에 받는 것과 같다.

스프링 부트의 장점:

- **자동 설정**: 대부분의 설정을 알아서 해준다
- **내장 서버**: 톰캣 같은 서버가 이미 들어있다
- **스타터 의존성**: 필요한 라이브러리를 묶어서 제공한다
- **실행 가능한 JAR**: 만든 프로그램을 바로 실행할 수 있다

### 3. 스프링 부트 퀵스타트

#### 3.1 스프링 부트 프로젝트 만들기

**스프링 부트 프로젝트 만들기**는 회사에서 새로운 프로젝트 폴더를 만드는 거다. Spring Initializr(start.spring.io)를 사용하면 클릭 몇 번으로 프로젝트를 만들 수 있다.

프로젝트 생성 시 선택사항:

- **Group**: 회사명 (com.mycompany)
- **Artifact**: 프로젝트명 (employee-system)
- **Dependencies**: 필요한 기능들 (Web, JPA, Security 등)

#### 3.2 스프링 부트 프로젝트 구조 및 실행

**프로젝트 구조**는 이렇게 생겼다:

text

```
employee-system/
├── src/
│   ├── main/
│   │   ├── java/           # 실제 업무 코드를 넣는 곳이다
│   │   │   └── com/mycompany/
│   │   │       ├── controller/   # 요청을 받는 창구다
│   │   │       ├── service/      # 업무를 처리하는 곳이다
│   │   │       ├── repository/   # 데이터를 관리하는 곳이다
│   │   │       └── entity/       # 데이터 구조를 정의하는 곳이다
│   │   └── resources/      # 설정 파일을 넣는 곳이다
│   │       ├── application.properties  # 메인 설정 파일이다
│   │       ├── static/     # CSS, JS 파일을 넣는 곳이다
│   │       └── templates/  # HTML 파일을 넣는 곳이다
│   └── test/              # 테스트 코드를 넣는 곳이다
└── pom.xml               # 프로젝트 설정과 의존성 목록이다
```

#### 3.3 스프링 부트 프로젝트 둘러보기

**메인 클래스**는 프로그램의 시작점이다:

Java

```
@SpringBootApplication  // 이거는 "스프링 부트 앱이다"라고 선언하는 거다
public class EmployeeSystemApplication {
    public static void main(String[] args) {
        SpringApplication.run(EmployeeSystemApplication.class, args);
        // 이 한 줄로 웹 서버가 시작된다!
    }
}
```

**application.properties**는 프로그램 설정을 담는 파일이다:

properties

```
# 서버 포트 설정
server.port=8080

# 데이터베이스 연결 정보
spring.datasource.url=jdbc:mysql://localhost:3306/company
spring.datasource.username=admin
spring.datasource.password=password123

# JPA 설정
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

#### 3.4 웹 애플리케이션 작성하기

**첫 번째 컨트롤러 만들기**:

Java

```
@RestController  // 이거는 "웹 요청을 받는 컨트롤러다"라고 선언하는 거다
public class HelloController {
    
    @GetMapping("/hello")  // GET 요청을 받는다
    public String hello() {
        return "안녕하세요, 김대리님!";
    }
    
    @GetMapping("/employee/{name}")  // URL에서 이름을 받는다
    public String getEmployee(@PathVariable String name) {
        return name + "님의 정보를 조회합니다.";
    }
    
    @PostMapping("/employee")  // POST 요청을 받는다
    public String createEmployee(@RequestBody Employee employee) {
        return employee.getName() + "님이 등록되었습니다.";
    }
}
```