# Chapter 06: 스프링 부트 화면 개발

## 1. 화면 개발

### 1.1 웹 애플리케이션 화면 개발하기

스프링 부트에서는 다양한 템플릿 엔진을 사용해 웹 화면을 개발할 수 있다. 이거는 회사의 인트라넷이나 고객용 웹사이트를 만드는 것과 같다.

기본 컨트롤러 구조:

Java

```
@Controller  // 이 클래스가 웹 요청을 처리하는 컨트롤러임을 나타낸다
public class EmployeeController {
    
    private final EmployeeService employeeService;
    
    // 생성자 주입
    public EmployeeController(EmployeeService employeeService) {
        this.employeeService = employeeService;
    }
    
    // 직원 목록 페이지
    @GetMapping("/employees")
    public String listEmployees(Model model) {
        List<Employee> employees = employeeService.findAll();
        model.addAttribute("employees", employees);  // 모델에 데이터 추가
        
        return "employee/list";  // 뷰 이름 반환 (templates/employee/list.html)
    }
    
    // 직원 상세 페이지
    @GetMapping("/employee/{id}")
    public String viewEmployee(@PathVariable Long id, Model model) {
        Employee employee = employeeService.findById(id);
        model.addAttribute("employee", employee);
        
        return "employee/view";
    }
    
    // 직원 등록 폼
    @GetMapping("/employee/new")
    public String newEmployeeForm(Model model) {
        model.addAttribute("employee", new Employee());
        model.addAttribute("departments", departmentService.findAll());
        
        return "employee/form";
    }
    
    // 직원 등록 처리
    @PostMapping("/employee")
    public String createEmployee(@Valid Employee employee, BindingResult result) {
        if (result.hasErrors()) {
            return "employee/form";  // 유효성 검사 실패 시 폼으로 돌아감
        }
        
        employeeService.save(employee);
        return "redirect:/employees";  // 성공 시 목록 페이지로 리다이렉트
    }
    
    // 직원 수정 폼
    @GetMapping("/employee/{id}/edit")
    public String editEmployeeForm(@PathVariable Long id, Model model) {
        Employee employee = employeeService.findById(id);
        model.addAttribute("employee", employee);
        model.addAttribute("departments", departmentService.findAll());
        
        return "employee/form";
    }
    
    // 직원 수정 처리
    @PostMapping("/employee/{id}")
    public String updateEmployee(@PathVariable Long id, @Valid Employee employee, BindingResult result) {
        if (result.hasErrors()) {
            return "employee/form";
        }
        
        employee.setId(id);  // ID 설정
        employeeService.save(employee);  // 저장 (update)
        
        return "redirect:/employee/" + id;  // 상세 페이지로 리다이렉트
    }
    
    // 직원 삭제
    @GetMapping("/employee/{id}/delete")
    public String deleteEmployee(@PathVariable Long id) {
        employeeService.delete(id);
        return "redirect:/employees";
    }
}
```

### 1.2 데이터베이스 연동하기

웹 애플리케이션에서 데이터베이스 연동은 다음과 같은 계층 구조로 이루어진다:

1. **Controller**: 웹 요청을 처리하고 뷰로 데이터를 전달한다
2. **Service**: 비즈니스 로직을 처리한다
3. **Repository**: 데이터베이스에 접근한다
4. **Entity**: 데이터베이스 테이블과 매핑되는 객체다

Java

```
// 1. Entity
@Entity
public class Employee {
    @Id
    @GeneratedValue
    private Long id;
    private String name;
    private String department;
    // ...
}

// 2. Repository
public interface EmployeeRepository extends JpaRepository<Employee, Long> {
    List<Employee> findByDepartment(String department);
}

// 3. Service
@Service
public class EmployeeService {
    private final EmployeeRepository repository;
    
    // 비즈니스 로직
    public List<Employee> findByDepartment(String department) {
        return repository.findByDepartment(department);
    }
}

// 4. Controller
@Controller
public class EmployeeController {
    private final EmployeeService service;
    
    @GetMapping("/department/{dept}/employees")
    public String listByDepartment(@PathVariable String dept, Model model) {
        model.addAttribute("employees", service.findByDepartment(dept));
        return "employee/department-list";
    }
}
```

이 구조는 관심사 분리(Separation of Concerns)를 통해 코드를 유지보수하기 쉽게 만든다.

### 1.3 게시판 구현하기

간단한 게시판 기능을 구현해보자:

Java

