## 7.1 통계정보와 비용 계산 원리

### 7.1.1 선택도와 카디널리티

- 선택도(Selectivity): 어떤 조건이 전체 행 중에서 몇 퍼센트를 만족하는지 비율. 0~1 사이 값.
- 카디널리티(Cardinality): 해당 단계에서 예상되는 결과 행 수. 보통 “입력 행 수 × 선택도”로 계산.
- 기본 공식
    - 단일 컬럼 동등 비교: selectivity ≈ 1 / NDV(해당 컬럼의 distinct 개수). 히스토그램 없고 균등 분포 가정일 때.
    - 범위 조건: selectivity ≈ (요청 범위 길이) / (전체 도메인 길이). 실제론 분포와 히스토그램 영향.
    - 결합 조건(AND): 컬럼 독립 가정 시 각 선택도의 곱.
    - 결합 조건(OR): 각 선택도의 합에서 겹치는 부분(교집합)을 뺀 값. 옵티마이저는 근사치를 쓴다.
    - 조인 선택도(동등 조인): selectivity ≈ 1 / max(NDV(A), NDV(B)). 결과 카디널리티 = 외부×내부×선택도.
- 한계 포인트
    - 컬럼 간 상관관계가 크면(예: 지역과 도시), 독립 가정이 틀려서 카디널리티가 크게 빗나간다.
    - 데이터가 편향(skew)되어 있으면 균등 가정이 깨진다. 이럴 때 히스토그램/확장 통계가 필요했다.

예제: 편향과 히스토그램 영향

SQL

```
-- 편향된 데이터: status='A' 99%, 'B' 1%
create table t_skew (id number, status varchar2(1));
insert /*+ append */ into t_skew
select level, case when level <= 990000 then 'A' else 'B' end
from dual connect by level <= 1000000;
commit;

-- 기본 통계(히스토그램 자동)
exec dbms_stats.gather_table_stats(user, 'T_SKEW', method_opt=>'FOR COLUMNS SIZE AUTO', estimate_percent=>dbms_stats.auto_sample_size);

-- 히스토그램 없이 가정하면 'A'와 'B' 모두 50%로 볼 수 있지만(=NDV 2),
-- 히스토그램 덕분에 'B'는 1%로 추정 가능.
select /*+ gather_plan_statistics */ count(*)
from t_skew
where status='B';

select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));
-- E-Rows(예상)와 A-Rows(실제) 비교로 추정 정확도 확인
```

예제: 컬럼 그룹(확장 통계)로 상관성 반영

SQL

```
create table t_corr as
select level id,
       case when mod(level,10)=0 then 'KR' else 'US' end as country,
       case when mod(level,10)=0 then 'Seoul' else 'NY' end as city
from dual connect by level <= 100000;

-- 단일 컬럼만 보면 'country=KR' 10%, 'city=Seoul' 10% → 독립 가정이면 AND는 1%로 추정.
-- 실제로는 KR↔Seoul이 100% 상관이라 실제 선택도는 10%.
exec dbms_stats.gather_table_stats(user,'T_CORR');

-- 확장 통계 생성(컬럼 그룹)
select dbms_stats.create_extended_stats(user,'T_CORR','(COUNTRY, CITY)') from dual;
exec dbms_stats.gather_table_stats(user,'T_CORR', method_opt=>'FOR ALL COLUMNS SIZE AUTO');

-- 이제 where country='KR' and city='Seoul'의 카디널리티 추정이 개선된다.
```

---

### 7.1.2 통계정보

- 테이블 통계: num_rows, blocks, avg_row_len. 실행 비용의 기본 단위가 된다.
- 컬럼 통계: num_distinct(NDV), num_nulls, density, histogram 유형(12c+는 hybrid/top-frequency 포함).
- 인덱스 통계: blevel(루트~리프 깊이), leaf_blocks, clustering_factor(테이블 순서와 인덱스 키 정렬 일치도), distinct_keys.
- 파티션 통계: 파티션별/서브파티션별 통계, 증분 통계(synopsis)로 빠른 갱신 가능.
- 시스템 통계(System stats): CPU 속도, I/O 처리량, MBRC(멀티블록 읽기), 병렬 비용 모델 등. CPU cost 모델이 기본이라 시스템 통계가 중요하다.
- 고정 객체 통계(Fixed object stats): X$ 테이블 대상. 드물지만 라이브러리 캐시/딕셔너리 접근 많은 시스템에서 중요.

실무에서 내가 주로 쓰는 DBMS_STATS 패턴

SQL

