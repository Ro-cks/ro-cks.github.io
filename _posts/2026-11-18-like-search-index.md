---
title: "인덱스를 만들었는데 LIKE 검색이 빨라지지 않은 이유"
date: 2026-11-18 20:00:00 +0900
categories: [Database, MySQL]
tags: [mysql, index, b-tree, explain, like, jpa]
---

> 실행 환경: MySQL 8.4(Docker), Spring Boot 4.1.x, Spring Data JPA(Hibernate)

## 상황

게시판에 제목과 내용 검색 기능을 추가했다. Spring Data JPA의 쿼리 메서드로 구현했다.

```java
public interface PostRepository extends JpaRepository<Post, Long> {

    Page<Post> findByTitleContaining(String title, Pageable pageable);
    Page<Post> findByContentContaining(String content, Pageable pageable);
    Page<Post> findByTitleContainingOrContentContaining(String title, String content, Pageable pageable);
}
```

`Containing`은 SQL의 `LIKE '%키워드%'`로 바뀐다. 즉 키워드가 **어딘가에 포함된** 값을 찾는다.

글이 많아지면 검색이 느려질 테니, 자주 검색하는 `title` 컬럼에 인덱스를 만들면 빨라지지 않을까 하는 의문이 들었다.

## 예상

인덱스를 만들어도 `LIKE` 검색은 전부 전체 스캔일 것 같았다. 다만 **왜 그런지는 설명할 수 없었다.** 검색어로 인덱스를 어떻게 찾는지 몰랐기 때문이다. 그래서 실행 계획으로 직접 확인해 보기로 했다.

## 실험 준비

글이 십여 개뿐이면 MySQL은 인덱스가 있어도 그냥 전체를 읽는 쪽을 고르는 경우가 많다. 읽을 데이터가 적으면 인덱스를 거치는 비용이 더 크기 때문이다. 그래서 재귀 CTE로 테스트 데이터 1만 건을 넣었다.

```sql
SET SESSION cte_max_recursion_depth = 20000;

INSERT INTO posts (title, content, author, created_at, updated_at)
WITH RECURSIVE seq AS (
  SELECT 1 AS n UNION ALL SELECT n + 1 FROM seq WHERE n < 10000
)
SELECT CONCAT('Title ', n), CONCAT('Content ', n), 'bulk', NOW(6), NOW(6) FROM seq;
```

제목은 `Title 1`부터 `Title 10000`까지 만들어진다.

## EXPLAIN 읽는 법

SQL 앞에 `EXPLAIN`을 붙이면, 쿼리를 실행하지 않고 **어떻게 실행할지**만 보여준다. 이번 실험에서는 세 열만 보면 된다.

| 열 | 의미 |
|---|---|
| `type` | 데이터에 접근하는 방식. `ALL`은 전체 스캔, `range`는 인덱스로 범위를 찾음 |
| `key` | 실제로 사용한 인덱스. `NULL`이면 사용하지 않음 |
| `rows` | 읽을 것으로 예상하는 행 수 |

## 실험

### A. 인덱스 없이 중간 일치 검색

```sql
EXPLAIN SELECT * FROM posts WHERE title LIKE '%Title 123%';
```