```
// 게시글 엔티티
@Entity
public class Board {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(nullable = false)
    private String title;
    
    @Lob  // 대용량 텍스트
    private String content;
    
    @ManyToOne
    @JoinColumn(name = "writer_id")
    private Employee writer;
    
    private LocalDateTime createdDate;
    
    private LocalDateTime modifiedDate;
    
    private int viewCount;
    
    // 생성자, Getter, Setter 등
}

// 게시판 리포지토리
public interface BoardRepository extends JpaRepository<Board, Long> {
    
    List<Board> findByTitleContaining(String keyword);
    
    List<Board> findByWriter(Employee writer);
    
    @Query("SELECT b FROM Board b ORDER BY b.createdDate DESC")
    List<Board> findLatestBoards(Pageable pageable);
}

// 게시판 서비스
@Service
public class BoardService {
    
    private final BoardRepository boardRepository;
    private final EmployeeRepository employeeRepository;
    
    // 게시글 목록 조회
    public Page<Board> findAll(Pageable pageable) {
        return boardRepository.findAll(pageable);
    }
    
    // 게시글 상세 조회 (조회수 증가)
    public Board findById(Long id) {
        Board board = boardRepository.findById(id)
            .orElseThrow(() -> new BoardNotFoundException(id));
        
        // 조회수 증가
        board.setViewCount(board.getViewCount() + 1);
        return boardRepository.save(board);
    }
    
    // 게시글 저장
    public Board save(Board board, Long writerId) {
        Employee writer = employeeRepository.findById(writerId)
            .orElseThrow(() -> new EmployeeNotFoundException(writerId));
        
        board.setWriter(writer);
        board.setCreatedDate(LocalDateTime.now());
        board.setModifiedDate(LocalDateTime.now());
        
        return boardRepository.save(board);
    }
    
    // 게시글 수정
    public Board update(Long id, Board updatedBoard) {
        Board board = boardRepository.findById(id)
            .orElseThrow(() -> new BoardNotFoundException(id));
        
        // 제목과 내용만 업데이트
        board.setTitle(updatedBoard.getTitle());
        board.setContent(updatedBoard.getContent());
        board.setModifiedDate(LocalDateTime.now());
        
        return boardRepository.save(board);
    }
    
    // 게시글 삭제
    public void delete(Long id) {
        boardRepository.deleteById(id);
    }
    
    // 키워드로 검색
    public List<Board> search(String keyword) {
        return boardRepository.findByTitleContaining(keyword);
    }
}

// 게시판 컨트롤러
@Controller
@RequestMapping("/boards")
public class BoardController {
    
    private final BoardService boardService;
    
    // 게시글 목록
    @GetMapping
    public String list(Model model, 
                      @RequestParam(defaultValue = "0") int page, 
                      @RequestParam(defaultValue = "10") int size) {
        
        Pageable pageable = PageRequest.of(page, size, Sort.by("createdDate").descending());
        Page<Board> boards = boardService.findAll(pageable);
        
        model.addAttribute("boards", boards);
        
        return "board/list";
    }
    
    // 게시글 상세
    @GetMapping("/{id}")
    public String view(@PathVariable Long id, Model model) {
        Board board = boardService.findById(id);
        model.addAttribute("board", board);
        
        return "board/view";
    }
    
    // 게시글 작성 폼
    @GetMapping("/new")
    public String newBoardForm(Model model) {
        model.addAttribute("board", new Board());
        
        return "board/form";
    }
    
    // 게시글 저장
    @PostMapping
    public String create(@Valid Board board, BindingResult result, 
                        @RequestParam Long writerId) {
        
        if (result.hasErrors()) {
            return "board/form";
        }
        
        Board savedBoard = boardService.save(board, writerId);
        
        return "redirect:/boards/" + savedBoard.getId();
    }
    
    // 게시글 수정 폼
    @GetMapping("/{id}/edit")
    public String editForm(@PathVariable Long id, Model model) {
        Board board = boardService.findById(id);
        model.addAttribute("board", board);
        
        return "board/form";
    }
    
    // 게시글 수정
    @PostMapping("/{id}")
    public String update(@PathVariable Long id, @Valid Board board, BindingResult result) {
        if (result.hasErrors()) {
            return "board/form";
        }
        
        boardService.update(id, board);
        
        return "redirect:/boards/" + id;
    }
    
    // 게시글 삭제
    @GetMapping("/{id}/delete")
    public String delete(@PathVariable Long id) {
        boardService.delete(id);
        
        return "redirect:/boards";
    }
    
    // 게시글 검색
    @GetMapping("/search")
    public String search(@RequestParam String keyword, Model model) {
        List<Board> boards = boardService.search(keyword);
        model.addAttribute("boards", boards);
        model.addAttribute("keyword", keyword);
        
        return "board/search-result";
    }
}
```

