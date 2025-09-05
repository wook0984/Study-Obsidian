# Chapter 08: 웹 애플리케이션 통합

## 1. 비즈니스 레이어 개발

### 1.1 비즈니스 컴포넌트 구조 이해하기

비즈니스 레이어는 애플리케이션의 핵심 로직을 담당하는 계층이다. 이거는 회사의 핵심 업무 프로세스를 관리하는 부서와 같다.

일반적인 계층 구조:

1. **컨트롤러(Controller)**: 사용자 요청을 처리하고 응답을 반환한다
2. **서비스(Service)**: 비즈니스 로직을 처리한다
3. **리포지토리(Repository)**: 데이터 접근을 담당한다
4. **도메인(Domain)**: 비즈니스 개체와 규칙을 표현한다

각 계층의 역할:

- **컨트롤러**: 사용자 입력 검증, URL 매핑, 뷰 선택
- **서비스**: 트랜잭션 관리, 비즈니스 규칙 적용, 여러 리포지토리 조합
- **리포지토리**: 데이터 CRUD 작업, 쿼리 최적화
- **도메인**: 비즈니스 상태와 행동 정의

### 1.2 비즈니스 컴포넌트 개발하기

직원 관리 시스템의 비즈니스 로직:

1. **도메인 모델**:

Java

```
@Entity
public class Employee {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    private String email;
    private String position;
    
    @ManyToOne
    @JoinColumn(name = "department_id")
    private Department department;
    
    private LocalDate joinDate;
    private Integer salary;
    
    // 비즈니스 메소드
    public boolean isEligibleForPromotion() {
        // 승진 자격 조건 (예: 3년 이상 근무)
        return ChronoUnit.YEARS.between(joinDate, LocalDate.now()) >= 3;
    }
    
    public void applyRaise(double percentage) {
        this.salary = (int)(this.salary * (1 + percentage / 100));
    }
    
    // 생성자, Getter, Setter 등
}

@Entity
public class Department {
    
    @Id
    @GeneratedValue
    private Long id;
    
    private String name;
    private String location;
    
    @OneToMany(mappedBy = "department")
    private List<Employee> employees = new ArrayList<>();
    
    // 비즈니스 메소드
    public int getTotalEmployees() {
        return employees.size();
    }
    
    public int getTotalSalary() {
        return employees.stream()
            .mapToInt(Employee::getSalary)
            .sum();
    }
    
    // 생성자, Getter, Setter 등
}
```

2. **리포지토리**:

Java

```
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    
    List<Employee> findByDepartment(Department department);
    
    List<Employee> findByPositionAndSalaryGreaterThan(String position, Integer minSalary);
    
    @Query("SELECT e FROM Employee e WHERE YEAR(e.joinDate) = :year")
    List<Employee> findByJoinYear(@Param("year") int year);
}

public interface DepartmentRepository extends JpaRepository<Department, Long> {
    
    Optional<Department> findByName(String name);
    
    @Query("SELECT d FROM Department d LEFT JOIN FETCH d.employees")
    List<Department> findAllWithEmployees();
}
```

3. **서비스**:

Java

