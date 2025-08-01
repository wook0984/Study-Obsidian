## 5장. 소트 튜닝

### 5.1 소트 연산에 대한 이해

#### 5.1.1 소트 수행 과정

소트는 많은 자원을 사용하는 작업이다.

**소트 실행 과정**

1. PGA의 SORT_AREA_SIZE만큼 메모리 할당
2. 메모리에서 정렬 시도
3. 메모리 부족 시 디스크 사용 (TEMP 테이블스페이스)
4. 정렬된 데이터 결과 반환

SQL

```
-- 소트 발생 쿼리
SELECT 직원ID, 이름, 급여
FROM 직원
ORDER BY 급여 DESC;

-- 처리 과정
1. 직원 테이블 데이터 읽기
2. PGA 메모리에서 급여 기준 정렬
3. 정렬 결과 반환
```

#### 5.1.2 소트 오퍼레이션

주요 소트 유형:

1. **SORT ORDER BY**
    
    - ORDER BY 절로 인한 정렬
    
2. **SORT GROUP BY**
    
    - GROUP BY 절로 인한 정렬
    
3. **SORT UNIQUE**
    
    - DISTINCT, UNION 등으로 인한 중복 제거 정렬
    
4. **SORT JOIN**
    
    - 소트 머지 조인을 위한 정렬
    
5. **SORT AGGREGATE**
    
    - 집계 함수 계산을 위한 정렬
    

### 5.2 소트가 발생하지 않도록 SQL 작성

#### 5.2.1 Union vs. Union All

UNION은 중복 제거를 위해 소트 발생, UNION ALL은 소트 없음

SQL

```
-- 소트 발생 (중복 제거)
SELECT 고객ID FROM 온라인고객
UNION
SELECT 고객ID FROM 오프라인고객;

-- 소트 없음 (중복 허용)
SELECT 고객ID FROM 온라인고객
UNION ALL
SELECT 고객ID FROM 오프라인고객;

-- 중복이 없다고 확신하면 UNION ALL 사용
```

#### 5.2.2 Exists 활용

IN 대신 EXISTS 사용으로 소트 회피

SQL

```
-- 소트 발생 가능성 (IN)
SELECT * FROM 주문
WHERE 고객ID IN (
    SELECT 고객ID FROM 고객 WHERE 지역 = '서울'
);

-- 소트 없음 (EXISTS)
SELECT * FROM 주문 o
WHERE EXISTS (
    SELECT 1 FROM 고객 c
    WHERE c.고객ID = o.고객ID
    AND c.지역 = '서울'
);
```

### 5.3 인덱스를 이용한 소트 연산 생략

#### 5.3.1 Sort Order By 생략

SQL

```
-- 인덱스: (부서코드, 급여 DESC)
CREATE INDEX idx_emp ON 직원(부서코드, 급여 DESC);

SELECT 직원ID, 이름, 급여
FROM 직원
WHERE 부서코드 = 'A001'
ORDER BY 급여 DESC;

-- 인덱스 순서대로 읽기만 하면 되므로 소트 연산 생략
```

#### 5.3.2 Top N 쿼리

상위 N개만 필요한 경우 전체 정렬 불필요

SQL

```
-- Oracle
SELECT * FROM (
    SELECT 직원ID, 이름, 급여
    FROM 직원
    ORDER BY 급여 DESC
) WHERE ROWNUM <= 10;

-- 표준 SQL
SELECT 직원ID, 이름, 급여
FROM 직원
ORDER BY 급여 DESC
FETCH FIRST 10 ROWS ONLY;

-- 인덱스: 급여 DESC
-- 인덱스에서 상위 10개만 읽고 중단
```

#### 5.3.3 최소값/최대값 구하기

SQL

```
-- 소트 발생
SELECT MAX(급여) FROM 직원;

-- 소트 없음 (인덱스 사용)
SELECT 급여 FROM 직원 
WHERE ROWNUM = 1
ORDER BY 급여 DESC;
-- 급여 DESC 인덱스의 첫 번째 값만 읽음
```

#### 5.3.4 이력 조회

최신 이력 한 건만 필요한 경우 많음

SQL

```
-- 소트 발생
SELECT * FROM (
    SELECT * FROM 직원이력
    WHERE 직원ID = 100
    ORDER BY 변경일자 DESC
) WHERE ROWNUM = 1;

-- 인덱스: (직원ID, 변경일자 DESC)
-- 인덱스 첫 레코드만 읽고 종료
```

#### 5.3.5 Sort Group By 생략

SQL

```
-- 인덱스: (부서코드, 직급)
CREATE INDEX idx_emp ON 직원(부서코드, 직급);

SELECT 부서코드, 직급, COUNT(*)
FROM 직원
GROUP BY 부서코드, 직급;

-- 인덱스가 이미 (부서코드, 직급) 순으로 정렬되어 있으므로
-- GROUP BY를 위한 소트 생략 가능
```

### 5.4 Sort Area를 적게 사용하도록 SQL 작성

#### 5.4.1 소트 데이터 줄이기

정렬 전에 데이터 필터링하기

SQL

```
-- 소트 비효율 (많은 데이터 정렬 후 10개만 사용)
SELECT * FROM (
    SELECT * FROM 주문
    ORDER BY 주문일자 DESC
) WHERE ROWNUM <= 10;

-- 소트 효율화 (필터링 먼저)
SELECT * FROM (
    SELECT * FROM 주문
    WHERE 주문일자 >= '2025-01-01'
    ORDER BY 주문일자 DESC
) WHERE ROWNUM <= 10;

-- 또는
SELECT * FROM 주문
WHERE 주문일자 >= '2025-01-01'
ORDER BY 주문일자 DESC
FETCH FIRST 10 ROWS ONLY;
```

#### 5.4.2 Top N 쿼리의 소트 부하 경감 원리

SQL

```
-- 일반적인 Top N (전체 정렬 후 상위 10개)
SELECT * FROM (
    SELECT * FROM 직원
    ORDER BY 급여 DESC
) WHERE ROWNUM <= 10;

-- 옵티마이저 최적화
-- 내부적으로 힙 구조를 사용해 상위 10개만 유지하는 알고리즘 사용
-- 전체 소트보다 메모리 사용량 적음
```

#### 5.4.3 Top N 쿼리가 아닐 때 발생하는 소트 부하

SQL

```
-- 정렬 후 특정 범위만 가져오기
SELECT * FROM (
    SELECT 직원ID, 이름, 급여,
           ROW_NUMBER() OVER (ORDER BY 급여 DESC) AS rn
    FROM 직원
) WHERE rn BETWEEN 11 AND 20;

-- 전체 정렬이 필요함
-- 인덱스를 사용해도 모든 데이터를 읽어야 함
```

#### 5.4.4 분석함수에서의 Top N 소트

SQL

```
-- 부서별 급여 상위 3명
SELECT * FROM (
    SELECT 직원ID, 이름, 부서코드, 급여,
           ROW_NUMBER() OVER (PARTITION BY 부서코드 ORDER BY 급여 DESC) AS rn
    FROM 직원
) WHERE rn <= 3;

-- 각 부서별로 소트 필요
-- 부서별 데이터가 적으면 부담 적음
```

성능 좋은 SQL을 작성하려면 인덱스의 구조와 동작 원리를 깊이 이해하고, 데이터 액세스 경로와 조인 방식을 적절히 선택하며, 소트 연산을 최소화하는 것이 핵심이라고 생각한다. 실행계획을 분석하고 병목 지점을 찾아 최적화하는 습관을 들여야겠다.