## 2. 타임리프 적용

### 2.1 타임리프 퀵스타트

타임리프(Thymeleaf)는 스프링 부트에서 권장하는 템플릿 엔진이다. 이거는 회사 문서에 실시간으로 데이터를 채워넣는 시스템과 같다.

타임리프 의존성 추가:

XML

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-thymeleaf</artifactId>
</dependency>
```

기본 타임리프 사용법:

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>직원 목록</title>
</head>
<body>
    <h1>직원 목록</h1>
    
    <!-- 조건문 -->
    <div th:if="${employees.isEmpty()}">
        <p>등록된 직원이 없습니다.</p>
    </div>
    
    <!-- 반복문 -->
    <table th:unless="${employees.isEmpty()}">
        <thead>
            <tr>
                <th>사번</th>
                <th>이름</th>
                <th>부서</th>
                <th>직급</th>
                <th>관리</th>
            </tr>
        </thead>
        <tbody>
            <!-- 반복 처리 -->
            <tr th:each="employee : ${employees}">
                <td th:text="${employee.id}">1001</td>
                <td th:text="${employee.name}">홍길동</td>
                <td th:text="${employee.department}">개발팀</td>
                <td th:text="${employee.position}">사원</td>
                <td>
                    <!-- URL 생성 -->
                    <a th:href="@{/employee/{id}(id=${employee.id})}">상세</a>
                    <a th:href="@{/employee/{id}/edit(id=${employee.id})}">수정</a>
                    <a th:href="@{/employee/{id}/delete(id=${employee.id})}"
                       onclick="return confirm('정말 삭제하시겠습니까?')">삭제</a>
                </td>
            </tr>
        </tbody>
    </table>
    
    <!-- 링크 -->
    <a th:href="@{/employee/new}" class="btn">새 직원 등록</a>
</body>
</html>
```

### 2.2 타임리프로 게시판 프로그램 개발하기

타임리프를 이용한 게시판 화면 개발:

1. 게시글 목록 화면 (board/list.html):

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>게시판</title>
</head>
<body>
    <h1>게시판</h1>
    
    <!-- 검색 폼 -->
    <form th:action="@{/boards/search}" method="get">
        <input type="text" name="keyword" placeholder="제목 검색..." />
        <button type="submit">검색</button>
    </form>
    
    <!-- 게시글 목록 -->
    <table>
        <thead>
            <tr>
                <th>번호</th>
                <th>제목</th>
                <th>작성자</th>
                <th>작성일</th>
                <th>조회수</th>
            </tr>
        </thead>
        <tbody>
            <tr th:each="board : ${boards}">
                <td th:text="${board.id}">1</td>
                <td>
                    <a th:href="@{/boards/{id}(id=${board.id})}" th:text="${board.title}">게시글 제목</a>
                </td>
                <td th:text="${board.writer.name}">작성자명</td>
                <td th:text="${#temporals.format(board.createdDate, 'yyyy-MM-dd')}">2023-01-01</td>
                <td th:text="${board.viewCount}">0</td>
            </tr>
        </tbody>
    </table>
    
    <!-- 페이지네이션 -->
    <div th:if="${boards.totalPages > 0}">
        <ul class="pagination">
            <li th:class="${boards.first ? 'disabled' : ''}">
                <a th:if="${!boards.first}" th:href="@{/boards(page=0)}">처음</a>
                <span th:if="${boards.first}">처음</span>
            </li>
            
            <li th:each="pageNum : ${#numbers.sequence(0, boards.totalPages - 1)}"
                th:class="${pageNum == boards.number ? 'active' : ''}">
                <a th:if="${pageNum != boards.number}" 
                   th:href="@{/boards(page=${pageNum})}" 
                   th:text="${pageNum + 1}">1</a>
                <span th:if="${pageNum == boards.number}" th:text="${pageNum + 1}">1</span>
            </li>
            
            <li th:class="${boards.last ? 'disabled' : ''}">
                <a th:if="${!boards.last}" 
                   th:href="@{/boards(page=${boards.totalPages - 1})}">마지막</a>
                <span th:if="${boards.last}">마지막</span>
            </li>
        </ul>
    </div>
    
    <!-- 글쓰기 버튼 -->
    <a th:href="@{/boards/new}" class="btn">글쓰기</a>
