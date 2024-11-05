---
title: "[Spring/JPA] JPA Batch Insert에 대해서 깊게 알아보자."
date: 2024-10-21 13:43:53 +09:00
categories: [Spring, JPA]
tags:
  [
    Spring,
    Spring Boot,
    Spring Framework,
    JPA,
    Spring Data JPA,
    Batch Insert,
    saveAll,
    스프링,
  ]
---

# Batch Insert

개발을 진행하던 중에 대량의 데이터를 insert 해야하는 일이 있었는데 당시 환경에서 소요 시간이 꽤 걸려서 성능 향상을 위해 batch insert를 도입했던 내용을 정리해보고자 한다.

## Batch Insert란?

```sql
# 일반적인 개별 insert
insert into member (username, age) values (?, ?)
;

# 배치 insert
insert into member (username, age)
    values
            ('안유진', 21),
            ('이영지', 23),
            ('미미', 30),
            ('이은지', 32)
;
```

DB에 여러 레코드를 삽입 시에 단건 insert문을 N번 반복하는 것이 아닌 여러 row를 한 번에 연결해서 한 번에 insert하는 방식을 `batch insert`라고 한다. 이 때 batch insert는 하나의 트랜잭션에 묶이게 된다.

## Hibernate에서 Batch Insert의 제약 사항

