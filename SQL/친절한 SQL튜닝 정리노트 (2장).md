## 2장. 인덱스 기본

### 2.1 인덱스 구조 및 탐색

#### 2.1.1 미리 보는 인덱스 튜닝

인덱스 튜닝의 핵심은 다음 세 가지다:

1. 인덱스 스캔 효율화
2. 랜덤 액세스 최소화
3. 소트 연산 생략

#### 2.1.2 인덱스 구조

**B-Tree 인덱스 구조**

text

```
         루트 블록
         [40|80]
        /   |   \
    브랜치 브랜치 브랜치
   [10|25] [50|65] [85|95]
   /  |  \
리프 리프 리프 ... (정렬된 키값 + ROWID)
```

**구성 요소**

- 루트 블록: 최상위 블록
- 브랜치 블록: 중간 블록
- 리프 블록: 최하위 블록 (실제 데이터 위치 정보)
- ROWID: 테이블 레코드의 물리적 주소

#### 2.1.3 인덱스 수직적 탐색

루트에서 리프까지 찾아가는 과정이다.

SQL

```
-- 인덱스: 이름
SELECT * FROM 직원 WHERE 이름 = '김철수';

탐색 과정:
1. 루트 블록 읽기: '김철수'가 어느 브랜치로 가야 하는지 확인
2. 브랜치 블록 읽기: '김철수'가 어느 리프로 가야 하는지 확인  
3. 리프 블록 도달: '김철수'의 ROWID 획득
```

#### 2.1.4 인덱스 수평적 탐색

리프 블록에서 옆으로 스캔하는 과정이다.

SQL

```
-- 범위 검색
SELECT * FROM 직원 WHERE 이름 BETWEEN '김' AND '박';

탐색 과정:
1. 수직적 탐색으로 '김'이 있는 첫 리프 블록 찾기
2. 수평적 탐색으로 '박'까지 리프 블록들을 순차적으로 읽기
3. 각 리프 블록에서 ROWID를 이용해 테이블 액세스
```

#### 2.1.5 결합 인덱스 구조와 탐색

여러 컬럼을 합친 인덱스다.

SQL

```
-- 인덱스: (부서코드, 직급, 입사일자)
CREATE INDEX idx_emp ON 직원(부서코드, 직급, 입사일자);

-- 리프 블록의 정렬 순서
(A001, 과장, 2020-01-01)
(A001, 과장, 2020-03-15)
(A001, 대리, 2019-05-20)
(A001, 대리, 2021-07-10)
(B001, 과장, 2018-09-01)
...
```

### 2.2 인덱스 기본 사용법

#### 2.2.1 인덱스를 사용한다는 것

인덱스를 사용한다는 것은 리프 블록에서 얻은 ROWID로 테이블을 액세스한다는 의미다.

SQL

```
-- 인덱스 사용 과정
SELECT 이름, 급여 FROM 직원 WHERE 직원ID = 100;

1. 직원ID 인덱스에서 100 검색
2. 리프 블록에서 ROWID 획득
3. ROWID로 테이블 블록 액세스
4. 이름, 급여 컬럼 값 읽기
```

#### 2.2.2 인덱스를 Range Scan 할 수 없는 이유

**인덱스 컬럼 가공**

SQL

```
-- 인덱스 사용 불가
SELECT * FROM 직원 WHERE SUBSTR(주민번호, 8, 1) = '1';
SELECT * FROM 직원 WHERE 급여 * 12 = 36000000;
SELECT * FROM 주문 WHERE TO_CHAR(주문일자, 'YYYYMM') = '202401';

-- 인덱스 사용 가능하게 수정
SELECT * FROM 직원 WHERE 주민번호 LIKE '______1%';
SELECT * FROM 직원 WHERE 급여 = 36000000 / 12;
SELECT * FROM 주문 WHERE 주문일자 >= '2025-08-01' AND 주문일자 < '2025-09-01';
```

**부정형 조건**

SQL

```
-- 인덱스 사용 불가
SELECT * FROM 직원 WHERE 부서코드 <> 'A001';
SELECT * FROM 직원 WHERE 부서코드 NOT IN ('A001', 'B001');

-- 인덱스 사용 가능하게 수정 (가능한 경우)
SELECT * FROM 직원 WHERE 부서코드 IN ('A002', 'A003', 'B002', ...);
```

#### 2.2.3 더 중요한 인덱스 사용 조건

**NULL 조건**

SQL

```
-- 인덱스에 NULL은 저장되지 않는다 (Oracle 기준)
SELECT * FROM 직원 WHERE 퇴사일자 IS NULL;  -- Full Table Scan

-- NOT NULL 컬럼이거나 함수 기반 인덱스 사용
CREATE INDEX idx_retire ON 직원(NVL(퇴사일자, '9999-12-31'));
```

**OR 조건**

SQL

