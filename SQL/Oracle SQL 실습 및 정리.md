Oracle SQL developer를 활용한 실습방법 및 개념정리
## 1. 직원 테이블(s_emp)의 모든 행을 삭제하는 문장을 적으시오.

SQL

```
DELETE FROM S_EMP; -- S_EMP 테이블의 모든 행(데이터)을 삭제합니다.
```

---

## 2. 직원 테이블에 존재하는 모든 직급(title)을 중복없이 출력하시오.

SQL

```
SELECT DISTINCT TITLE -- 중복을 제거하고
FROM S_EMP;           -- S_EMP 테이블에서 TITLE 컬럼만 조회합니다.
```

---

## 3. 직원 테이블을 부서별(dept_id) 내림차순, 연봉(salary) 오름차순으로 정렬하시오.

SQL

```
SELECT *                -- 모든 컬럼을 조회합니다.
FROM S_EMP              -- S_EMP 테이블에서
ORDER BY
  DEPT_ID DESC,         -- 부서번호(DEPT_ID)를 내림차순(DESC)으로 정렬하고,
  SALARY ASC;           -- 같은 부서 내에서는 급여(SALARY)를 오름차순(ASC)으로 정렬합니다.
```

---

## 4. 직원 테이블에서 2015년도 이전에 입사한 직원의 수를 출력하시오.

SQL

```
SELECT COUNT(*)                       -- 조건에 맞는 행의 개수를 셉니다.
FROM S_EMP                            -- S_EMP 테이블에서
WHERE START_DATE < TO_DATE('2015-01-01', 'YYYY-MM-DD'); 
-- START_DATE(입사일)가 2015년 1월 1일보다 이전인 경우만 선택합니다.
-- TO_DATE('2015-01-01', 'YYYY-MM-DD')는 문자열을 날짜 타입으로 변환합니다.
-- 'YYYY-MM-DD'는 입력 문자열의 형식(연도-월-일)을 의미합니다.
```

---

## 5. 연봉이 1000이상 5000이하인 직원을 모두 출력하시오.

SQL

```
SELECT *                        -- 모든 컬럼을 조회합니다.
FROM S_EMP                      -- S_EMP 테이블에서
WHERE SALARY BETWEEN 1000 AND 5000; 
-- SALARY(급여)가 1000 이상 5000 이하인 행만 선택합니다.
-- BETWEEN A AND B는 A 이상 B 이하의 범위를 의미합니다.
```

---

## 6. 각 부서(dept_id)별 평균 급여를 계산해서 보여주시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       AVG(SALARY) AS AVG_SALARY -- 해당 부서의 평균 급여를 계산하여 AVG_SALARY라는 별칭으로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY DEPT_ID;               -- DEPT_ID(부서번호)별로 그룹화하여 집계합니다.
```

---

## 7. 각 부서(dept_id)별 직책이 사원인 직원들의 평균 급여를 계산해서 보여주시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       AVG(SALARY) AS AVG_SALARY -- 해당 부서의 평균 급여를 계산하여 AVG_SALARY라는 별칭으로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
WHERE TITLE = '사원'            -- TITLE(직책)이 '사원'인 행만 선택합니다.
GROUP BY DEPT_ID;               -- DEPT_ID(부서번호)별로 그룹화하여 집계합니다.
```

---

## 8. 각 지역(region_id)별로 몇 개의 부서가 있는지를 나타내시오.

SQL

```
SELECT REGION_ID,               -- 지역번호(REGION_ID)를 조회하고,
       COUNT(*) AS DEPT_COUNT   -- 해당 지역에 속한 부서의 개수를 DEPT_COUNT라는 별칭으로 출력합니다.
FROM S_DEPT                     -- S_DEPT(부서) 테이블에서
GROUP BY REGION_ID;             -- REGION_ID(지역번호)별로 그룹화하여 집계합니다.
```

---

