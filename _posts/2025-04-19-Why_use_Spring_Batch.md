---
title: "[Spring/Spring Batch] 배치 처리와 Spring Batch란 무엇일까?"
date: 2025-04-19 13:59:23 +09:00
categories: [Spring, Spring Batch]
tags:
  [
    Spring Batch,
    Spring Boot,
    스프링 배치,
    스프링 부트,
  ]
---

# 서론 및 배경

IT 기술이 발전하면서 한 번에 처리해야하는 데이터 양이 늘었고 이를 효율적이고 안정적으로 처리하기 위한 기술이 필요해졌다.  
이를 '배치 처리'라고 표현한다. 다시 요약하면 '대량의 데이터를 일정 시간에 모아서 한꺼번에 처리한다.'는 의미이다.  
보통 사용자의 활동 로그 정리, 정산, 데이터 마이그레이션, 정해진 시각에 특정 사용자에게 이메일 발송 등 다양한 상황에서 배치 처리가 사용될 수 있다.  
그 중에서 Java, Spring에서 배치 처리에 대해 표준으로 사용되는 Spring Batch 프레임워크에 대해 조금 더 깊게 공부하고자 한다.

## 배치 처리의 기본적인 흐름

> 단순하게 얘기하면 Read, Process, Write로 나뉜다. 이는 ETL(Extract, Transform, Load) 프로세스로 칭한다.

- Read : DB, 파일, 큐에서 데이터를 조회한다.
- Process : Read에서 조회한 데이터를 가공한다.
- Write : 가공된 데이터를 포맷(규격)에 맞게 새로 저장/처리한다.

## Spring Batch란 무엇인가?

Spring Batch는 대량의 데이터를 효율적으로 처리할 수 있도록 도와주는 Java, Spring 기반의 표준 배치 처리 프레임워크이다.  
[Spring Batch Introduction](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/spring-batch-intro.html) (Spring Batch 공식 문서)에서 특징을 확인할 수 있다.

## Spring Batch 시나리오

- 배치 프로세스를 주기적으로 커밋
- 동시에 진행되는 Job의 배치 처리 (= 대용량 병렬 처리, 멀티스레드로 Job 처리)
- 실패 후 수동 또는 스케줄링에 의한 재시작
- 의존 관계가 있는 로직(Step)을 순차적으로 처리
- 조건적인 흐름, 로직을 구성해서 체계적이고 유연한 배치 구성
- 실패 후 재시도, 반복, Skip, 실패 처리

## Spring Batch 구조

![이미지](../assets/img/develop/spring-batch-layers.png)

Spring Batch의 구조는 Application, Batch Core, Batch Infrastructure로 구성되어있다.

- Application : 어플리케이션에는 사용자가 정의한 배치의 Job과 Step의 코드가 포함되어있다.
- Batch Core : 배치 작업(Job)을 시작하고 제어하는 데에 필요한 핵심 런타임 클래스가 포함되어있다. 예를 들어 `JobLauncher`, `Job`, `Step`이 있다.
- Batch Infrastructure : Application과 Core 모두 Infrastructure에서 빌드되며 Job의 실행 처리와 흐름의 틀을 제공한다. `Reader`, `Processor`, `Writer`, `Retry`, `Skip`과 같은 로직이 포함된다.

### 출처

- [Spring Batch - 정수원님 인프런 강의](https://www.inflearn.com/course/%EC%8A%A4%ED%94%84%EB%A7%81-%EB%B0%B0%EC%B9%98/dashboard)
- [Spring Batch Introduction](https://docs.spring.io/spring-batch/docs/4.3.x/reference/html/spring-batch-intro.html)
