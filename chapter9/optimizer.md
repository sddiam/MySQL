# 📘 [ 9장, 옵티마이저와 힌트]

### 9.3.1.11 퍼스트 매치(firstmatch)
> First Match 최적화 전략은 IN형태의 세미 조인을 EXISTS 형태로 튜닝한 것과 비슷한 방법으로 실행된다.

```sql
EXPLAIN SELECT *
FROM employees e WHERE e.first_name='Matt'
  AND e.emp_no IN (
    SELECT t.emp_no FROM titles t
    WHERE t.from_data BETWEEN '1995-01-01' AND '1995-01-30'
    );
```
위 쿼리는 이름이 'Matt'인 사원 중 1995년 01월 01일 부터 1995년 1월 30일 사이에 직급이 변경된 적 있는 사원을 조회하고 있다.
![image](https://github.com/user-attachments/assets/a9ef36bc-96b5-408d-965c-4a6354880025)

employees 테이블에서 first_name 칼럼 값이 ‘Matt’인 사원 정보를 ix_firstname 인덱스를 이용하여 레인지 스캔으로 읽은 결과가 위 그림의 왼쪽 테이블인 employees 테이블이다.
employees 테이블의 emp_no를 titles 과 조인하여 from_date의 조건에 일치하는지 확인하여 일치하면 더이상 titles 테이블을 검색하지 않고 바로 레코드를 반환한다.


**장점**
- 가끔 여러 테이블이 조인되는 경우 옵티마이저가 동등 조건을 추가하는 형태의 최적화가 실행된다.
  기존 IN-to-EXISTS 최적화에서는 서브쿼리 내에서만 가능했지만 FirstMatch에서는 조인 형태로 처리되기에 서브쿼리뿐만 아니라 아우터 쿼리의 테이블까지 전파될 수 있다.
  이렇게 되면 더 많은 조건이 주어져서 더 나은 실행 계획을 수립할 수 있게 된다.
- FirstMatch 최적화에서는 서브쿼리의 모든 테이블에 대해 최적화를 수행할지 아니면 일부 테이블에 대해서만 수행할지 취사선택할 수 있다는 것이 장점이다.

**특징**
- FirstMatch는 서브쿼리에서 하나의 레코드만 검색되면 더이상 검색을 멈추는 단축 실행 경로(Shor-cut path)이기 때문에 FirstMatch 최적화에서 서브쿼리는 그 서브쿼리가 참조하는 모든 아우터 테이블이 먼저 조회된 이후에 실행된다.
- FirstMatch 최적화는 상관 서브쿼리에서도 사용될 수 있다.
- FirstMatch 최적화는 GROUP BY나 집합 함수가 사용된 서브쿼리의 최적화에서는 사용될 수 없다.


### 9.3.1.12 루스 스캔
> 세미 조인 서브쿼리 최적화의 LooseScan은 인덱스를 사용하는 GROUP BY 최적화 방법에서 살펴본 "Using index for group-by"의 루스 인덱스 스캔과 비슷한 읽기 방식을 사용한다.

인덱스를 건너뛰며 읽으면서 효율적으로 인덱스를 스캔하는 방법이다.
중복된 레코드도 제거하면서 실행 된다.
```sql
EXPLAIN
SELECT * FROM departments d WHERE d.dept_no IN (
  SELECT de.dept_no FROM dept_emp de );
```
![image](https://github.com/user-attachments/assets/ab289041-0df0-4c62-97dd-47cffae0a874)


**특징**
- LooseScan 최적화는 루스 인덱스 스캔으로 서브쿼리 테이블을 읽고, 그 다음으로 아우터 테이블을 드리븐으로 사용해서 조인을 수행한다. 그래서 서브쿼리 부분이 루스 인덱스 스캔을 사용할 수 있는 조건이 갖춰져야 사용할 수 있는 최적화다. 루스 인덱스 스캔 최적화는 다음과 같은 형태의 서브쿼리들에서 사용할 수 있다.



### 9.3.1.13 구체화
> Meterialization 최적화는 세미 조인에 사용된 서브쿼리를 통째로 구체화해서 쿼리를 최적화한다는 의미다.
> 구체화란 내부 임시 테이블을 생성하는 것이다.

```sql
EXPLAIN
SELECT *
FROM emplyees e
WHERE e.emp_no IN
     (SELECT de.emp_no FROM dept_emp de
      WHERE de.from_data='2025-01-01');
```
위 쿼리를 실행하고 그 결과로 임시 테이블을 만들게 된다. 그리고 그걸 이용해 쿼리를 빠르게 처리하는 방식이다.

**특징**
- IN(subquery)에서 서브쿼리는 상관 서브쿼리가 아니어야 한다.(바깥 쿼리의 값에 따라 매번 결과가 달라지기 때문에)
- 서브쿼리는 GROUP BY나 집합 함수들이 사용돼도 구체화를 사용할 수 있다.
- 구체화가 사용된 경우에는 내부 임시 테이블이 사용된다.



### 9.3.1.14 중복 제거
> Duplicate Weedout은 세미 조인 서브쿼리를 일반적인 INNER JOIN 쿼리로 바꿔서 실행하고 마지막에 중복된 레코드를 제거하는 방법으로 처리되는 최적화 알고리즘이다.

```sql
EXPLAIN
SELECT * FROM employees e
WHERE e.emp_no IN (SELECT s.emp_no FROM salaries s WHERE s.salary>150000);
```

**과정**
1. salaries 테이블의 ix_salary 인덱스를 스캔해서 salary가 150000보다 큰 사원을 검색해 employees 테이블 조인을 실행
2. 조인된 결과를 임시 테이블에 저장
3. 임시 테이블에 저장된 결과에서 emp_no 기준으로 중복 제거
4. 중복을 제거하고 남은 레코드를 최종적으로 반환
![image](https://github.com/user-attachments/assets/fcef8fa6-4f91-4fd1-aae7-12cc89ecc32f)


**장점 및 제약 사항**
- 서브쿼리가 상관 서브쿼리라고 하더라도 사용할 수 있는 최적화다.
- 서브쿼리가 GROUP BY나 집합 함수가 사용된 경우에는 사용될 수 없다.
- Duplicate Weedout은 서브쿼리의 테이블을 조인으로 처리하기 때문에 최적화할 수 있는 방법이 많다.

### 9.3.1.15 컨디션 팬아웃
> MySQL 옵티마이저는 여러 테이블이 조인되는 경우 가능하다면 일치하는 레코드 건수가 적은 순서대로 조인을 실행한다.

조인 실행할 때, 테이블 순서는 쿼리 성능에 매우 큰 영향을 미침.
A 테이블과 B 테이블 조인 시, A 테이블이 조건에 일치하는 레코드가 1만 건이고, B 테이블이 조건에 일치하는 레코드가 10건이라고 가정하자.
A 테이블을 조인의 드라이빙 테이블로 결정하면 B 테이블을 1만번 읽어야함.
이때, B 테이블의 인덱스를 이용하여 조인을 실행하더라도 레코드를 읽을 때마다 B 테이블의 인덱스를 구성하는 B-Tree의 루트 노드부터 검색해야 함.

```sql
mysql> SELECT *
             FROM employees e
                 INNER JOIN salaries s ON s.emp_no=e.emp_no
             WHERE e.first_name='Matt'
                 AND e.hire_date BETWEEN '1985-11-21' AND '1986-11-21';

-- condition_fandout_filter=off
+----+-------+------+--------------+------+----------+-------------+
| id | table | type | key          | rows | filtered | Extra       |
+----+-------+------+--------------+------+----------+-------------+
| 1  | e     | ref  | ix_firstname |  233 |   100.00 | Using where |
| 1  | s     | ref  | PRIMARY      |   10 |   100.00 | NULL        |
+----+-------+------+--------------+------+----------+-------------+

-- condition_fandout_filter=on
+----+-------+------+--------------+------+----------+-------------+
| id | table | type | key          | rows | filtered | Extra       |
+----+-------+------+--------------+------+----------+-------------+
| 1  | e     | ref  | ix_firstname |  233 |    23.20 | Using where |
| 1  | s     | ref  | PRIMARY      |   10 |   100.00 | NULL        |
+----+-------+------+--------------+------+----------+-------------+
```
**실행 계획 순서**
1. employees 테이블에서 ix_firstname 인덱스를 이용하여 first_name=’Matt’ 조건에 일치하는 233건의 레코드를 검색한다.
2. 검색된 233건 레코드 중, hire_date가 1985-11-21 ~ 1986-11-21 사이인 레코드만 걸러냄. 이 때, 실행 계획에서 filtered 칼럼의 값이 100인 것은 옵티마이저가 233건 모두 hire_date 칼럼의 조건을 만족할 것으로 예측했다는 것을 의미함.
3. employees 테이블을 읽은 결과, 233건에 대해 salaries 테이블의 프라이머리 키를 이용해 salaries 테이블의 레코드를 읽음.

컨디션 팬 아웃 최적화가 활성화되면 MySQL 옵티마이저는 인덱스를 사용할 수 있는 first_name 칼럼 조건 이외의 나머지 조건(hire_date 칼럼의 조건)에 대해서도 얼마나 조건을 충족할지를 고려했다는 뜻임.
MySQL 옵티마이저가 조건을 만족하는 레코드 건수를 정확하게 예측할 수 있다면 더 빠른 실행 계획을 만들어 낼 수 있음

**MySQL 옵티마이저가 실행 계획 수립할 때, 테이블이나 인덱스의 통계 정보만 사용하는 것이 아니라, 다음의 순서대로 사용 가능한 방식을 선택함.**
1.  레인지 옵티마이저(Range optimizer)를 이용한 예측
2. 히스토그램을 이용한 예측
3. 인덱스 통계를 이용한 예측
4. 추측에 기반한 예측(Guesstimates = Guess + Estimate)

### 9.3.1.16 파생 테이블 머지
> 예전 버전의 MySQL 서버에서, 아래와 같은 FROM 절에 사용된 서브쿼리는, 먼저 실행해서 그 결과를 임시 테이블로 만든 다음, 외부 쿼리 부분을 처리했다.

```sql
EXPLAIN
SELECT * FROM (
  SELECT * FROM employees WHERE first_name='Matt'
) derived_table
WHERE derived_table.hire_date='1986-04-03';

+----+-------------+------------+------+--------------+
| id | select_type | table      | type | key          |
+----+-------------+------------+------+--------------+
| 1  | PRIMARY     | <derived2> | ref  | <auto_key0>  |
| 2  | DERIVED     | employees  | ref  | ix_firstname |
+----+-------------+------------+------+--------------+
```
FROM절의 서브쿼리는 '파생 테이블(Derived Table)'로 처리되어 임시 테이블을 생성하고 다시 읽는 오버헤드가 발생하게 된다.
MySQL 5.7부터 'derived_merge' 옵션으로 서브쿼리를 외부 쿼리와 병합하는 최적화가 도입되었다.
```sql
+----+-------------+-----------+-------------+--------------------------+
| id | select_type | table     | type        | key                      |
+----+-------------+-----------+-------------+--------------------------+
| 1  |  SIMPLE     | employees | index_merge | ix_hiredate,ix_firstname |
+----+-------------+-----------+-------------+--------------------------+
```
병합 최적화 적용 시 실행 계획에서 DERIVED 라인이 사라지고 단순화된 형태로 변경되었다.
이제 DBA가 수작업으로 쿼리를 재작성할 필요가 줄었지만, 모든 서브쿼리가 병합 가능한 것은 아니다.
최적화를 통해 임시 테이블 생성과 데이터 복사 오버헤드를 줄일 수 있다.


**다음과 같은 서브쿼리는 외부 쿼리로 수동으로 병합해서 작성하는 것이 쿼리 향상에 도움이 된다.**
- SUM() 또는 MIN(), MAX() 같은 집계 함수와 윈도우 함수가 사용된 서브쿼리
- DISTINCT가 사용된 서브쿼리
- GROUP BY나 HAVING이 사용된 서브쿼리
- LIMIT이 사용된 서브쿼리
- UNION 또는 UNION ALL을 포함하는 서브쿼리
- SELECT 절에 사용된 서브쿼리
- 값이 변경되는 사용자 변수가 사용된 서브쿼리



### 9.3.1.17 인비저블 인덱스
> MySQL 8.0 버전부터 인덱스를 삭제하지 않고도 해당 인덱스를 사용하지 못하게 제어하는 기능이 제공되었다.

**옵티마이저의 인덱스 가시성 변경 명령**
```sql
-- 인덱스를 사용하도록 변경
ALTER TABLE employees ALTER INDEX ix_hiredate INVISIBLE;

-- 인덱스를 사용하지 못하게 변경
ALTER TABLE employees ALTER INDEX ix_hiredate VISIBLE;
```

### 9.3.1.18 스킵 스캔
> 인덱스 스킵 스캔은 인덱스의 선행 칼럼이 조건절에 사용되지 않더라도 후행 칼럼의 조건만으로도 인덱스를 이용한 쿼리 성능 개선이 가능하도록 하는 것이다.

```sql
ALTER TABLE employees
ADD INDEX ix_gender_birthdate (gender, birth_date);
```
위와 같이 employees 테이블에 gender와 birth_date라는 인덱스가 있다고 한다면
gender라는 칼럼이 먼저 WHERE 조건절에 오지 않는다면 birth_date가 사용된 쿼리에서 인덱스를 사용할 수 없었다.

MySQL 8.0 버전부터 인덱스 스킵 스캔 최적화 기능으로 인해 제한적이긴 하지만 인덱스의 이러한 제약 사항을 뛰어넘을 수 있는 최적화 기법이 생기게 되었다.

*그런데 인덱스의 선행 칼럼이 매우 다양한 값을 가지는 경우에는 인덱스 스캡 스캔 최적화가 비효율적일 수 있다. 그래서 MySQL 8.0 옵티마이저는 인덱스의 선행 칼럼이 소수의 유니크한 값을 가질 때만 인덱스 스킵 스캔 최적화를 사용한다.*

현재 세션에서 인덱스 스킵 스캔 최적화를 활성화 하는 쿼리
```sql
SET optimizer_switch='skip_scan=on';
```


### 9.3.1.19 해시 조인
해시 조인은 기존의 네스티드 루프 조인보다 빠르다고 생각하여 해시 조인을 기대하는 사용자들이 존재하는데 이는 항상 옳다고는 할 수 없다.

![image](https://github.com/user-attachments/assets/8c7bafc0-bd59-4ba9-a539-e888d63a333e)

사진을 보면 해시조인은 첫 번째 레코드를 찾는데 시간이 많이 걸리지만 최종 레코드를 찾는 데까지는 시간이 많이 걸리지 않는다.
그에 반해 네스티드 루프 조인은 마지막 레코드를 찾는 데까지는 시간이 많이 걸리지만 첫 번째 레코드를 찾는 것은 해시조인보다 훨씬 빠르다.

즉, 해시 조인은 최고 스루풋 전략에 적합하고, 네스티드 루프 조인은 최고 응답 속도 전략에 적합하다. 적합한 전략이 다른만큼 더 어울리는 서비스가 다른데
일반적인 웹 서비스는 응답 속도를 더 중요시하고 분석과 같은 서비스는 전체 스루풋을 중요시한다.

*그래서 대용량 분석을 위해 사용되지 않는 MySQL도 응답 속도에 더 최적화하고, 조인 대상 테이블 중 일부의 레코드 건수가 매우 적은 경우에만 해시 조인 알고리즘을 사용한다.*


일반적으로 해시 조인은 빌드 단계(Build-phsae)와 프로브 단계(Probe-phsae)로 나뉘어 처리된다.
- 빌드 단계: 레코드 수가 적은 테이블을 선택하여 메모리에 해시 테이블 생성
- 프로브 단계: 다른 테이블의 레코드를 읽어 해시 테이블에서 일치하는 레코드 검색

**메모리 관리 전략**
- 해시 테이블은 join_buffer_size(기본 256KB) 시스템 변수로 제어되는 조인 버퍼에 저장
- 해시 테이블이 메모리보다 클 경우, 테이블을 청크로 분할하여 처리(1차 처리)
- 빌드 테이블 크기에 따라 클래식 해시 조인 또는 그레이스 해시 조인 알고리즘을 적용

**실행 계획 분석**
- 일반 EXPLAIN으로는 빌드/프로브 테이블 구분이 어려움
- EXPLAIN FORMAT=TREE 또는 EXPLAIN ANALYZE 명령으로 더 상세한 실행 계획 확인 가능
- 실행 계획의 "Using join buffer (hash join)" 표시로 해시 조인 사용 여부 식별

**최적화 고려사항**
- 해시 조인이 항상 최선은 아니므로 옵티마이저 힌트로 강제하는 것은 권장되지 않음
- 메모리 부족 시 디스크 기반 처리로 전환되어 성능 저하 가능성 존재
- 데이터 분포와 테이블 크기를 고려한 적절한 조인 전략 선택이 중요

### 9.3.1.20 인덱스 정렬 선호
> MySQL 옵티마이저는 ORDER BY 또는 GROUP BY 처리를 위해 쿼리의 실행 계획에서 이 인덱스의 가중치를 높이 설정해서 실행된다.

**방법**
- 인덱스를 이용해 조건에 일치하는 레코드를 찾은 다음 정렬 조건으로 정렬 후 결과 반환
- 정렬 기준이 PK이면 PK 정순으로 읽은 후 조건에 일치하는지 비교 후 결과 반환

상황에 따라 효율적인 방법이 달라진다. 조건에 부합하는 레코드 건수가 많지 않다면 대부분 1번이 효율적이다. 가끔 레코드 건수가 많지 않은 경우에도 옵티마이저가 2번 방법을 실행하는 경우가 있는데, 이 때는 옵티마이저 옵션을 이용할 수 있다.

**인덱스 정렬 선호 활성화/비활성화**
```sql
SET optimizer_switch='prefer_ordering_index=on'; -- 활성화
SET optimizer_switch='prefer_ordering_index=off'; -- 비활성화
```


## 9.3.2 조인 최적화 알고리즘
### 9.3.2.1 Exhaustive 검색 알고리즘
> Exhaustive 검색 알고리즘은 MySQL 5.0 이전 버전에서 사용되던 조인 최적화 기법이다. FROM 절에 명시된 모든 테이블의 조합에 대해 실행 계획의 비용을 계산해서 최적의 조합 1개를 찾는 방법이다.

![image](https://github.com/user-attachments/assets/f6e08550-7db8-44a6-8cf2-655ab20ee187)
위 그림은 4개의 테이블이 Exhaustive 검색 알고리즘으로 처리될 때 최적의 조인 순서를 찾는 방법을 표현한 것이다. 만약 테이블의 갯수가 20개라면 가능한 조인 조합은 총 3628800개로 무수히 많아진다.


### 9.3.2.2 Greedy 검색 알고리즘
> Greedy 검색 알고리즘은 Exhaustive 검색 알고리즘의 시간 소모적인 문제점을 해결하기 위해 도입된 최적화 기법이다.
> 여러 개의 테이블을 조인할 때에도 탐색 속도가 빠르다.

![image](https://github.com/user-attachments/assets/a02c143e-d546-4b2a-bb46-ca244e0b4c7f)
**작동 순서**
1. 전체 N개의 테이블 중에서 optimizer_search_depth 시스템 설정 변수에 정의된 개수의 테이블로 가능한 조인 조합을 생성
2. 1번에서 생성된 조인 조합 중에서 최소 비용의 실행 계획 하나를 선정
3. 2번에서 선정된 실행 계획의 첫 번째 테이블을 "부분 실행 계획"의 첫 번째 테이블로 선정
4. 전체 N-1개의 테이블 중에서 optimizer_search_depth 시스템 설정 변수에 정의된 개수의 테이블로 가능한 조인 조합을 생성
5. 4번에서 생성된 조인 조합들을 하나씩 3번에서 생성된 "부분 실행 계획"에 대입해 실행 비용을 계산
6. 5번의 비용 계산 결과, 최적의 실행 계획에서 두 번째 테이블을 3번에서 생성된 "부분 실행 계획"의 두 번째 테이블로 선정
7. 남은 테이블이 모두 없어질 때까지 4~6번까지의 과정을 반복 실행하면서 "부분 실행 계획"에 테이블의 조인 순서를 기록
8. 최종적으로 "부분 실행 계획"이 테이블의 조인 순서로 결정됨



# 9.4 쿼리 힌트
> 쿼리 힌트는 옵티마이저가 실행 계획을 수립하는 방식을 사용자가 옵티마이저에게 알려주기 위해 제공된다.

**종류**
- 인덱스 힌트
- 옵티마이저 힌트

## 9.4.1 인덱스 힌트
인덱스 힌트는 옵티마이저에게 특정 인덱스를 사용하거나 무시하도록 지시하는 힌트이다.

### 9.4.1.1 STRAIGHT_JOIN
> STRAIGHT_JOIN은 SELECT, UPDATE, DELETE 쿼리에서 여러 개의 테이블이 조인되는 경우 조인 순서를 고정하는 역할을 한다.

옵티마이저는 여러 개의 테이블 중 그때그때 각 테이블의 통계 정보와 쿼리의 조건을 기반으로 가장 최적이라고 판단되는 순서로 조인을 진행한다.
하지만 쿼리의 조인 순서를 변경하고 싶은 경우에는 STRAIGHT_JOIN 힌트를 사용할 수 있다.

```sql
-- 사용 전
EXPLAIN
SELECT *
FROM employees e, dept_emp de, departments d
WHERE e.emp_no=de.emp_no AND d.dept_no=de.dept_no;

--사용 후 (순서대로 조인 수행)
SELECT STRAIGHT_JOIN
  e.first_name, e.last_name, d.dept_name
FROM employees e, dept_emp de, departments d
WHERE e.emp_no=de.emp_no
  AND d.dept_no=de.dept_no;
```

**STRAIGHT_JOIN 힌트로 조인 순서를 조정하는 것이 좋은 경우**
- 임시 테이블과 일반 테이블의 조인: 일반적으로 임시 테이블을 드라이빙 테이블로 선정하는 것이 좋고, 옵티마이저가 실행 계획을 제대로 수립하지 못해서 심각한 성능 저하가 있는 경우에 힌트를 사용하면 된다.
- 임시 테이블끼리 조인: 크기가 작은 테이블을 드라이빙으로 선택해주는 것이 좋다.
- 일반 테이블끼리 조인: 양쪽 테이블 모두 조인 칼럼에 인덱스가 있거나 없는 경우에는 레코드 건수가 적은 테이블을 드라이빙으로 선택해주는 것이 좋고, 그 이외에는 조인 칼럼에 인덱스가 없는 테이블을 드라이빙으로 선택하는 것이 좋다.

**비슷한 역할의 옵티마이저 힌트(9.4.2.7)**
- JOIN_FIXED_ORDER
- JOIN_ORDER
- JOIN_PREFIX
- JOIN_SUFFIX


### 9.4.1.2 USE INDEX / FORCE INDEX / IGNORE INDEX
> 테이블 액세스 시 사용할 인덱스를 제안하거나 강제하는 힌트

- USE INDEX : 가장 자주 사용되는 인덱스 힌트로, 옵티마이저에서 특정 테이블의 인덱스를 사용하도록 권장하는 힌트이다. 대부분 힌트를 채택하지만 옵티마이저는 제안을 무시하고 더 나은 인덱스를 선택할 수도 있다.
- FORCE INDEX : USE INDEX보다 영향이 더 강한 힌트이다. (굳이 사용되지는 않는다)
- IGNORE INDEX : 특정 인덱스를 사용하지 못하게 하는 용도의 힌트이다.

```sql
-- 동일한 실행 계획으로 쿼리 진행
SELECT * FROM employees WHERE emp_no='10001';
SELECT * FROM employees FORCE INDEX(primary) WHERE emp_no='10001';
SELECT * FROM employees USE INDEX(primary) WHERE emp_no='10001';

-- 프라이머리 키를 무시하고 풀 테이블 스캔으로 처리하도록 유도하는 힌트
SELECT * FROM employees IGNORE INDEX(primary) WHERE emp_no='10001';
-- 관계없는 인덱스를 사용해 진행
SELECT * FROM employees FORCE INDEX(ix_firstname) WHERE emp_no='10001';
```

### 9.4.1.3 SQL_CALC_FOUND_ROWS
> LIMIT 절이 포함된 쿼리에서 LIMIT을 무시하고 전체 결과 행의 수를 계산하는 힌트

```sql
--기존 2개의 쿼리로 쪼개어 실행하는 방법
SELECT COUNT(*) FROM employees WHERE first_name='Georgi';
SELECT * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;

--SQL_CALC_FOUND_ROWS 사용법
SELECT SQL_CALC_FOUND_ROWS * FROM employees WHERE first_name='Georgi' LIMIT 0, 20;
SELECT FOUND_FOWS() AS total_record_count; --전체 행 수 조회
```

기존 방법과 SQL_CALC_FOUND_ROWS 방법 모두 쿼리를 2번 실행해야 한다. 만약 실제 이 조건을 만족하는 레코드가 253건일 때,
SQL_CALC_FOUND_ROWS 힌트는 인덱스를 통해 실제 데이터 레코드를 찾아가는 작업을 253번 실행해야 하고, 디스크 헤드가 특정 위치로 움직일 때까지 기다려야 하는 랜덤 I/O가 253번 일어난다.
하지만 기존 방법으로는 인덱스를 레인지 스캔으로 접근한 후, 실제로 데이터 레코드를 읽으러 갈 때 랜덤 I/O가 20번만 실행된다. 


## 9.4.2 옵티마이저 힌트
옵티마이저 힌트는 옵티마이저의 최적화 전략과 인덱스 선택 등에 영향을 주는 지시 도구이다.

### 9.4.2.1 옵티마이저 힌트 종류

**영향 범위에 따른 구분**
- 인덱스 : 특정 인덱스의 이름을 사용할 수 있는 옵티마이저 힌트
- 테이블 : 특정 테이블의 이름을 사용할 수 있는 옵티마이저 힌트
- 쿼리 블록 : 특정 쿼리 블록에 사용할 수 있는 옵티마이저 힌트로서, 특정 쿼리 블록의 이름을 명시하는 것이 아니라 힌트가 명시된 쿼리 블록에 대해서만 영향을 미치는 옵티마이저 힌트
- 글로벌(쿼리 전체) : 전체 쿼리에 대해서 영향을 미치는 힌트


### 9.4.2.2 MAX_EXECUTION_TIME
> 쿼리의 실행 시간 제한

```sql
SELECT /*+ MAX_EXECUTION_TIME(100) */*
FROM employees
ORDER BY last_name LIMIT 1;
```

### 9.4.2.3 SET_VAR
> 쿼리 실행을 위한 시스템 변수 제어
```sql
EXPLAIN
SELECT /*+ SET_VAR(optimizer_switch='index_merge_intersection=off') */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```

### 9.4.2.4 SEMIJOIN & NO_SEMIJOIN
> 서브쿼리의 세미 조인 최적화 (DUPSWEEDOUT, FIRSTMATCH, LOOSESCAN, MATERIALIZATION) 전략 제어
```sql
EXPLAIN
SELECT /*+ SEMIJOIN(@subq1 MATERIALIZATION) */ *
FROM departments d
WHERE d.dept_no IN
    (SELECT /*+ QB_NAME(subq1) */ de.dept_no
    FROM dept_emp de);
```

### 9.4.2.5 SUBQUERY
> 서브쿼리의 세미 조인 최적화(MATERIALIZATION과 INTOEXISTS) 전략 제어


### 9.4.2.6 BNL & NO_BNL & HASHJOIN & NO_HASHJOIN
> BNL & NO_BNL : 블록 네스티드 루프 조인(BNL) 사용 여부 제어

> HASHJOIN & NO_HASHJOIN : 해시 조인 사용 여부 제어
```sql
EXPLAIN
SELECT /*+ BNL(e, de) */ *
FROM employees e
INNER JOIN dept_emp de ON de.emp_no=e.emp_no;
```


### 9.4.2.7 JOIN_FIXED_ORDER & JOIN_ORDER & JOIN_PREFIX & JOIN_SUFFIX
- JOIN_FIXED_ORDER : FROM 절에 명시된 테이블 순서대로 조인 실행
```sql
SELECT /*+ JOIN_FIXED_ORDER() */ *
FROM employees e
  INNER JOIN dept_emp de ON de.emp_no=e.emp_no
  INNER JOIN departments d ON d.dept_no=de.dept_no;
```
- JOIN_ORDER : 힌트에 명시된 테이블 순서대로 조인 실행
```sql
SELECT /*+ JOIN_ORDER(d, de) */ *
FROM employees e
  INNER JOIN dept_emp de ON de.emp_no=e.emp_no
  INNER JOIN departments d ON d.dept_no=de.dept_no;
```
- JOIN_PREFIX : 힌트에 명시된 테이블을 조인의 드라이빙 테이블로 조인 실행
```sql
SELECT /*+ JOIN_PREFIX(e. de) */ *
FROM employees e
  INNER JOIN dept_emp de ON de.emp_no=e.emp_no
  INNER JOIN departments d ON d.dept_no=de.dept_no;
```
- JOIN_SUFFIX : 힌트에 명시된 테이블을 조인의 드리븐 테이블로 조인 실행
```sql
SELECT /*+ JOIN_SUFFIX(de, e) */ *
FROM employees e
  INNER JOIN dept_emp de ON de.emp_no=e.emp_no
  INNER JOIN departments d ON d.dept_no=de.dept_no;
```

### 9.4.2.8 MERGE & NO_MERGE
> FROM 절의 서브쿼리나 뷰를 외부 쿼리 블록으로 병합하는 최적화를 수행할지 여부 제어
> 
> 내부 임시테이블 생성과 내부 쿼리를 외부 쿼리와 병합하는 것을 선택
```sql
--외부 쿼리와 병합
EXPLAIN
SELECT /*+ MERGE(sub) */ *
FROM (SELECT *
      FROM employees
      WHERE first_name='Matt') sub LIMIT 10;

--내부 임시 테이블로 처리
EXPLAIN
SELECT /*+ NO_MERGE(sub)*/ *
FROM (SELECT *
      FROM employees
      WHERE first_name='Matt') sub LIMIT 10;
```




### 9.4.2.9 INDEX_MERGE & NO_INDEX_MERGE
> 인덱스 병합 실행 계획 사용 여부 제어
```sql
--여러 개의 인덱스를 동시에 사용하는 실행 계획 허용
EXPLAIN
SELECT /*+ NO_INDEX_MERGE(employees PRIMARY) */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;

--여러 인덱스를 병합하지 않도록 차단
EXPLAIN
SELECT /*+ INDEX_MERGE(employees ix_firstname, PRIMARY) */ *
FROM employees
WHERE first_name='Georgi' AND emp_no BETWEEN 10000 AND 20000;
```


### 9.4.2.10 NO_ICP
> ICP(인덱스 컨디션 푸시다운) 최적화 전략 사용 여부 제어
```sql
--인덱스 컨디션 푸시다운 최적화 비활성화
EXPLAIN
SELECT /*+ NO_ICP(employees ix_lastname_firstname) */ *
FROM employees
WHERE last_name='Acton' AND first_name LIKE '%sal';
```

### 9.4.2.11 SKIP_SCAN & NO_SKIP_SCAN
> 인덱스 스킵 스캔 사용 여부 제어
```sql
EXPLAIN
SELECT gender, birth_date
FROM employees
WHERE birth_date>='2025-02-01';

--인덱스 스킵 스캔 비활성화
EXPLAIN
SELECT /*+ NO_SKIP_SCAN(employees ix_gender_birthdate) */ gender, birth_date
FROM employees
WHERE birth_date>='2025-02-01'
```

### 9.4.2.12 INDEX & NO_INDEX
> GROUP BY, ORDER BY, WHERE절의 처리를 위한 인덱스 사용 여부 제어
```sql
EXPLAIN
SELECT /*+ INDEX(employees ix_firstname) */ *
FROM employees
WHERE first_name='Matt';
```

