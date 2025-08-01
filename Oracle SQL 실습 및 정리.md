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

