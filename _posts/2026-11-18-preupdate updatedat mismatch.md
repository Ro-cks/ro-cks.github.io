---
title: "@PreUpdate 때문에 API 응답의 updatedAt이 틀렸던 이유"
date: 2026-11-18 19:00:00 +0900
categories: [Spring, JPA]
tags: [spring-boot, jpa, hibernate, entity-listener, flush, dirty-checking]
---

> 실행 환경: Java 21, Spring Boot 4.1.x, Spring Data JPA(Hibernate), MySQL 8.4

## 상황

스프링 부트로 게시판 CRUD를 만들면서 게시글 수정 API를 구현했다. 작성일(`createdAt`)과 수정일(`updatedAt`)은 직접 넣지 않고, JPA 엔티티 생명주기 콜백으로 자동으로 채우도록 했다.

```java
@Entity
@Table(name = "posts")
public class Post {

    // id, title, content, author 생략

    @Column(nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Column(nullable = false)
    private LocalDateTime updatedAt;

    public void update(String title, String content) {
        this.title = title;
        this.content = content;
    }

    @PrePersist
    void onCreate() {
        LocalDateTime now = LocalDateTime.now();
        this.createdAt = now;
        this.updatedAt = now;
    }

    @PreUpdate
    void onUpdate() {
        this.updatedAt = LocalDateTime.now();
    }
}
```

수정 API의 서비스 코드는 다음과 같다.

```java
@Transactional
public PostResponse update(Long id, PostUpdateRequest request) {
    Post post = postRepository.findById(id)
            .orElseThrow(() -> new PostNotFoundException(id));

    post.update(request.title(), request.content());

    return PostResponse.from(post);
}
```

`save()`를 호출하지 않아도 DB에 반영된다. 트랜잭션 안에서 조회한 엔티티는 영속성 컨텍스트가 관리하고, 트랜잭션이 끝날 때 JPA가 처음 상태와 현재 값을 비교해 `UPDATE` SQL을 자동으로 실행하기 때문이다. 이를 **변경 감지(dirty checking)**라고 한다.

## 예상

`@PreUpdate`가 수정될 때 `updatedAt`을 현재 시각으로 바꿔 주니, PUT 응답의 `updatedAt`에는 수정 시각이 담기고 직후 GET으로 조회한 값과도 같을 거라고 생각했다.

## 확인

생성과 수정 사이에 시간 차이가 분명히 나도록 3초를 기다린 뒤, 세 응답의 `updatedAt`을 비교했다.

```bash
BASE="http://localhost:8080/api/posts"

CREATED=$(curl -s -X POST "$BASE" -H "Content-Type: application/json" \
  -d '{"title":"Before","content":"Before content","author":"rocks"}')
ID=$(echo "$CREATED" | python -c "import sys, json; print(json.load(sys.stdin)['id'])")

sleep 3

UPDATED=$(curl -s -X PUT "$BASE/$ID" -H "Content-Type: application/json" \
  -d '{"title":"After","content":"After content"}')
FOUND=$(curl -s "$BASE/$ID")

echo "POST 응답: $(echo "$CREATED" | python -c "import sys, json; print(json.load(sys.stdin)['updatedAt'])")"
echo "PUT 응답:  $(echo "$UPDATED" | python -c "import sys, json; print(json.load(sys.stdin)['updatedAt'])")"
echo "GET 응답:  $(echo "$FOUND" | python -c "import sys, json; print(json.load(sys.stdin)['updatedAt'])")"
```

```text
POST 응답: 2026-11-18T19:51:55.1700273
PUT 응답:  2026-11-18T19:51:55.170027
GET 응답:  2026-11-18T19:51:59.678376
```

제목과 내용은 정상적으로 바뀌었고, 로그에도 `update posts` SQL이 찍혔다. 그런데 **PUT 응답의 `updatedAt`은 생성 시각 그대로**였고, 직후 GET으로 조회해야 새 값이 보였다. POST와 PUT의 소수점 마지막 자리가 다른 이유는 글 뒷부분에서 따로 다룬다. DB에는 제대로 저장됐지만, 수정 API가 틀린 값을 응답한 것이다.

