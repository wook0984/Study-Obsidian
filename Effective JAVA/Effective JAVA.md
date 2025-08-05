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

싱글턴을 만드는 방법은 세 가지가 있습니다:

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
아이템 15. 클래스와 멤버의 접근 권한을 최소화하라  
아이템 16. public 클래스에서는 public 필드가 아닌 접근자 메서드를 사용하라  
아이템 17. 변경 가능성을 최소화하라  
아이템 18. 상속보다는 컴포지션을 사용하라  
아이템 19. 상속을 고려해 설계하고 문서화하라. 그러지 않았다면 상속을 금지하라  
아이템 20. 추상 클래스보다는 인터페이스를 우선하라  
아이템 21. 인터페이스는 구현하는 쪽을 생각해 설계하라  
아이템 22. 인터페이스는 타입을 정의하는 용도로만 사용하라  
아이템 23. 태그 달린 클래스보다는 클래스 계층구조를 활용하라  
아이템 24. 멤버 클래스는 되도록 static으로 만들라  
아이템 25. 톱레벨 클래스는 한 파일에 하나만 담으라  
  
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