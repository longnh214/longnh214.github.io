---
title: "[Java/자료구조] Stack vs Queue"
date: 2025-04-21 20:12:23 +09:00
categories: [Java, Data Structure, 알고리즘]
tags: [
    Java,
    algorithm,
    알고리즘,
    Stack,
    Queue,
    스택,
    큐,
    자바,
]
---

# 서론

평소에 Java로 알고리즘 문제를 풀면서 사용되는 `Stack`, `Queue` 자료구조와 `java.util.*` 패키지에 구현된 스택과 큐 클래스에 대해 더 알아보고자 한다.

## Stack과 Queue의 정의

> 간단하게 Stack과 Queue를 정의하면 Stack은 후입선출, LIFO(Last-In-First-Out) 형태의 자료구조이고, Queue는 선입선출, FIFO(First-In-First-Out)형태의 자료구조이다.

## Stack과 Queue가 사용되는 예시

### Stack

- Java 내에서 Stack 영역에 지역 변수와 함수 콜스택이 관리되는 부분
- HTML 태그와 같이 괄호를 검증
- 하노이 탑, DFS 구현 시 재귀

### Queue

- 운영체제의 라운드로빈 방식의 프로세스 스케줄링
- BFS 구현 시 큐
- Kafka, RabbitMQ와 같은 메시지 큐
- 예매 사이트의 대기열

## 왜 자바 공식 문서에서는 Stack을 권장하지 않을까?

> "A more complete and consistent set of LIFO stack operations is provided by the Deque interface and its implementations, which should be used in preference to this class."
>
> 보다 완전하고 일관된 LIFO 스택 작업은 Deque 인터페이스 및 해당 구현에 의해 제공됩니다.

[Java 공식 문서](https://docs.oracle.com/javase/8/docs/api/java/util/Stack.html)에는 위와 같이 `Stack` 사용을 지양하고, `Deque` 인터페이스 사용을 권장하고 있다. 이유가 무엇일까?

### Stack == class

`Stack`은 클래스이다. Java에서 클래스는 다중 상속을 지원하지 않고 구현 클래스 변경에 취약하다.
반면 `Deque`는 인터페이스이다. 다중 상속이 지원되고 객체지향적으로 유연하게 사용할 수 있다.

### Stack은 Vector 기반의 클래스이다.

`public class Stack<E> extends Vector<E> { ... }` 형태로 Stack은 Vector를 상속받은 클래스이다.

```java
public synchronized E pop() {
    E obj;
    int len = size();

    obj = peek();
    removeElementAt(len - 1);

    return obj;
}
```

Stack의 `push()`, `pop()`메소드에 `synchronized`키워드가 붙어있어 실행 때마다 스레드의 동기화를 체크하므로 성능이 좋지 않다.

### Stack은 인덱스 접근이 가능하다.

```java
public synchronized E get(int index) {
    if (index >= elementCount)
        throw new ArrayIndexOutOfBoundsException(index);

    return elementData(index);
}
```

- Stack은 push, pop, peek 메소드만 있으면 되는데 Vector 클래스를 상속받았기 때문에 Vector 내의 `get(int index)`메소드가 호출이 가능해져 인덱스 접근이 가능하다.

- Stack은 ISP(인터페이스 분리 원칙)에 어긋난 클래스이기 때문에 객체지향적으로 복잡하고 원하지 않는 기능까지 따라올 경우가 (get 메소드처럼) 있다.

## Stack과 Queue는 내부적으로 어떤 자료구조를 이용할까?

### Stack

```java
public class Vector<E>
    extends AbstractList<E>
    implements List<E>, RandomAccess, Cloneable, java.io.Serializable{
    protected Object[] elementData;
    //...
}
```

> Stack의 내부 Vector 클래스 안에 요소는 배열로 선언되어있는 것을 알 수 있다.

### Queue (ArrayQueue)

```java
public class ArrayQueue<T> extends AbstractList<T> {
    private T[] queue;
    //...
}
```

> Queue의 구현체 중 하나인 `ArrayQueue`클래스 내에는 T[] 배열 타입으로 요소가 이루어져 있다.

### Queue (LinkedList)

#### LinkedList.java

```java
public class LinkedList<E>
    extends AbstractSequentialList<E>
    implements List<E>, Deque<E>, Cloneable, java.io.Serializable{
    transient Node<E> first;
    transient Node<E> last;
    //...
}
```

#### Node.java

```java
private static class Node<E> {
    E item;
    Node<E> next;        
    Node<E> prev;

    Node(Node<E> prev, E element, Node<E> next) {
        this.item = element;
        this.next = next;
        this.prev = prev;
    }
}
```

> Queue의 구현체 `LinkedList`에는 내부에 `Node<E>` 노드 클래스 기반의 양방향 탐색이 가능한 이중 연결 리스트로 구현이 되어있다.

## Deque의 구현체 : ArrayDeque vs LinkedList

### ArrayDeque

```java
public class ArrayDeque<E> extends AbstractCollection<E>
                           implements Deque<E>, Cloneable, Serializable{
    transient Object[] elements;
    //...
}
```

> Deque의 구현체 ArrayDeque는 내부적으로 `Object[] elements;`와 같이 배열로 구성되어있다.

### LinkedList

> LinkedList는 위의 Queue 구현체 LinkedList와 같이 내부적으로 addFirst, addLast, pollFirst, pollLast 메소드가 구현되어있다.

## 결론

`Stack` 클래스는 동기화 문제와 객체지향에 벗어난 클래스이기 때문에 Java 공식 문서에서도 사용을 지양해야한다는 의견이 있었다. 스택을 구현하기 위해서는 `Deque` 인터페이스를 이용하거나 직접 구현하는 것이 권장된다.  
`Queue`인터페이스와 `Deque`인터페이스는 Array와 LinkedList로 구현이 가능하고, 실제로 구현한다면 삽입과 삭제의 로직이 빈번하게 일어나는 자료구조이기 때문에 LinkedList가 유리할 것 같았지만 배열 기반 Array* 구현체가 메모리 효율 상 성능이 더 좋다고 한다.

### 출처

- [Java 공식 문서](https://docs.oracle.com/javase/8/docs/api/java/util/Stack.html)
- `java.util.*` 패키지
