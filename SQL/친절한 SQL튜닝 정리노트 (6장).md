## 6.1 기본 DML 튜닝

### 6.1.1 DML 성능에 영향을 미치는 요소

- 인덱스가 많으면 DML이 느려진다. 행을 바꿀 때마다 관련 인덱스 키도 수정해야 하기 때문이다. 넓은 컬럼 인덱스나 대량 갱신일수록 영향이 커진다.
- 제약조건과 트리거는 대량 DML에서 비용이 크게 나온다. 행마다 검사와 실행이 붙기 때문에, 필요에 따라 일시적으로 비활성화하고 끝난 뒤에 검증하는 전략이 유리했다.
- redo/undo는 변경량에 비례해서 늘어난다. 대량 작업에서 redo를 줄이려면 direct-path insert와 NOLOGGING 조합이 효과적이었다. 다만 복구 정책을 명확히 정해두는 것이 필요했다.
- 바인드 변수를 쓰지 않으면 하드 파싱이 과도하게 발생한다. 공유 풀 경합과 파싱 비용이 누적되므로 같은 형태의 SQL은 바인드로 재사용하는 것이 기본이라고 이해했다.
- 커밋을 자주 하면 매우 느려진다. 커밋은 LGWR redo flush를 강제하기 때문에 매 행 커밋 같은 패턴은 피해야 했다. 수천~수만 건 단위 배치 커밋이 안정적이었다.
- 네트워크 라운드 트립이 많으면 느려진다. 애플리케이션에서 한 행씩 DML을 호출하는 방식은 반드시 피하고, 집합 기반 처리나 최소한 배치 호출을 적용하는 것이 효과적이었다.

예제: 인덱스 유지 비용 체감

SQL

```
create table t_big as
select level id, rpad('x',100,'x') pad
from dual connect by level <= 1e6;

create index t_big_idx1 on t_big(id);
create index t_big_idx2 on t_big(pad);

-- 인덱스 2개 유지 상태에서 대량 UPDATE
set timing on
update t_big set pad = pad where id between 1 and 500000;

-- 보조 인덱스 제거 후 동일 UPDATE 수행
drop index t_big_idx2;
update t_big set pad = pad where id between 1 and 500000;
```

내가 테스트했을 때는 보조 인덱스를 제거한 후가 확실히 빨랐다. 각 행의 인덱스 유지 비용이 사라진 영향이라고 본다.

---

### 6.1.2 데이터베이스 Call과 성능

- Oracle과의 통신은 Parse/Execute/Fetch 호출로 이뤄진다. 호출 횟수가 많을수록 느려지므로, 집합(Set) 단위 처리와 바인드 변수 재사용으로 파싱을 줄이는 것이 중요했다.
- 커밋은 배치 단위로 모아서 해야 했다. 매 행 커밋은 redo flush가 너무 잦아져 비용이 과도하게 커졌다.

예제: 매 행 커밋 vs 배치 커밋

plsql

```
-- 느린 패턴: 매 행 커밋
begin
  for i in 1..100000 loop
    insert into t_big values(i, rpad('x',100,'x'));
    commit;
  end loop;
end;
/

-- 권장 패턴: 배치 커밋
begin
  for i in 1..100000 loop
    insert into t_big values(i, rpad('x',100,'x'));
    if mod(i, 10000) = 0 then commit; end if;
  end loop;
  commit;
end;
/
```

같은 적재량이라도 배치 커밋이 훨씬 빨랐다. JDBC 배치 등 클라이언트에서도 동일한 원리로 최적화할 수 있었다.

---

### 6.1.3 Array Processing 활용

- PL/SQL에서 SQL을 반복 호출하면 SQL-PL/SQL 컨텍스트 전환 비용이 커진다. BULK COLLECT와 FORALL로 전환 횟수를 크게 줄일 수 있었다.
- 메모리 사용을 안정화하려면 BULK COLLECT에 LIMIT를 주는 것이 좋았다. 부분 실패를 분리하려면 SAVE EXCEPTIONS가 유용했다.

예제: 벌크 처리 + LIMIT + 예외 수집

plsql

```
declare
  cursor c is select id from t_big where id <= 100000;
  type t_ids is table of number index by pls_integer;
  v_ids t_ids;
begin
  open c;
  loop
    fetch c bulk collect into v_ids limit 10000;  -- PGA 사용량 제어
    exit when v_ids.count = 0;

    begin
      forall i in indices of v_ids save exceptions
        update t_big
           set pad = pad || '!'
         where id = v_ids(i);
    exception
      when others then
        -- 실패 건 로깅(테이블/파일/테EMP 테이블 등)
        null;
    end;
  end loop;
  close c;
  commit;
end;
/
```

