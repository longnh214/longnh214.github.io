---
title: "[Spring/Java] Reflection"
date: 2025-05-20 11:19:43 +09:00
categories: [Java, Spring]
tags:
  [
    Java,
    Spring,
    ComponentScan,
    자바,
    스프링,
    리플렉션,
    Reflection,
    Reflect,
    Reflection API,
  ]
---

# 서론

Reflection은 Java의 기본 입문서에 구성되어있지 않지만 강력한 기능이고, 객체의 암/복호화 기능을 구현할 때 사용했었는데 깊게 공부하고 정리해보고자 한다.

<br>
<br>

# Reflection이 무엇일까?

> 런타임 시점에 클래스를 분석해서 동적으로 값을 수정하고, 생성자와 필드 이름, 메소드 이름과 같은 메타 데이터 정보를 알고 조작할 수 있다.  
> 그 뿐아니라 새로운 객체를 인스턴스화하고, 메소드를 호출할 수도 있다.  
> 번외로 C#, Javascript, Python, PHP도 Reflection을 지원한다.

<br>
<br>

## Reflection이 사용되는 예시

- DI(의존성 주입) 기반 프레임워크 (ex : Spring, Google Guice 등) : 클래스의 생성자나 필드를 런타임에 분석하고 객체를 생성한 뒤 주입한다.

- Spring의 컴포넌트 스캔 : `@ComponentScan`, `@Service`, `@Repository`와 같은 어노테이션에 이용된다.

- Jackson, Gson 라이브러리 : 데이터를 직렬화, 역직렬화 할 때 각 클래스의 구조를 분석해서 동적으로 필드에 값을 주입한다.

- ORM 프레임워크 (Hibernate, JPA) : DB에서 불러온 값을 객체의 각 필드에 값을 넣고 `persist`할 때 이용된다.

<br>
<br>

## 예시 코드

### Reflection 필수 패키지 import

```java
import java.lang.reflect.*;
```

### private 접근 시

```java
field.setAccessible(true);
```

### 클래스의 메소드 조회

```java
public static void main(String[] args) {
    Class clazz = String.class;

    for(Method method : clazz.getDeclaredMethods()){
        System.out.println(method);
    }
}
```
```text
[결과]
byte[] java.lang.String.value()
public boolean java.lang.String.equals(java.lang.Object)
public int java.lang.String.length()
public java.lang.String java.lang.String.toString()
public int java.lang.String.hashCode()
//...
```

> Java에서 Reflection API에는 getXXX 메소드와 getDeclaredXXX 메소드가 아래와 같은 차이가 있다.

| 항목      | `getXXX()`                    | `getDeclaredXXX()`                                                                     |
| --------- | ----------------------------- | -------------------------------------------------------------------------------------- |
| 대상 멤버 | **public** 멤버만 포함        | **모든 접근 제어자**의 멤버 포함 (`private`, `protected`, `package-private`, `public`) |
| 상속 멤버 | **상속된 멤버도 포함**        | **자기 클래스에 선언된 멤버만 포함**                                                   |
| 접근 허용 | 기본적으로 public 멤버만 접근 | private 등 접근 불가 멤버는 `setAccessible(true)` 필요                                 |

<br>
<br>

### 클래스의 필드 설정

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException {
    Class<?> clazz = Class.forName("java.lang.Integer");

    Field field = clazz.getDeclaredField("value");
    Integer i = 100;

    field.setAccessible(true);
    field.set(i, 5);

    System.out.println(i);
}
```
```text
[실행 결과]
WARNING: An illegal reflective access operation has occurred
WARNING: Illegal reflective access by ReflectionTest (file:/bin/) to field java.lang.Integer.value
WARNING: Please consider reporting this to the maintainers of ReflectionTest
WARNING: Use --illegal-access=warn to enable warnings of further illegal reflective access operations
WARNING: All illegal access operations will be denied in a future release
5
```

> 임의로 `java.lang.Integer` 클래스의 value를 100에서 5로 수정해서 출력했다. `setAccessible(true)`옵션을 설정해서 값이 수정은 됐으나, 위와 같이 WARNING 메시지로 경고를 주고 있다.  
> (⚠️ 해당 실행 결과는 Java 11 버전에서 실행했을 때의 결과이다.)
>
> WARNING 로그의 내용을 요약하면 추후 JDK 버전에서는 private 제어자 필드의 접근이 차단될 수 있다는 경고, `setAccessible(true)` 기반 강제 접근은 비공식의 필드 접근 방식이라는 의미이다.
>
> JVM `--illegal-access` 옵션의 값을 수정해서 비공식 필드 접근을 차단할 수 있다. 이 값이 Java 9~15까지는 기본값이 `permit`이다.  
>
> Java 16 이후부터는 `--illegal-access` 옵션의 값이 기본으로 `deny`이며, JVM의 `--add-opens` 옵션이 필수로 존재해야 `setAccessible(true)`가 적용될 수 있다고 한다.

### 특정 생성자를 찾을 때

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException {
    Class<?> clazz = Class.forName("java.lang.String");

    Constructor<?> constructor = clazz.getDeclaredConstructor(String.class);
    Constructor<?> constructor2 = clazz.getDeclaredConstructor(byte[].class, int.class);

    String str = (String) constructor.newInstance("hi");
    String str2 = (String) constructor2.newInstance("hello".getBytes(), 0);

    System.out.println(str);
    System.out.println(str2);
}
```
```text
[실행 결과]
hi
hello
```

