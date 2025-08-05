1장 들어가기  
  
2장 객체 생성과 파괴  
## 아이템 1. 생성자 대신 정적 팩터리 메서드를 고려하라

정적 팩터리 메서드는 클래스의 인스턴스를 반환하는 정적 메서드입니다.

**장점:**

1. **이름을 가질 수 있다**: 생성자와 달리 메서드명으로 반환될 객체의 특성을 설명할 수 있습니다.

Java

```
// 생성자
BigInteger(int, int, Random)  // 무엇을 하는지 불명확

// 정적 팩터리 메서드
BigInteger.probablePrime(int, Random)  // 소수를 반환함이 명확
```

2. **호출될 때마다 인스턴스를 새로 생성하지 않아도 된다**: 불변 클래스는 인스턴스를 캐싱하여 재활용할 수 있습니다.

Java

```
Boolean.valueOf(boolean b) {
    return b ? Boolean.TRUE : Boolean.FALSE;  // 미리 생성된 인스턴스 반환
}
```

3. **반환 타입의 하위 타입 객체를 반환할 수 있다**: 유연성이 높아집니다.

Java

```
public static <E> List<E> of(E e1, E e2) {
    return new ImmutableList<>(e1, e2);  // 구현체는 숨기고 인터페이스만 공개
}
```

4. **입력 매개변수에 따라 매번 다른 클래스의 객체를 반환할 수 있다**
5. **정적 팩터리 메서드를 작성하는 시점에는 반환할 객체의 클래스가 존재하지 않아도 된다**

**단점:**

- 상속을 하려면 public이나 protected 생성자가 필요한데, 정적 팩터리 메서드만 제공하면 하위 클래스를 만들 수 없다
- 정적 팩터리 메서드는 프로그래머가 찾기 어렵다

**흔히 사용하는 명명 방식:**

- `from`: 매개변수 하나로 인스턴스 생성
- `of`: 여러 매개변수로 인스턴스 생성
- `valueOf`: from과 of의 더 자세한 버전
- `instance` 또는 `getInstance`: 인스턴스 반환
- `create` 또는 `newInstance`: 새로운 인스턴스 생성
- `getType`: 다른 타입의 인스턴스 반환
- `newType`: 다른 타입의 새로운 인스턴스 생성

## 아이템 2. 생성자에 매개변수가 많다면 빌더를 고려하라

매개변수가 많을 때 점층적 생성자 패턴이나 자바빈즈 패턴 대신 빌더 패턴을 사용합니다.

Java

```
public class NutritionFacts {
    private final int servingSize;
    private final int servings;
    private final int calories;
    private final int fat;
    private final int sodium;
    
    public static class Builder {
        // 필수 매개변수
        private final int servingSize;
        private final int servings;
        
        // 선택 매개변수 - 기본값으로 초기화
        private int calories = 0;
        private int fat = 0;
        private int sodium = 0;
        
        public Builder(int servingSize, int servings) {
            this.servingSize = servingSize;
            this.servings = servings;
        }
        
        public Builder calories(int val) {
            calories = val;
            return this;
        }
        
        public Builder fat(int val) {
            fat = val;
            return this;
        }
        
        public Builder sodium(int val) {
            sodium = val;
            return this;
        }
        
        public NutritionFacts build() {
            return new NutritionFacts(this);
        }
    }
    
    private NutritionFacts(Builder builder) {
        servingSize = builder.servingSize;
        servings = builder.servings;
        calories = builder.calories;
        fat = builder.fat;
        sodium = builder.sodium;
    }
}

// 사용 예
NutritionFacts cocaCola = new NutritionFacts.Builder(240, 8)
    .calories(100)
    .sodium(35)
    .build();
```

**장점:**

- 가독성이 좋다
- 불변 객체를 만들 수 있다
- 계층적으로 설계된 클래스와 함께 쓰기 좋다
- 가변인수 매개변수를 여러 개 사용할 수 있다

## 아이템 3. private 생성자나 열거 타입으로 싱글턴임을 보증하라

싱글톤을 만드는 방법은 세 가지가 있습니다:

1. **public static final 필드 방식**

Java

```
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
}
```

2. **정적 팩터리 방식**

Java

```
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    public static Elvis getInstance() { return INSTANCE; }
}
```

3. **열거 타입 방식 (가장 바람직함)**

Java

```
public enum Elvis {
    INSTANCE;
    
    public void leaveTheBuilding() { ... }
}
```

열거 타입 방식이 가장 간결하고, 직렬화 상황이나 리플렉션 공격에도 안전합니다.

## 아이템 4. 인스턴스화를 막으려거든 private 생성자를 사용하라

정적 메서드와 정적 필드만을 담은 유틸리티 클래스는 인스턴스화를 막아야 합니다.

Java

```
public class UtilityClass {
    // 인스턴스화 방지
    private UtilityClass() {
        throw new AssertionError();
    }
    
    public static String processString(String s) { ... }
}
```

## 아이템 5. 자원을 직접 명시하지 말고 의존 객체 주입을 사용하라

사용하는 자원에 따라 동작이 달라지는 클래스는 정적 유틸리티 클래스나 싱글턴 방식이 적합하지 않습니다.

Java

```
// 나쁜 예 - 유연하지 않고 테스트하기 어렵다
public class SpellChecker {
    private static final Lexicon dictionary = new KoreanDictionary();
    
    private SpellChecker() {} // 객체 생성 방지
    
    public static boolean isValid(String word) { ... }
}

// 좋은 예 - 의존 객체 주입
public class SpellChecker {
    private final Lexicon dictionary;
    
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }
    
    public boolean isValid(String word) { ... }
}
```

## 아이템 6. 불필요한 객체 생성을 피하라

같은 기능의 객체를 매번 생성하기보다는 객체 하나를 재사용하는 편이 좋습니다.

Java