에러 로그가 전혀 남지 않기 때문에, 응답 값을 직접 비교하지 않으면 놓치기 쉬운 버그다.

## 원인: 응답을 만든 시점과 @PreUpdate가 실행된 시점

수정 요청이 처리되는 순서를 따라가 보면 원인이 보인다.

| 순서 | 동작 | `updatedAt` |
|---|---|---|
| 1 | `findById()`로 엔티티 조회, 영속성 컨텍스트가 관리 시작 | 이전 값 |
| 2 | `post.update()`로 제목과 내용 변경, 아직 SQL 없음 | 이전 값 |
| 3 | `PostResponse.from(post)`로 응답 DTO 생성 | **이전 값이 복사됨** |
| 4 | 메서드 종료, 트랜잭션 커밋 전에 flush 시작 | 이전 값 |
| 5 | 변경 감지 후 `@PreUpdate` 실행 | 새 값 |
| 6 | `UPDATE` SQL 실행 후 커밋 | DB에 새 값 저장 |

핵심은 `@PreUpdate`가 실행되는 시점이다. 이름의 `Update`는 엔티티의 `update()` 메서드가 아니라 **DB의 `UPDATE` SQL**을 가리킨다. 즉 `@PreUpdate`는 값을 바꾼 순간이 아니라, 변경 내용을 SQL로 DB에 보내는 **flush 직전**에 실행된다.

그런데 응답 DTO는 3번에서 이미 만들어졌다. 그래서 5번에서 바뀐 `updatedAt`은 응답에 담기지 못하고 DB에만 저장됐다.

## 해결 방법 두 가지

### A. 응답을 만들기 전에 flush하기

```java
post.update(request.title(), request.content());
postRepository.flush();
return PostResponse.from(post);
```

`flush()`를 먼저 호출하면 4~5번이 응답을 만들기 전에 일어나므로, `@PreUpdate`가 바꾼 값이 응답에 담긴다.

처음에는 "flush를 먼저 하면, 그 뒤에 응답을 만들다 실패했을 때 수정만 저장되는 것 아닌가?"라고 생각했다. 하지만 **flush와 commit은 다르다.**

- **flush:** 변경 내용을 SQL로 DB에 보낸다. 트랜잭션은 아직 끝나지 않는다.
- **commit:** 트랜잭션을 확정한다.

flush 뒤에 예외가 발생해도 commit 전이라 `UPDATE`는 롤백된다. 따라서 A 방법이 데이터 정합성을 깨뜨리지는 않는다.

### B. update()에서 updatedAt을 직접 바꾸기

```java
public void update(String title, String content) {
    this.title = title;
    this.content = content;
    this.updatedAt = LocalDateTime.now();
}
```

그리고 `@PreUpdate`가 붙은 `onUpdate()` 메서드를 지운다.

`@PreUpdate`를 남겨 두면 안 된다. `update()`에서 한 번, flush 때 `@PreUpdate`에서 또 한 번 시각이 바뀌기 때문이다. 그러면 응답에는 `update()` 때의 시각이, DB에는 flush 때의 시각이 저장되어 두 값이 다시 어긋난다.

### 두 방법 비교

| 방법 | 장점 | 단점 |
|---|---|---|
| A. `flush()` | 실제로 값이 바뀔 때만 `updatedAt`이 갱신됨 | 서비스가 flush 시점을 직접 신경 써야 함 |
| B. `update()`에서 갱신 | 코드가 단순하고, DB 없이 단위 테스트로 검증 가능 | 같은 내용으로 수정해도 `updatedAt`이 바뀌고 `UPDATE` SQL이 실행됨 |

