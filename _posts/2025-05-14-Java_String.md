---
title: "[Java] Java의 String (Java 11)"
date: 2025-05-14 14:22:31 +09:00
categories: [Java, String, 알고리즘]
tags: [
    Java,
    algorithm,
    알고리즘,
    String,
    LPS,
    KMP,
    문자열,
    contains,
    indexOf,
    replace,
    compareTo,
    substring
]
---

# 서론

Java에서 문자열 중 특정 정규 표현식이나 문자열이 포함되어있는지 확인하는 메소드 `String.contains()` 를 포함해서 `String` 객체에서 자주 사용하는 메소드들의 내부 로직이 어떻게 구성되어있는지 알아보자.  

⚠️ 대부분의 코드는 Java 11을 예시로 든다.

# 목차

* 데이터 관리 자료구조
* `compareTo()`
* `contains()`와 `indexOf()`
* `startsWith()`와 `endsWith()`
* `substring()`
* `replace(char oldChar, char newChar)`

# 데이터 관리 자료구조

```java
public final class String
    implements java.io.Serializable, Comparable<String>, CharSequence {
    // ~ Java 8
    @Stable
    private final char [] value;

    // Java 9 ~
    @Stable
    private final byte [] value;
    private final byte coder; // LATIN1(1 byte) / UTF-16(2 byte)
    //...
}
```

> Java 8 이전의 `String.class`는 값을 `char []` 형태로 관리하고 있다.  
> Java 9 이후 버전의 `String.class`는 값을 `byte []` 형태로 관리하고 있으며, `coder`라는 변수에서 byte 배열을 1 byte(Latin1)로 해석할 것인지, 2 byte(UTF-16)로 해석할 것인지 변수화 했다. `byte []` 형태로 값을 관리함으로써 ASCII 문자만 사용하는 경우에 `char []` 형태보다 메모리를 절약할 수 있다.

<br>
<br>

# `compareTo()`

> `compareTo()`는 `String.compareTo(String anotherString)` 형태로 다른 문자열과 비교할 때 사용하는 메소드로 각각 결과값이 음수, 0, 양수에 따라 다음과 같은 의미를 가진다.

| result | `str1.compareTo(str2)` 의미                  |
| ------ | -------------------------------------------- |
| 음수   | `str1`이 `str2`에 비해 사전순으로 앞에 있다. |
| 0      | `str1`과 `str2`의 문자열이 같다.             |
| 양수   | `str1`이 `str2`에 비해 사전순으로 뒤에 있다. |

```java
//... String.java (Java 11)
public int compareTo(String anotherString) {
    byte v1[] = value;
    byte v2[] = anotherString.value;
    if (coder() == anotherString.coder()) {
        return isLatin1() ? StringLatin1.compareTo(v1, v2)
                          : StringUTF16.compareTo(v1, v2);
    }
    return isLatin1() ? StringLatin1.compareToUTF16(v1, v2)
                      : StringUTF16.compareToLatin1(v1, v2);
}
//...
```

```java
// StringLatin1.java 
@HotSpotIntrinsicCandidate
public static int compareTo(byte[] value, byte[] other) {
    int len1 = value.length;
    int len2 = other.length;
    return compareTo(value, other, len1, len2);
}

public static int compareTo(byte[] value, byte[] other, int len1, int len2) {
    int lim = Math.min(len1, len2);
    for (int k = 0; k < lim; k++) {
        if (value[k] != other[k]) {
            return getChar(value, k) - getChar(other, k);
        }
    }
    return len1 - len2;
}
```

```java
// StringUTF16.java
@HotSpotIntrinsicCandidate
public static int compareTo(byte[] value, byte[] other) {
    int len1 = length(value);
    int len2 = length(other);
    return compareValues(value, other, len1, len2);
}

private static int compareValues(byte[] value, byte[] other, int len1, int len2) {
    int lim = Math.min(len1, len2);
    for (int k = 0; k < lim; k++) {
        char c1 = getChar(value, k);
        char c2 = getChar(other, k);
        if (c1 != c2) {
            return c1 - c2;
        }
    }
    return len1 - len2;
}
```