```
// 나쁜 예 - 호출할 때마다 String 인스턴스 생성
String s = new String("bikini");

// 좋은 예 - 하나의 인스턴스 재사용
String s = "bikini";

// 나쁜 예 - 비싼 객체를 반복 생성
static boolean isRomanNumeral(String s) {
    return s.matches("^(?=.)M*(C[MD]|D?C{0,3})"
            + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");
}

// 좋은 예 - Pattern 인스턴스 캐싱
private static final Pattern ROMAN = Pattern.compile(
    "^(?=.)M*(C[MD]|D?C{0,3})"
    + "(X[CL]|L?X{0,3})(I[XV]|V?I{0,3})$");

static boolean isRomanNumeral(String s) {
    return ROMAN.matcher(s).matches();
}
```

## 아이템 7. 다 쓴 객체 참조를 해제하라

메모리 누수의 주범들:

1. **자기 메모리를 직접 관리하는 클래스**

Java

```
public class Stack {
    private Object[] elements;
    private int size = 0;
    
    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        Object result = elements[--size];
        elements[size] = null; // 다 쓴 참조 해제
        return result;
    }
}
```

2. **캐시**: WeakHashMap을 활용하거나 주기적으로 청소
3. **리스너와 콜백**: 약한 참조(weak reference)로 저장

## 아이템 8. finalizer와 cleaner 사용을 피하라

finalizer와 cleaner는 예측할 수 없고, 상황에 따라 위험할 수 있으니 기본적으로 사용하지 말아야 합니다. 대신 AutoCloseable을 구현하고 try-with-resources를 사용하세요.

Java

```
public class Room implements AutoCloseable {
    private static final Cleaner cleaner = Cleaner.create();
    
    // 청소가 필요한 자원. 절대 Room을 참조해서는 안 된다!
    private static class State implements Runnable {
        int numJunkPiles; // 방 안의 쓰레기 수
        
        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }
        
        @Override public void run() {
            System.out.println("방 청소");
            numJunkPiles = 0;
        }
    }
    
    private final State state;
    private final Cleaner.Cleanable cleanable;
    
    public Room(int numJunkPiles) {
        state = new State(numJunkPiles);
        cleanable = cleaner.register(this, state);
    }
    
    @Override public void close() {
        cleanable.clean();
    }
}
```

## 아이템 9. try-finally보다는 try-with-resources를 사용하라

자원을 회수할 때는 try-with-resources를 사용하세요. 코드가 더 짧고 분명하며, 예외 정보도 훨씬 유용합니다.

Java

```
// 나쁜 예 - try-finally
static String firstLineOfFile(String path) throws IOException {
    BufferedReader br = new BufferedReader(new FileReader(path));
    try {
        return br.readLine();
    } finally {
        br.close();
    }
}

// 좋은 예 - try-with-resources
static String firstLineOfFile(String path) throws IOException {
    try (BufferedReader br = new BufferedReader(new FileReader(path))) {
        return br.readLine();
    }
}

// 복수의 자원을 처리하는 경우
static void copy(String src, String dst) throws IOException {
    try (InputStream in = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {
        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
}
```
  
3장 모든 객체의 공통 메서드  
## 아이템 10. equals는 일반 규약을 지켜 재정의하라

equals 메서드는 다음 경우에는 재정의하지 않는 것이 최선입니다:

- 각 인스턴스가 본질적으로 고유하다 (Thread 등)
- 인스턴스의 '논리적 동치성'을 검사할 일이 없다
- 상위 클래스에서 재정의한 equals가 하위 클래스에도 적절하다
- 클래스가 private이거나 package-private이고 equals 메서드를 호출할 일이 없다

**equals 메서드의 일반 규약 (동치관계):**

1. **반사성(reflexivity)**: x.equals(x)는 true
2. **대칭성(symmetry)**: x.equals(y)가 true면 y.equals(x)도 true
3. **추이성(transitivity)**: x.equals(y)가 true이고 y.equals(z)가 true면 x.equals(z)도 true
4. **일관성(consistency)**: x.equals(y)를 반복 호출해도 항상 같은 결과
5. **null-아님**: x.equals(null)은 항상 false

**올바른 equals 메서드 구현 방법:**

Java

```
@Override
public boolean equals(Object o) {
    // 1. 자기 자신의 참조인지 확인
    if (this == o) return true;
    
    // 2. instanceof 연산자로 입력이 올바른 타입인지 확인
    if (!(o instanceof PhoneNumber)) return false;
    
    // 3. 입력을 올바른 타입으로 형변환
    PhoneNumber pn = (PhoneNumber) o;
    
    // 4. 입력 객체와 자기 자신의 대응되는 '핵심' 필드들이 모두 일치하는지 검사
    return pn.lineNum == lineNum 
        && pn.prefix == prefix 
        && pn.areaCode == areaCode;
}
```

**주의사항:**

- equals를 재정의할 때는 hashCode도 반드시 재정의하라
- 너무 복잡하게 해결하려 들지 말라
- Object 외의 타입을 매개변수로 받는 equals 메서드는 선언하지 말라

## 아이템 11. equals를 재정의하려거든 hashCode도 재정의하라

**hashCode 일반 규약:**

1. equals 비교에 사용되는 정보가 변경되지 않았다면, hashCode도 변하지 않아야 한다
2. equals가 두 객체를 같다고 판단했다면, 두 객체의 hashCode는 같아야 한다
3. equals가 두 객체를 다르다고 판단했더라도, hashCode가 꼭 다를 필요는 없다 (하지만 다른 것이 성능상 유리)

**좋은 hashCode 구현 방법:**

Java

```
@Override
public int hashCode() {
    int result = Short.hashCode(areaCode);
    result = 31 * result + Short.hashCode(prefix);
    result = 31 * result + Short.hashCode(lineNum);
    return result;
}

// 또는 Objects.hash 사용 (성능은 조금 떨어짐)
@Override
public int hashCode() {
    return Objects.hash(lineNum, prefix, areaCode);
}

// 캐싱을 사용한 지연 초기화
private int hashCode; // 자동으로 0으로 초기화

@Override
public int hashCode() {
    int result = hashCode;
    if (result == 0) {
        result = Short.hashCode(areaCode);
        result = 31 * result + Short.hashCode(prefix);
        result = 31 * result + Short.hashCode(lineNum);
        hashCode = result;
    }
    return result;
}
```