```
-- 테이블 통계 수집(+ 히스토그램 자동 판단)
exec dbms_stats.gather_table_stats(user, 'T', estimate_percent=>dbms_stats.auto_sample_size, method_opt=>'FOR ALL COLUMNS SIZE AUTO', cascade=>true, degree=>4);

-- 파티션 테이블: 증분 통계 활성화
exec dbms_stats.set_table_prefs(user,'SALES','INCREMENTAL','TRUE');
exec dbms_stats.set_table_prefs(user,'SALES','GRANULARITY','AUTO');
exec dbms_stats.gather_table_stats(user,'SALES', partname=>'P202502');

-- 컬럼 그룹(확장 통계)
select dbms_stats.create_extended_stats(user,'ORDERS','(CUST_ID, REGION)') from dual;
exec dbms_stats.gather_table_stats(user,'ORDERS');

-- 통계 잠금/해제(예상치 못한 자동 갱신 방지)
exec dbms_stats.lock_table_stats(user,'CRITICAL_T');
exec dbms_stats.unlock_table_stats(user,'CRITICAL_T');

-- 통계 공개 지연(PUBLISH=FALSE로 테스트 후 반영)
exec dbms_stats.set_table_prefs(user,'T','PUBLISH','FALSE');
exec dbms_stats.gather_table_stats(user,'T');
exec dbms_stats.publish_pending_stats(user,'T');
```

메모

- 자동 통계는 대부분 잘 동작하지만, 편향/상관관계가 강한 컬럼은 METHOD_OPT, 확장 통계를 명시적으로 손봐야 했다.
- 시스템 통계가 없으면 디폴트 가정에 의존한다. OLTP/OLAP 특성에 맞게 수집을 고려했다.

---

### 7.1.3 비용 계산 원리

- CBO(비용 기반 옵티마이저)는 “작업량 추정”으로 비용을 비교한다. 단위는 시간이 아니라 추상화된 cost이며, I/O + CPU 모델 기반.
- 접근 경로 비용(대략적 직관)
    - Full Table Scan: cost ≈ blocks / MBRC + CPU. 대량 범위/병렬 시 유리.
    - Index Range Scan: cost ≈ blevel + 선택된 리프 블록 수 + 테이블 랜덤 액세스 비용. 클러스터링 팩터가 나쁘면 테이블 랜덤 I/O가 커진다.
- 조인 방식 선택
    - Nested Loops: 외부가 아주 작고 내부가 인덱스로 빠르게 찾을 때 유리.
    - Hash Join: 큰 집합끼리 조인할 때 유리. 해시 빌드/프로브 비용 + 메모리 사용.
    - Sort Merge Join: 정렬 비용이 들지만, 양쪽이 이미 정렬/인덱스 스캔이면 유리할 수 있음.
- 병렬 비용: 병렬도(DOP)만큼 I/O/CPU를 나눠 갖는 가정으로 비용을 낮춰 잡는다(오버헤드도 존재).
- 계획 검증 팁: DBMS_XPLAN으로 E-Rows(예상)와 A-Rows(실제)를 비교한다. 큰 차이가 나면 통계/히스토그램/상관성 문제일 가능성이 크다.

예제: FTS vs 인덱스 스캔 비용 감

SQL

```
-- 준비: 대량 테이블 + 인덱스
create table t_cost as
select level id, trunc(dbms_random.value(1, 10000)) k, rpad('x',100,'x') pad
from dual connect by level <= 1000000;
create index t_cost_k on t_cost(k);
exec dbms_stats.gather_table_stats(user,'T_COST',cascade=>true);

-- 특정 k 값은 매우 선택적
select /*+ gather_plan_statistics */ count(*)
from t_cost where k = 42;

select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));
-- NL+Index Range Scan일지, FTS일지 비교

-- 넓은 범위는 보통 FTS
select /*+ gather_plan_statistics */ count(*)
from t_cost where k between 1 and 9000;
select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));
```

메모

- 클러스터링 팩터가 나쁘면(테이블 물리 순서와 인덱스 키 순서가 뒤섞임) 인덱스 스캔 후 테이블 랜덤 I/O가 폭증한다. 이럴 땐 FTS가 선택되기도 한다.

---

## 7.2 옵티마이저에 대한 이해

### 7.2.1 옵티마이저 종류

- 규칙 기반 옵티마이저(RBO)는 역사 속으로 사라졌다. 지금은 전적으로 비용 기반 옵티마이저(CBO)다.
- CBO 내부 구성(내가 이해한 흐름)
    - Query Transformer: 서브쿼리 unnesting, view merging, predicate pushdown, OR-expansion, star transformation 등 변형을 시도.
    - Estimator: 통계/히스토그램으로 선택도·카디널리티를 추정.
    - Plan Generator: 다양한 조인 순서/방식을 탐색하고 비용이 가장 낮은 계획을 고른다.