```
@Service
@Transactional
public class EmployeeService {
    
    private final EmployeeRepository employeeRepository;
    private final DepartmentRepository departmentRepository;
    private final EmailService emailService;
    private final SalaryCalculator salaryCalculator;
    
    // 직원 등록
    public Employee registerEmployee(EmployeeDTO dto) {
        // 부서 조회
        Department department = departmentRepository.findByName(dto.getDepartmentName())
            .orElseThrow(() -> new DepartmentNotFoundException(dto.getDepartmentName()));
        
        // 초기 급여 계산
        int initialSalary = salaryCalculator.calculateInitialSalary(
            dto.getPosition(), dto.getExperience());
        
        // 직원 생성
        Employee employee = new Employee();
        employee.setName(dto.getName());
        employee.setEmail(dto.getEmail());
        employee.setPosition(dto.getPosition());
        employee.setDepartment(department);
        employee.setJoinDate(LocalDate.now());
        employee.setSalary(initialSalary);
        
        // 저장
        Employee savedEmployee = employeeRepository.save(employee);
        
        // 환영 이메일 발송
        emailService.sendWelcomeEmail(savedEmployee);
        
        return savedEmployee;
    }
    
    // 부서 이동
    public void transferEmployee(Long employeeId, String newDepartmentName) {
        // 직원 조회
        Employee employee = employeeRepository.findById(employeeId)
            .orElseThrow(() -> new EmployeeNotFoundException(employeeId));
        
        // 새 부서 조회
        Department newDepartment = departmentRepository.findByName(newDepartmentName)
            .orElseThrow(() -> new DepartmentNotFoundException(newDepartmentName));
        
        // 기존 부서
        Department oldDepartment = employee.getDepartment();
        
        // 부서 변경
        employee.setDepartment(newDepartment);
        employeeRepository.save(employee);
        
        // 이동 알림 발송
        emailService.sendTransferNotification(employee, oldDepartment, newDepartment);
    }
    
    // 연봉 인상
    public void applyAnnualRaise(String departmentName, double percentage) {
        // 부서 조회
        Department department = departmentRepository.findByName(departmentName)
            .orElseThrow(() -> new DepartmentNotFoundException(departmentName));
        
        // 해당 부서 직원 조회
        List<Employee> employees = employeeRepository.findByDepartment(department);
        
        // 연봉 인상 적용
        for (Employee employee : employees) {
            employee.applyRaise(percentage);
        }
        
        // 일괄 저장
        employeeRepository.saveAll(employees);
        
        // 알림 발송
        emailService.sendSalaryRaiseNotification(employees, percentage);
    }
    
    // 승진 대상자 조회
    public List<Employee> findPromotionCandidates() {
        List<Employee> allEmployees = employeeRepository.findAll();
        
        return allEmployees.stream()
            .filter(Employee::isEligibleForPromotion)
            .collect(Collectors.toList());
    }
    
    // 기타 비즈니스 메소드...
}

@Service
@Transactional
public class DepartmentService {
    
    private final DepartmentRepository departmentRepository;
    
    // 부서 생성
    public Department createDepartment(String name, String location) {
        // 이미 존재하는 부서인지 확인
        if (departmentRepository.findByName(name).isPresent()) {
            throw new DepartmentAlreadyExistsException(name);
        }
        
        // 부서 생성
        Department department = new Department();
        department.setName(name);
        department.setLocation(location);
        
        return departmentRepository.save(department);
    }
    
    // 부서별 인건비 분석
    public List<DepartmentCostDTO> analyzeDepartmentCosts() {
        List<Department> departments = departmentRepository.findAllWithEmployees();
        
        return departments.stream()
            .map(department -> {
                DepartmentCostDTO dto = new DepartmentCostDTO();
                dto.setDepartmentName(department.getName());
                dto.setEmployeeCount(department.getTotalEmployees());
                dto.setTotalSalary(department.getTotalSalary());
                dto.setAverageSalary(department.getTotalSalary() / Math.max(1, department.getTotalEmployees()));
                return dto;
            })
            .collect(Collectors.toList());
    }
    
    // 기타 비즈니스 메소드...
}
```

## 2. 프레젠테이션 레이어 개발

### 2.1 프레젠테이션 개발 준비하기

프레젠테이션 레이어는 사용자 인터페이스와 상호작용을 담당한다. 이거는 회사의 고객 접점 부서(고객 센터, 영업부)와 같다.

DTO(Data Transfer Object) 클래스:

Java