## 아이템 12. toString을 항상 재정의하라

toString을 잘 구현한 클래스는 사용하기 편하고 디버깅이 쉽습니다.

Java

```
/**
 * 이 전화번호의 문자열 표현을 반환한다.
 * 이 문자열은 "XXX-YYY-ZZZZ" 형태의 12글자로 구성된다.
 * XXX는 지역 코드, YYY는 프리픽스, ZZZZ는 가입자 번호다.
 */
@Override
public String toString() {
    return String.format("%03d-%03d-%04d", areaCode, prefix, lineNum);
}
```

**toString 구현 시 주의사항:**

- 객체가 가진 주요 정보를 모두 반환하는 것이 좋다
- 반환값의 포맷을 문서화할지 정해야 한다
- toString이 반환한 값에 포함된 정보를 얻어올 수 있는 API를 제공하자

## 아이템 13. clone 재정의는 주의해서 진행하라

Cloneable은 복제해도 되는 클래스임을 명시하는 믹스인 인터페이스지만, clone 메서드는 Object에 protected로 선언되어 있습니다.

**clone 메서드의 일반 규약:**

- x.clone() != x (참조가 다름)
- x.clone().getClass() == x.getClass() (같은 클래스)
- x.clone().equals(x) (논리적 동치)

**가변 상태를 참조하지 않는 클래스의 clone:**

Java

```
@Override
public PhoneNumber clone() {
    try {
        return (PhoneNumber) super.clone();
    } catch (CloneNotSupportedException e) {
        throw new AssertionError(); // 일어날 수 없는 일
    }
}
```

**가변 상태를 참조하는 클래스의 clone:**

Java

```
public class Stack implements Cloneable {
    private Object[] elements;
    private int size = 0;
    
    @Override
    public Stack clone() {
        try {
            Stack result = (Stack) super.clone();
            result.elements = elements.clone(); // 배열의 clone은 런타임 타입과 컴파일타임 타입 모두 원본과 같은 배열 반환
            return result;
        } catch (CloneNotSupportedException e) {
            throw new AssertionError();
        }
    }
}
```

**더 나은 방법: 복사 생성자와 복사 팩터리**

Java

```
// 복사 생성자
public Yum(Yum yum) {
    this.field1 = yum.field1;
    this.field2 = new ArrayList<>(yum.field2);
}

// 복사 팩터리
public static Yum newInstance(Yum yum) {
    return new Yum(yum);
}
```

## 아이템 14. Comparable을 구현할지 고려하라

Comparable 인터페이스의 유일한 메서드인 compareTo는 단순 동치성 비교에 더해 순서까지 비교할 수 있으며, 제네릭합니다.

**compareTo 메서드의 일반 규약:**

1. sgn(x.compareTo(y)) == -sgn(y.compareTo(x))
2. 추이성: x.compareTo(y) > 0 && y.compareTo(z) > 0이면 x.compareTo(z) > 0
3. x.compareTo(y) == 0이면 sgn(x.compareTo(z)) == sgn(y.compareTo(z))
4. (권고) x.compareTo(y) == 0이면 x.equals(y)여야 한다

**compareTo 구현 예시:**

Java

```
// 기본 타입 필드가 여럿일 때의 비교
public int compareTo(PhoneNumber pn) {
    int result = Short.compare(areaCode, pn.areaCode);
    if (result == 0) {
        result = Short.compare(prefix, pn.prefix);
        if (result == 0)
            result = Short.compare(lineNum, pn.lineNum);
    }
    return result;
}

// 비교자 생성 메서드를 활용한 비교자
private static final Comparator<PhoneNumber> COMPARATOR =
    comparingInt((PhoneNumber pn) -> pn.areaCode)
        .thenComparingInt(pn -> pn.prefix)
        .thenComparingInt(pn -> pn.lineNum);

public int compareTo(PhoneNumber pn) {
    return COMPARATOR.compare(this, pn);
}
```

**주의사항:**

- 값의 차를 기준으로 하는 compareTo나 compare 메서드는 정수 오버플로를 일으킬 수 있으니 사용하지 말자

Java

```
// 잘못된 예 - 정수 오버플로 가능성
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return o1.hashCode() - o2.hashCode(); // 위험!
    }
};

// 올바른 예
static Comparator<Object> hashCodeOrder = new Comparator<>() {
    public int compare(Object o1, Object o2) {
        return Integer.compare(o1.hashCode(), o2.hashCode());
    }
};

// 또는 비교자 생성 메서드 사용
static Comparator<Object> hashCodeOrder = 
    Comparator.comparingInt(o -> o.hashCode());
```

4장 클래스와 인터페이스  
## 아이템 15. 클래스와 멤버의 접근 권한을 최소화하라

### 정보 은닉(캡슐화)의 개념

정보 은닉은 소프트웨어 설계의 근간이 되는 원리로, 시스템의 구성요소를 서로 독립시켜 개발, 테스트, 최적화, 사용, 이해, 수정을 개별적으로 할 수 있게 합니다. 이는 개발 속도를 높이고, 관리 비용을 낮추며, 성능 최적화에 도움을 주고, 재사용성을 높입니다.

### 자바의 접근 제한자

- **private**: 멤버를 선언한 톱레벨 클래스에서만 접근 가능
- **package-private(default)**: 멤버가 소속된 패키지 안의 모든 클래스에서 접근 가능
- **protected**: package-private의 접근 범위를 포함하며, 이 멤버를 선언한 클래스의 하위 클래스에서도 접근 가능
- **public**: 모든 곳에서 접근 가능

### 접근성을 최소화하는 방법

Java