- 11g 이후: Bind Peeking + Adaptive Cursor Sharing(ACS)로 바인드 값 편향에 대응. 12c: Adaptive Plans/Statistics, SQL Plan Directives(버전에 따라 제한/비활성 기본). 19c 기준으로 Adaptive Plans는 유지, Adaptive Statistics/Directives는 보수적으로 동작.

---

### 7.2.2 옵티마이저 모드

- optimizer_mode
    - ALL_ROWS: 전체 처리량 최적화(기본). 배치/리포트 성격 유리.
    - FIRST_ROWS_n: 초기 응답 최적화(n은 1,10,100,1000 등). OLTP 인터랙션에서 첫 페이지 빨리 주는 데 유리.
- 힌트로도 제어 가능: ALL_ROWS, FIRST_ROWS(n).
- 병렬 DOP, star_transformation_enabled, parallel_degree_policy 같은 세팅도 계획에 영향.

예제: 모드 차이 체감

SQL

```
alter session set optimizer_mode=all_rows;
select /*+ gather_plan_statistics */ * from t_cost where k between 1 and 500 order by pad;
select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));

alter session set optimizer_mode=first_rows_10;
select /*+ gather_plan_statistics */ * from t_cost where k between 1 and 500 order by pad;
select * from table(dbms_xplan.display_cursor(null,null,'ALLSTATS LAST'));
-- 초기 응답 최적화 모드에서 NL + 인덱스 스캔이 선택될 여지가 있다.
```

---

### 7.2.3 옵티마이저에 영향을 미치는 요소

- 통계/히스토그램/확장 통계: 선택도와 조인 카디널리티의 기반.
- 시스템 통계: I/O/CPU 비중. 미수집 시 디폴트 가정으로 비용 계산.
- 바인드 피킹 + Adaptive Cursor Sharing: 처음 바인드로 계획을 잡고, 편향이 관찰되면 바인드 값 범위에 따라 child cursor를 분기.
- 동적 샘플링(Dynamic Sampling): 통계가 없거나 신뢰 낮으면 실행 시 샘플링으로 추정 보정. 힌트로 강제 가능.
- 제약조건/인덱스 메타데이터: NOT NULL, CHECK, FK, RELY/VALIDATE가 변형과 카디널리티에 영향. FK 인덱스 유무는 조인/락 경합에도 중요.
- 파티션 정보: 파티션 프루닝/파티션 와이즈 조인/집계를 가능하게 하며 비용에 큰 영향.
- 함수 기반 인덱스/가상 컬럼: 표현식 사용시 SARGable하게 만들어 인덱스 사용 여지를 줌.
- 힌트/프로파일/베이스라인/패치
    - Hints: 개발자가 직접 계획 유도.
    - SQL Profile: 옵티마이저 통계를 보정(카디널리티/선택도 조정 등).
    - SQL Plan Baseline: 승인된 계획만 사용하도록 안정화.
    - SQL Patch: 특정 SQL에 힌트를 주입.

예제: 동적 샘플링, 힌트, SPM

SQL

```
-- 동적 샘플링: 통계 없는 테이블에 유용
select /*+ dynamic_sampling(11) */ count(*) from t_nostats where col = :b;

-- 힌트로 조인 순서/방식 유도
select /*+ leading(o c) use_hash(c) */ *
from orders o join customers c on c.id = o.cust_id
where c.country = 'KR';

-- SQL Plan Baseline 캡처(간단 예)
-- 1) 원하는 힌트로 좋은 계획을 만든 뒤
-- 2) 커서 캐시에서 베이스라인으로 로드
declare
  l_cnt pls_integer;
begin
  l_cnt := dbms_spm.load_plans_from_cursor_cache(
             sql_id => '<원하는 SQL_ID>');
end;
/
```

---

### 7.2.4 옵티마이저의 한계

- 컬럼 독립 가정: 실제로는 상관관계가 강한 경우가 많아 카디널리티가 어긋난다. 확장 통계를 쓰지 않으면 해결이 어렵다.
- 데이터 편향(skew): 히스토그램이 없거나 바인드 기반(값을 모름)이면 균등 가정으로 오판하기 쉽다.
- 사용자 정의 함수/카탈로그 정보 부족: 비용/선택도를 추정하기 어렵다. 함수가 non-deterministic이면 더 보수적으로 본다.
- 바인드 피킹의 양날의 검: 첫 실행 값에 맞춘 계획이 이후 다른 값엔 부적합할 수 있다. ACS가 있더라도 완벽하지 않다.
- 변형 제한: 뷰 머지/unnesting이 특정 힌트, 함수 사용, 권한/머티리얼라이즈드 뷰 등으로 제한될 수 있다.

메모