## 9. 각 부서별로 평균 급여를 구하여 평균 급여가 2000이상인 부서만 나타내시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       AVG(SALARY) AS AVG_SALARY -- 해당 부서의 평균 급여를 계산하여 AVG_SALARY라는 별칭으로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY DEPT_ID                -- DEPT_ID(부서번호)별로 그룹화하여 집계하고,
HAVING AVG(SALARY) >= 2000;     -- 집계 결과 중 평균 급여가 2000 이상인 부서만 선택합니다.
```

---

## 10. 각 직책별로 급여의 총합을 구하되 직책이 부장인 사람은 제외하시오.

단, 급여총합이 8000(만원) 이상인 직책만 나타내며, 급여 총합에 대한 오름차순으로 정렬하시오.

SQL

```
SELECT TITLE,                   -- 직책(TITLE)을 조회하고,
       SUM(SALARY) AS TOTAL_SALARY -- 해당 직책의 급여 총합을 TOTAL_SALARY라는 별칭으로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
WHERE TITLE != '부장'           -- TITLE(직책)이 '부장'이 아닌 행만 선택합니다.
GROUP BY TITLE                  -- TITLE(직책)별로 그룹화하여 집계하고,
HAVING SUM(SALARY) >= 8000      -- 집계 결과 중 급여 총합이 8000 이상인 직책만 선택합니다.
ORDER BY TOTAL_SALARY ASC;      -- 급여 총합을 오름차순(ASC)으로 정렬합니다.
```

---
## 11. 각 부서별로 직책이 사원인 직원들에 대해서만 평균 급여를 구하시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       AVG(SALARY) AS AVG_SALARY -- 해당 부서의 '사원' 직책 평균 급여를 AVG_SALARY로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
WHERE TITLE = '사원'            -- TITLE(직책)이 '사원'인 행만 선택합니다.
GROUP BY DEPT_ID;               -- DEPT_ID(부서번호)별로 그룹화하여 집계합니다.
```

---

## 12. 각 부서내에서 각 직책별로 몇 명의 인원이 있는지를 나타내시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       TITLE,                   -- 직책(TITLE)을 조회하고,
       COUNT(*) AS CNT          -- 해당 부서/직책 조합의 인원수를 CNT로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY DEPT_ID, TITLE;        -- DEPT_ID와 TITLE별로 그룹화하여 집계합니다.
```

---

## 13. 각 부서내에서 몇 명의 직원이 근무하는지를 나타내시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       COUNT(*) AS CNT          -- 해당 부서의 인원수를 CNT로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY DEPT_ID;               -- DEPT_ID(부서번호)별로 그룹화하여 집계합니다.
```

---

## 14. 각 부서별로 급여의 최소값과 최대값을 나타내시오.

단, 최소값과 최대값이 같은 부서는 출력하지 마시오.

SQL

```
SELECT DEPT_ID,                 -- 부서번호(DEPT_ID)를 조회하고,
       MIN(SALARY) AS MIN_SALARY, -- 해당 부서의 최소 급여를 MIN_SALARY로 출력,
       MAX(SALARY) AS MAX_SALARY  -- 해당 부서의 최대 급여를 MAX_SALARY로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY DEPT_ID                -- DEPT_ID별로 그룹화하여 집계하고,
HAVING MIN(SALARY) <> MAX(SALARY); -- 최소값과 최대값이 다른 부서만 출력합니다.
```

---

## 15. 직원(s_emp) 테이블과 부서(s_dept) 테이블을 JOIN하여, 사원의 이름과 부서, 부서명을 나타내시오.

SQL

```
SELECT E.NAME,                  -- 사원 이름을 조회하고,
       E.DEPT_ID,               -- 사원의 부서번호를 조회하고,
       D.NAME AS DEPT_NAME      -- 부서명을 DEPT_NAME으로 출력합니다.
FROM S_EMP E                    -- S_EMP 테이블을 E로 별칭,
JOIN S_DEPT D                   -- S_DEPT 테이블을 D로 별칭,
  ON E.DEPT_ID = D.ID;          -- 사원의 부서번호와 부서 테이블의 ID를 연결(JOIN)합니다.
```