```text
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | posts | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9696 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

인덱스가 없으니 전체 스캔이다. 예상대로다.

### B. title에 인덱스 생성

```sql
CREATE INDEX idx_posts_title ON posts (title);
```

### C. 인덱스를 만든 뒤 중간 일치 검색

```sql
EXPLAIN SELECT * FROM posts WHERE title LIKE '%Title 123%';
```

```text
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
| id | select_type | table | partitions | type | possible_keys | key  | key_len | ref  | rows | filtered | Extra       |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
|  1 | SIMPLE      | posts | NULL       | ALL  | NULL          | NULL | NULL    | NULL | 9696 |    11.11 | Using where |
+----+-------------+-------+------------+------+---------------+------+---------+------+------+----------+-------------+
```

인덱스를 만들었는데도 A와 완전히 같다. `possible_keys`까지 `NULL`이다. 쓸 수 있는 인덱스 후보조차 없다는 뜻이다.

### D. 인덱스를 만든 뒤 앞부분 일치 검색

```sql
EXPLAIN SELECT * FROM posts WHERE title LIKE 'Title 123%';
```

```text
+----+-------------+-------+------------+-------+-----------------+-----------------+---------+------+------+----------+-----------------------+
| id | select_type | table | partitions | type  | possible_keys   | key             | key_len | ref  | rows | filtered | Extra                 |
+----+-------------+-------+------------+-------+-----------------+-----------------+---------+------+------+----------+-----------------------+
|  1 | SIMPLE      | posts | NULL       | range | idx_posts_title | idx_posts_title | 402     | NULL |   11 |   100.00 | Using index condition |
+----+-------------+-------+------------+-------+-----------------+-----------------+---------+------+------+----------+-----------------------+
```

앞의 `%`만 뺐을 뿐인데 `type`이 `range`로 바뀌고, 인덱스를 사용해 **11건만** 읽는다.

### 결과 비교

| 실험 | 조건 | `type` | `key` | `rows` |
|---|---|---|---|---|
| A | 인덱스 없음, `%Title 123%` | `ALL` | `NULL` | 9696 |
| C | 인덱스 있음, `%Title 123%` | `ALL` | `NULL` | 9696 |
| D | 인덱스 있음, `Title 123%` | `range` | `idx_posts_title` | 11 |

## 원인: 인덱스는 정렬된 목록이다

MySQL(InnoDB)의 기본 인덱스는 B-Tree 구조로, **값을 정렬한 상태로** 저장한다. 국어사전을 떠올리면 이해하기 쉽다.

- **"가"로 시작하는 단어 찾기:** 사전은 가나다순으로 정렬되어 있으니 "가" 항목으로 바로 넘어가, "나"가 나올 때까지만 보면 된다.
- **"가"가 들어간 단어 찾기:** 어느 쪽에 있는지 알 수 없으니 사전 전체를 처음부터 끝까지 넘겨야 한다.

`LIKE`도 똑같다.

- **D의 `Title 123%`:** 앞부분이 정해져 있다. 정렬된 인덱스에서 `Title 123`이 시작하는 위치로 바로 이동하고, 접두사가 달라지는 지점에서 멈추면 된다. 그래서 `Title 123`과 `Title 1230`부터 `Title 1239`까지, 정확히 11건만 읽었다.
- **C의 `%Title 123%`:** 앞부분이 무엇이든 될 수 있다. 시작 위치를 정할 수 없으니 정렬이 아무 도움이 되지 않고, 모든 행을 확인해야 한다.

결국 인덱스를 쓸 수 있는지는 **검색 조건이 시작 위치를 정해 줄 수 있는지**에 달려 있다.

## 결과에서 본 다른 숫자들

- **`rows` 9696:** 실제로는 1만 건이 넘는데 9696으로 나왔다. `rows`는 테이블 통계로 낸 **예상값**이라 실제 행 수와 다를 수 있다.
- **`key_len` 402:** 인덱스에서 사용한 키의 바이트 수다. `title`은 `varchar(100)`이고 `utf8mb4`는 한 글자에 최대 4바이트이므로 400바이트, 여기에 문자열 길이를 저장하는 2바이트를 더해 402가 된다.
- **`Using index condition`:** 테이블의 행을 읽기 전에 인덱스 단계에서 조건을 먼저 걸러내는 최적화(Index Condition Pushdown)가 적용됐다는 뜻이다.

## 덤: 소문자로 검색했는데 대문자 제목이 나온 이유

검색 기능을 테스트할 때 `spring`으로 제목을 검색했는데 `Spring guide`가 결과에 나왔다. Java 코드에서는 대소문자를 따로 처리하지 않았다.

원인은 **콜레이션(collation)**, 즉 DB가 문자열을 비교하고 정렬하는 규칙이다. MySQL 8.0부터 기본 콜레이션은 `utf8mb4_0900_ai_ci`인데, 끝의 두 부분이 비교 규칙을 나타낸다.

- **`ai`(accent insensitive):** 악센트를 구분하지 않는다.
- **`ci`(case insensitive):** 대소문자를 구분하지 않는다.

컬럼의 콜레이션은 아래 명령으로 확인할 수 있다.

```sql
SHOW FULL COLUMNS FROM posts;
```

같은 Java 코드라도 DB의 콜레이션에 따라 검색 결과가 달라질 수 있다는 점을 알아 두면 좋다.

## 그래서 게시판 검색은 어떻게 했나

`Containing`은 항상 `LIKE '%키워드%'`를 만든다. 따라서 **`title`에 인덱스를 만들어도 지금 검색은 빨라지지 않는다.** 데이터가 많아질 때 고려할 수 있는 방법은 세 가지다.

| 방법 | 장점 | 단점 |
|---|---|---|
| 앞부분 일치(`StartingWith`) | B-Tree 인덱스를 그대로 사용 | "제목이 ~로 시작"하는 글만 찾을 수 있음 |
| MySQL FULLTEXT 인덱스(ngram 파서) | DB 안에서 해결, 한국어도 글자 단위로 검색 가능 | 인덱스가 커지고, 검색어 길이와 검색 방식에 제약이 있음 |
| Elasticsearch, OpenSearch | 형태소 분석, 관련도 정렬 등 검색 기능이 가장 풍부 | 별도 서버 운영과 DB와의 데이터 동기화가 필요 |

지금 게시판 규모에서는 전체 스캔으로도 충분하다고 판단해 검색 방식은 그대로 두었다. 대신 실험용으로 만든 인덱스는 지웠다.

```sql
DROP INDEX idx_posts_title ON posts;
```

지운 이유는 두 가지다.

- **쓰기 비용:** 인덱스는 글을 쓰거나 수정할 때마다 함께 갱신해야 한다. 쓰이지 않는 인덱스는 비용만 남긴다.
- **코드와 DB의 불일치:** JPA 엔티티에 정의되지 않은 인덱스가 DB에만 있으면, 코드만 보고는 DB 구조를 알 수 없다.

## 정리

- MySQL의 B-Tree 인덱스는 값을 정렬해서 저장하므로, **앞부분이 정해진 조건**에서만 시작 위치를 찾을 수 있다.
- `LIKE 'abc%'`는 인덱스로 범위를 찾지만(`range`), `LIKE '%abc%'`는 인덱스가 있어도 전체 스캔한다(`ALL`).
- Spring Data JPA의 `Containing`은 `LIKE '%키워드%'`를 만들기 때문에 일반 인덱스의 도움을 받을 수 없다.
- 인덱스가 효과가 있는지는 추측하지 말고 `EXPLAIN`으로 확인한다.
- 대소문자 구분 여부처럼 Java 코드가 아닌 DB 설정이 결과를 바꾸는 경우도 있다.