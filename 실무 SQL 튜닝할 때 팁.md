
### 실행계획 해석의 중요성

실행계획은 SQL 튜닝의 시작점이라고 생각한다.

SQL

```
EXPLAIN PLAN FOR
SELECT * FROM 주문 o, 고객 c
WHERE o.고객ID = c.고객ID
AND o.주문일자 >= '2025-01-01';

SELECT * FROM TABLE(DBMS_XPLAN.DISPLAY);
```

실행계획 해석 시 주의점

- 실행 순서는 들여쓰기 순으로 아래에서 위로
- 비용이 가장 큰 부분 먼저 확인
- 예상 행 수(Cardinality)와 실제 행 수 비교

### 통계 정보의 중요성

옵티마이저는 통계 정보를 기반으로 결정한다.

SQL

```
-- 통계 정보 수집
ANALYZE TABLE 주문 COMPUTE STATISTICS;
-- 또는
EXEC DBMS_STATS.GATHER_TABLE_STATS('스키마명', '주문');

-- 컬럼 히스토그램 수집
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  '스키마명', '주문', 
  method_opt => 'FOR COLUMNS 주문상태 SIZE 10'
);
```

통계 수집 주기:

- OLTP: 주 1회 (업무 외 시간)
- DW: 데이터 로드 후 즉시

### 힌트 사용의 장단점

**장점**

- 특정 상황에서 옵티마이저보다 정확한 판단 가능
- 성능 예측 가능성 증가

**단점**

- 데이터 변화에 적응하지 못함
- 유지보수성 저하
- 과도한 의존은 위험

SQL

```
-- 힌트 사용 예시
SELECT /*+ LEADING(c o) USE_NL(o) INDEX(c idx_cust_region) */
       o.주문번호, c.고객명, o.금액
FROM 주문 o, 고객 c
WHERE o.고객ID = c.고객ID
  AND c.지역 = '서울';
```

### 바인드 변수 사용과 리터럴 SQL의 균형

**바인드 변수 장점**

- 하드 파싱 감소
- 라이브러리 캐시 효율성 증가
- 실행계획 안정성

**리터럴 SQL이 필요한 경우**

- 데이터 분포가 극단적으로 불균등할 때
- 특정 값에 최적화된 실행계획이 필요할 때

SQL

```
-- 히스토그램이 있으면 바인드 변수 피킹 가능
EXEC DBMS_STATS.GATHER_TABLE_STATS(
  '스키마명', '주문', 
  method_opt => 'FOR COLUMNS 주문상태 SIZE 10'
);

-- 옵티마이저가 바인드 변수 값을 알 수 있음
SELECT * FROM 주문 WHERE 주문상태 = :status;
```

### 파티셔닝 활용

대용량 테이블의 효율적 관리:

SQL

```
-- 범위 파티션
CREATE TABLE 주문 (
  주문ID NUMBER,
  주문일자 DATE,
  고객ID NUMBER,
  금액 NUMBER
)
PARTITION BY RANGE (주문일자) (
  PARTITION p2024_q1 VALUES LESS THAN (TO_DATE('2024-04-01','YYYY-MM-DD')),
  PARTITION p2024_q2 VALUES LESS THAN (TO_DATE('2024-07-01','YYYY-MM-DD')),
  PARTITION p2024_q3 VALUES LESS THAN (TO_DATE('2024-10-01','YYYY-MM-DD')),
  PARTITION p2024_q4 VALUES LESS THAN (TO_DATE('2025-01-01','YYYY-MM-DD')),
  PARTITION p2025_q1 VALUES LESS THAN (TO_DATE('2025-04-01','YYYY-MM-DD')),
  PARTITION p_max VALUES LESS THAN (MAXVALUE)
);

-- 파티션 프루닝
SELECT * FROM 주문
WHERE 주문일자 BETWEEN '2024-01-01' AND '2024-04-31';
-- p2024_q1 파티션만 접근
```

### 병렬 처리 활용

대량 데이터 처리 시간 단축:

SQL

```
-- 병렬 쿼리 실행
SELECT /*+ PARALLEL(8) */ 
       고객유형, COUNT(*), AVG(구매금액)
FROM 고객이력
WHERE 구매일자 BETWEEN '2025-01-01' AND '2025-12-31'
GROUP BY 고객유형;

-- 병렬 인덱스 생성
CREATE INDEX /*+ PARALLEL(8) */ idx_cust_date
ON 고객이력(구매일자);
```

병렬 처리 고려사항:

- CPU 코어 수를 고려한 병렬도 설정
- 소량 데이터는 오히려 오버헤드
- 병렬 처리 간 자원 경합 주의

### 실제 업무에서의 SQL 튜닝 프로세스

1. **문제 식별**
    
    - AWR/ASH 리포트로 고부하 SQL 식별
    - 사용자 응답 시간 불만 접수
    - 시스템 모니터링 경고
    
2. **실행계획 분석**
    
    - 현재 실행계획 확인
    - 인덱스, 조인 방식, 테이블 액세스 방식 분석
    - 예상치 못한 Full Table Scan, 소트 연산 확인
    
3. **튜닝 포인트 식별**
    
    - 인덱스 추가/변경 필요성
    - 조인 순서 최적화
    - 불필요한 소트 제거
    - 통계 정보 최신화
    
4. **SQL 재작성 또는 힌트 추가**
    
    - SQL 로직 개선
    - 적절한 힌트 적용
    - 인덱스 변경 제안
    
5. **테스트 및 검증**
    
    - 개발/테스트 환경에서 변경 효과 검증
    - 성능 지표 측정 (수행 시간, I/O, CPU 사용량)
    - 부작용 확인
    
6. **적용 및 모니터링**
    
    - 변경사항 적용
    - 지속적 모니터링
    - 피드백 수집
    

### 최신 데이터베이스 버전의 자동 튜닝 기능

최신 DBMS들은 자동 튜닝 기능을 제공한다:

SQL

```
-- Oracle SQL Tuning Advisor
DECLARE
  task_name VARCHAR2(30);
  sql_id    VARCHAR2(13) := 'abcd1234efgh';
BEGIN
  task_name := DBMS_SQLTUNE.CREATE_TUNING_TASK(
    sql_id => sql_id,
    scope => DBMS_SQLTUNE.SCOPE_COMPREHENSIVE,
    time_limit => 60,
    task_name => 'tuning_task_' || sql_id
  );
  DBMS_SQLTUNE.EXECUTE_TUNING_TASK(task_name);
END;
/

-- 결과 확인
SELECT DBMS_SQLTUNE.REPORT_TUNING_TASK('tuning_task_abcd1234efgh') FROM DUAL;
```

하지만 자동 튜닝 기능에만 의존하지 말고, 기본 원리를 이해하는 것이 중요하다고 생각한다. 
특히 SQL 처리 과정, 인덱스 구조, 조인 메커니즘, 소트 연산 원리를 제대로 이해하면 어떤 상황에서도 
문제를 해결할 수 있다고 생각한다.

실무에서 이론을 바탕으로 실제 환경에 맞게 적용하고 끊임없이 테스트와 검증을 통해 경험을 쌓는 것이 중요하다고 생각한다.