```
// 직원 등록 DTO
public class EmployeeRegistrationDTO {
    
    @NotBlank(message = "이름은 필수입니다.")
    private String name;
    
    @NotBlank(message = "이메일은 필수입니다.")
    @Email(message = "유효한 이메일 형식이 아닙니다.")
    private String email;
    
    @NotBlank(message = "직급은 필수입니다.")
    private String position;
    
    @NotBlank(message = "부서는 필수입니다.")
    private String departmentName;
    
    @Min(value = 0, message = "경력은 0년 이상이어야 합니다.")
    private int experience;
    
    // Getter, Setter 등
}

// 직원 정보 DTO
public class EmployeeInfoDTO {
    
    private Long id;
    private String name;
    private String email;
    private String position;
    private String departmentName;
    private LocalDate joinDate;
    private int yearsOfService;
    private Integer salary;
    
    // 엔티티에서 DTO로 변환하는 정적 메소드
    public static EmployeeInfoDTO fromEntity(Employee employee) {
        EmployeeInfoDTO dto = new EmployeeInfoDTO();
        dto.setId(employee.getId());
        dto.setName(employee.getName());
        dto.setEmail(employee.getEmail());
        dto.setPosition(employee.getPosition());
        dto.setDepartmentName(employee.getDepartment().getName());
        dto.setJoinDate(employee.getJoinDate());
        dto.setYearsOfService((int) ChronoUnit.YEARS.between(employee.getJoinDate(), LocalDate.now()));
        dto.setSalary(employee.getSalary());
        return dto;
    }
    
    // Getter, Setter 등
}
```

컨트롤러 기본 구조:

Java

```
@Controller
@RequestMapping("/employees")
public class EmployeeController {
    
    private final EmployeeService employeeService;
    private final DepartmentService departmentService;
    
    // 생성자 주입
    public EmployeeController(EmployeeService employeeService, DepartmentService departmentService) {
        this.employeeService = employeeService;
        this.departmentService = departmentService;
    }
    
    // 직원 목록 페이지
    @GetMapping
    public String listEmployees(Model model) {
        List<Employee> employees = employeeService.findAll();
        
        // 엔티티에서 DTO로 변환
        List<EmployeeInfoDTO> employeeDTOs = employees.stream()
            .map(EmployeeInfoDTO::fromEntity)
            .collect(Collectors.toList());
        
        model.addAttribute("employees", employeeDTOs);
        
        return "employee/list";
    }
    
    // 나머지 핸들러 메소드...
}
```

### 2.2 게시판 기능 구현하기

공지사항 게시판 구현:

1. **엔티티 클래스**:

Java

```
@Entity
public class Notice {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @NotBlank
    private String title;
    
    @Lob
    private String content;
    
    @ManyToOne
    @JoinColumn(name = "author_id")
    private Employee author;
    
    private LocalDateTime createdAt;
    
    private boolean important;
    
    // 생성자, Getter, Setter 등
}
```

2. **리포지토리**:

Java

```
public interface NoticeRepository extends JpaRepository<Notice, Long> {
    
    List<Notice> findByImportantOrderByCreatedAtDesc(boolean important);
    
    List<Notice> findByTitleContaining(String keyword);
    
    @Query("SELECT n FROM Notice n ORDER BY n.important DESC, n.createdAt DESC")
    List<Notice> findAllOrderByImportantAndCreatedAt();
}
```

3. **서비스**:

Java

```
@Service
@Transactional
public class NoticeService {
    
    private final NoticeRepository noticeRepository;
    private final EmployeeRepository employeeRepository;
    
    // 공지사항 등록
    public Notice createNotice(NoticeDTO dto, Long authorId) {
        Employee author = employeeRepository.findById(authorId)
            .orElseThrow(() -> new EmployeeNotFoundException(authorId));
        
        Notice notice = new Notice();
        notice.setTitle(dto.getTitle());
        notice.setContent(dto.getContent());
        notice.setAuthor(author);
        notice.setCreatedAt(LocalDateTime.now());
        notice.setImportant(dto.isImportant());
        
        return noticeRepository.save(notice);
    }
    
    // 공지사항 목록 조회
    public List<Notice> findAllNotices() {
        return noticeRepository.findAllOrderByImportantAndCreatedAt();
    }
    
    // 중요 공지사항만 조회
    public List<Notice> findImportantNotices() {
        return noticeRepository.findByImportantOrderByCreatedAtDesc(true);
    }
    
    // 공지사항 상세 조회
    public Notice findById(Long id) {
        return noticeRepository.findById(id)
            .orElseThrow(() -> new NoticeNotFoundException(id));
    }
    
    // 공지사항 수정
    public Notice updateNotice(Long id, NoticeDTO dto) {
        Notice notice = findById(id);
        
        notice.setTitle(dto.getTitle());
        notice.setContent(dto.getContent());
        notice.setImportant(dto.isImportant());
        
        return noticeRepository.save(notice);
    }
    
    // 공지사항 삭제
    public void deleteNotice(Long id) {
        noticeRepository.deleteById(id);
    }
    
    // 공지사항 검색
    public List<Notice> searchNotices(String keyword) {
        return noticeRepository.findByTitleContaining(keyword);
    }
}
```

