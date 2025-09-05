# Chapter 07: 스프링 부트 시큐리티

## 1. 스프링 부트 시큐리티 퀵스타트

### 1.1 스프링 부트 시큐리티 적용하기

스프링 시큐리티는 인증과 권한 부여를 관리하는 강력한 보안 프레임워크다. 이거는 회사의 보안팀이 출입 통제와 정보 접근 권한을 관리하는 것과 같다.

의존성 추가:

XML

```
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-security</artifactId>
</dependency>
```

기본 보안 설정:

Java

```
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
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
                // 부서 관리 페이지는 MANAGER 권한 필요
                .requestMatchers("/department/**").hasRole("MANAGER")
                // 그 외 모든 요청은 인증 필요
                .anyRequest().authenticated()
            )
            .formLogin(form -> form
                .loginPage("/login")  // 커스텀 로그인 페이지
                .defaultSuccessUrl("/dashboard")  // 로그인 성공 시 이동할 페이지
                .failureUrl("/login?error=true")  // 로그인 실패 시 이동할 페이지
                .permitAll()
            )
            .logout(logout -> logout
                .logoutUrl("/logout")  // 로그아웃 URL
                .logoutSuccessUrl("/login?logout=true")  // 로그아웃 성공 시 이동할 페이지
                .invalidateHttpSession(true)  // 세션 무효화
                .permitAll()
            );
        
        return http.build();
    }
    
    // 테스트용 인메모리 사용자 설정
    @Bean
    public UserDetailsService userDetailsService() {
        UserDetails admin = User.builder()
            .username("admin")
            .password(passwordEncoder().encode("admin123"))
            .roles("ADMIN", "USER")
            .build();
            
        UserDetails manager = User.builder()
            .username("manager")
            .password(passwordEncoder().encode("manager123"))
            .roles("MANAGER", "USER")
            .build();
            
        UserDetails user = User.builder()
            .username("user")
            .password(passwordEncoder().encode("user123"))
            .roles("USER")
            .build();
            
        return new InMemoryUserDetailsManager(admin, manager, user);
    }
    
    @Bean
    public PasswordEncoder passwordEncoder() {
        return new BCryptPasswordEncoder();
    }
}
```

### 1.2 시큐리티 커스터마이징하기

로그인 성공/실패 핸들러:

Java

```
@Component
public class CustomAuthenticationSuccessHandler implements AuthenticationSuccessHandler {
    
    private final Logger logger = LoggerFactory.getLogger(getClass());
    
    @Override
    public void onAuthenticationSuccess(HttpServletRequest request, 
                                      HttpServletResponse response, 
                                      Authentication authentication) throws IOException {
        
        // 로그인 성공 로그 기록
        logger.info("사용자 로그인 성공: {}", authentication.getName());
        
        // 관리자는 관리자 페이지로, 일반 사용자는 대시보드로 이동
        if (authentication.getAuthorities().stream()
                .anyMatch(a -> a.getAuthority().equals("ROLE_ADMIN"))) {
            response.sendRedirect("/admin/dashboard");
        } else {
            response.sendRedirect("/dashboard");
        }
    }
}

@Component
public class CustomAuthenticationFailureHandler implements AuthenticationFailureHandler {
    
    private final Logger logger = LoggerFactory.getLogger(getClass());
    
    @Override
    public void onAuthenticationFailure(HttpServletRequest request, 
                                      HttpServletResponse response, 
                                      AuthenticationException exception) throws IOException {
        
        // 로그인 실패 로그 기록
        logger.warn("로그인 실패: {}, 원인: {}", 
                  request.getParameter("username"), 
                  exception.getMessage());
        
        // 오류 메시지 전달
        String errorMessage = "아이디 또는 비밀번호가 일치하지 않습니다.";
        response.sendRedirect("/login?error=true&message=" + URLEncoder.encode(errorMessage, "UTF-8"));
    }
}
```

보안 설정에 핸들러 적용:

Java

```
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    private final CustomAuthenticationSuccessHandler successHandler;
    private final CustomAuthenticationFailureHandler failureHandler;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        http
            .authorizeHttpRequests(...)
            .formLogin(form -> form
                .loginPage("/login")
                .successHandler(successHandler)  // 성공 핸들러 적용
                .failureHandler(failureHandler)  // 실패 핸들러 적용
                .permitAll()
            )
            .logout(...);
        
        return http.build();
    }
    
    // 기타 빈 설정...
}
```

## 2. 시큐리티 이해 및 데이터베이스 연동

### 2.1 스프링 시큐리티 동작 원리

스프링 시큐리티는 필터 체인을 통해 웹 요청을 가로채고 보안을 적용한다:

1. **SecurityContextPersistenceFilter**: 보안 컨텍스트 정보를 로드하고 저장한다
2. **LogoutFilter**: 로그아웃 요청을 처리한다
3. **UsernamePasswordAuthenticationFilter**: 사용자 인증을 처리한다
4. **FilterSecurityInterceptor**: 접근 제어 결정을 내린다

인증 과정:

1. 사용자가 로그인 정보를 제출한다
2. `AuthenticationManager`가 인증을 시도한다
3. `UserDetailsService`가 사용자 정보를 로드한다
4. `PasswordEncoder`가 비밀번호를 검증한다
5. 인증 성공 시 `SecurityContext`에 인증 정보가 저장된다

