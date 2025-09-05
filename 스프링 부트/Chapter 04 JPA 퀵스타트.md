# Chapter 04: JPA 퀵스타트

## 1. 스프링과 JPA

### 1.1 JPA 개념 이해하기

JPA(Java Persistence API)는 자바에서 데이터베이스를 쉽게 사용할 수 있게 해주는 기술이다. 이거는 회사에서 복잡한 엑셀 작업 대신 사용하기 쉬운 업무 관리 시스템을 사용하는 것과 같다.

JPA의 핵심 개념:

- **객체-관계 매핑(ORM)**: 자바 객체와 데이터베이스 테이블을 연결한다
- **엔티티(Entity)**: 데이터베이스 테이블과 매핑되는 자바 클래스다
- **영속성 컨텍스트**: 엔티티를 관리하는 환경이다
- **트랜잭션**: 데이터베이스 작업의 논리적 단위다

JPA를 사용하면:

- SQL 쿼리를 직접 작성하지 않아도 된다
- 데이터베이스 벤더(MySQL, Oracle 등)가 바뀌어도 코드를 수정할 필요가 없다
- 객체 지향적인 코드를 유지할 수 있다

### 1.2 JPA 퀵스타트

간단한 JPA 엔티티 클래스:

Java

```
@Entity  // 이 클래스는 데이터베이스 테이블과 매핑된다
@Table(name = "employees")  // 테이블 이름 지정
public class Employee {
    
    @Id  // 기본 키(Primary Key) 지정
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // 자동 증가 설정
    private Long id;  // 사번
    
    @Column(nullable = false)  // NOT NULL 제약조건
    private String name;  // 이름
    
    @Column(name = "dept_name")  // 컬럼 이름 지정
    private String department;  // 부서
    
    private String position;  // 직급
    
    @Temporal(TemporalType.DATE)  // 날짜 타입 지정
    private Date joinDate;  // 입사일
    
    @Transient  // 이 필드는 데이터베이스에 저장되지 않는다
    private int serviceYears;  // 근속년수
    
    // 생성자, Getter, Setter 등
}
```

이 코드는:

1. `Employee` 클래스를 데이터베이스의 "employees" 테이블과 매핑한다
2. `id` 필드를 기본 키로 지정하고, 자동 증가하도록 설정한다
3. `name` 필드는 NULL이 될 수 없게 한다
4. `department` 필드는 "dept_name" 컬럼과 매핑한다
5. `joinDate` 필드는 날짜 타입으로 저장한다
6. `serviceYears` 필드는 데이터베이스에 저장하지 않는다

## 2. JPA 설정

### 2.1 영속성 유닛 설정

스프링 부트에서는 application.properties 파일로 JPA 설정을 관리한다:

properties

```
# 데이터소스 설정
spring.datasource.url=jdbc:mysql://localhost:3306/employeedb
spring.datasource.username=admin
spring.datasource.password=password
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# JPA 설정
spring.jpa.hibernate.ddl-auto=update  # 테이블 자동 생성/수정
spring.jpa.show-sql=true  # SQL 로깅 활성화
spring.jpa.properties.hibernate.format_sql=true  # SQL 포맷팅
spring.jpa.database-platform=org.hibernate.dialect.MySQL8Dialect  # 데이터베이스 방언
```

`spring.jpa.hibernate.ddl-auto` 옵션:

- **create**: 애플리케이션 시작 시 테이블을 삭제하고 다시 생성한다
- **create-drop**: create와 같지만 애플리케이션 종료 시 테이블을 삭제한다
- **update**: 엔티티와 테이블을 비교해 필요한 변경사항만 적용한다 (개발 환경에 적합)
- **validate**: 엔티티와 테이블이 일치하는지만 확인한다 (운영 환경에 적합)
- **none**: 아무 작업도 하지 않는다

### 2.2 엔티티 매핑 설정하기

다양한 엔티티 매핑 방법:

Java

```
@Entity
public class Department {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;  // 부서명
    
    @OneToMany(mappedBy = "department")  // 1:N 관계 설정
    private List<Employee> employees = new ArrayList<>();
}

@Entity
public class Project {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;  // 프로젝트명
    
    private LocalDate startDate;  // 시작일
    
    private LocalDate endDate;  // 종료일
    
    @Enumerated(EnumType.STRING)  // 열거형 타입 매핑
    private ProjectStatus status;  // 프로젝트 상태
}

// 열거형 타입
public enum ProjectStatus {
    PLANNING, IN_PROGRESS, COMPLETED, ON_HOLD
}
```

### 2.3 식별자 값 자동 증가시키기

기본 키(Primary Key)의 자동 생성 전략:

Java

