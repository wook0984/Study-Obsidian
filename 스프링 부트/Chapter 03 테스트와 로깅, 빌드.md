# Chapter 03: 테스트와 로깅, 빌드

## 1. 스프링 부트 테스트

### 1.1 스프링 부트에서 테스트하기

스프링 부트는 테스트를 쉽게 할 수 있는 도구를 제공한다. 이거는 회사에서 업무 프로세스가 제대로 작동하는지 검증하는 품질 관리 시스템과 같다.

기본 테스트 예제:

Java

```
@SpringBootTest  // 스프링 부트 테스트를 사용한다
class EmployeeServiceTest {
    
    @Autowired  // 스프링이 자동으로 객체를 주입한다
    private EmployeeService employeeService;
    
    @Test  // 이 메소드는 테스트 케이스다
    void 직원등록_테스트() {
        // Given (준비)
        Employee employee = new Employee("김사원", "개발팀");
        
        // When (실행)
        Employee savedEmployee = employeeService.register(employee);
        
        // Then (검증)
        assertNotNull(savedEmployee.getId());  // ID가 생성되었는지 확인
        assertEquals("김사원", savedEmployee.getName());  // 이름이 저장되었는지 확인
    }
}
```

이 테스트는 직원 등록 기능이 제대로 동작하는지 확인한다. "Given-When-Then" 패턴을 사용하여 테스트를 구성한다:

- **Given**: 테스트를 위한 준비 (테스트 데이터 설정)
- **When**: 테스트할 기능 실행
- **Then**: 결과 검증

### 1.2 MockMvc 이용해서 컨트롤러 테스트하기

MockMvc는 웹 요청을 테스트하기 위한 도구다. 이거는 실제 웹 브라우저 없이 웹 요청을 시뮬레이션하는 것과 같다.

Java

```
@WebMvcTest(EmployeeController.class)  // 웹 계층만 테스트한다
class EmployeeControllerTest {
    
    @Autowired
    private MockMvc mockMvc;  // MockMvc 객체를 주입받는다
    
    @MockBean  // 가짜 서비스 객체를 만든다
    private EmployeeService employeeService;
    
    @Test
    void 직원목록_페이지_로드_테스트() throws Exception {
        // Given
        List<Employee> employees = Arrays.asList(
            new Employee("김사원", "개발팀"),
            new Employee("박대리", "영업팀")
        );
        given(employeeService.findAll()).willReturn(employees);  // 서비스 동작 설정
        
        // When & Then
        mockMvc.perform(get("/employees"))  // GET /employees 요청 테스트
               .andExpect(status().isOk())  // HTTP 200 상태 확인
               .andExpect(view().name("employee/list"))  // 뷰 이름 확인
               .andExpect(model().attributeExists("employees"))  // 모델 데이터 확인
               .andExpect(content().string(containsString("김사원")))  // 응답 내용 확인
               .andExpect(content().string(containsString("박대리")));
    }
}
```

이 테스트는 `/employees` 페이지에 접속했을 때:

1. HTTP 상태가 200(OK)인지 확인한다
2. "employee/list" 뷰를 사용하는지 확인한다
3. 모델에 "employees" 데이터가 있는지 확인한다
4. 응답 내용에 "김사원"과 "박대리"가 포함되어 있는지 확인한다

### 1.3 서비스 계층을 연동하는 컨트롤러 테스트하기

실제 서비스 계층과 연동하는 통합 테스트도 가능하다. 이거는 회사의 여러 부서가 함께 참여하는 업무 프로세스를 테스트하는 것과 같다.

Java

```
@SpringBootTest
@AutoConfigureMockMvc
class EmployeeIntegrationTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Autowired
    private EmployeeRepository employeeRepository;  // 실제 리포지토리 사용
    
    @BeforeEach
    void setup() {
        employeeRepository.deleteAll();  // 테스트 전에 데이터 초기화
    }
    
    @Test
    void 직원등록_통합테스트() throws Exception {
        // Given
        EmployeeDTO dto = new EmployeeDTO("이과장", "인사팀", "과장");
        
        // When & Then
        mockMvc.perform(post("/employee/register")
               .contentType(MediaType.APPLICATION_FORM_URLENCODED)
               .param("name", dto.getName())
               .param("department", dto.getDepartment())
               .param("position", dto.getPosition()))
               .andExpect(status().is3xxRedirection())  // 리다이렉트 확인
               .andExpect(redirectedUrl("/employees"));  // 리다이렉트 URL 확인
        
        // 데이터베이스에 저장되었는지 확인
        List<Employee> employees = employeeRepository.findAll();
        assertEquals(1, employees.size());
        assertEquals("이과장", employees.get(0).getName());
        assertEquals("인사팀", employees.get(0).getDepartment());
    }
}
```