```
// 잘못된 예 - public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다
public class Point {
    public double x;
    public double y;
}

// 개선된 예 - 필드를 private으로 만들고 접근자 제공
public class Point {
    private double x;
    private double y;
    
    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

### 주의사항

- 클래스의 공개 API를 세심히 설계한 후, 그 외의 모든 멤버는 private으로 만들자
- public 클래스의 인스턴스 필드는 되도록 public이 아니어야 한다
- public static final 필드는 기본 타입 값이나 불변 객체를 참조해야 한다

Java

```
// 보안 허점이 있는 예
public static final Thing[] VALUES = { ... };

// 해결책 1: private 배열과 public 불변 리스트
private static final Thing[] PRIVATE_VALUES = { ... };
public static final List<Thing> VALUES = 
    Collections.unmodifiableList(Arrays.asList(PRIVATE_VALUES));

// 해결책 2: private 배열과 복사본 반환 메서드
private static final Thing[] PRIVATE_VALUES = { ... };
public static final Thing[] values() {
    return PRIVATE_VALUES.clone();
}
```

## 아이템 16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라

### 데이터 캡슐화의 중요성

public 클래스가 필드를 공개하면 이를 사용하는 클라이언트가 생겨나므로 내부 표현을 바꾸기 어려워집니다.

Java

```
// 나쁜 예 - 퇴보한 클래스
class Point {
    public double x;
    public double y;
}

// 좋은 예 - 접근자와 변경자 메서드를 활용한 데이터 캡슐화
public class Point {
    private double x;
    private double y;
    
    public Point(double x, double y) {
        this.x = x;
        this.y = y;
    }
    
    public double getX() { return x; }
    public double getY() { return y; }
    
    public void setX(double x) { this.x = x; }
    public void setY(double y) { this.y = y; }
}
```

### 예외 상황

package-private 클래스나 private 중첩 클래스라면 데이터 필드를 노출한다 해도 문제가 없습니다. 클래스가 표현하려는 추상 개념만 올바르게 표현해주면 됩니다.

## 아이템 17. 변경 가능성을 최소화하라

### 불변 클래스의 개념

불변 클래스란 인스턴스의 내부 값을 수정할 수 없는 클래스입니다. 불변 인스턴스에 간직된 정보는 고정되어 객체가 파괴되는 순간까지 절대 달라지지 않습니다.

### 불변 클래스를 만드는 다섯 가지 규칙

1. **객체의 상태를 변경하는 메서드(변경자)를 제공하지 않는다**
2. **클래스를 확장할 수 없도록 한다** (final 클래스로 선언)
3. **모든 필드를 final로 선언한다**
4. **모든 필드를 private으로 선언한다**
5. **자신 외에는 내부의 가변 컴포넌트에 접근할 수 없도록 한다**

### 불변 클래스 예시

Java

```
public final class Complex {
    private final double re;
    private final double im;
    
    public Complex(double re, double im) {
        this.re = re;
        this.im = im;
    }
    
    public double realPart() { return re; }
    public double imaginaryPart() { return im; }
    
    public Complex plus(Complex c) {
        return new Complex(re + c.re, im + c.im);
    }
    
    public Complex minus(Complex c) {
        return new Complex(re - c.re, im - c.im);
    }
    
    public Complex times(Complex c) {
        return new Complex(re * c.re - im * c.im,
                          re * c.im + im * c.re);
    }
    
    public Complex dividedBy(Complex c) {
        double tmp = c.re * c.re + c.im * c.im;
        return new Complex((re * c.re + im * c.im) / tmp,
                          (im * c.re - re * c.im) / tmp);
    }
    
    @Override public boolean equals(Object o) {
        if (o == this) return true;
        if (!(o instanceof Complex)) return false;
        Complex c = (Complex) o;
        return Double.compare(c.re, re) == 0
            && Double.compare(c.im, im) == 0;
    }
    
    @Override public int hashCode() {
        return 31 * Double.hashCode(re) + Double.hashCode(im);
    }
    
    @Override public String toString() {
        return "(" + re + " + " + im + "i)";
    }
}
```

### 불변 클래스의 장점

1. **단순하다**: 생성 시점의 상태를 파괴될 때까지 그대로 간직
2. **스레드 안전하며 따로 동기화할 필요 없다**
3. **안심하고 공유할 수 있다**
4. **불변 객체끼리는 내부 데이터를 공유할 수 있다**
5. **객체를 만들 때 다른 불변 객체들을 구성요소로 사용하면 이점이 많다**
6. **실패 원자성을 제공한다**: 메서드에서 예외가 발생해도 객체는 여전히 유효한 상태

### 불변 클래스의 단점과 대처법

값이 다르면 반드시 독립된 객체로 만들어야 하므로, 값의 가짓수가 많다면 비용이 큽니다.

Java

```
// 가변 동반 클래스를 제공하는 예
// String(불변)의 가변 동반 클래스는 StringBuilder
BigInteger moby = new BigInteger("1000000");
moby = moby.flipBit(0); // 새로운 BigInteger 인스턴스 생성

// 다단계 연산을 위한 가변 동반 클래스 사용
StringBuilder sb = new StringBuilder();
for (String s : strings) {
    sb.append(s);
}
String result = sb.toString();
```

## 아이템 18. 상속보다는 컴포지션을 사용하라

### 상속의 위험성

상속은 코드를 재사용하는 강력한 수단이지만, 잘못 사용하면 오류를 내기 쉬운 소프트웨어를 만들게 됩니다. 상위 클래스와 하위 클래스를 모두 같은 프로그래머가 통제하는 패키지 안에서라면 상속도 안전한 방법이지만, 일반적인 구체 클래스를 패키지 경계를 넘어 상속하는 일은 위험합니다.

### 메서드 호출과 달리 상속은 캡슐화를 깨뜨린다

Java

```
// 잘못된 예 - 상속을 잘못 사용했다!
public class InstrumentedHashSet<E> extends HashSet<E> {
    private int addCount = 0;
    
    public InstrumentedHashSet() { }
    