권한 부여 과정:

1. 사용자가 보호된 리소스에 접근을 시도한다
2. `AccessDecisionManager`가 접근 권한을 확인한다
3. 권한이 있으면 요청을 처리하고, 없으면 403 오류를 반환한다

### 2.2 JPA 연동하기

스프링 시큐리티와 JPA를 연동하여 데이터베이스에서 사용자 정보를 관리:

1. 사용자 및 권한 엔티티 정의:

Java

```
@Entity
@Table(name = "users")
public class User {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @Column(unique = true, nullable = false)
    private String username;
    
    @Column(nullable = false)
    private String password;
    
    private boolean enabled = true;
    
    @OneToMany(mappedBy = "user", cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private Set<Authority> authorities = new HashSet<>();
    
    // 생성자, Getter, Setter 등
}

@Entity
@Table(name = "authorities")
public class Authority {
    
    @Id
    @GeneratedValue
    private Long id;
    
    @ManyToOne
    @JoinColumn(name = "user_id", nullable = false)
    private User user;
    
    @Column(nullable = false)
    private String authority;
    
    // 생성자, Getter, Setter 등
}
```

2. 사용자 리포지토리 생성:

Java

```
public interface UserRepository extends JpaRepository<User, Long> {
    Optional<User> findByUsername(String username);
}
```

3. UserDetailsService 구현:

Java

```
@Service
public class CustomUserDetailsService implements UserDetailsService {
    
    private final UserRepository userRepository;
    
    @Override
    public UserDetails loadUserByUsername(String username) throws UsernameNotFoundException {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("사용자를 찾을 수 없습니다: " + username));
        
        return new org.springframework.security.core.userdetails.User(
            user.getUsername(),
            user.getPassword(),
            user.isEnabled(),
            true, // accountNonExpired
            true, // credentialsNonExpired
            true, // accountNonLocked
            getAuthorities(user.getAuthorities())
        );
    }
    
    private Collection<? extends GrantedAuthority> getAuthorities(Set<Authority> authorities) {
        return authorities.stream()
            .map(authority -> new SimpleGrantedAuthority(authority.getAuthority()))
            .collect(Collectors.toList());
    }
}
```

4. 보안 설정 업데이트:

Java

```
@Configuration
@EnableWebSecurity
public class SecurityConfig {
    
    private final CustomUserDetailsService userDetailsService;
    
    @Bean
    public SecurityFilterChain securityFilterChain(HttpSecurity http) throws Exception {
        // 기존 설정...
        
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

### 2.3 PasswordEncoder 사용하기

PasswordEncoder는 사용자 비밀번호를 안전하게 저장하고 검증하는 인터페이스다. 이거는 회사에서 중요 문서를 암호화하여 보관하는 것과 같다.

BCrypt 암호화 방식:

Java

```
@Service
public class UserService {
    
    private final UserRepository userRepository;
    private final PasswordEncoder passwordEncoder;
    
    // 사용자 등록
    public User registerUser(String username, String rawPassword, Set<String> roles) {
        // 비밀번호 암호화
        String encodedPassword = passwordEncoder.encode(rawPassword);
        
        // 사용자 생성
        User user = new User();
        user.setUsername(username);
        user.setPassword(encodedPassword);
        user.setEnabled(true);
        
        // 권한 설정
        for (String role : roles) {
            Authority authority = new Authority();
            authority.setUser(user);
            authority.setAuthority("ROLE_" + role.toUpperCase());
            user.getAuthorities().add(authority);
        }
        
        return userRepository.save(user);
    }
    
    // 비밀번호 변경
    public void changePassword(String username, String currentPassword, String newPassword) {
        User user = userRepository.findByUsername(username)
            .orElseThrow(() -> new UsernameNotFoundException("사용자를 찾을 수 없습니다."));
        
        // 현재 비밀번호 확인
        if (!passwordEncoder.matches(currentPassword, user.getPassword())) {
            throw new BadCredentialsException("현재 비밀번호가 일치하지 않습니다.");
        }
        
        // 새 비밀번호 암호화 및 저장
        user.setPassword(passwordEncoder.encode(newPassword));
        userRepository.save(user);
    }
}
```

비밀번호 변경 폼:

HTML

```
<!DOCTYPE html>
<html xmlns:th="http://www.thymeleaf.org">
<head>
    <title>비밀번호 변경</title>
</head>
<body>
    <h1>비밀번호 변경</h1>
    
    <div th:if="${error}" class="error" th:text="${error}">
        오류 메시지
    </div>
    
    <div th:if="${success}" class="success" th:text="${success}">
        성공 메시지
    </div>
    
    <form th:action="@{/change-password}" method="post">
        <div>
            <label for="currentPassword">현재 비밀번호</label>
            <input type="password" id="currentPassword" name="currentPassword" required />
        </div>
        
        <div>
            <label for="newPassword">새 비밀번호</label>
            <input type="password" id="newPassword" name="newPassword" required />
        </div>
        
        <div>
            <label for="confirmPassword">비밀번호 확인</label>
            <input type="password" id="confirmPassword" name="confirmPassword" required />
        </div>
        
        <button type="submit">변경</button>
    </form>
</body>
</html>
```