---

## 16. 서울 지역에 근무하는 사원에 각 사원의 이름과 근무하는 부서명을 나타내시오.

SQL

```
SELECT E.NAME,                  -- 사원 이름을 조회하고,
       D.NAME AS DEPT_NAME      -- 부서명을 DEPT_NAME으로 출력합니다.
FROM S_EMP E                    -- S_EMP 테이블을 E로 별칭,
JOIN S_DEPT D ON E.DEPT_ID = D.ID -- 사원의 부서번호와 부서 테이블의 ID를 연결(JOIN)하고,
JOIN S_REGION R ON D.REGION_ID = R.ID -- 부서의 지역번호와 지역 테이블의 ID를 연결(JOIN)합니다.
WHERE R.NAME = '서울';          -- 지역명이 '서울'인 경우만 선택합니다.
```

---

## 17. 직원 테이블(s_emp)과 급여 테이블(salgrade)을 JOIN하여 사원의 이름과 급여, 그리고 해당 급여등급을 나타내시오.

SQL

```
SELECT E.NAME,                  -- 사원 이름을 조회하고,
       E.SALARY,                -- 사원 급여를 조회하고,
       S.GRADE                  -- 급여등급(GRADE)을 조회합니다.
FROM S_EMP E                    -- S_EMP 테이블을 E로 별칭,
JOIN SALGRADE S                 -- SALGRADE 테이블을 S로 별칭,
  ON E.SALARY BETWEEN S.LOSAL AND S.HISAL; -- 사원의 급여가 SALGRADE의 LOSAL~HISAL 범위에 해당하는 등급만 JOIN합니다.
```

---

## 18. 직원(s_emp)테이블과 고객(s_customer)테이블에서 사원의 이름과 사번, 그리고 각 사원의 담당고객 이름을 나타내시오.

단, 고객에 해당하는 담당영업사원이 없더라도 모든 고객의 이름을 나타내고, 사번 순으로 정렬하시오.

SQL

```
SELECT C.NAME AS CUSTOMER_NAME, -- 고객 이름을 CUSTOMER_NAME으로 출력하고,
       E.ID AS EMP_ID,          -- 담당 영업사원의 사번을 EMP_ID로 출력하고,
       E.NAME AS EMP_NAME       -- 담당 영업사원의 이름을 EMP_NAME으로 출력합니다.
FROM S_CUSTOMER C               -- S_CUSTOMER 테이블을 C로 별칭,
LEFT JOIN S_EMP E               -- S_EMP 테이블을 E로 별칭, LEFT JOIN을 사용하여
  ON C.SALES_REP_ID = E.ID      -- 고객의 SALES_REP_ID와 사원의 ID를 연결합니다.
ORDER BY E.ID;                  -- 담당 영업사원 사번(EMP_ID) 순으로 정렬합니다.
```

---

## 19. 직원 중에 '김정기'와 같은 직책(title)을 가지는 사원의 이름과 직책, 급여, 부서번호를 나타내시오. (SELF JOIN을 사용할 것)

SQL

```
SELECT E2.NAME,                 -- 같은 직책을 가진 사원의 이름을 조회하고,
       E2.TITLE,                -- 같은 직책을 가진 사원의 직책을 조회하고,
       E2.SALARY,               -- 같은 직책을 가진 사원의 급여를 조회하고,
       E2.DEPT_ID               -- 같은 직책을 가진 사원의 부서번호를 조회합니다.
FROM S_EMP E1                   -- S_EMP 테이블을 E1로 별칭,
JOIN S_EMP E2                   -- S_EMP 테이블을 E2로 별칭,
  ON E1.TITLE = E2.TITLE        -- E1과 E2의 직책이 같은 경우만 JOIN,
WHERE E1.NAME = '김정기';       -- E1의 이름이 '김정기'인 경우만 선택합니다.
```