    public InstrumentedHashSet(int initCap, float loadFactor) {
        super(initCap, loadFactor);
    }
    
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c); // HashSet의 addAll은 add를 호출한다!
    }
    
    public int getAddCount() {
        return addCount;
    }
}
```

### 컴포지션(composition)을 사용한 해결책

기존 클래스를 확장하는 대신, 새로운 클래스를 만들고 private 필드로 기존 클래스의 인스턴스를 참조하게 합니다.

Java

```
// 래퍼 클래스 - 상속 대신 컴포지션을 사용했다
public class InstrumentedSet<E> extends ForwardingSet<E> {
    private int addCount = 0;
    
    public InstrumentedSet(Set<E> s) {
        super(s);
    }
    
    @Override public boolean add(E e) {
        addCount++;
        return super.add(e);
    }
    
    @Override public boolean addAll(Collection<? extends E> c) {
        addCount += c.size();
        return super.addAll(c);
    }
    
    public int getAddCount() {
        return addCount;
    }
}

// 재사용할 수 있는 전달 클래스
public class ForwardingSet<E> implements Set<E> {
    private final Set<E> s;
    public ForwardingSet(Set<E> s) { this.s = s; }
    
    public void clear() { s.clear(); }
    public boolean contains(Object o) { return s.contains(o); }
    public boolean isEmpty() { return s.isEmpty(); }
    public int size() { return s.size(); }
    public Iterator<E> iterator() { return s.iterator(); }
    public boolean add(E e) { return s.add(e); }
    public boolean remove(Object o) { return s.remove(o); }
    public boolean containsAll(Collection<?> c) { return s.containsAll(c); }
    public boolean addAll(Collection<? extends E> c) { return s.addAll(c); }
    public boolean removeAll(Collection<?> c) { return s.removeAll(c); }
    public boolean retainAll(Collection<?> c) { return s.retainAll(c); }
    public Object[] toArray() { return s.toArray(); }
    public <T> T[] toArray(T[] a) { return s.toArray(a); }
    @Override public boolean equals(Object o) { return s.equals(o); }
    @Override public int hashCode() { return s.hashCode(); }
    @Override public String toString() { return s.toString(); }
}
```

### 데코레이터 패턴

이러한 설계를 데코레이터 패턴이라고 합니다. 컴포지션과 전달의 조합은 넓은 의미로 위임(delegation)이라고 부릅니다.

## 아이템 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라

### 상속용 클래스 설계 시 주의사항

1. **메서드를 재정의하면 어떤 일이 일어나는지 정확히 정리하여 문서로 남겨야 한다**
2. **클래스의 내부 동작 과정 중간에 끼어들 수 있는 훅(hook)을 잘 선별하여 protected 메서드 형태로 공개해야 할 수도 있다**
3. **상속용 클래스의 생성자는 직접적으로든 간접적으로든 재정의 가능 메서드를 호출해서는 안 된다**

Java

```
public class Super {
    // 잘못된 예 - 생성자가 재정의 가능 메서드를 호출한다
    public Super() {
        overrideMe();
    }
    
    public void overrideMe() { }
}

public final class Sub extends Super {
    private final Instant instant;
    
    Sub() {
        instant = Instant.now();
    }
    
    @Override public void overrideMe() {
        System.out.println(instant); // 첫 번째 호출에서는 null을 출력한다!
    }
}
```

### 상속을 금지하는 방법

1. **클래스를 final로 선언한다
2. **모든 생성자를 private이나 package-private으로 선언하고 public 정적 팩터리를 만들어준다**

Java

```
// final 클래스
public final class ImmutableClass {
    private final String value;
    
    public ImmutableClass(String value) {
        this.value = value;
    }
}

// 생성자를 private으로 하고 정적 팩터리 제공
public class ComplexNumber {
    private final double re;
    private final double im;
    
    private ComplexNumber(double re, double im) {
        this.re = re;
        this.im = im;
    }
    
    public static ComplexNumber valueOf(double re, double im) {
        return new ComplexNumber(re, im);
    }
    
    public static ComplexNumber valueOfPolar(double r, double theta) {
        return new ComplexNumber(r * Math.cos(theta), r * Math.sin(theta));
    }
}
```

## 아이템 20. 추상 클래스보다는 인터페이스를 우선하라

### 인터페이스와 추상 클래스의 차이점

자바가 제공하는 다중 구현 메커니즘은 인터페이스와 추상 클래스입니다. Java 8부터 인터페이스도 디폴트 메서드를 제공할 수 있게 되어, 두 메커니즘 모두 인스턴스 메서드를 구현 형태로 제공할 수 있습니다.

**가장 큰 차이점:**

- 추상 클래스가 정의한 타입을 구현하는 클래스는 반드시 추상 클래스의 하위 클래스가 되어야 한다
- 인터페이스가 선언한 메서드를 모두 정의하고 일반 규약을 잘 지킨 클래스라면 어떤 클래스를 상속했든 같은 타입으로 취급된다

### 인터페이스의 장점

1. **기존 클래스에도 손쉽게 새로운 인터페이스를 구현해넣을 수 있다**

Java

```
// Comparable을 구현하도록 기존 클래스를 수정하는 것은 간단하다
public class Employee implements Comparable<Employee> {
    private String name;
    private int salary;
    
    @Override
    public int compareTo(Employee other) {
        return Integer.compare(this.salary, other.salary);
    }
}
```

2. **인터페이스는 믹스인(mixin) 정의에 안성맞춤이다**  
    믹스인이란 클래스가 구현할 수 있는 타입으로, 주된 타입 외에도 특정 선택적 행위를 제공한다고 선언하는 효과를 줍니다.

Java

```
public class Singer implements Comparable<Singer> {
    @Override
    public int compareTo(Singer other) {
        return this.name.compareTo(other.name);
    }
}
```

3. **인터페이스로는 계층구조가 없는 타입 프레임워크를 만들 수 있다**

Java

```
public interface Singer {
    AudioClip sing(Song s);
}

public interface Songwriter {
    Song compose(int chartPosition);
}