[Hibernate ORM User Guide - 12.2.1 Batch Inserts](https://docs.jboss.org/hibernate/orm/5.4/userguide/html_single/Hibernate_User_Guide.html#batch-session-batch-insert)를 참고하면 아래와 같은 문구가 있다.

> Hibernate disables insert batching at the JDBC level transparently if you use an identity identifier generator.

식별자 방식에 `IDENTITY` 방식은 batch insert에 사용될 수 없다는 내용이다.

식별자 생성 방식은 `GenerationType.AUTO`, `GenerationType.IDENTITY`, `GenerationType.TABLE`, `GenerationType.SEQUENCE`가 있다.

기본키 생성 전략에 대해 간단하게 얘기하면, 아래와 같이 네 종류가 있다.

- AUTO : DB dialect에 따른 Hibernate의 식별자 생성 타입 default 설정하는 방식
- IDENTITY : MySQL의 auto increment와 같이 DB에 식별자 생성을 위임하는 방식 (DB에 레코드를 insert한 후 기본 키 값을 조회할 수 있다.)
- SEQUENCE : **DB SEQUENCE**를 사용해 기본 키를 할당하는 방식으로 메모리를 활용해 Application 단에서 sequence 값을 할당할 수 있다. (DB에 레코드를 insert하기 전에 기본키 값을 알 수 있다.)
- TABLE : 키 생성 전용 테이블을 별도로 사용하는 전략이다. DB SEQUENCE를 지원하지 않을 때에 흉내내는 전략으로 DB에서 기본키에 넣을 값을 별도 테이블에서 조회해서 insert 후 값 증가를 위해 update 쿼리를 한 번 더 사용해야한다.

## 왜 Batch Insert에서 IDENTITY 키 생성 방식이 지원되지 않는가?

[Why does hibernate disable insert batching when using an identity identifier generator](https://stackoverflow.com/questions/27697810/why-does-hibernate-disable-insert-batching-when-using-an-identity-identifier-gen/27732138#27732138)를 참고하면, hibernate에서는 `Transactional Write behind` 방식(쓰기 지연)을 사용하므로 entity를 persist하기 위해서는 @Id 어노테이션의 멤버 변수에 값이 필요한데 `IDENTITY`방식으로는 insert를 실행하기 전에 기본키에 할당할 값을 알 수 없기 때문에 Batch insert를 지원할 수 없다.

## SEQUENCE 방식 도입

MySQL을 사용하던 프로젝트에서 IDENTITY 방식을 사용할 수 없다면 매우 치명적이다. 그래서 키 생성 방식을 변경하지 않고 도입하려면 `Spring Data JDBC`의 `JdbcTemplate.batchUpdate()`메소드를 통해 구현할 수 있다.

하지만 다른 이슈로 인해 프로젝트 DB 스펙이 MySQL에서 MariaDB로 변경되었고, 이를 기점으로 Sequence 방식으로 변경하며 도입했다.

## Batch Insert 테스트 스펙

> DB는 h2를 이용해서 테스트를 진행하였다.

### Member Entity class

```java
@Entity
@Getter
@NoArgsConstructor(access = AccessLevel.PROTECTED)
@AllArgsConstructor
public class Member {
    @Id
    @SequenceGenerator(
          name = "MEMBER_SEQ_GENERATOR",
          sequenceName = "MEMBER_SEQ",  // 데이터베이스에 등록되어있는 시퀀스 이름: DB에는 해당 이름으로 매핑된다.
          initialValue = 1,  // DDL 생성시에만 사용되며 시퀀스 DDL을 생성할 때 처음 시작하는 수를 지정
          allocationSize = 50  // 시퀀스 한 번 호출에 증가하는 수
    )
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "MEMBER_SEQ_GENERATOR")

//  @GeneratedValue(strategy = GenerationType.IDENTITY) //IDENTITY와 SEQUENCE 테스트용
    @Column(name = "member_id")
    private Long id;
    private String username;
    private int age;

    public Member(String username) {
        this.username = username;
    }

    public Member(String username, int age) {
        this.username = username;
        this.age = age;
    }
}
```
SequenceGenerator 어노테이션을 통해 시퀀스 생성을 해주었다.

### applicaton.yml

```yml
spring:
  jpa:
    hibernate:
      ddl-auto: create
    properties:
      hibernate:
        generate_statistics: true
        show_sql: true
        format_sql: true
        jdbc:
          batch_size: 1000
        use_sql_comments: true
            
```
DB의 캐시 초기화를 위해 `ddl-auto` 값을 `create`로 설정했고, 쿼리의 소요 시간을 조회하기 위한 `generate_statistics`값을 `true`로 주었다.

### saveAll test code

```java
@Test
public void saveAllMember100Test() throws IOException {
    BufferedWriter bw = new BufferedWriter(new OutputStreamWriter(System.out));
    long start = System.currentTimeMillis();
    List<Member> memberList = new ArrayList<>();

    for(int i=0;i<100000;i++){
        memberList.add(new Member("memberName" + i));
    }
    memberRepository.saveAll(memberList);

    long end = System.currentTimeMillis();

    bw.write("total time : " + (end - start) + "\n");
    bw.flush();
    bw.close();
}
```
10만 건을 대상으로 테스트 진행을 하였다.

## 테스트 진행

### 키 생성 IDENTITY 방식 소요 시간 측정

<img width="722" alt="identitySaveAll100000" src="https://github.com/user-attachments/assets/438aa67f-c86a-4f1e-b9d8-2c891f8929a4">
<img width="846" alt="identitySaveAll100000H2" src="https://github.com/user-attachments/assets/0a1821d5-a129-4172-bf31-adb855d00af6">

saveAll의 소요 시간이 약 110,000ms이고, batch insert는 적용되지 않은 모습을 볼 수 있다.

### 키 생성 SEQUENCE 방식 소요 시간 측정

<img width="534" alt="SequenceSaveAll100000" src="https://github.com/user-attachments/assets/1ff32fde-0df0-462c-85f1-7a2acfaf0939">
<img width="862" alt="SequenceSaveAll100000H2" src="https://github.com/user-attachments/assets/2a9e7cf2-5797-4059-b28e-50ed796831f9">

Batch insert가 적용되어 JDBC batches 부분이 100,000 / 1,000(batch_size) = 100번으로 확인되었고, saveAll 소요시간은 약 2,700ms로 확인되어 현저히 줄어들었다.
`call next value for member_seq`로 인해 `SEQUENCE`방식으로 기본키를 채번하는 것을 알 수 있다.

하지만 statistics의 flush nanosecond 소요 시간을 보면, SEQUENCE 방식 보다 IDENTITY 방식의 소요 시간이 적다는 것을 알 수 있다. 여기에서 주의해야 할 부분이 있다.

## 주의해야 할 부분

> batch_size가 높으면 높을 수록 좋을까?

아니다. batch_size가 커지면 쓰기 지연 저장소에 batch_size만큼의 데이터가 영속화되어 일시적으로 저장되기 때문에 메모리에 부하가 생기고, 트랜잭션의 크기가 커져서 하나의 레코드에 이상이 있다면 batch_size 만큼 모든 레코드의 롤백이 일어날 수 있다. 메모리와 DB 성능에 따라 batch_size를 조절해야한다.

> 그렇다면 SEQUENCE 방식이 IDENTITY 방식보다 성능이 좋을까?

SEQUENCE 방식으로 테스트를 진행함으로서 batch insert가 적용된 것을 알 수 있었다.

하지만 flush 과정에서 14배 차이가 날 정도로 SEQUENCE 방식이 오래걸리는 것을 알 수 있었다. 이 부분은 SEQUENCE 방식은 application 단계에서 기본키를 채번하는데 이 과정에서 flush 해야할 내용이 많아진 듯 보인다. 하지만 모든 소요 시간을 합했을 때 SEQUENCE 방식이 적었다.

## 설정

- order_insert
- order_update

위와 같은 설정들을 `application.yml`에 설정할 수 있다. 이 설정들은 SQL 쿼리의 실행 순서를 정렬하는 옵션이다.

예를 들어 아래와 같이 한 트랜잭션에 부모의 save, 자식의 save 번갈아서 하는 로직이 있을 때 정렬하지 않는다면 쿼리도 번갈아서 나갈 것이다.

```sql
insert into 부모 values a1;
insert into 자식 values b1;
insert into 부모 values a2;
insert into 자식 values b2;
```

하지만 order 설정을 해준다면, 아래와 같이 쿼리를 묶어서 수행할 수 있다.

```sql
insert into 부모 values (a1, a2);
insert into 자식 values (b1, b2);
```

## 결론

- 대량의 데이터를 insert 해야할 때에는 Spring data JDBC의 `batchUpdate()`메소드를 이용한 batch insert를 고려해야겠다.

- Spring Data JPA를 이용해서 batch insert를 해야한다면 기본키를 batch로 채번하고 sequence 방식을 이용한다.

- insert 소요 시간이 전부가 아니고 총 소요시간이 중요하다는 것을 알 수 있었다.

## 출처

- [JPA Generation별 INSERT 성능 비교](https://github.com/HomoEfficio/dev-tips/blob/master/JPA-GenerationType-별-INSERT-성능-비교.md)
- [Spring Data에서 Batch Insert 최적화](https://homoefficio.github.io/2020/01/25/Spring-Data%EC%97%90%EC%84%9C-Batch-Insert-%EC%B5%9C%EC%A0%81%ED%99%94/)
- [Batch Insert 성능 향상기 1편](https://cheese10yun.github.io/jpa-batch-insert/)