> `getConstructor()`혹은 `getDeclaredConstructor()` 메소드에 생성자의 인자의 순서와 타입에 맞게 호출하면 해당 생성자에 대한 `Constructor` 클래스를 반환받을 수 있다.  
> ⚠️ 하지만 생성자의 타입과 순서를 모른다면 원하는 `Constructor`클래스를 반환받을 수 없다.  
> 그리고 `newInstance()`메소드를 통해서 해당 생성자를 실행한다. (반환 타입으로 캐스팅이 필요하다.)

### 특정 메소드를 실행할 때

```java
public static void main(String[] args) throws NoSuchFieldException, IllegalAccessException, ClassNotFoundException, NoSuchMethodException, InvocationTargetException, InstantiationException {
    Class<?> clazz = Class.forName("java.lang.String");

    Constructor<?> constructor = clazz.getDeclaredConstructor(String.class);
    Method substringMethod = clazz.getMethod("substring", int.class);

    String str = (String) constructor.newInstance("Hi, My Name is Mac.");
    String subStr = (String) substringMethod.invoke(str, 7);

    System.out.println(subStr);
}
```
```text
[실행 결과]
Name is Mac.
```

> `getMethod()`혹은 `getDeclaredMethod()`의 인자로 메소드의 이름과 인자의 타입을 담아서 `Method` 객체를 반환받을 수 있다.  
> 그리고 해당 `Method` 객체를 실행하기 위해서 `invoke()` 메소드에 인자를 담아서 호출한다.

<br>
<br>

# Reflection의 장점

| 항목                             | 설명                                                                                       |
| -------------------------------- | ------------------------------------------------------------------------------------------ |
| **유연성 (Flexibility)**         | 컴파일 시점에 타입을 알지 못해도 런타임에 클래스, 필드, 메소드 등에 접근 가능              |
| **프레임워크 개발에 필수**       | Spring, Hibernate, Jackson 등은 객체 자동 생성, 주입, 직렬화/역직렬화 등에 Reflection 활용 |
| **플러그인 구조 구현 가능**      | 외부 모듈이나 클래스 로딩 시 클래스 이름만 알고 있어도 동적으로 로딩하여 사용 가능         |
| **테스트 및 디버깅 도구에 유용** | 객체 내부 상태 점검, 비공개 메소드 호출 등에 활용 가능                                     |

<br>
<br>

# Reflection의 단점

| 항목                        | 설명                                                                           |
| --------------------------- | ------------------------------------------------------------------------------ |
| **성능 저하**               | JVM 최적화에서 제외되기 쉬우며, 직접 호출보다 느림                             |
| **컴파일 타임 안전성 없음** | 오타나 잘못된 접근은 컴파일 에러가 아닌 런타임 에러로 발생                     |
| **보안 이슈**               | `setAccessible(true)`를 사용하면 private 필드도 조작 가능 → 보안상 취약점 가능 |
| **코드 유지보수 어려움**    | 명확하지 않은 코드 흐름으로 인해 디버깅이 어렵고 유지보수성 저하               |

<br>
<br>

## 결론

Reflection의 장점과 단점, 기본적인 사용법에 대해서 복습할 수 있었다.
장점인 확장성과 유연성에 따라 실제로 사용했던 이유도 런타임 환경에서 클래스의 값을 조작(암호화, 복호화)하기 위해서 사용했었다.  
Hibernate, JPA, 스프링에서도 사용되어지고 있는 Java 기술인데, 특수한 경우가 아니라면 어플리케이션 개발자가 사용할 일은 적을 것이다.  
추후에는 Hibernate와 Reflection의 관계, JDK 버전 별 Module과 Reflection의 관계에 대해서 다뤄볼 것이다.

<br>
<br>

## 출처

- https://www.baeldung.com/java-reflection
- https://www.baeldung.com/java-reflection-benefits-drawbacks
- https://www.oracle.com/technical-resources/articles/java/javareflection.html
- java.lang.reflect.*