---

## 20. 가장 적은 평균급여를 받는 직책에 대해 그 직책과 평균급여를 나타내시오.

SQL

```
SELECT TITLE,                   -- 직책(TITLE)을 조회하고,
       AVG(SALARY) AS AVG_SALARY -- 해당 직책의 평균 급여를 AVG_SALARY로 출력합니다.
FROM S_EMP                      -- S_EMP 테이블에서
GROUP BY TITLE                  -- TITLE(직책)별로 그룹화하여 집계하고,
ORDER BY AVG_SALARY ASC         -- 평균 급여를 오름차순(ASC)으로 정렬하여
FETCH FIRST 1 ROW ONLY;         -- 가장 적은 평균 급여를 받는 직책 1개만 출력합니다. (Oracle 12c 이상)
```
---

## 21. S_EMP테이블에서 각 사원의 이름과 급여, 급여등급을 나타내시오.

급여가 4000만원 이상이면 A등급, 3000만원 이상이면 B등급, 2000만원 이상이면 C등급, 1000만원 이상이면 D등급, 1000만원 미만이면 E등급으로 나타내시오.

SQL

```
SELECT NAME,                    -- 사원 이름을 조회하고,
       SALARY,                  -- 사원 급여를 조회하고,
       CASE                     -- 급여에 따라 등급을 나눕니다.
         WHEN SALARY >= 4000 THEN 'A등급' -- 4000 이상이면 A등급,
         WHEN SALARY >= 3000 THEN 'B등급' -- 3000 이상이면 B등급,
         WHEN SALARY >= 2000 THEN 'C등급' -- 2000 이상이면 C등급,
         WHEN SALARY >= 1000 THEN 'D등급' -- 1000 이상이면 D등급,
         ELSE 'E등급'                    -- 그 외는 E등급
       END AS GRADE             -- 등급을 GRADE라는 별칭으로 출력합니다.
FROM S_EMP;                     -- S_EMP 테이블에서
```

---

## 22. 자신의 급여가 자신이 속한 부서의 평균 급여보다 적은 직원에 대해 이름, 급여, 부서번호를 출력하시오.

SQL

```
SELECT NAME,                    -- 사원 이름을 조회하고,
       SALARY,                  -- 사원 급여를 조회하고,
       DEPT_ID                  -- 사원의 부서번호를 조회합니다.
FROM S_EMP E1                   -- S_EMP 테이블을 E1로 별칭,
WHERE SALARY < (                -- 자신의 급여가
  SELECT AVG(SALARY)            -- 같은 부서의 평균 급여보다
  FROM S_EMP                    -- S_EMP 테이블에서
  WHERE DEPT_ID = E1.DEPT_ID    -- E1과 같은 부서(DEPT_ID)만 집계
);                              -- 작은 경우만 선택합니다.
```

---

## 23. 본인의 급여가 각 부서별 평균 급여 중 어느 한 부서의 평균급여보다 적은 직원을 출력하시오.

(ANY 사용)

SQL

```
SELECT NAME,                    -- 사원 이름을 조회하고,
       SALARY,                  -- 사원 급여를 조회하고,
       DEPT_ID                  -- 사원의 부서번호를 조회합니다.
FROM S_EMP                      -- S_EMP 테이블에서
WHERE SALARY < ANY (            -- 자신의 급여가
  SELECT AVG(SALARY)            -- 각 부서별 평균 급여 중
  FROM S_EMP                    -- S_EMP 테이블에서
  GROUP BY DEPT_ID              -- 부서별로 그룹화하여 집계한 값보다
);                              -- 작은 경우만 선택합니다.
-- ANY는 여러 값 중 하나라도 조건을 만족하면 참입니다.
```