`String.compareTo()` 메소드에서 Latin1, UTF-16 인코딩에 따라 compareTo도 분기처리 되어있는 점을 알 수 있고, 두 문자열을 각 인덱스 별 문자끼리 비교해서 다르다면 인덱스 차이만큼 반환하고, 이외에는 두 문자열의 길이 차이만큼 반환한다.

<br>
<br>

# `contains()`와 `indexOf()`

문자열 안에 특정 문자열이 포함되어있는지 판별하는 `contains()`는 내부적으로 `indexOf()` 메소드가 호출된다.

```java
//...
public boolean contains(CharSequence s) {
    return indexOf(s.toString()) >= 0;
}

public int indexOf(String str) {
    if (coder() == str.coder()) {
        return isLatin1() ? StringLatin1.indexOf(value, str.value)
                            : StringUTF16.indexOf(value, str.value);
        }
    if (coder() == LATIN1) {  // str.coder == UTF16
        return -1;
    }
    return StringUTF16.indexOfLatin1(value, str.value);
}
//...
```

```java
// StringLatin1.java
@HotSpotIntrinsicCandidate
public static int indexOf(byte[] value, int valueCount, byte[] str, int strCount, int fromIndex) {
    byte first = str[0];
    int max = (valueCount - strCount);
    for (int i = fromIndex; i <= max; i++) {
        // Look for first character.
        if (value[i] != first) {
             while (++i <= max && value[i] != first);
        }
        // Found first character, now look at the rest of value
        if (i <= max) {
            int j = i + 1;
            int end = j + strCount - 1;
            for (int k = 1; j < end && value[j] == str[k]; j++, k++);
            if (j == end) {
                // Found whole string.
                return i;
            }
        }
    }
    return -1;
}
//...
```
```java
// StringUTF16.java
@HotSpotIntrinsicCandidate
public static int indexOf(byte[] value, byte[] str) {
    if (str.length == 0) {
        return 0;
    }
    if (value.length < str.length) {
        return -1;
    }
    return indexOfUnsafe(value, length(value), str, length(str), 0);
}

private static int indexOfUnsafe(byte[] value, int valueCount, byte[] str, int strCount, int fromIndex) {
    assert fromIndex >= 0;
    assert strCount > 0;
    assert strCount <= length(str);
    assert valueCount >= strCount;
    char first = getChar(str, 0);
    int max = (valueCount - strCount);
    for (int i = fromIndex; i <= max; i++) {
            // Look for first character.
        if (getChar(value, i) != first) {
            while (++i <= max && getChar(value, i) != first);
        }
        // Found first character, now look at the rest of value
        if (i <= max) {
            int j = i + 1;
            int end = j + strCount - 1;
            for (int k = 1; j < end && getChar(value, j) == getChar(str, k); j++, k++);
            if (j == end) {
                // Found whole string.
                return i;
            }
        }
    }
    return -1;
}
//...
```

Latin1과 UTF-16로 분기처리 되어지지만 `indexOf` 메소드 내부 로직은 똑같아 보인다. 아래의 순서대로 진행된다.

1. 원본 문자열(origin)의 각 문자 위치를 순회하면서,
2. 찾고자 하는 문자열(pattern)의 첫 번째 문자와 일치하는 위치를 발견하면
3. 그 위치부터 pattern 길이만큼의 문자들이 모두 일치하는지 확인한다.
4. 일치하면 인덱스를 반환, 아니면 다음 위치로 이동해서 다시 시도한다.