</body>
</html>
```

2. 게시글 상세 화면 (board/view.html):

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${board.title}">게시글 제목</title>
</head>
<body>
    <h1 th:text="${board.title}">게시글 제목</h1>
    
    <div class="info">
        <p>
            작성자: <span th:text="${board.writer.name}">작성자명</span> | 
            작성일: <span th:text="${#temporals.format(board.createdDate, 'yyyy-MM-dd HH:mm')}">2023-01-01</span> | 
            조회수: <span th:text="${board.viewCount}">0</span>
        </p>
    </div>
    
    <div class="content">
        <!-- pre 태그로 줄바꿈 유지 -->
        <pre th:text="${board.content}">게시글 내용</pre>
    </div>
    
    <div class="actions">
        <a th:href="@{/boards}" class="btn">목록</a>
        <a th:href="@{/boards/{id}/edit(id=${board.id})}" class="btn">수정</a>
        <a th:href="@{/boards/{id}/delete(id=${board.id})}" 
           class="btn"
           onclick="return confirm('정말 삭제하시겠습니까?')">삭제</a>
    </div>
</body>
</html>
```

3. 게시글 폼 화면 (board/form.html):

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title th:text="${board.id == null ? '게시글 작성' : '게시글 수정'}">게시글 작성</title>
</head>
<body>
    <h1 th:text="${board.id == null ? '게시글 작성' : '게시글 수정'}">게시글 작성</h1>
    
    <!-- 폼 제출 경로 분기 -->
    <form th:action="${board.id == null ? @{/boards} : @{/boards/{id}(id=${board.id})}}"
          th:object="${board}"
          method="post">
        
        <!-- 숨겨진 필드 (수정 시 사용) -->
        <input type="hidden" th:if="${board.id != null}" th:field="*{id}" />
        
        <div>
            <label for="title">제목</label>
            <input type="text" id="title" th:field="*{title}" required />
            <span th:if="${#fields.hasErrors('title')}" th:errors="*{title}">제목 오류</span>
        </div>
        
        <div>
            <label for="content">내용</label>
            <textarea id="content" th:field="*{content}" rows="10" required></textarea>
            <span th:if="${#fields.hasErrors('content')}" th:errors="*{content}">내용 오류</span>
        </div>
        
        <!-- 신규 작성 시에만 작성자 선택 -->
        <div th:if="${board.id == null}">
            <label for="writerId">작성자</label>
            <select id="writerId" name="writerId" required>
                <option value="">작성자 선택</option>
                <option th:each="employee : ${employees}"
                        th:value="${employee.id}"
                        th:text="${employee.name}">작성자명</option>
            </select>
        </div>
        
        <div class="actions">
            <button type="submit" th:text="${board.id == null ? '등록' : '수정'}">등록</button>
            <a th:href="@{/boards}" class="btn">취소</a>
        </div>
    </form>
</body>
</html>
```

## 3. 사용자 인증과 예외처리

### 3.1 로그인 인증 처리하기

간단한 로그인 기능 구현:

Java

```
// 로그인 컨트롤러
@Controller
public class LoginController {
    
    private final EmployeeService employeeService;
    
    @GetMapping("/login")
    public String loginForm() {
        return "login";
    }
    
    @PostMapping("/login")
    public String login(@RequestParam String username,
                       @RequestParam String password,
                       HttpSession session,
                       RedirectAttributes redirectAttributes) {
        
        try {
            // 사용자 인증
            Employee employee = employeeService.authenticate(username, password);
            
            // 세션에 사용자 정보 저장
            session.setAttribute("loggedInUser", employee);
            
            // 로그인 후 이동할 페이지
            return "redirect:/dashboard";
            
        } catch (AuthenticationException e) {
            // 인증 실패 시 오류 메시지 전달
            redirectAttributes.addFlashAttribute("error", "아이디 또는 비밀번호가 일치하지 않습니다.");
            return "redirect:/login";
        }
    }
    
    @GetMapping("/logout")
    public String logout(HttpSession session) {
        // 세션 무효화
        session.invalidate();
        return "redirect:/login";
    }
}

// 인증 관련 서비스
@Service
public class EmployeeService {
    