4. **컨트롤러**:

Java

```
@Controller
@RequestMapping("/notices")
public class NoticeController {
    
    private final NoticeService noticeService;
    
    // 공지사항 목록
    @GetMapping
    public String listNotices(Model model) {
        List<Notice> notices = noticeService.findAllNotices();
        model.addAttribute("notices", notices);
        
        return "notice/list";
    }
    
    // 공지사항 상세
    @GetMapping("/{id}")
    public String viewNotice(@PathVariable Long id, Model model) {
        Notice notice = noticeService.findById(id);
        model.addAttribute("notice", notice);
        
        return "notice/view";
    }
    
    // 공지사항 작성 폼
    @GetMapping("/new")
    public String newNoticeForm(Model model) {
        model.addAttribute("notice", new NoticeDTO());
        
        return "notice/form";
    }
    
    // 공지사항 등록
    @PostMapping
    public String createNotice(@Valid NoticeDTO notice, BindingResult result,
                              @AuthenticationPrincipal UserDetails userDetails) {
        
        if (result.hasErrors()) {
            return "notice/form";
        }
        
        // 현재 로그인한 사용자 정보 가져오기
        Employee author = employeeService.findByUsername(userDetails.getUsername());
        
        noticeService.createNotice(notice, author.getId());
        
        return "redirect:/notices";
    }
    
    // 공지사항 수정 폼
    @GetMapping("/{id}/edit")
    public String editNoticeForm(@PathVariable Long id, Model model) {
        Notice notice = noticeService.findById(id);
        
        // 엔티티를 DTO로 변환
        NoticeDTO dto = new NoticeDTO();
        dto.setTitle(notice.getTitle());
        dto.setContent(notice.getContent());
        dto.setImportant(notice.isImportant());
        
        model.addAttribute("notice", dto);
        model.addAttribute("noticeId", id);
        
        return "notice/form";
    }
    
    // 공지사항 수정
    @PostMapping("/{id}")
    public String updateNotice(@PathVariable Long id, 
                              @Valid NoticeDTO notice, 
                              BindingResult result) {
        
        if (result.hasErrors()) {
            return "notice/form";
        }
        
        noticeService.updateNotice(id, notice);
        
        return "redirect:/notices/" + id;
    }
    
    // 공지사항 삭제
    @GetMapping("/{id}/delete")
    public String deleteNotice(@PathVariable Long id) {
        noticeService.deleteNotice(id);
        
        return "redirect:/notices";
    }
    
    // 공지사항 검색
    @GetMapping("/search")
    public String searchNotices(@RequestParam String keyword, Model model) {
        List<Notice> notices = noticeService.searchNotices(keyword);
        
        model.addAttribute("notices", notices);
        model.addAttribute("keyword", keyword);
        
        return "notice/search-result";
    }
}
```

5. **타임리프 템플릿**:

HTML