- E-Rows와 A-Rows가 크게 다르면(특히 몇 배~몇 십 배) 통계/히스토그램/확장 통계를 우선 점검하고, 필요시 프로파일/힌트/베이스라인으로 보정했다.

---

### 7.2.5 개발자의 역할

- SARGable한 조건으로 쓰기: 컬럼에 함수 씌우지 않기(to_char(date), upper(col) 등). 불가피하면 함수 기반 인덱스나 가상 컬럼로 보완.
- 바인드 변수 사용: 파싱 비용/플랜 캐시 안정성. 다만 편향 컬럼은 히스토그램/ACS와 함께 고려.
- 인덱스/파티션 설계: 자주 조인/필터되는 컬럼 우선, FK 인덱스 유지, 로컬 인덱스 선호(관리성), 클러스터링 팩터 고려.
- 통계 전략: 자동 통계 기본 + 편향 컬럼은 히스토그램/확장 통계. 파티션 테이블은 증분 통계.
- 계획 검증 루틴화: DBMS_XPLAN으로 E/A Rows 비교, SQL Monitor로 병목 파악, AWR/ASH로 시스템 관점 점검.
- 안정화 도구: 반복적으로 중요 SQL은 SQL Plan Baseline으로 통제. 힌트 남발 대신 근본 원인(통계/설계)을 먼저 고친다.

예제: 함수로 비SARGable → 함수 기반 인덱스로 보완

SQL

```
create table t_date (id number, dt date);
create index t_date_idx on t_date(dt); -- 일반 인덱스

-- 비SARGable: 인덱스 활용 어렵다
select * from t_date where trunc(dt) = date '2025-01-01';

-- 보완: 가상 컬럼 + 함수 기반 인덱스
alter table t_date add dt_d as (trunc(dt));
create index t_date_d_idx on t_date(dt_d);

-- 이제 SARGable
select * from t_date where dt_d = date '2025-01-01';
```

---

### 7.2.6 튜닝 전문가 되는 공부방법

- 반복 가능한 실험 환경 만들기: 데이터 생성 스크립트, 통계 수집, 다양한 바인드 값/분포를 쉽게 바꿔볼 수 있게 준비.
- 관찰 포인트 고정
    - DBMS_XPLAN.DISPLAY_CURSOR(‘ALLSTATS LAST’)로 E-Rows vs A-Rows를 항상 확인.
    - SQL Monitor(enterprise)로 실시간 병목 구간 파악.
    - AWR/ASH에서 대기 이벤트/자원 사용 흐름 확인.
- 원인 우선 접근
    1. 카디널리티 오차 확인(E vs A).
    2. 통계/히스토그램/확장 통계 점검 및 보완.
    3. 설계(인덱스/파티션/데이터 모델) 점검.
    4. 불가피할 때 힌트/프로파일/베이스라인/패치.
- 고급 도구 익히기
    - 10053 trace(코스트 계산/변형 결정 과정), SQLT(SQLTXPLAIN), SQL Patch/Profiles/SPM API.
- 추천 레퍼런스
    - Oracle Concepts/Performance Tuning Guide
    - Tom Kyte(Ask Tom), Jonathan Lewis(CBO/Stats/Skew), Kerry Osborne(SQL Tuning)

미니 실습 아이디어

- 편향 컬럼 + 바인드 → 히스토그램/ACS 전후 플랜 비교
- 컬럼 상관관계 → 확장 통계 전후 카디널리티 비교
- 클러스터링 팩터 극단(정렬 insert vs 랜덤 insert) → 인덱스 스캔 효율 비교
- 시스템 통계 수집 전후 비용 변화 비교
- 동일 SQL의 모드/힌트/병렬 바꿔가며 플랜과 실행시간 비교

---

## 실전 체크리스트

- E-Rows vs A-Rows부터 본다. 크게 어긋나면 통계/히스토그램/확장 통계를 먼저 손본다.
- 바인드 편향이 의심되면 히스토그램과 ACS를 확인한다. 필요한 경우 리터럴로 hard parse 테스트도 해본다.
- FK 인덱스, 조인/필터 컬럼 인덱스, 파티션 프루닝 가능성, 클러스터링 팩터를 점검한다.
- 파티션 테이블은 증분 통계, 로컬 인덱스, 파티션 와이즈 조인/집계의 활용을 염두에 둔다.
- 함수로 인덱스 사용을 막고 있지 않은지 확인한다. 필요시 함수 기반 인덱스/가상 컬럼을 쓴다.
- 계획이 출렁이면 베이스라인으로 안정화한다. 단, 설계/통계 문제는 근본적으로 수정한다.
- 동적 샘플링/시스템 통계/병렬 정책/optimizer_mode 등 환경 설정이 계획에 끼치는 영향도 체크한다.