💡 즉, 요약하면 Java의 String 내부 `contains()`와 `indexOf()`는 완전탐색(brute-force)으로 이루어져 있다. 효율적으로 구성하려면 [KMP 알고리즘](https://longnh214.github.io/posts/%EB%AC%B8%EC%9E%90%EC%97%B4_%EA%B2%80%EC%83%89_%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98(KMP)/)이나 [Boyer-Moore 알고리즘](https://en.wikipedia.org/wiki/Boyer%E2%80%93Moore_string-search_algorithm), [Rabin-Karp 알고리즘](https://namu.wiki/w/%EB%AC%B8%EC%9E%90%EC%97%B4%20%EC%95%8C%EA%B3%A0%EB%A6%AC%EC%A6%98#:~:text=2.4.-,Rabin%2DKarp%20string%20search%20algorithm,-%5B%ED%8E%B8%EC%A7%91%5D)을 직접 구현해서 문자열을 검색하는 것이 좋을 듯 하다.

<br>
<br>

# `startsWith()`와 `endsWith()`

```java
//...
public boolean startsWith(String prefix) {
    return startsWith(prefix, 0);
}

public boolean endsWith(String suffix) {
    return startsWith(suffix, length() - suffix.length());
}

public boolean startsWith(String prefix, int toffset) {
    // Note: toffset might be near -1>>>1.
    if (toffset < 0 || toffset > length() - prefix.length()) {
        return false;
    }
    byte ta[] = value;
    byte pa[] = prefix.value;
    int po = 0;
    int pc = pa.length;
    if (coder() == prefix.coder()) {
        int to = isLatin1() ? toffset : toffset << 1;
        while (po < pc) {
            if (ta[to++] != pa[po++]) {
                return false;
            }
         }
    } else {
        if (isLatin1()) {  // && pcoder == UTF16
             return false;
        }
        // coder == UTF16 && pcoder == LATIN1)
        while (po < pc) {
            if (StringUTF16.getChar(ta, toffset++) != (pa[po++] & 0xff)) {
                return false;
            }
        }
    }
    return true;
}
//...
```

> `startsWith()`와 `endsWith()`는 내부적으로 `startsWith(prefix, offset)`을 공통으로 이용하고 있었다. `offset`을 `0`으로 하는지 `length() - suffix.length()`로 하는 지의 차이였다.  
> 내부에서는 인코딩 기법이 같은지, 인코딩 기법이 Latin1인지 UTF-16인지에 따라 인덱스 별로 각각 byte나 char 값을 비교해서 prefix와 같다면 true, 아니라면 false를 반환했다.

<br>
<br>

# `substring()`

```java
//...
public String substring(int beginIndex) {
    if (beginIndex < 0) {
        throw new StringIndexOutOfBoundsException(beginIndex);
    }
    int subLen = length() - beginIndex;
    if (subLen < 0) {
        throw new StringIndexOutOfBoundsException(subLen);
    }
    if (beginIndex == 0) {
        return this;
    }
    return isLatin1() ? StringLatin1.newString(value, beginIndex, subLen)
                        : StringUTF16.newString(value, beginIndex, subLen);
}

public String substring(int beginIndex, int endIndex) {
    int length = length();
    checkBoundsBeginEnd(beginIndex, endIndex, length);
    int subLen = endIndex - beginIndex;
    if (beginIndex == 0 && endIndex == length) {
        return this;
    }
    return isLatin1() ? StringLatin1.newString(value, beginIndex, subLen)
                          : StringUTF16.newString(value, beginIndex, subLen);
}
//...
```

> `substring()`은 인자가 하나인 메소드와 두 개인 메소드로 나뉘는데, 공통적으로 입력받은 인자에 따라서 `StringIndexOutOfBoundsException`이 발생하는 지 예외 처리를 한다.  
> `substring(beginIndex)`은 문자열 s의 길이가 n이라고 하면, `s[beginIndex...n-1]` 값을 반환하고, `substring(beginIndex, endIndex)`메소드는 `s[beginIndex...endIndex-1]`를 반환한다. 두 번째 인자의 -1번째 인덱스까지 반환하는 것을 알 수 있다.

<br>
<br>

# `replace(char oldChar, char newChar)`

```java
//String.java
public String replace(char oldChar, char newChar) {
    if (oldChar != newChar) {
        String ret = isLatin1() ? StringLatin1.replace(value, oldChar, newChar)
                                : StringUTF16.replace(value, oldChar, newChar);
        if (ret != null) {
            return ret;
        }
    }
    return this;
}
```

```java
//StringLatin1.java
public static String replace(byte[] value, char oldChar, char newChar) {
    if (canEncode(oldChar)) {
        int len = value.length;
        int i = -1;
        while (++i < len) {
            if (value[i] == (byte)oldChar) {
                break;
            }
        }
        if (i < len) {
            if (canEncode(newChar)) {
                byte buf[] = new byte[len];
                for (int j = 0; j < i; j++) {    // TBD arraycopy?
                    buf[j] = value[j];
                 }
                while (i < len) {
                    byte c = value[i];
                        buf[i] = (c == (byte)oldChar) ? (byte)newChar : c;
                    i++;
                }
                return new String(buf, LATIN1);
            } else {
                byte[] buf = StringUTF16.newBytesFor(len);
                // inflate from latin1 to UTF16
                inflate(value, 0, buf, 0, i);
                while (i < len) {
                        char c = (char)(value[i] & 0xff);
                        StringUTF16.putChar(buf, i, (c == oldChar) ? newChar : c);
                        i++;
                }
                return new String(buf, UTF16);
            }
        }
    }
    return null; // for string to return this;
}
```

```java
//StringUTF16.java
public static String replace(byte[] value, char oldChar, char newChar) {
    int len = value.length >> 1;
    int i = -1;
    while (++i < len) {
        if (getChar(value, i) == oldChar) {
                break;
        }
    }
    if (i < len) {
        byte buf[] = new byte[value.length];
        for (int j = 0; j < i; j++) {
            putChar(buf, j, getChar(value, j)); // TBD:arraycopy?
         }
        while (i < len) {
            char c = getChar(value, i);
                putChar(buf, i, c == oldChar ? newChar : c);
            i++;
        }
        // Check if we should try to compress to latin1
        if (String.COMPACT_STRINGS &&
            !StringLatin1.canEncode(oldChar) &&
               StringLatin1.canEncode(newChar)) {
            byte[] val = compress(buf, 0, len);
            if (val != null) {
                return new String(val, LATIN1);
            }
        }
        return new String(buf, UTF16);
    }
    return null;
}
```

### 💡 `replace()` 로직 흐름

1. 문자열의 처음부터 순차적으로 oldChar와 비교한다.
2. oldChar를 찾았다면 새 byte 배열을 만들어서 교체를 진행한다.
3. 교체를 전부 진행한 뒤에 배열 기준으로 새로운 문자열을 생성한다.
4. Latin1 -> UTF-16, UTF-16 -> Latin1이 가능하다면 인코딩을 바꾸고 최적화를 해준 뒤에 반환한다.

<br>
<br>

## 그렇다면 Latin1과 UTF-16이 뭐지?

> 우선 문자 인코딩은 문자를 컴퓨터가 이해할 수 있는 숫자(Byte or Bit)로 변환하는 방법을 의미한다. 예를 들어서 `B`라는 문자를 66라는 숫자로 바꾸고 1바이트로 저장하거나 더 복잡한 방식으로 2바이트로 저장할 수도 있다.

### Latin1 (ISO-8859-1)

Latin1은 유럽계 언어를 표현하기 위한 인코딩으로 256개의 문자를 1바이트로 표현한다. ASCII 코드 문자(0~127)는 그대로 맵핑되고, 그 외에는 스페인어, 독일어, 프랑스어 등 문자가 포함된다.

#### 특징

* 1바이트당 문자 하나를 표현한다.
* 표현 가능한 문자는 256개이다. (0x00 ~ 0xFF)
* 유럽의 언어 범위로 제한적이다.
* 메모리 사용이 적고, 빠르다.

### UTF-16

UTF-16은 유니코드를 표현하기 위한 인코딩 기법으로, 대부분의 문자는 2바이트로 표현하고 특정 문자는 4바이트로 표현하기도 한다.

#### 특징

* 전 세계의 모든 문자를 표현할 수 있다.
* 표현할 수 있는 문자의 범위가 넓다.
* Latin1에 비해서 메모리 점유량이 높아진다.

> 💡 Java는 상황에 따라 두 인코딩을 전환하면서 최적의 문자열 변환을 내부적으로 수행한다.

<br>
<br>

## 결론

Java의 String 클래스 내부에 대해서 공부하고 파악할 수 있었고, 문자열 처리에 대해서는 brute-force로 구현되어있는 부분이 많아서 여건이 된다면 직접 구현하는 것이 효율적이라고 생각된다.  
뿐만 아니라 내부적으로 인코딩 기법에 따라 변환 로직을 수행하고 있다는 점도 흥미로웠다.

## 출처

* java.lang.String (Java 11)