// 가수 겸 작곡가
public interface SingerSongwriter extends Singer, Songwriter {
    AudioClip strum();
    void actSensitive();
}
```

4. **래퍼 클래스 관용구와 함께 사용하면 인터페이스는 기능을 향상시키는 안전하고 강력한 수단이 된다**

### 디폴트 메서드 제공

Java

```
public interface Calculator {
    double calculate(double x, double y);
    
    // 디폴트 메서드
    default double sqrt(double a) {
        return Math.sqrt(a);
    }
}
```

### 인터페이스와 추상 골격 구현 클래스

인터페이스와 추상 골격 구현(skeletal implementation) 클래스를 함께 제공하는 식으로 인터페이스와 추상 클래스의 장점을 모두 취할 수 있습니다.

Java

```
// 인터페이스
public interface List<E> {
    boolean add(E e);
    E get(int index);
    // ... 기타 메서드들
}

// 골격 구현 클래스
public abstract class AbstractList<E> implements List<E> {
    // 변하지 않는 핵심 메서드들을 구현
    public boolean addAll(int index, Collection<? extends E> c) {
        boolean modified = false;
        for (E e : c) {
            add(index++, e);
            modified = true;
        }
        return modified;
    }
    
    // 하위 클래스에서 구현해야 하는 메서드는 abstract로 선언
    public abstract E get(int index);
    public abstract boolean add(E e);
}
```

### 골격 구현 클래스 작성 방법

Java

```
// 1단계: 인터페이스를 잘 정의한다
public interface Vending {
    void start();
    void chooseProduct(int productId);
    void stop();
    boolean isRunning();
}

// 2단계: 골격 구현 클래스를 작성한다
public abstract class AbstractVending implements Vending {
    private boolean running = false;
    
    @Override
    public void start() {
        running = true;
    }
    
    @Override
    public void stop() {
        running = false;
    }
    
    @Override
    public boolean isRunning() {
        return running;
    }
    
    // 하위 클래스가 구현해야 하는 메서드
    @Override
    public abstract void chooseProduct(int productId);
}
```

## 아이템 21. 인터페이스는 구현하는 쪽을 생각해 설계하라

### 디폴트 메서드의 위험성

Java 8에서 기존 인터페이스에 메서드를 추가할 수 있도록 디폴트 메서드가 도입되었지만, 위험이 완전히 사라진 것은 아닙니다.

Java

```
// Collection 인터페이스에 추가된 디폴트 메서드
default boolean removeIf(Predicate<? super E> filter) {
    Objects.requireNonNull(filter);
    boolean removed = false;
    final Iterator<E> each = iterator();
    while (each.hasNext()) {
        if (filter.test(each.next())) {
            each.remove();
            removed = true;
        }
    }
    return removed;
}
```

### 디폴트 메서드의 문제점

1. **모든 구현체와 잘 어우러지는 디폴트 메서드를 작성하기는 어렵다**
2. **디폴트 메서드는 컴파일에 성공하더라도 기존 구현체에 런타임 오류를 일으킬 수 있다**

### 인터페이스 설계 시 주의사항

- 인터페이스를 설계할 때는 여전히 세심한 주의를 기울여야 한다
- 디폴트 메서드로 기존 인터페이스에 새 메서드를 추가하는 일은 꼭 필요한 경우가 아니면 피해야 한다
- 새로운 인터페이스라면 릴리스 전에 반드시 테스트를 거쳐야 한다

## 아이템 22. 인터페이스는 타입을 정의하는 용도로만 사용하라

### 인터페이스의 올바른 용도

인터페이스는 자신을 구현한 클래스의 인스턴스를 참조할 수 있는 타입 역할을 합니다. 클래스가 어떤 인터페이스를 구현한다는 것은 자신의 인스턴스로 무엇을 할 수 있는지를 클라이언트에 얘기해주는 것입니다.

### 안티패턴: 상수 인터페이스

Java

```
// 따라 하지 말 것! - 상수 인터페이스 안티패턴
public interface PhysicalConstants {
    // 아보가드로 수 (1/몰)
    static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    
    // 볼츠만 상수 (J/K)
    static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    
    // 전자 질량 (kg)
    static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

### 상수를 공개하는 올바른 방법

Java

```
// 1. 클래스나 인터페이스 자체에 추가
public class Integer {
    public static final int MIN_VALUE = 0x80000000;
    public static final int MAX_VALUE = 0x7fffffff;
    // ...
}

// 2. 열거 타입으로 만들기
public enum Planet {
    MERCURY(3.302e+23, 2.439e6),
    VENUS(4.869e+24, 6.052e6),
    EARTH(5.975e+24, 6.378e6);
    
    private final double mass;
    private final double radius;
    
    Planet(double mass, double radius) {
        this.mass = mass;
        this.radius = radius;
    }
}

// 3. 인스턴스화할 수 없는 유틸리티 클래스
public class PhysicalConstants {
    private PhysicalConstants() { }  // 인스턴스화 방지
    
    public static final double AVOGADROS_NUMBER = 6.022_140_857e23;
    public static final double BOLTZMANN_CONSTANT = 1.380_648_52e-23;
    public static final double ELECTRON_MASS = 9.109_383_56e-31;
}
```

## 아이템 23. 태그 달린 클래스보다는 클래스 계층구조를 활용하라

### 태그 달린 클래스의 문제점

태그 달린 클래스는 두 가지 이상의 의미를 표현할 수 있으며, 그중 현재 표현하는 의미를 태그 값으로 알려주는 클래스입니다.

Java

```
// 태그 달린 클래스 - 클래스 계층구조보다 훨씬 나쁘다!
class Figure {
    enum Shape { RECTANGLE, CIRCLE };
    
    // 태그 필드 - 현재 모양을 나타낸다
    final Shape shape;
    
    // 다음 필드들은 모양이 사각형일 때만 쓰인다
    double length;
    double width;
    
    // 다음 필드는 모양이 원일 때만 쓰인다
    double radius;
    
    // 원용 생성자
    Figure(double radius) {
        shape = Shape.CIRCLE;
        this.radius = radius;
    }
    
    // 사각형용 생성자
    Figure(double length, double width) {
        shape = Shape.RECTANGLE;
        this.length = length;
        this.width = width;
    }
    
    double area() {
        switch(shape) {
            case RECTANGLE:
                return length * width;
            case CIRCLE:
                return Math.PI * (radius * radius);
            default:
                throw new AssertionError(shape);
        }
    }
}
```

### 클래스 계층구조로 변환

Java

```
// 태그 달린 클래스를 클래스 계층구조로 변환
abstract class Figure {
    abstract double area();
}

class Circle extends Figure {
    final double radius;
    
    Circle(double radius) { 
        this.radius = radius; 
    }
    
    @Override
    double area() { 
        return Math.PI * (radius * radius); 
    }
}

class Rectangle extends Figure {
    final double length;
    final double width;
    
    Rectangle(double length, double width) {
        this.length = length;
        this.width = width;
    }
    
    @Override
    double area() { 
        return length * width; 
    }
}

// 정사각형도 지원하도록 확장
class Square extends Rectangle {
    Square(double side) {
        super(side, side);
    }
}
```

### 클래스 계층구조의 장점

1. 간결하고 명확하다
2. 각 의미를 독립된 클래스에 담아 관련 없는 데이터 필드를 제거할 수 있다
3. 살아 있는 코드만 남는다
4. 각 클래스의 생성자가 모든 필드를 남김없이 초기화한다
5. 타입 사이의 자연스러운 계층 관계를 반영할 수 있다

## 아이템 24. 멤버 클래스는 되도록 static으로 만들라

### 중첩 클래스의 종류

중첩 클래스(nested class)는 다른 클래스 안에 정의된 클래스를 말하며, 네 가지 종류가 있습니다:

1. **정적 멤버 클래스(static member class)**
2. **비정적 멤버 클래스(nonstatic member class)**
3. **익명 클래스(anonymous class)**
4. **지역 클래스(local class)**

### 정적 멤버 클래스

Java

```
public class Calculator {
    // 정적 멤버 클래스
    public static class Operation {
        public enum Type { PLUS, MINUS, MULTIPLY, DIVIDE }
        
        private final Type type;
        private final double operand;
        
        public Operation(Type type, double operand) {
            this.type = type;
            this.operand = operand;
        }
        
        public double apply(double value) {
            switch (type) {
                case PLUS: return value + operand;
                case MINUS: return value - operand;
                case MULTIPLY: return value * operand;
                case DIVIDE: return value / operand;
```
text

```
            default: throw new AssertionError("Unknown operation: " + type);
        }
    }
}

public double calculate(double value, Operation... operations) {
    for (Operation op : operations) {
        value = op.apply(value);
    }
    return value;
}
```

}