```
<!-- notice/list.html -->
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>공지사항</title>
</head>
<body>
    <h1>공지사항</h1>
    
    <div class="search">
        <form th:action="@{/notices/search}" method="get">
            <input type="text" name="keyword" placeholder="제목으로 검색..." />
            <button type="submit">검색</button>
        </form>
    </div>
    
    <table>
        <thead>
            <tr>
                <th>번호</th>
                <th>제목</th>
                <th>작성자</th>
                <th>작성일</th>
            </tr>
        </thead>
        <tbody>
            <tr th:each="notice : ${notices}">
                <td th:text="${notice.id}">1</td>
                <td>
                    <!-- 중요 공지는 강조 표시 -->
                    <span th:if="${notice.important}" class="important">[중요]</span>
                    <a th:href="@{/notices/{id}(id=${notice.id})}" th:text="${notice.title}">공지사항 제목</a>
                </td>
                <td th:text="${notice.author.name}">작성자명</td>
                <td th:text="${#temporals.format(notice.createdAt, 'yyyy-MM-dd')}">2023-01-01</td>
            </tr>
        </tbody>
    </table>
    
    <!-- 글쓰기 버튼 (관리자만 표시) -->
    <div sec:authorize="hasRole('ADMIN')">
        <a th:href="@{/notices/new}" class="btn">공지사항 작성</a>
    </div>
</body>
</html>
```

### 2.3 시큐리티 적용하기

스프링 시큐리티 설정:

Java

```
@Configuration
@EnableWebSecurity
@EnableMethodSecurity  // 메소드 수준 보안 활성화
public class SecurityConfig {
    
    private final CustomUserDetailsService userDetailsService;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(authorize -> authorize
                // 정적 리소스는 누구나 접근 가능
                .requestMatchers("/css/**", "/js/**", "/images/**").permitAll()
                // 로그인, 회원가입 페이지는 누구나 접근 가능
                .requestMatchers("/login", "/register").permitAll()
                // 관리자 페이지는 ADMIN 권한 필요
                .requestMatchers("/admin/**").hasRole("ADMIN")
                // 공지사항 작성/수정/삭제는 ADMIN 권한 필요
                .requestMatchers("/notices/new", "/notices/*/edit", "/notices/*/delete").hasRole("ADMIN")
                // 부서 관리 페이지는 MANAGER 권한 필요
                .requestMatchers("/departments/**").hasRole("MANAGER")
                // 그 외 모든 요청은 인증 필요
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")
                .defaultSuccessUrl("/dashboard")
                .failureUrl("/login?error=true")
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")
                .logoutSuccessUrl("/login?logout=true")
                .invalidateHttpSession(true)
                .permitAll()
            )
            .exceptionHandling(exceptions -> exceptions
                .accessDeniedPage("/access-denied")  // 접근 거부 시 페이지
            );
        
        return http.build();
    }
    
    @Bean
    public DaoAuthenticationProvider authenticationProvider() {
        DaoAuthenticationProvider provider = new DaoAuthenticationProvider();
        provider.setUserDetailsService(userDetailsService);
        provider.setPasswordEncoder(passwordEncoder());
        return provider;
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

메소드 수준 보안:

Java

```
@Service
public class EmployeeService {
    
    // 관리자만 접근 가능
    @PreAuthorize("hasRole('ADMIN')")
    public void deleteEmployee(Long id) {
        employeeRepository.deleteById(id);
    }
    
    // 관리자 또는 해당 직원만 접근 가능
    @PreAuthorize("hasRole('ADMIN') or @securityUtil.isCurrentUser(#id)")
    public EmployeeDTO updateEmployee(Long id, EmployeeDTO dto) {
        // 업데이트 로직
    }
    
    // 부서 관리자 또는 관리자만 접근 가능
    @PreAuthorize("hasRole('ADMIN') or @securityUtil.isDepartmentManager(#departmentId)")
    public List<EmployeeDTO> findEmployeesByDepartment(Long departmentId) {
        // 조회 로직
    }
}
```

Thymeleaf 보안 통합:

HTML

```
<!-- 로그인한 사용자만 볼 수 있는 콘텐츠 -->
<div sec:authorize="isAuthenticated()">
    <p>환영합니다, <span sec:authentication="name">사용자</span>님!</p>
</div>

<!-- 특정 권한을 가진 사용자만 볼 수 있는 콘텐츠 -->
<div sec:authorize="hasRole('ADMIN')">
    <a th:href="@{/admin/dashboard}">관리자 페이지</a>