```
@Entity
public class Employee {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)  // MySQL의 AUTO_INCREMENT
    private Long id;
    
    // 다른 필드들...
}

@Entity
public class Department {
    
    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE)  // 시퀀스 사용 (Oracle)
    private Long id;
    
    // 다른 필드들...
}

@Entity
public class Project {
    
    @Id
    @GeneratedValue(strategy = GenerationType.TABLE)  // 테이블을 사용한 키 생성
    private Long id;
    
    // 다른 필드들...
}

@Entity
public class Task {
    
    @Id
    @GeneratedValue(strategy = GenerationType.AUTO)  // 데이터베이스에 맞게 자동 선택
    private Long id;
    
    // 다른 필드들...
}
```

이러한 전략들은 데이터베이스 종류에 따라 선택할 수 있다:

- **IDENTITY**: MySQL, SQL Server의 자동 증가 기능
- **SEQUENCE**: Oracle, PostgreSQL의 시퀀스 기능
- **TABLE**: 키 생성 전용 테이블을 사용
- **AUTO**: 데이터베이스에 맞는 방식을 자동 선택

## 3. JPA API 이해

### 3.1 EntityManagerFactory와 EntityManager 이해하기

JPA의 핵심 API는 EntityManager다. 이거는 회사에서 데이터베이스 관리자와 같은 역할을 한다.

스프링 부트에서 EntityManager 사용하기:

Java

```
@Service
@Transactional  // 트랜잭션 관리를 위한 어노테이션
public class EmployeeService {
    
    @PersistenceContext  // EntityManager 주입
    private EntityManager entityManager;
    
    public void save(Employee employee) {
        // 저장
        entityManager.persist(employee);
    }
    
    public Employee find(Long id) {
        // 조회
        return entityManager.find(Employee.class, id);
    }
    
    public void update(Employee employee) {
        // 업데이트 (merge는 준영속 상태의 엔티티를 영속 상태로 변경)
        entityManager.merge(employee);
    }
    
    public void delete(Long id) {
        // 삭제
        Employee employee = find(id);
        entityManager.remove(employee);
    }
    
    public List<Employee> findByDepartment(String department) {
        // JPQL 쿼리 사용
        return entityManager.createQuery(
            "SELECT e FROM Employee e WHERE e.department = :dept", Employee.class)
            .setParameter("dept", department)
            .getResultList();
    }
}
```

EntityManager의 주요 메소드:

- **persist()**: 엔티티를 영속화(저장)한다
- **find()**: 기본 키로 엔티티를 조회한다
- **merge()**: 준영속 상태의 엔티티를 영속 상태로 병합한다
- **remove()**: 엔티티를 삭제한다
- **createQuery()**: JPQL 쿼리를 생성한다

### 3.2 영속성 컨텍스트와 엔티티 상태

영속성 컨텍스트는 엔티티를 관리하는 환경이다. 이거는 회사의 문서 관리 시스템과 같다.

엔티티의 생명주기:

1. **비영속(New/Transient)**: 영속성 컨텍스트와 관련이 없는 상태
2. **영속(Managed)**: 영속성 컨텍스트에 저장된 상태
3. **준영속(Detached)**: 영속성 컨텍스트에 저장되었다가 분리된 상태
4. **삭제(Removed)**: 삭제된 상태

Java

```
// 비영속 상태
Employee employee = new Employee("김사원", "개발팀");

// 영속 상태
entityManager.persist(employee);

// 준영속 상태
entityManager.detach(employee);

// 영속 상태로 다시 병합
employee = entityManager.merge(employee);

// 삭제 상태
entityManager.remove(employee);
```

### 3.3 영속성 컨텍스트와 SQL 저장소 이해하기

영속성 컨텍스트의 주요 특징:

1. **1차 캐시**: 한 번 조회한 엔티티는 메모리에 캐시된다
2. **동일성 보장**: 같은 엔티티를 여러 번 조회해도 같은 객체를 반환한다
3. **트랜잭션 쓰기 지연**: SQL을 바로 실행하지 않고 모아서 실행한다
4. **변경 감지(Dirty Checking)**: 엔티티의 변경을 자동으로 감지한다
5. **지연 로딩(Lazy Loading)**: 연관 엔티티를 실제 사용할 때 로딩한다

Java

```
@Transactional
public void updateEmployeeSalary(Long id, int newSalary) {
    // 영속성 컨텍스트에 로드
    Employee employee = entityManager.find(Employee.class, id);
    
    // 엔티티 수정 - SQL이 즉시 실행되지 않음
    employee.setSalary(newSalary);
    
    // 트랜잭션 종료 시 변경 감지가 동작하여 UPDATE SQL이 실행됨
    // entityManager.flush() 호출하지 않아도 자동으로 처리됨
}
```

이 코드에서:

1. `find()` 메소드로 엔티티를 조회하면 영속성 컨텍스트에 캐시된다
2. `setSalary()` 메소드로 엔티티를 수정해도 즉시 SQL이 실행되지 않는다
3. 트랜잭션이 종료될 때 변경 감지가 동작하여 필요한 UPDATE SQL이 실행된다
4. 이 모든 과정이 자동으로 처리되므로 개발자는 객체지향적인 코드에 집중할 수 있다