---

## 24. 본인이 다른 사람의 관리자(manager_id)로 되어 있는 직원의 사번, 이름, 직책, 부서번호를 나타내시오. (exists 사용)

SQL

```
SELECT ID,                      -- 사원의 사번(ID)을 조회하고,
       NAME,                    -- 사원 이름을 조회하고,
       TITLE,                   -- 사원 직책을 조회하고,
       DEPT_ID                  -- 사원 부서번호를 조회합니다.
FROM S_EMP E1                   -- S_EMP 테이블을 E1로 별칭,
WHERE EXISTS (                   -- 다음 조건을 만족하는 경우만 선택합니다.
  SELECT 1
  FROM S_EMP E2                 -- S_EMP 테이블을 E2로 별칭,
  WHERE E2.MANAGER_ID = E1.ID   -- E2의 관리자(MANAGER_ID)가 E1의 사번(ID)인 경우
);
-- 즉, 자신이 다른 직원의 관리자 역할을 하고 있는 경우만 출력합니다.
```

---

## 25. 직원(s_emp) 테이블에서 이름을 사전순으로 정렬하여 5개의 데이터만 나타내시오.

SQL

```
SELECT *                        -- 모든 컬럼을 조회합니다.
FROM S_EMP                      -- S_EMP 테이블에서
ORDER BY NAME ASC               -- 이름(NAME)을 오름차순(사전순)으로 정렬하고,
FETCH FIRST 5 ROWS ONLY;        -- 상위 5개 행만 출력합니다. (Oracle 12c 이상)
```

---
# Oracle SQL 심화 학습

---

## I. 고급 SQL 쿼리 및 개념

### 1. 윈도우 함수 (Window Functions)

윈도우 함수는 `GROUP BY`와 유사하게 행의 집합에 대해 연산을 수행하지만, `GROUP BY`처럼 여러 행을 하나의 결과 행으로 그룹화하지 않습니다. 대신, 각 행에 대한 계산 결과를 해당 행에 그대로 반환합니다. 분석 및 순위 계산에 매우 유용합니다.

-   **주요 윈도우 함수:**
    -   `RANK()`: 순위를 매기며, 같은 값이면 같은 순위를 부여하고 다음 순위는 해당 개수만큼 건너뜁니다. (예: 1, 2, 2, 4)
    -   `DENSE_RANK()`: 같은 값에 같은 순위를 부여하지만, 다음 순위를 건너뛰지 않습니다. (예: 1, 2, 2, 3)
    -   `ROW_NUMBER()`: 순위와 상관없이 고유한 번호를 부여합니다. (예: 1, 2, 3, 4)
    -   `LEAD(컬럼, N)`: 현재 행에서 N번째 뒤에 있는 행의 컬럼 값을 가져옵니다.
    -   `LAG(컬럼, N)`: 현재 행에서 N번째 앞에 있는 행의 컬럼 값을 가져옵니다.
    -   `SUM() OVER (PARTITION BY ... ORDER BY ...)`: 파티션(그룹) 내에서 누적 합계를 계산합니다.

#### **실습 1: 부서별 급여 순위 매기기**

각 부서(dept_id) 내에서 직원들의 급여(salary)가 높은 순서대로 순위를 매겨 이름, 부서, 급여, 순위를 출력하시오.

```sql
SELECT
    NAME,
    DEPT_ID,
    SALARY,
    RANK() OVER (PARTITION BY DEPT_ID ORDER BY SALARY DESC) AS SALARY_RANK,
    DENSE_RANK() OVER (PARTITION BY DEPT_ID ORDER BY SALARY DESC) AS SALARY_DENSE_RANK,
    ROW_NUMBER() OVER (PARTITION BY DEPT_ID ORDER BY SALARY DESC) AS SALARY_ROW_NUM
FROM S_EMP;
```

---

### 2. 공통 테이블 표현식 (CTE - Common Table Expressions)

