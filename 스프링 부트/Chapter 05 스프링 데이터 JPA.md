# Chapter 05: 스프링 데이터 JPA

## 1. 스프링 데이터 JPA 퀵스타트

### 1.1 스프링 데이터 JPA 사용하기

스프링 데이터 JPA는 JPA를 더 쉽게 사용할 수 있게 해주는 도구다. 이거는 회사에서 복잡한 데이터 처리 작업을 자동화된 시스템으로 대체하는 것과 같다.

기본 리포지토리 인터페이스:

Java

```
// 인터페이스만 정의하면 구현체는 스프링이 자동으로 만들어준다
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    // 기본 CRUD 메소드가 자동으로 제공된다:
    // - save(entity): 저장 및 수정
    // - findById(id): ID로 조회
    // - findAll(): 전체 조회
    // - delete(entity): 삭제
    // - count(): 개수 조회
}
```

이 인터페이스 하나만으로 기본적인 CRUD(Create, Read, Update, Delete) 기능을 모두 사용할 수 있다.

사용 예시:

Java

```
@Service
public class EmployeeService {
    
    private final EmployeeRepository employeeRepository;
    
    // 생성자 주입
    public EmployeeService(EmployeeRepository employeeRepository) {
        this.employeeRepository = employeeRepository;
    }
    
    // 모든 직원 조회
    public List<Employee> findAllEmployees() {
        return employeeRepository.findAll();
    }
    
    // ID로 직원 조회
    public Employee findEmployee(Long id) {
        return employeeRepository.findById(id)
            .orElseThrow(() -> new EmployeeNotFoundException(id));
    }
    
    // 직원 저장 (신규 등록 또는 수정)
    public Employee saveEmployee(Employee employee) {
        return employeeRepository.save(employee);
    }
    
    // 직원 삭제
    public void deleteEmployee(Long id) {
        employeeRepository.deleteById(id);
    }
}
```

### 1.2 쿼리 메소드 사용하기

스프링 데이터 JPA는 메소드 이름만으로 쿼리를 생성해준다. 이거는 회사에서 "영업팀 직원 목록 보고서 작성"이라는 요청만으로 보고서가 자동으로 생성되는 것과 같다.

쿼리 메소드 예시:

Java

```
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    // 부서로 직원 찾기
    List<Employee> findByDepartment(String department);
    
    // 이름에 특정 키워드가 포함된 직원 찾기
    List<Employee> findByNameContaining(String keyword);
    
    // 특정 급여 이상 받는 직원 찾기
    List<Employee> findBySalaryGreaterThanEqual(Integer minSalary);
    
    // 부서와 직급으로 찾기
    List<Employee> findByDepartmentAndPosition(String department, String position);
    
    // 입사일 기준으로 찾기
    List<Employee> findByJoinDateBetween(Date start, Date end);
    
    // 부서별로 정렬해서 찾기
    List<Employee> findByDepartmentOrderByNameAsc(String department);
    
    // 특정 조건의 직원이 존재하는지 확인
    boolean existsByDepartmentAndPosition(String department, String position);
    
    // 특정 부서의 직원 수 카운트
    long countByDepartment(String department);
    
    // 특정 조건의 첫 번째 직원만 찾기
    Employee findFirstByDepartmentOrderBySalaryDesc(String department);
    
    // 특정 조건의 상위 3명만 찾기
    List<Employee> findTop3ByOrderBySalaryDesc();
}
```

이러한 메소드 이름 규칙을 사용하면 SQL을 직접 작성하지 않고도 다양한 쿼리를 실행할 수 있다.

### 1.3 @Query 어노테이션 사용하기

복잡한 쿼리는 @Query 어노테이션으로 직접 작성할 수 있다. 이거는 회사에서 표준 보고서 양식으로는 표현하기 어려운 특별한 보고서를 직접 작성하는 것과 같다.

Java

```
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    // JPQL 사용
    @Query("SELECT e FROM Employee e WHERE e.department = :dept AND e.salary > :minSalary")
    List<Employee> findHighPaidEmployees(
        @Param("dept") String department, 
        @Param("minSalary") Integer minSalary
    );
    
    // 네이티브 SQL 사용
    @Query(value = "SELECT * FROM employees WHERE YEAR(join_date) = :year", 
           nativeQuery = true)
    List<Employee> findEmployeesJoinedInYear(@Param("year") int year);
    
    // 특정 필드만 선택 (프로젝션)
    @Query("SELECT e.id, e.name, e.department FROM Employee e WHERE e.position = :position")
    List<Object[]> findEmployeeInfoByPosition(@Param("position") String position);
    
    // DTO로 결과 매핑
    @Query("SELECT new com.mycompany.dto.EmployeeSummary(e.id, e.name, e.department) " +
           "FROM Employee e WHERE e.position = :position")
    List<EmployeeSummary> findEmployeeSummaryByPosition(@Param("position") String position);
    
    // 데이터 수정 쿼리
    @Modifying
    @Query("UPDATE Employee e SET e.salary = e.salary * 1.1 WHERE e.department = :dept")
    int giveRaiseToDepartment(@Param("dept") String department);
    
    // 데이터 삭제 쿼리
    @Modifying
    @Query("DELETE FROM Employee e WHERE e.department = :dept")
    int deleteEmployeesByDepartment(@Param("dept") String department);
}
```

@Modifying 어노테이션은 수정이나 삭제와 같이 데이터를 변경하는 쿼리에 사용된다.

### 1.4 QueryDSL을 이용한 동적 쿼리 적용하기

QueryDSL은 타입 안전한 방식으로 동적 쿼리를 작성할 수 있게 해주는 도구다. 이거는 회사에서 상황에 따라 다양한 조건으로 보고서를 생성할 수 있는 유연한 시스템과 같다.