// 사용 예  
Calculator calc = new Calculator();  
double result = calc.calculate(10,  
new Calculator.Operation(Calculator.Operation.Type.PLUS, 5),  
new Calculator.Operation(Calculator.Operation.Type.MULTIPLY, 2));

text

````

### 비정적 멤버 클래스
```java
public class OuterClass {
    private int outerField = 10;
    
    // 비정적 멤버 클래스
    public class InnerClass {
        public void printOuterField() {
            // 바깥 클래스의 private 필드에 접근 가능
            System.out.println(outerField);
            // 바깥 인스턴스 참조: OuterClass.this
            System.out.println(OuterClass.this.outerField);
        }
    }
    
    public InnerClass createInner() {
        return new InnerClass();
    }
}

// 사용 예
OuterClass outer = new OuterClass();
OuterClass.InnerClass inner = outer.createInner();
// 또는
OuterClass.InnerClass inner2 = outer.new InnerClass();
````

**중요한 차이점:**

- 비정적 멤버 클래스의 인스턴스는 바깥 클래스의 인스턴스와 암묵적으로 연결됨
- 비정적 멤버 클래스의 인스턴스 메서드에서 바깥 클래스의 메서드를 호출하거나 바깥 인스턴스의 참조를 가져올 수 있음
- 비정적 멤버 클래스는 바깥 인스턴스 없이는 생성할 수 없음

### 익명 클래스

Java

```
public class ButtonExample {
    public void addActionListener(Button button) {
        // 익명 클래스 사용
        button.setOnClickListener(new OnClickListener() {
            @Override
            public void onClick() {
                System.out.println("Button clicked!");
            }
        });
    }
    
    // 람다로 대체 가능한 경우
    public void addActionListenerWithLambda(Button button) {
        button.setOnClickListener(() -> System.out.println("Button clicked!"));
    }
}
```

### 지역 클래스

Java

```
public class LocalClassExample {
    public void process(int[] numbers) {
        // 지역 클래스
        class NumberProcessor {
            int countEvens() {
                int count = 0;
                for (int num : numbers) {
                    if (num % 2 == 0) count++;
                }
                return count;
            }
        }
        
        NumberProcessor processor = new NumberProcessor();
        System.out.println("짝수의 개수: " + processor.countEvens());
    }
}
```

### 정적 멤버 클래스 vs 비정적 멤버 클래스

멤버 클래스가 바깥 인스턴스에 접근할 일이 없다면 무조건 static을 붙여서 정적 멤버 클래스로 만드는 것이 좋습니다. static을 생략하면:

- 바깥 인스턴스로의 숨은 참조를 갖게 됨
- 이 참조를 저장하려면 시간과 공간이 소비됨
- 가비지 컬렉션이 바깥 클래스의 인스턴스를 수거하지 못하는 메모리 누수가 발생할 수 있음

## 아이템 25. 톱레벨 클래스는 한 파일에 하나만 담으라

### 여러 톱레벨 클래스를 한 파일에 담을 때의 문제점

한 소스 파일에 여러 톱레벨 클래스를 선언하면 한 클래스를 여러 가지로 정의할 수 있고, 그중 어느 것을 사용할지는 어떤 소스 파일을 먼저 컴파일하냐에 따라 달라지는 위험이 있습니다.

Java

```
// Main.java 파일
public class Main {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
}