    private final EmployeeRepository employeeRepository;
    private final PasswordEncoder passwordEncoder;
    
    // 사용자 인증 메소드
    public Employee authenticate(String username, String password) {
        Employee employee = employeeRepository.findByUsername(username)
            .orElseThrow(() -> new AuthenticationException("사용자를 찾을 수 없습니다."));
        
        // 비밀번호 검증
        if (!passwordEncoder.matches(password, employee.getPassword())) {
            throw new AuthenticationException("비밀번호가 일치하지 않습니다.");
        }
        
        return employee;
    }
    
    // 비밀번호 변경 메소드
    public void changePassword(Long employeeId, String currentPassword, String newPassword) {
        Employee employee = findById(employeeId);
        
        // 현재 비밀번호 확인
        if (!passwordEncoder.matches(currentPassword, employee.getPassword())) {
            throw new AuthenticationException("현재 비밀번호가 일치하지 않습니다.");
        }
        
        // 새 비밀번호 암호화 후 저장
        employee.setPassword(passwordEncoder.encode(newPassword));
        employeeRepository.save(employee);
    }
}
```

로그인 페이지 (login.html):

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>로그인</title>
</head>
<body>
    <h1>로그인</h1>
    
    <!-- 오류 메시지 표시 -->
    <div th:if="${error}" class="error" th:text="${error}">
        오류 메시지
    </div>
    
    <form th:action="@{/login}" method="post">
        <div>
            <label for="username">아이디</label>
            <input type="text" id="username" name="username" required />
        </div>
        
        <div>
            <label for="password">비밀번호</label>
            <input type="password" id="password" name="password" required />
        </div>
        
        <button type="submit">로그인</button>
    </form>
</body>
</html>
```

### 3.2 예외 처리

전역 예외 처리기를 사용한 예외 처리:

Java

```
// 사용자 정의 예외 클래스
public class EmployeeNotFoundException extends RuntimeException {
    public EmployeeNotFoundException(Long id) {
        super("직원을 찾을 수 없습니다. ID: " + id);
    }
}

public class BoardNotFoundException extends RuntimeException {
    public BoardNotFoundException(Long id) {
        super("게시글을 찾을 수 없습니다. ID: " + id);
    }
}

public class AuthenticationException extends RuntimeException {
    public AuthenticationException(String message) {
        super(message);
    }
}

// 전역 예외 처리기
@ControllerAdvice
public class GlobalExceptionHandler {
    
    private final Logger logger = LoggerFactory.getLogger(GlobalExceptionHandler.class);
    
    // 리소스를 찾을 수 없는 경우
    @ExceptionHandler({EmployeeNotFoundException.class, BoardNotFoundException.class})
    public String handleNotFoundException(Exception e, Model model) {
        logger.error("리소스를 찾을 수 없습니다: {}", e.getMessage());
        
        model.addAttribute("errorMessage", e.getMessage());
        return "error/not-found";  // 404 오류 페이지
    }
    
    // 인증 예외
    @ExceptionHandler(AuthenticationException.class)
    public String handleAuthenticationException(AuthenticationException e, 
                                              RedirectAttributes redirectAttributes) {
        logger.error("인증 오류: {}", e.getMessage());
        
        redirectAttributes.addFlashAttribute("error", e.getMessage());
        return "redirect:/login";  // 로그인 페이지로 리다이렉트
    }
    
    // 권한 부족 예외
    @ExceptionHandler(AccessDeniedException.class)
    public String handleAccessDeniedException(AccessDeniedException e, Model model) {
        logger.error("접근 권한 없음: {}", e.getMessage());
        
        model.addAttribute("errorMessage", "이 작업을 수행할 권한이 없습니다.");
        return "error/forbidden";  // 403 오류 페이지
    }
    
    // 기타 모든 예외
    @ExceptionHandler(Exception.class)
    public String handleException(Exception e, Model model) {
        logger.error("예상치 못한 오류 발생: {}", e.getMessage(), e);
        
        model.addAttribute("errorMessage", "요청을 처리하는 중 오류가 발생했습니다.");
        return "error/server-error";  // 500 오류 페이지
    }
}
```

오류 페이지 (error/not-found.html):

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>리소스를 찾을 수 없음</title>
</head>
<body>
    <h1>요청하신 페이지를 찾을 수 없습니다</h1>
    
    <p th:text="${errorMessage}">오류 메시지</p>
    
    <a th:href="@{/}">홈으로 돌아가기</a>
</body>
</html>
```