이 테스트는 폼 제출을 시뮬레이션하고, 데이터가 실제로 데이터베이스에 저장되는지 확인한다.

## 2. 스프링 부트 로깅

### 2.1 스프링 부트 로깅

로깅은 프로그램의 동작 상태와 오류를 기록하는 것이다. 이거는 회사에서 업무 일지나 문제 해결 보고서를 작성하는 것과 같다.

스프링 부트는 기본적으로 SLF4J와 Logback을 사용한다:

Java

```
@Service
public class EmployeeService {
    
    private final Logger log = LoggerFactory.getLogger(EmployeeService.class);
    
    public Employee register(Employee employee) {
        log.info("신규 직원 등록: {}", employee.getName());  // 정보 로그
        
        try {
            // 직원 등록 로직
            return employeeRepository.save(employee);
        } catch (Exception e) {
            log.error("직원 등록 중 오류 발생: {}", e.getMessage(), e);  // 오류 로그
            throw e;
        }
    }
    
    public List<Employee> findByDepartment(String department) {
        log.debug("부서별 직원 조회: {}", department);  // 디버그 로그
        return employeeRepository.findByDepartment(department);
    }
}
```

로그 레벨:

- **ERROR**: 심각한 오류 (예: 데이터베이스 연결 실패)
- **WARN**: 경고 (예: 잘못된 요청, 하지만 처리 가능)
- **INFO**: 일반 정보 (예: 애플리케이션 시작, 중요 이벤트)
- **DEBUG**: 디버깅 정보 (개발 중에 유용)
- **TRACE**: 매우 상세한 정보 (거의 사용하지 않음)

### 2.2 스프링 부트 로깅 수정하기

로깅 설정은 application.properties 파일에서 변경할 수 있다:

properties

```
# 로그 레벨 설정
logging.level.root=INFO  # 기본 로그 레벨
logging.level.com.mycompany=DEBUG  # 특정 패키지의 로그 레벨
logging.level.org.springframework=WARN  # 스프링 프레임워크 로그 레벨

# 로그 파일 설정
logging.file.name=logs/employee-system.log  # 로그 파일 위치
logging.file.max-size=10MB  # 최대 파일 크기
logging.file.max-history=7  # 보관할 로그 파일 수

# 로그 패턴 설정
logging.pattern.console=%d{yyyy-MM-dd HH:mm:ss} [%thread] %-5level %logger{36} - %msg%n
```

이 설정은:

1. 루트 로그 레벨을 INFO로 설정한다
2. 우리 회사 코드(com.mycompany)는 상세한 DEBUG 로그를 남긴다
3. 스프링 프레임워크는 중요한 WARN 로그만 남긴다
4. 로그를 'logs/employee-system.log' 파일에 저장한다
5. 로그 파일이 10MB를 넘으면 새 파일을 만들고, 최대 7개 파일을 보관한다

## 3. 독립적으로 실행 가능한 JAR

### 3.1 스프링 부트 빌드 이해하기

스프링 부트는 모든 의존성을 포함한 실행 가능한 JAR 파일을 만들 수 있다. 이거는 회사 프로젝트의 모든 문서와 자원을 하나의 패키지로 묶는 것과 같다.

Maven을 사용한 빌드:

XML

```
<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
        </plugin>
    </plugins>
</build>
```

이 플러그인은 `mvn package` 명령을 실행할 때 실행 가능한 JAR 파일을 생성한다.

빌드 과정:

1. 소스 코드 컴파일
2. 테스트 실행
3. 리소스 파일 복사
4. 의존성 라이브러리 포함
5. 실행 가능한 JAR 파일 생성

### 3.2 Runnable JAR 실행하기

생성된 JAR 파일은 Java 명령으로 쉽게 실행할 수 있다:

Bash

```
java -jar target/employee-system-0.0.1-SNAPSHOT.jar
```

추가 옵션을 지정할 수도 있다:

Bash

```
# 프로파일 지정
java -jar -Dspring.profiles.active=production target/employee-system-0.0.1-SNAPSHOT.jar

# 포트 변경
java -jar -Dserver.port=9090 target/employee-system-0.0.1-SNAPSHOT.jar

# JVM 메모리 설정
java -Xms512m -Xmx1024m -jar target/employee-system-0.0.1-SNAPSHOT.jar
```

이렇게 생성된 JAR 파일은:

1. 독립적으로 실행 가능하다 (별도의 서버 설치 불필요)
2. 모든 의존성을 포함한다 (추가 라이브러리 설치 불필요)
3. 어떤 환경에서도 동일하게 동작한다 (개발, 테스트, 운영 환경의 일관성)

이거는 회사 프로젝트를 완벽히 패키징하여 어디서든 동일하게 실행할 수 있게 해주는 것이다.