CTE는 `WITH` 절을 사용하여 정의하는 임시 결과 집합입니다. 복잡한 쿼리를 여러 개의 논리적인 부분으로 나누어 가독성을 높이고 재사용성을 증대시킵니다. 특히, 여러 단계의 서브쿼리가 필요한 경우에 유용합니다.

#### **실습 2: 부서별 평균 급여보다 많이 받는 직원 찾기 (CTE 활용)**

CTE를 사용하여 각 부서의 평균 급여를 먼저 계산한 후, 이 정보를 바탕으로 해당 부서의 평균 급여보다 많이 받는 직원들의 이름, 부서, 급여를 출력하시오.

```sql
WITH DEPT_AVG_SALARY AS (
    SELECT
        DEPT_ID,
        AVG(SALARY) AS AVG_SAL
    FROM S_EMP
    GROUP BY DEPT_ID
)
SELECT
    E.NAME,
    E.DEPT_ID,
    E.SALARY
FROM S_EMP E
JOIN DEPT_AVG_SALARY D
  ON E.DEPT_ID = D.DEPT_ID
WHERE E.SALARY > D.AVG_SAL;
```

---

### 3. 집합 연산자 (Set Operators)

여러 `SELECT` 문의 결과 집합을 하나로 결합하는 데 사용됩니다.

-   `UNION`: 두 결과 집합을 합치되, 중복된 행은 제거합니다.
-   `UNION ALL`: 두 결과 집합을 합치되, 중복된 행을 그대로 포함합니다. (성능상 이점)
-   `INTERSECT`: 두 결과 집합에 모두 존재하는 행만 반환합니다. (교집합)
-   `MINUS`: 첫 번째 결과 집합에는 있지만 두 번째 결과 집합에는 없는 행을 반환합니다. (차집합)

#### **실습 3: 특정 부서 직원과 특정 직책 직원 합치기**

101번 부서에 속한 직원과 직책이 '과장'인 모든 직원의 목록을 중복을 제거하여 출력하시오.

```sql
SELECT ID, NAME, TITLE, DEPT_ID FROM S_EMP WHERE DEPT_ID = 101
UNION
SELECT ID, NAME, TITLE, DEPT_ID FROM S_EMP WHERE TITLE = '과장';
```

---

## II. 데이터 정의어(DDL) 및 데이터 조작어(DML) 심화

### 1. 테이블 관리 (Table Management)

-   `CREATE TABLE`: 새로운 테이블을 생성합니다.
-   `ALTER TABLE`: 기존 테이블의 구조를 변경합니다. (`ADD`, `MODIFY`, `DROP COLUMN`)
-   `TRUNCATE TABLE`: 테이블의 모든 행을 삭제하며, `DELETE`보다 빠르고 롤백이 불가능합니다.

#### **실습 4: 제약조건을 포함한 신규 테이블 생성**

```sql
CREATE TABLE S_PROJECT (
    ID NUMBER(7) PRIMARY KEY, -- 프로젝트 ID (기본 키)
    NAME VARCHAR2(50) NOT NULL, -- 프로젝트명 (NULL 불허)
    START_DATE DATE, -- 시작일
    END_DATE DATE, -- 종료일
    MANAGER_ID NUMBER(7) REFERENCES S_EMP(ID) -- 담당 매니저 (S_EMP 테이블의 ID 참조)
);
```

### 2. DML 심화

-   `MERGE`: 조건에 따라 데이터가 있으면 `UPDATE`, 없으면 `INSERT`를 수행하여 테이블을 병합합니다.

#### **실습 5: MERGE를 이용한 직원 정보 동기화**

`S_EMP_BACKUP` 테이블에 `S_EMP` 테이블의 데이터를 동기화합니다. 사번(ID)이 존재하면 이름과 급여를 업데이트하고, 존재하지 않으면 새로운 직원 정보를 추가합니다.