LIMIT를 주면 메모리가 안정적으로 유지되었고, SAVE EXCEPTIONS로 일부 실패가 있어도 전체 배치를 진행할 수 있었다.

---

### 6.1.4 인덱스 및 제약 해제를 통한 대량 DML 튜닝

- 대량 INSERT/UPDATE/DELETE를 보조 인덱스를 유지한 채 진행하면 행마다 인덱스 키를 갱신해야 해서 시간이 오래 걸렸다. 불필요 인덱스를 일시적으로 UNUSABLE로 두거나 삭제했다가, 적재 후 리빌드하는 방식이 전체적으로 더 빨랐다.
- 제약조건은 DISABLE로 끄고, 마지막에 ENABLE VALIDATE로 검증을 도는 방식이 일반적이었다. 다만 기본키/유니크/외래키는 무결성 요구사항에 따라 신중하게 결정해야 했다.
- Direct-path insert와 NOLOGGING를 함께 쓰면 redo가 크게 줄어들었다. 하지만 데이터베이스나 테이블스페이스에 FORCE LOGGING이 활성화되어 있으면 최소 로깅이 적용되지 않기 때문에 성능 향상이 제한적이었다.

예제: 대량 적재 플로우

SQL

```
-- 1) 로그 최소화
alter table sales nologging;

-- 2) 보조 인덱스 비활성화(또는 드롭)
alter index sales_idx1 unusable;

-- 3) 일부 제약 임시 해제(정책에 따라 선택)
alter table sales disable constraint sales_ck1;

-- 4) Direct-path insert로 대량 적재
insert /*+ append */ into sales
select * from sales_stage;
commit;

-- 5) 인덱스 리빌드(필요 시 NOLOGGING로 빠르게 생성 후 LOGGING 복구)
alter index sales_idx1 rebuild nologging;
alter index sales_idx1 logging;

-- 6) 제약 재활성화 및 검증
alter table sales enable validate constraint sales_ck1;
```

> 메모: FK 컬럼에는 인덱스가 있어야 부모/자식 갱신 시 TM 경합을 줄일 수 있었다.

---

### 6.1.5 수정가능 조인 뷰

- 조인 뷰로도 업데이트가 가능한데, 키 보존 테이블의 컬럼만 수정할 수 있었다. DISTINCT/GROUP BY가 들어가면 업데이트가 불가능해지고, 아우터 조인이 포함된 경우에도 제한이 있었다.
- 뷰를 통해 한 번에 두 개 이상의 기본 테이블을 동시에 갱신하는 것은 허용되지 않았다.

예제

SQL

```
create table dept (deptno number primary key, dname varchar2(30));
create table emp  (
  empno number primary key,
  ename varchar2(30),
  sal   number,
  deptno number references dept(deptno)
);

create or replace view emp_dept_v as
select e.empno, e.ename, e.sal, d.deptno, d.dname
from emp e join dept d on e.deptno = d.deptno;

-- emp는 키 보존이므로 emp 컬럼 갱신은 허용된다.
update emp_dept_v
   set sal = sal * 1.1
 where dname = 'SALES';
```

---

### 6.1.6 MERGE 문 활용

- MERGE는 소스와 타깃을 맞춰서, 있으면 UPDATE 없으면 INSERT를 한 번에 처리한다. 대량 동기화에 적합했다.
- UPDATE 절에 불필요 갱신을 막는 조건을 넣으면 redo/undo가 줄어든다. 오류 레코드는 DBMS_ERRLOG로 별도 테이블에 기록해 전체 진행을 막지 않게 할 수 있었다.

예제

SQL

```
begin
  dbms_errlog.create_error_log('TARGET_T');
end;
/

merge into target_t t
using staging_t s
   on (t.pk = s.pk)
 when matched then
   update set t.val = s.val, t.updated_at = systimestamp
   where (t.val != s.val or t.val is null)
 when not matched then
   insert (pk, val, created_at) values (s.pk, s.val, systimestamp)
 log errors into err$_target_t reject limit unlimited;

commit;
```

---

## 6.2 Direct Path I/O 활용

### 6.2.1 Direct Path I/O

- Direct path read는 버퍼 캐시를 거치지 않고 세그먼트에서 PGA로 직접 읽는다. 대용량 Full Scan이나 병렬 쿼리에서 주로 발생했고, 버퍼 캐시 경합을 피할 수 있어 큰 범위 읽기에 유리했다.
- 직렬 쿼리에서도 버전에 따라 옵티마이저가 direct path read를 선택하는 경우가 있었다. 테이블 크기, 버퍼 캐시 상태, 병렬 힌트 등이 판단에 영향을 줬다.