// Utensil.java 파일
class Utensil {
    static final String NAME = "pan";
}

class Dessert {
    static final String NAME = "cake";
}

// Dessert.java 파일 (의도치 않게 같은 이름의 클래스 정의)
class Utensil {
    static final String NAME = "pot";
}

class Dessert {
    static final String NAME = "pie";
}
```

### 해결책

톱레벨 클래스들을 서로 다른 소스 파일로 분리하는 것이 가장 좋습니다. 꼭 한 파일에 담고 싶다면 정적 멤버 클래스를 사용하는 방법을 고려하세요.

Java

```
// 톱레벨 클래스들을 정적 멤버 클래스로 변환
public class Test {
    public static void main(String[] args) {
        System.out.println(Utensil.NAME + Dessert.NAME);
    }
    
    private static class Utensil {
        static final String NAME = "pan";
    }
    
    private static class Dessert {
        static final String NAME = "cake";
    }
}
```



5장 제네릭  
아이템 26. 로 타입은 사용하지 말라  
아이템 27. 비검사 경고를 제거하라  
아이템 28. 배열보다는 리스트를 사용하라  
아이템 29. 이왕이면 제네릭 타입으로 만들라  
아이템 30. 이왕이면 제네릭 메서드로 만들라  
아이템 31. 한정적 와일드카드를 사용해 API 유연성을 높이라  
아이템 32. 제네릭과 가변인수를 함께 쓸 때는 신중하라  
아이템 33. 타입 안전 이종 컨테이너를 고려하라  
  
6장 열거 타입과 애너테이션  
아이템 34. int 상수 대신 열거 타입을 사용하라  
아이템 35. ordinal 메서드 대신 인스턴스 필드를 사용하라  
아이템 36. 비트 필드 대신 EnumSet을 사용하라  
아이템 37. ordinal 인덱싱 대신 EnumMap을 사용하라  
아이템 38. 확장할 수 있는 열거 타입이 필요하면 인터페이스를 사용하라  
아이템 39. 명명 패턴보다 애너테이션을 사용하라  
아이템 40. @Override 애너테이션을 일관되게 사용하라  
아이템 41. 정의하려는 것이 타입이라면 마커 인터페이스를 사용하라  
  
7장 람다와 스트림  
아이템 42. 익명 클래스보다는 람다를 사용하라  
아이템 43. 람다보다는 메서드 참조를 사용하라  
아이템 44. 표준 함수형 인터페이스를 사용하라  
아이템 45. 스트림은 주의해서 사용하라  
아이템 46. 스트림에서는 부작용 없는 함수를 사용하라  
아이템 47. 반환 타입으로는 스트림보다 컬렉션이 낫다  
아이템 48. 스트림 병렬화는 주의해서 적용하라  
  
8장 메서드  
아이템 49. 매개변수가 유효한지 검사하라  
아이템 50. 적시에 방어적 복사본을 만들라  
아이템 51. 메서드 시그니처를 신중히 설계하라  
아이템 52. 다중정의는 신중히 사용하라  
아이템 53. 가변인수는 신중히 사용하라  
아이템 54. null이 아닌, 빈 컬렉션이나 배열을 반환하라  
아이템 55. 옵셔널 반환은 신중히 하라  
아이템 56. 공개된 API 요소에는 항상 문서화 주석을 작성하라  
  
9장 일반적인 프로그래밍 원칙  
아이템 57. 지역변수의 범위를 최소화하라  
아이템 58. 전통적인 for 문보다는 for-each 문을 사용하라  
아이템 59. 라이브러리를 익히고 사용하라  
아이템 60. 정확한 답이 필요하다면 float와 double은 피하라  
아이템 61. 박싱된 기본 타입보다는 기본 타입을 사용하라  
아이템 62. 다른 타입이 적절하다면 문자열 사용을 피하라  
아이템 63. 문자열 연결은 느리니 주의하라  
아이템 64. 객체는 인터페이스를 사용해 참조하라  
아이템 65. 리플렉션보다는 인터페이스를 사용하라  
아이템 66. 네이티브 메서드는 신중히 사용하라  
아이템 67. 최적화는 신중히 하라  
아이템 68. 일반적으로 통용되는 명명 규칙을 따르라  
  
10장 예외  
아이템 69. 예외는 진짜 예외 상황에만 사용하라  
아이템 70. 복구할 수 있는 상황에는 검사 예외를, 프로그래밍 오류에는 런타임 예외를 사용하라  
아이템 71. 필요 없는 검사 예외 사용은 피하라  
아이템 72. 표준 예외를 사용하라  
아이템 73. 추상화 수준에 맞는 예외를 던지라  
아이템 74. 메서드가 던지는 모든 예외를 문서화하라  
아이템 75. 예외의 상세 메시지에 실패 관련 정보를 담으라  
아이템 76. 가능한 한 실패 원자적으로 만들라  
아이템 77. 예외를 무시하지 말라  
  
11장 동시성  
아이템 78. 공유 중인 가변 데이터는 동기화해 사용하라  
아이템 79. 과도한 동기화는 피하라  
아이템 80. 스레드보다는 실행자, 태스크, 스트림을 애용하라  
아이템 81. wait와 notify보다는 동시성 유틸리티를 애용하라  
아이템 82. 스레드 안전성 수준을 문서화하라  
아이템 83. 지연 초기화는 신중히 사용하라  
아이템 84. 프로그램의 동작을 스레드 스케줄러에 기대지 말라  
  
12장 직렬화  
아이템 85. 자바 직렬화의 대안을 찾으라  
아이템 86. Serializable을 구현할지는 신중히 결정하라  
아이템 87. 커스텀 직렬화 형태를 고려해보라  
아이템 88. readObject 메서드는 방어적으로 작성하라  
아이템 89. 인스턴스 수를 통제해야 한다면 readResolve보다는 열거 타입을 사용하라  
아이템 90. 직렬화된 인스턴스 대신 직렬화 프록시 사용을 검토하라