QueryDSL 설정:

Java

```
// QueryDSL 설정
public interface EmployeeRepository extends JpaRepository<Employee, Long>, 
                                           QuerydslPredicateExecutor<Employee> {
    // 기존 메소드들...
}
```

QueryDSL 사용 예시:

Java

```
@Service
public class EmployeeSearchService {
    
    private final EmployeeRepository employeeRepository;
    
    // 동적 쿼리로 직원 검색
    public List<Employee> searchEmployees(EmployeeSearchCondition condition) {
        QEmployee employee = QEmployee.employee;  // QueryDSL 엔티티
        
        // 동적 조건 생성
        BooleanBuilder builder = new BooleanBuilder();
        
        // 부서 조건이 있으면 추가
        if (condition.getDepartment() != null) {
            builder.and(employee.department.eq(condition.getDepartment()));
        }
        
        // 직급 조건이 있으면 추가
        if (condition.getPosition() != null) {
            builder.and(employee.position.eq(condition.getPosition()));
        }
        
        // 최소 급여 조건이 있으면 추가
        if (condition.getMinSalary() != null) {
            builder.and(employee.salary.goe(condition.getMinSalary()));
        }
        
        // 최대 급여 조건이 있으면 추가
        if (condition.getMaxSalary() != null) {
            builder.and(employee.salary.loe(condition.getMaxSalary()));
        }
        
        // 이름 검색 조건이 있으면 추가
        if (condition.getNameKeyword() != null) {
            builder.and(employee.name.contains(condition.getNameKeyword()));
        }
        
        // 조건에 맞는 직원 조회
        return (List<Employee>) employeeRepository.findAll(builder);
    }
}
```

이렇게 하면 사용자의 검색 조건에 따라 동적으로 쿼리를 생성할 수 있다.

## 2. 연관관계 매핑

### 2.1 단방향 연관관계 설정하기

단방향 연관관계는 한쪽 엔티티만 다른 엔티티를 참조하는 관계다. 이거는 회사에서 직원이 자신의 부서를 알지만, 부서는 소속 직원을 모르는 것과 같다.

Java

```
@Entity
public class Employee {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 다대일(N:1) 관계: 여러 직원이 하나의 부서에 속한다
    @ManyToOne
    @JoinColumn(name = "department_id")  // 외래 키 컬럼 이름 지정
    private Department department;
    
    // 다대일(N:1) 관계: 여러 직원이 한 명의 관리자를 가질 수 있다
    @ManyToOne
    @JoinColumn(name = "manager_id")
    private Employee manager;
    
    // 일대일(1:1) 관계: 직원 한 명은 하나의 컴퓨터를 가진다
    @OneToOne
    @JoinColumn(name = "computer_id")
    private Computer computer;
    
    // 다대다(N:M) 관계: 직원은 여러 프로젝트에 참여할 수 있다
    @ManyToMany
    @JoinTable(
        name = "employee_project",  // 연결 테이블 이름
        joinColumns = @JoinColumn(name = "employee_id"),  // 현재 엔티티를 참조하는 외래 키
        inverseJoinColumns = @JoinColumn(name = "project_id")  // 상대 엔티티를 참조하는 외래 키
    )
    private Set<Project> projects = new HashSet<>();
    
    // 생성자, Getter, Setter 등
}

@Entity
public class Department {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 생성자, Getter, Setter 등
}
```

### 2.2 양방향 연관관계 매핑하기

양방향 연관관계는 양쪽 엔티티가 서로를 참조하는 관계다. 이거는 회사에서 직원이 자신의 부서를 알고, 부서도 소속 직원 목록을 아는 것과 같다.

Java

```
@Entity
public class Department {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    // 일대다(1:N) 관계: 하나의 부서에 여러 직원이 속한다
    @OneToMany(mappedBy = "department")  // Employee 클래스의 department 필드와 매핑
    private List<Employee> employees = new ArrayList<>();
    
    // 생성자, Getter, Setter 등
}

@Entity
public class Project {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    
    private LocalDate startDate;
    
    private LocalDate endDate;
    
    // 다대다(N:M) 관계: 프로젝트는 여러 직원이 참여한다
    @ManyToMany(mappedBy = "projects")  // Employee 클래스의 projects 필드와 매핑
    private Set<Employee> members = new HashSet<>();
    
    // 생성자, Getter, Setter 등
}
```

양방향 관계에서 주의할 점:

1. 연관관계의 주인은 외래 키가 있는 쪽이다 (보통 @ManyToOne 쪽)
2. 주인이 아닌 쪽은 mappedBy 속성으로 주인을 지정한다
3. 주인만이 외래 키 값을 변경할 수 있다
4. 양쪽 모두 참조를 설정해야 한다

Java

```
// 올바른 양방향 연관관계 설정
public void assignEmployeeToDepartment(Employee employee, Department department) {
    // 양쪽 모두 설정
    employee.setDepartment(department);  // 연관관계의 주인 쪽 설정
    department.getEmployees().add(employee);  // 주인이 아닌 쪽 설정
}
```

연관관계 편의 메소드를 사용하면 더 쉽게 관리할 수 있다:

Java

```
@Entity
public class Employee {
    // ...
    
    // 연관관계 편의 메소드
    public void setDepartment(Department department) {
        // 기존 부서에서 제거
        if (this.department != null) {
            this.department.getEmployees().remove(this);
        }
        
        this.department = department;
        
        // 새 부서에 추가
        if (department != null) {
            department.getEmployees().add(this);
        }
    }
}
```

이렇게 하면 한 쪽만 설정해도 양쪽 관계가 올바르게 유지된다.