예제: 병렬 Full Scan로 direct path read 유도

SQL

```
select /*+ full(b) parallel(b 8) */ count(*)
from big_table b;
-- 실행 후 ASH/AWR 또는 autotrace 대기 이벤트에서 'direct path read' 확인
```

---

### 6.2.2 Direct Path Insert

- APPEND 힌트를 주면 세그먼트 HWM(High Water Mark) 뒤쪽에 바로 쓰면서 버퍼 캐시를 우회한다. NOLOGGING과 함께 쓰면 최소 로깅이 가능해져 대량 적재가 빨랐다.
- Direct-path insert는 세그먼트 단위 잠금을 유발한다. 동시성이 중요한 OLTP 테이블에는 신중하게 적용해야 했다. 또한 내부의 빈 공간을 재사용하지 않기 때문에 공간 관리 정책과도 맞춰봐야 했다.
- Force Logging이 활성화된 데이터베이스나 테이블스페이스에서는 최소 로깅이 적용되지 않는다. 이 경우 redo를 줄이는 이점을 기대하기 어려웠다.

예제

SQL

```
alter table sales nologging;

insert /*+ append */ into sales
select * from sales_stage;

commit;
```

---

### 6.2.3 병렬 DML

- 병렬 DML은 세션에서 먼저 parallel dml을 활성화해야 했다. 그렇지 않으면 SELECT는 병렬이더라도 DML은 직렬로 실행됐다.
- INSERT/UPDATE/DELETE/MERGE 모두 병렬화할 수 있지만, 트리거나 외래키, 함수 호출, 물리적 구조 등에 따라 옵티마이저가 병렬을 제한하거나 직렬로 강등할 수 있었다. 그래서 대량 작업 전에 실제 실행 계획을 확인하는 절차를 두는 것이 안전했다.
- 병렬 DML은 TEMP/UNDO/REDO 사용량과 I/O 대역폭 요구가 커진다. 공간과 리소스를 사전에 확보하는 게 중요했다.

예제

SQL

```
alter session enable parallel dml;

-- 병렬 INSERT
insert /*+ append parallel(sales 8) */ into sales
select /*+ parallel(sales_stage 8) */ *
from sales_stage;

-- 병렬 UPDATE
update /*+ parallel(sales 8) */ sales
   set amount = amount * 1.05
 where sale_dt >= date '2025-01-01';

commit;
```

---

## 6.3 파티션을 활용한 DML 튜닝

### 6.3.1 테이블 파티션

- 파티션을 나누면 파티션 프루닝으로 필요한 범위만 읽을 수 있어서 I/O가 크게 줄었다. 파티션 단위의 TRUNCATE, DROP, EXCHANGE 같은 관리 작업을 빠르게 수행할 수 있어 대량 DML에 유리했다.

예제: 월별 RANGE 파티션

SQL

```
create table sales (
  id        number,
  sale_dt   date,
  amount    number,
  channel   varchar2(10)
)
partition by range (sale_dt)
(
  partition p202501 values less than (date '2025-02-01'),
  partition p202502 values less than (date '2025-03-01')
);

-- 프루닝 확인
select sum(amount)
from sales
where sale_dt >= date '2025-01-01' and sale_dt < date '2025-02-01';
```

---

### 6.3.2 인덱스 파티션(Local vs Global)

- Local 인덱스는 테이블 파티션과 1:1로 매칭되므로 파티션을 트렁케이트하거나 드롭할 때 인덱스 관리가 매우 쉬웠다. 대량 관리 작업이 잦다면 로컬 인덱스가 실무적으로 유리했다.
- Global 인덱스는 전체 키 범위를 하나의 인덱스에 담기 때문에 파티션 DDL을 수행하면 인덱스가 UNUSABLE 상태가 되거나 유지 비용이 많이 들었다. 성능/기능상 글로벌 인덱스가 필요한 경우도 있지만, 관리 부담을 반드시 고려해야 했다.

예제

SQL

```
create index sales_local_idx  on sales(sale_dt) local;
create index sales_global_idx on sales(amount)  global;
```

---

### 6.3.3 파티션을 활용한 대량 UPDATE 튜닝

- 특정 파티션만 대상일 때는 파티션을 명시해서 UPDATE하면 불필요한 파티션 접근을 줄일 수 있었다.
- 대량 수정이 필요하면 EXCHANGE PARTITION으로 파티션을 임시 테이블로 빼서, 오프라인으로 수정 후 다시 교체하는 방식이 안정적이고 빨랐다. WITHOUT VALIDATION 옵션을 쓰면 파티션 범위 검증을 생략하는 대신 데이터가 범위를 충족한다는 가정은 내가 책임져야 한다.