B의 단점은 변경 감지와 관련이 있다. 제목과 내용이 그대로여도 `updatedAt`이 바뀌면 JPA는 엔티티가 변경됐다고 판단해 `UPDATE` SQL을 실행한다.

## 선택: B

게시판에서는 **수정 요청 자체를 수정 시각으로 봐도 문제가 없다**고 판단해 B를 선택했다. 같은 내용으로 다시 저장해도 수정일이 바뀌는 건 사용자가 이해하기 어려운 동작이 아니다.

대신 얻는 것이 더 크다.

- 서비스 코드가 flush 시점을 신경 쓰지 않아도 된다.
- `updatedAt`이 언제 바뀌는지 엔티티 코드만 보고 알 수 있다.
- DB 없이 `update()`만 호출해도 `updatedAt`이 바뀌는지 단위 테스트로 확인할 수 있다.

수정 후 같은 방법으로 다시 확인했다.

```text
POST 응답: 2026-11-18T19:53:46.9813528
PUT 응답:  2026-11-18T19:53:51.0743614
GET 응답:  2026-11-18T19:53:51.074361
```

이제 PUT 응답에도 수정 시각이 담긴다. 다만 GET 응답과 소수점 마지막 자리가 다르다.

## 덤: 소수점 자릿수가 다른 이유

두 실행 결과를 보면 값의 출처에 따라 소수점 자릿수가 다르다.

| 값 | 출처 | 자릿수 |
|---|---|---|
| POST 응답, 수정 후 PUT 응답 | Java에서 방금 만든 `LocalDateTime.now()` | 7자리 |
| GET 응답, 수정 전 PUT 응답 | DB에서 읽어 온 값 | 6자리 |

- **Java:** `LocalDateTime.now()`의 정밀도는 운영체제 시계에 따라 다르다. 이 Windows 환경에서는 100나노초 단위인 7자리까지 나왔다.
- **MySQL:** `created_at`, `updated_at` 컬럼은 `datetime(6)`이라 마이크로초인 6자리까지만 저장한다. `DESC posts;`로 확인할 수 있다.

수정 전 PUT 응답은 DB에서 조회한 엔티티의 값을 그대로 담았기 때문에 6자리였고, 수정 후 PUT 응답은 `update()`에서 방금 만든 값이라 7자리였다. 그래서 같은 시각인데도 응답마다 표현이 달라진다.

API를 쓰는 쪽에서 문자열로 값을 비교하면 문제가 될 수 있다. DB 정밀도에 맞춰 Java에서 미리 잘라 두면 응답이 항상 같아진다.

```java
this.updatedAt = LocalDateTime.now().truncatedTo(ChronoUnit.MICROS);
```

## 참고: Spring Data JPA Auditing을 써도 같은 문제가 생긴다

수정일을 자동으로 채울 때 `@LastModifiedDate`와 `AuditingEntityListener`를 많이 쓴다. 이 방식도 내부적으로 JPA 생명주기 콜백(`@PreUpdate`)을 이용하므로, 응답 DTO를 flush 전에 만들면 같은 문제가 생긴다. 어떤 방식을 쓰든 **값이 언제 채워지는지**를 알고 있어야 한다.

## 정리

- `@PreUpdate`는 엔티티 값을 바꾼 순간이 아니라 **flush 직전**에 실행된다.
- 응답 DTO를 flush 전에 만들면, 콜백이 바꾼 값은 응답에 담기지 않는다.
- flush는 SQL을 DB로 보낼 뿐이고, 확정은 commit에서 일어난다. flush 뒤에 예외가 나도 롤백된다.
- 해결 방법은 응답 전에 flush하거나, 엔티티 메서드에서 값을 직접 바꾸는 것이다. 이 프로젝트에서는 코드가 단순하고 테스트하기 쉬운 후자를 선택했다.
- Java와 DB의 시각 정밀도가 다를 수 있으니, 응답 값을 일관되게 하려면 DB 정밀도에 맞춰 잘라 두는 것이 좋다.