```sql
MERGE INTO S_EMP_BACKUP B
USING S_EMP E
ON (B.ID = E.ID)
WHEN MATCHED THEN
    UPDATE SET
        B.NAME = E.NAME,
        B.SALARY = E.SALARY
WHEN NOT MATCHED THEN
    INSERT (ID, NAME, TITLE, DEPT_ID, SALARY)
    VALUES (E.ID, E.NAME, E.TITLE, E.DEPT_ID, E.SALARY);
```

---

## III. 트랜잭션 제어 및 Oracle 주요 함수

### 1. 트랜잭션 제어어 (TCL)

-   `COMMIT`: 모든 변경사항을 데이터베이스에 영구적으로 저장합니다.
-   `ROLLBACK`: 마지막 `COMMIT` 이후의 모든 변경사항을 취소합니다.
-   `SAVEPOINT`: 트랜잭션 내에 저장점을 만들어, 전체가 아닌 특정 지점까지만 `ROLLBACK` 할 수 있도록 합니다.

### 2. Oracle 주요 내장 함수

-   **NULL 처리**: `NVL(A, B)` (A가 NULL이면 B 반환), `NVL2(A, B, C)` (A가 NULL이 아니면 B, NULL이면 C 반환), `COALESCE(A, B, ...)` (첫 번째로 NULL이 아닌 값 반환)
-   **조건부 함수**: `DECODE(값, 조건1, 결과1, 조건2, 결과2, ..., 기본결과)` (CASE 문과 유사하게 동작)

#### **실습 6: NVL과 DECODE를 활용한 보너스 계산**

직원의 직책(title)에 따라 보너스를 계산하여 이름, 직책, 급여, 보너스를 출력합니다. '부장'은 급여의 20%, '과장'은 15%, 나머지는 10%를 보너스로 지급하며, MANAGER_ID가 없는 경우(NULL) 보너스를 0으로 처리합니다.

```sql
SELECT
    NAME,
    TITLE,
    SALARY,
    NVL(DECODE(TITLE,
        '부장', SALARY * 0.2,
        '과장', SALARY * 0.15,
        SALARY * 0.1
    ), 0) AS BONUS
FROM S_EMP;
```

---

## IV. 데이터베이스 객체 및 성능

### 1. 뷰 (View)

뷰는 하나 이상의 테이블을 기반으로 만들어진 가상의 테이블입니다. 복잡한 쿼리를 단순화하고, 특정 데이터만 선택적으로 노출하여 보안을 강화하는 데 사용됩니다.

#### **실습 7: 서울 지역 근무 직원 뷰 생성**

서울 지역에 근무하는 직원들의 사번, 이름, 직책, 부서명, 지역명을 포함하는 뷰를 생성합니다.

```sql
CREATE OR REPLACE VIEW VW_EMP_SEOUL AS
SELECT
    E.ID,
    E.NAME,
    E.TITLE,
    D.NAME AS DEPT_NAME,
    R.NAME AS REGION_NAME
FROM S_EMP E
JOIN S_DEPT D ON E.DEPT_ID = D.ID
JOIN S_REGION R ON D.REGION_ID = R.ID
WHERE R.NAME = '서울';

-- 뷰 조회
SELECT * FROM VW_EMP_SEOUL;
```

### 2. 인덱스 (Index)

인덱스는 테이블의 특정 컬럼에 대한 검색 속도를 향상시키기 위한 데이터 구조입니다. `WHERE` 절이나 `JOIN` 조건에서 자주 사용되는 컬럼에 생성하면 효과적입니다.

#### **실습 8: 이름(NAME) 컬럼에 인덱스 생성**

`S_EMP` 테이블의 `NAME` 컬럼으로 직원을 검색하는 경우가 많다고 가정하고, 해당 컬럼에 인덱스를 생성합니다.

```sql
CREATE INDEX IDX_EMP_NAME ON S_EMP(NAME);
```