예제 A: 파티션 지정 UPDATE

SQL

```
update sales partition(p202501)
   set channel = 'ONLINE'
 where channel = 'OFFLINE';
commit;
```

예제 B: EXCHANGE PARTITION로 오프라인 변환

SQL

```
-- 파티션과 동일 구조의 임시 테이블 생성
create table sales_p202501_tmp as
select * from sales partition(p202501) where 1=2;

-- 데이터 복사 + 대량 수정
insert /*+ append */ into sales_p202501_tmp
select * from sales partition(p202501);

update sales_p202501_tmp
   set channel = 'ONLINE'
 where channel = 'OFFLINE';
commit;

-- 파티션 교체(로컬 인덱스면 즉시 반영이 매우 빠르다)
alter table sales exchange partition p202501
with table sales_p202501_tmp
including indexes
without validation;
```

---

### 6.3.4 파티션을 활용한 대량 DELETE 튜닝

- 파티션 전체를 지울 때는 TRUNCATE PARTITION이 가장 빨랐다. 실제 데이터를 지우기보다 메타데이터만 초기화하기 때문이다.
- 파티션이 더 이상 필요 없으면 DROP PARTITION을 썼다. 글로벌 인덱스가 있다면 추가 관리 작업을 고려해야 했다.
- 데이터를 외부로 보관해야 한다면 EXCHANGE로 파티션을 비파티션 테이블로 교환한 다음, 그 테이블을 관리하는 방식이 운영 테이블에 미치는 영향이 작았다.

예제

SQL

```
-- 1) 파티션 데이터 전체 삭제
alter table sales truncate partition p202501;

-- 2) 파티션 자체 삭제
alter table sales drop partition p202501;

-- 3) 교환 후 외부에서 정리
create table sales_p202501_keep as
select * from sales partition(p202501) where 1=2;

alter table sales exchange partition p202501
with table sales_p202501_keep
including indexes;

-- 보관이 끝났다면 외부 테이블을 트렁케이트 또는 아카이브
truncate table sales_p202501_keep;
```

---

### 6.3.5 파티션을 활용한 대량 INSERT 튜닝

- 해당 파티션만 대상으로 APPEND 적재를 하면 매우 빠른 로드가 가능했다. 파티션 범위를 만족하도록 입력 조건을 정확히 주는 것이 중요했다.
- 외부에서 미리 적재한 스테이징 테이블과 EXCHANGE PARTITION을 하면 거의 즉시 반영되는 효과가 있었다. 특히 로컬 인덱스 환경에서 체감 속도가 크게 빨랐다.

예제 A: 파티션 타깃 INSERT

SQL

```
insert /*+ append */ into sales partition(p202502)
select * 
from sales_stage
where sale_dt >= date '2025-02-01' and sale_dt < date '2025-03-01';
commit;
```

예제 B: 스테이징과 EXCHANGE

SQL

```
create table sales_stage_p202502 as
select * from sales where 1=2;  -- 컬럼 구조/순서/타입 일치 필요

-- 외부 적재 후 교환
alter table sales exchange partition p202502
with table sales_stage_p202502
including indexes
without validation;
```

---

## 6.4 Lock과 트랜잭션 동시성 제어

### 6.4.1 오라클 Lock

- 오라클은 행 수준 잠금(TX)과 테이블 DML 잠금(TM)을 사용한다. UPDATE/DELETE는 대상 행에 TX 잠금을 건다. 외래키가 인덱스되지 않은 상태에서 부모 테이블을 갱신하면 자식 테이블에 광범위한 TM 잠금이 걸려 경합이 심해졌다. 그래서 외래키 컬럼에는 인덱스를 두는 것이 안전했다.
- 블로킹 상황은 V$LOCK, V$SESSION, DBA_BLOCKERS/DBA_WAITERS 뷰로 추적할 수 있었다. 행 잠금 대기를 피하려면 FOR UPDATE NOWAIT 또는 SKIP LOCKED를 적절히 사용했다.

예제: 블로킹 세션 확인과 회피

SQL

```
-- 블로킹/대기 세션 매핑
select s1.sid as blocker_sid, s2.sid as waiter_sid, l1.type, o.object_name
from v$lock l1
join v$lock l2 on l1.id1 = l2.id1 and l1.id2 = l2.id2
join v$session s1 on l1.sid = s1.sid
join v$session s2 on l2.sid = s2.sid
left join dba_objects o on o.object_id = l1.id1
where l1.block = 1 and l2.request > 0;

-- 작업 큐 패턴: 잠긴 행은 건너뛴다.
select * from job_queue
 where status = 'READY'
 for update skip locked;
```