```
-- OR는 각각의 조건을 따로 처리한다
SELECT * FROM 직원 WHERE 부서코드 = 'A001' OR 직급 = '과장';

-- 각각의 인덱스를 사용하거나 Full Table Scan
-- UNION ALL로 변경 고려
SELECT * FROM 직원 WHERE 부서코드 = 'A001'
UNION ALL
SELECT * FROM 직원 WHERE 직급 = '과장' AND 부서코드 <> 'A001';
```

#### 2.2.4 인덱스를 이용한 소트 연산 생략

인덱스는 정렬되어 있으므로 ORDER BY를 생략할 수 있다.

SQL

```
-- 인덱스: 입사일자
SELECT * FROM 직원 ORDER BY 입사일자;
-- 별도 정렬 없이 인덱스 순서대로 읽기만 하면 된다

-- 인덱스: (부서코드, 급여)
SELECT * FROM 직원 
WHERE 부서코드 = 'A001'
ORDER BY 급여;
-- 부서코드 = 'A001'인 부분만 급여 순으로 읽으면 된다
```

#### 2.2.5 ORDER BY 절에서 컬럼 가공

SQL

```
-- 정렬 연산 발생
SELECT * FROM 직원 ORDER BY 급여 * 12;

-- 함수 기반 인덱스로 해결
CREATE INDEX idx_annual_sal ON 직원(급여 * 12);
```

#### 2.2.6 SELECT-LIST에서 컬럼 가공

SELECT 절의 컬럼 가공은 인덱스 사용에 영향을 주지 않는다.

SQL

```
-- 인덱스 사용에 문제없음
SELECT 이름, 급여 * 12 AS 연봉
FROM 직원
WHERE 부서코드 = 'A001';  -- 부서코드 인덱스 사용 가능
```

#### 2.2.7 자동 형변환

데이터 타입이 다르면 자동 형변환이 발생한다.

SQL

```
-- 직원번호가 VARCHAR2 타입일 때
SELECT * FROM 직원 WHERE 직원번호 = 100;
-- 내부적으로: WHERE TO_NUMBER(직원번호) = 100 (인덱스 사용 불가)

-- 올바른 사용
SELECT * FROM 직원 WHERE 직원번호 = '100';
```

### 2.3 인덱스 확장기능 사용법

#### 2.3.1 Index Range Scan

가장 일반적인 인덱스 스캔 방식이다.

SQL

```
SELECT * FROM 직원 WHERE 이름 = '김철수';
SELECT * FROM 직원 WHERE 이름 LIKE '김%';
SELECT * FROM 직원 WHERE 입사일자 BETWEEN '2025-01-01' AND '2025-12-31';
```

특징:

- 인덱스를 수직적 탐색 후 수평적 탐색
- B-Tree 인덱스의 가장 일반적인 형태
- 범위 검색에 적합

#### 2.3.2 Index Full Scan

인덱스 전체를 스캔한다.

SQL

```
-- 급여 인덱스가 있을 때
SELECT 급여 FROM 직원 ORDER BY 급여;

-- 실행계획
INDEX FULL SCAN IDX_SALARY
```

사용 조건:

- 인덱스에 포함된 컬럼만 필요할 때
- 정렬이 필요할 때
- 테이블보다 인덱스가 작을 때

#### 2.3.3 Index Unique Scan

Unique 인덱스를 = 조건으로 검색할 때 사용된다.

SQL

```
-- PK나 Unique 인덱스
SELECT * FROM 직원 WHERE 직원ID = 100;

-- 실행계획
INDEX UNIQUE SCAN PK_EMP
```

특징:

- 한 건만 찾으면 스캔 종료
- 가장 빠른 인덱스 스캔

#### 2.3.4 Index Skip Scan

선행 컬럼을 건너뛰고 스캔한다.

SQL

```
-- 인덱스: (성별, 부서코드)
-- 성별은 'M', 'F' 두 개 값만 존재

SELECT * FROM 직원 WHERE 부서코드 = 'A001';

-- 내부적으로 아래처럼 처리
SELECT * FROM 직원 WHERE 성별 = 'M' AND 부서코드 = 'A001'
UNION ALL
SELECT * FROM 직원 WHERE 성별 = 'F' AND 부서코드 = 'A001';
```

사용 조건:

- 선행 컬럼의 Distinct 값이 적을 때
- 후행 컬럼의 조건만 있을 때

#### 2.3.5 Index Fast Full Scan

인덱스를 Multiblock I/O로 읽는다.

SQL

```
-- 인덱스에 있는 컬럼만 필요하고 정렬 불필요
SELECT COUNT(*) FROM 직원;
SELECT 부서코드, COUNT(*) FROM 직원 GROUP BY 부서코드;
```

특징:

- 병렬 처리 가능
- 정렬 순서 보장 안 됨
- Full Table Scan보다 빠름

#### 2.3.6 Index Range Scan Descending

인덱스를 역순으로 읽는다.

SQL

```
-- 최신 데이터부터 조회
SELECT * FROM 주문 
WHERE 고객ID = 100
ORDER BY 주문일자 DESC;

-- 실행계획
INDEX RANGE SCAN DESCENDING IDX_ORDER
```