</div>

<!-- 현재 사용자가 특정 직원인 경우에만 표시 -->
<div sec:authorize="@securityUtil.isCurrentUser(${employee.id})">
    <a th:href="@{/employee/{id}/edit(id=${employee.id})}">내 정보 수정</a>
</div>
```

### 2.4 기타 기능 추가하기

1. **파일 업로드**:

Java

```
@Controller
@RequestMapping("/documents")
public class DocumentController {
    
    private final DocumentService documentService;
    
    @PostMapping("/upload")
    public String uploadDocument(@RequestParam("file") MultipartFile file,
                                @RequestParam("description") String description,
                                RedirectAttributes redirectAttributes) {
        
        try {
            Document document = documentService.storeDocument(file, description);
            redirectAttributes.addFlashAttribute("message", "파일이 성공적으로 업로드되었습니다: " + file.getOriginalFilename());
            return "redirect:/documents";
        } catch (IOException e) {
            redirectAttributes.addFlashAttribute("error", "파일 업로드 중 오류가 발생했습니다: " + e.getMessage());
            return "redirect:/documents";
        }
    }
    
    @GetMapping("/download/{id}")
    public ResponseEntity<Resource> downloadDocument(@PathVariable Long id) {
        Document document = documentService.getDocument(id);
        Resource resource = documentService.loadAsResource(document.getStoredFileName());
        
        return ResponseEntity.ok()
            .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + document.getOriginalFileName() + "\"")
            .body(resource);
    }
}
```

2. **엑셀 다운로드**:

Java

```
@Controller
@RequestMapping("/reports")
public class ReportController {
    
    private final ReportService reportService;
    
    @GetMapping("/employees/excel")
    public ResponseEntity<Resource> downloadEmployeeReport() {
        try {
            String filename = "직원목록_" + LocalDate.now() + ".xlsx";
            Resource resource = reportService.generateEmployeeExcelReport();
            
            return ResponseEntity.ok()
                .header(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename=\"" + filename + "\"")
                .contentType(MediaType.parseMediaType("application/vnd.openxmlformats-officedocument.spreadsheetml.sheet"))
                .body(resource);
        } catch (Exception e) {
            // 오류 처리
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).build();
        }
    }
}
```

3. **대시보드**:

Java

```
@Controller
public class DashboardController {
    
    private final DepartmentService departmentService;
    private final EmployeeService employeeService;
    private final NoticeService noticeService;
    
    @GetMapping("/dashboard")
    public String dashboard(Model model) {
        // 부서별 인원 통계
        List<DepartmentStatsDTO> departmentStats = departmentService.getDepartmentStatistics();
        model.addAttribute("departmentStats", departmentStats);
        
        // 최근 입사자
        List<EmployeeDTO> recentEmployees = employeeService.findRecentEmployees(5);
        model.addAttribute("recentEmployees", recentEmployees);
        
        // 중요 공지사항
        List<Notice> importantNotices = noticeService.findImportantNotices();
        model.addAttribute("importantNotices", importantNotices);
        
        // 이번 달 생일자
        List<EmployeeDTO> birthdayEmployees = employeeService.findEmployeesWithBirthdayThisMonth();
        model.addAttribute("birthdayEmployees", birthdayEmployees);
        
        return "dashboard";
    }
}
```

4. **알림 시스템**:

Java

```
@RestController
@RequestMapping("/api/notifications")
public class NotificationController {
    
    private final NotificationService notificationService;
    
    @GetMapping
    public List<NotificationDTO> getNotifications(@AuthenticationPrincipal UserDetails userDetails) {
        return notificationService.getUnreadNotifications(userDetails.getUsername());
    }
    
    @PostMapping("/{id}/read")
    public ResponseEntity<Void> markAsRead(@PathVariable Long id) {
        notificationService.markAsRead(id);
        return ResponseEntity.ok().build();
    }
}
```

이러한 기능들을 통합하여 완전한 웹 애플리케이션을 구축할 수 있다.