> 데모 팁: 부모-자식 테이블에서 자식 FK에 인덱스가 없을 때 부모를 업데이트하면 TM 경합이 쉽게 재현된다. FK 컬럼에 인덱스를 추가하면 경합이 크게 줄어드는 것을 확인할 수 있었다.

---

### 6.4.2 트랜잭션 동시성 제어

- 오라클은 MVCC로 일관된 읽기를 보장한다. 기본 격리 수준은 READ COMMITTED이고, 필요하면 SERIALIZABLE로 격리 수준을 높일 수 있었다. 다만 SERIALIZABLE에서는 갱신 충돌 시 ORA-08177(직렬화 오류)이 발생할 수 있으니 주의가 필요했다.
- 잠금 대기를 피하려면 FOR UPDATE NOWAIT로 즉시 실패하게 하거나, SKIP LOCKED로 잠긴 행을 건너뛰는 패턴을 사용했다. 작업 큐를 여러 세션이 병렬로 소비할 때 SKIP LOCKED가 특히 유용했다.

예제

SQL

```
-- 격리 수준 상향
set transaction isolation level serializable;

-- 잠금 대기 없이 점유 시도
select * from orders where id = :id for update nowait;

-- 작업 큐 소비자 패턴
select * from task
 where status = 'READY'
 for update skip locked;

-- 이후 현재 커서 행을 상태 변경(예시)
-- update task set status = 'RUNNING' where current of <cursor_name>;
commit;
```

---

### 6.4.3 채번 방식에 따른 INSERT 성능 비교

- 시퀀스가 가장 빠르고 안전했다. CACHE 값을 충분히 크게 잡으면 래치 경합이 줄고 성능이 좋아졌다. RAC 환경에서 NOORDER(기본값)를 사용하면 인스턴스별로 번호가 뒤섞일 수 있는데, 대부분의 시스템에서는 순서 보장보다 성능을 우선해 NOORDER를 사용했다. ORDER를 사용하면 순서가 보장되지만 동시성에서 병목이 생길 수 있었다.
- NOCACHE 시퀀스는 매번 디스크에 상태를 기록해야 해서 느려졌다. 시퀀스 값은 본질적으로 “갭 없는 연속”을 보장하지 않는다. 롤백이나 장애 시 번호가 건너뛰는 현상은 정상 동작이었다.
- 단조 증가 PK는 인덱스 우측 끝에 트래픽이 몰려 “핫 블록”을 만들 수 있다. 범위 스캔이 필요 없다면 reverse key 인덱스나 해시 파티션 전역 인덱스로 분산하는 방법이 있었다.

예제

SQL

```
-- 권장: 캐시 큰 시퀀스
create sequence s_order_id cache 1000 nocycle;

create table orders (
  id number primary key,
  val number
);

-- 대량 INSERT(배치 + FORALL)
declare
  type t_num is table of number;
  ids t_num := t_num();
begin
  for i in 1..100000 loop
    ids.extend; ids(ids.count) := s_order_id.nextval;
  end loop;

  forall i in 1..ids.count
    insert into orders(id, val) values (ids(i), i);

  commit;
end;
/
```

핫 블록 완화(범위 스캔이 필요 없을 때 고려)

SQL

```
-- reverse key 인덱스: 우측 핫스팟 분산(대신 범위 스캔은 불리)
create unique index orders_pk on orders(id) reverse;

-- 또는(엔터프라이즈 기능): 해시 파티션 전역 인덱스
-- create index orders_pk on orders(id) global partition by hash(id) partitions 16;
```

---

## 추가로 내가 정리한 체크리스트

- 대량 DML 이후 통계를 갱신해 옵티마이저가 최신 정보를 쓰도록 한다. 파티션 테이블은 파티션 단위 또는 증분 통계를 고려한다.
- Direct-path/병렬 DML을 사용할 때 TEMP/UNDO/REDO와 I/O 대역폭을 사전에 점검한다. 적재 도중 공간 부족으로 실패하면 되돌리는 비용이 훨씬 크다.
- Data Guard 환경에서 FORCE LOGGING이 켜져 있으면 NOLOGGING의 최소 로깅 효과가 적용되지 않는다. 이때는 병렬화나 파티션 전략 쪽에서 최적화를 찾는 것이 더 현실적이었다.
- 대량 작업에서 불량 데이터로 전체 트랜잭션이 실패하지 않도록 DBMS_ERRLOG와 FORALL SAVE EXCEPTIONS를 적극적으로 사용한다.