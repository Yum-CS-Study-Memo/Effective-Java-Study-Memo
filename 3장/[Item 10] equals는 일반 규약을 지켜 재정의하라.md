# 이펙티브 자바 아이템 10 - equals는 일반 규약을 지켜 재정의하라

---

## 📌 핵심 개념: equals가 뭐야?

Java에서 **두 객체가 "같은지" 비교하는 메서드**야.

"같다"는 게 두 가지 의미가 있어:

```java
String a = new String("hello");
String b = new String("hello");

System.out.println(a == b);       // false ← 주소가 달라서 (다른 객체)
System.out.println(a.equals(b));  // true  ← 내용이 같아서
```

| 비교 방식 | 설명 |
|---|---|
| `==` | 주소 비교 (같은 객체야?) |
| `equals()` | 내용 비교 (값이 같아?) |

---

## 📌 왜 equals를 재정의해야 해?

Java의 모든 클래스는 `Object`를 상속받아. `Object`에 기본 `equals()`가 있는데, **기본 동작이 `==`랑 똑같아** - 즉, 주소 비교야.

```java
class Person {
    String name;
    int age;
}

Person p1 = new Person("유미", 30);
Person p2 = new Person("유미", 30);

System.out.println(p1.equals(p2)); // false 😱
// 내용이 같은데 false가 나옴!
// 기본 equals()가 주소 비교를 하기 때문
```

그래서 **equals()를 재정의(override)해서 내용 비교로 바꾸는 거야.**

---

## 📌 equals를 재정의하지 않아도 되는 경우

### ① 각 인스턴스가 본질적으로 고유하다
클래스가 **값이 아니라 동작/기능을 나타낼 때**야.

```java
// ❌ 값을 표현하는 클래스 → equals 재정의 필요
class Person {
    String name;  // "유미"라는 값
    int age;      // 30이라는 값
}

// ✅ 동작을 표현하는 클래스 → 재정의 불필요
class Thread {
    // "실행 중인 작업" 그 자체
    // "내용이 같은 스레드"라는 개념 자체가 없음!
}
```

### ② 논리적 동치성을 검사할 일이 없다
설계자가 *"어차피 이 클래스 객체끼리 equals 비교할 일 없겠다"* 고 판단하면 그냥 놔둬도 OK.

### ③ 상위 클래스의 equals가 하위 클래스에도 딱 들어맞는다
부모 클래스가 이미 equals를 잘 만들어놨으면 자식 클래스가 또 만들 필요 없어.

```java
// HashSet도, TreeSet도 "원소가 같으면 같은 Set"
// 이 논리는 AbstractSet이 이미 구현해놨음
Set<String> s1 = new HashSet<>(Arrays.asList("a", "b"));
Set<String> s2 = new TreeSet<>(Arrays.asList("a", "b"));
s1.equals(s2); // true → AbstractSet의 equals 덕분에!
```

### ④ 클래스가 private이거나 package-private이고 equals를 호출할 일이 없다
실수로라도 equals가 호출되는 걸 막고 싶다면:

```java
@Override
public boolean equals(Object o) {
    throw new AssertionError(); // 누가 실수로 호출하면 에러!
}
```

---

## 📌 A.equals(B) 구조 이해 - 가장 중요!

```java
A.equals(B)
// A = equals를 실행하는 주체 (이 클래스의 equals 코드가 실행됨)
// B = 괄호 안에 들어오는 것 = "들어온 것" = o (비교 대상)
```

**equals는 항상 A(주체)의 클래스에 정의된 코드가 실행돼!**

---

## 📌 equals 5가지 규약

### 1. 반사성 (Reflexivity)
```java
a.equals(a) // 무조건 true
```

### 2. 대칭성 (Symmetry)
```java
a.equals(b) == true 이면
b.equals(a) == true 여야 해!
```

### 3. 추이성 (Transitivity)
```java
a.equals(b) == true 이고
b.equals(c) == true 이면
a.equals(c) == true 여야 해!
```

### 4. 일관성 (Consistency)
```java
// 몇 번을 호출해도 결과가 같아야 해
a.equals(b) // 오늘 → true
a.equals(b) // 내일 → true
a.equals(b) // 100번 호출해도 → true
```
> ⚠️ equals 안에서 네트워크, 외부 자원 같은 거 쓰지 마! 항상 같은 결과가 나와야 하는데 네트워크는 상황에 따라 결과가 달라질 수 있거든.

### 5. null-아님 (Non-nullity)
```java
a.equals(null) // 무조건 false여야 함
```
> instanceof는 o가 null이면 자동으로 false를 반환해줘서 null 검사를 따로 안 해도 돼!

---

## 📌 코드 10-1: 대칭성 위배 예제

### CaseInsensitiveString이 뭐야?
"Polish"와 "polish"를 같다고 보고 싶어서 만든 **대소문자를 무시하는 문자열 클래스**야.

```java
public final class CaseInsensitiveString {
    private final String s;
    // CIS 객체 안에 실제 문자열을 저장하는 공간
    // CIS("Polish") 를 만들면 → 이 s에 "Polish"가 저장됨

    public CaseInsensitiveString(String s) {
        this.s = Objects.requireNonNull(s);
        // null이면 에러 발생
    }

    // ❌ 대칭성 위배!
    @Override public boolean equals(Object o) {

        // Case 1: 들어온 것(o)이 CIS 타입이면
        if (o instanceof CaseInsensitiveString)
            return s.equalsIgnoreCase(((CaseInsensitiveString) o).s);
            // s          = 주체(A) 안에 저장된 String 필드 ex) "Polish"
            // (CIS) o).s = 들어온것(B) 안에 저장된 String 필드 ex) "polish"
            // 결국 String.equalsIgnoreCase(String) → 대소문자 무시 비교
            // "Polish".equalsIgnoreCase("polish") → true ✅

        // ❌ Case 2: 들어온 것(o)이 그냥 String이면
        if (o instanceof String)
            return s.equalsIgnoreCase((String) o);
            // 여기가 문제!! CIS는 String을 받아주는데
            // String은 CIS를 받아주지 않음 → 비대칭 발생

        return false;
    }
}
```

### 왜 대칭성 위배야?

```java
CaseInsensitiveString cis = new CaseInsensitiveString("Polish");
String s = "polish";

cis.equals(s)
// A = cis (주체) → CIS의 equals 실행
// o = s   (들어온 것)
// "o가 String이야?" → YES
// "Polish".equalsIgnoreCase("polish") → true ✅

s.equals(cis)
// A = s   (주체) → String의 equals 실행 (Java 내장, 우리가 못 바꿈)
// o = cis (들어온 것)
// String의 equals 내부: "o가 String이야?" → NO (cis는 CIS 타입)
// → false ❌

// cis.equals(s) = true
// s.equals(cis) = false
// 누가 먼저 equals를 부르냐에 따라 결과가 달라짐 = 대칭성 위배!
```

### 수정된 코드

```java
// ✅ 올바른 버전
@Override public boolean equals(Object o) {
    return o instanceof CaseInsensitiveString &&
            ((CaseInsensitiveString) o).s.equalsIgnoreCase(s);
    // CIS끼리만 비교! String이랑은 아예 비교 안 함
    // 비대칭 생길 여지 자체가 없음
}

// cis.equals(s) → "o가 CIS야?" → NO (String이니까) → false
// s.equals(cis) → String의 equals → false
// 둘 다 false → 대칭성 OK ✅
```

---

## 📌 instanceof vs getClass

### instanceof
```java
ColorPoint cp = new ColorPoint(1, 2, Color.RED);
cp instanceof Point       // true  ← 상속받았으니까 OK
cp instanceof ColorPoint  // true
```
- "이 타입이거나 자식이면 true"
- 상속관계를 인정함
- 비유: "정원이는 사람이야?" → YES (사람을 상속받은 존재니까)

### getClass
```java
cp.getClass() == Point.class        // false ← ColorPoint지 Point가 아님!
cp.getClass() == ColorPoint.class   // true
```
- "정확히 이 클래스여야만 true"
- 상속관계를 인정 안 함
- 비유: 주민등록증으로 정확히 대조

| | instanceof | getClass |
|---|---|---|
| 자식 클래스 | ✅ 인정 | ❌ 불인정 |
| 비교 방식 | 가족관계 | 정확한 클래스 |
| Point vs ColorPoint | true | false |

---

## 📌 코드 10-2, 10-3: 상속에서의 대칭성/추이성 위배

### 등장인물

```java
// Point: x, y 좌표만 있는 클래스
public class Point {
    private final int x;
    private final int y;

    @Override public boolean equals(Object o) {
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return p.x == x && p.y == y;
        // p.x == x 는 p.x == this.x 와 같음
        // x, y는 상수가 아니라 "객체 안에 저장된 필드값"이야!
    }
}

// ColorPoint: Point를 상속받고 color만 추가
public class ColorPoint extends Point {
    private final Color color;
    // extends Point = Point의 x, y를 그대로 가져오고, color만 추가!
}
```

### 코드 10-2: 대칭성 위배 ❌

```java
// ColorPoint의 equals
@Override public boolean equals(Object o) {
    if (!(o instanceof ColorPoint))
        return false;
    // 들어온 것이 ColorPoint가 아니면 무조건 false

    return super.equals(o) && ((ColorPoint) o).color == color;
    // super.equals(o) = 부모(Point)의 equals로 x,y 비교
    // ((ColorPoint) o).color == color = 색상도 비교
}
```

```java
Point p        = new Point(1, 2);
ColorPoint cp  = new ColorPoint(1, 2, Color.RED);

p.equals(cp)
// A = p (Point) → Point의 equals 실행
// "cp가 Point야?" → YES! (ColorPoint는 Point를 상속받았으니까)
// x, y 비교 → 같음 → true ✅

cp.equals(p)
// A = cp (ColorPoint) → ColorPoint의 equals 실행
// "p가 ColorPoint야?" → NO
// → false ❌

// p.equals(cp) = true
// cp.equals(p) = false
// = 대칭성 위배!
```

> 💡 **왜 ColorPoint가 Point instanceof에 걸리지 않아?**
> ColorPoint는 Point를 상속받았어. 자식은 부모 타입으로 볼 수 있어.
> `cp instanceof Point → true` (Point의 자식이니까!)
> 이게 상속의 특징이야.

### 코드 10-3: 추이성 위배 ❌

**추이성이 뭐야?**
```
A == B 이고
B == C 이면
A == C 여야 해!
```

```java
// ColorPoint의 equals (고치려다 더 망한 버전)
@Override public boolean equals(Object o) {
    if (!(o instanceof Point))
        return false;

    if (!(o instanceof ColorPoint))
        return o.equals(this);
    // 들어온 것이 그냥 Point면 → 색상 무시하고 Point의 equals로 비교

    return super.equals(o) && ((ColorPoint) o).color == color;
    // 들어온 것이 ColorPoint면 → x,y,color 모두 비교
}
```

```java
ColorPoint p1 = new ColorPoint(1, 2, Color.RED);
Point      p2 = new Point(1, 2);
ColorPoint p3 = new ColorPoint(1, 2, Color.BLUE);

p1.equals(p2)
// A=p1(ColorPoint), o=p2(Point)
// "p2가 Point야?" YES
// "p2가 ColorPoint야?" NO → 색상 무시하고 비교
// x,y 같음 → true ✅

p2.equals(p3)
// A=p2(Point), o=p3(ColorPoint)
// Point의 equals 실행 → x,y만 비교
// 같음 → true ✅

p1.equals(p3)
// A=p1(ColorPoint), o=p3(ColorPoint)
// "p3가 ColorPoint야?" YES → 색상까지 비교
// RED vs BLUE → false ❌

// p1==p2, p2==p3 인데 p1≠p3
// 추이성 위배!!
```

---

## 📌 코드 10-5: 올바른 해결책 - 컴포지션

### 상속 vs 컴포지션

```
❌ 상속 (extends):
ColorPoint가 Point를 부모로 삼는 것
= "나는 Point의 자식이야"

✅ 컴포지션 (composition):
ColorPoint 안에 Point를 필드로 갖는 것
= "나는 Point를 안에 품고 있어"
```

```java
// ❌ 상속 버전 (문제 있는 것)
public class ColorPoint extends Point {
    private final Color color;
}

// ✅ 컴포지션 버전 (올바른 것)
public class ColorPoint {
    private final Point point;  // Point를 상속 대신 필드로 가짐!
    private final Color color;
}
```

### 코드 10-5 전체 해설

```java
public class ColorPoint {

    private final Point point;
    // Point 객체를 상속 대신 필드로 가짐
    // ColorPoint(1, 2, RED) 만들면
    // → 안에 Point(1,2) 객체가 들어있는 구조

    private final Color color;

    public ColorPoint(int x, int y, Color color) {
        point = new Point(x, y);
        // Point 객체를 직접 만들어서 필드에 저장
        this.color = Objects.requireNonNull(color);
    }

    public Point asPoint() {
        return point;
        // "나를 그냥 Point로 봐줄 때 써"
        // ColorPoint → Point로 변환해주는 메서드
    }

    @Override public boolean equals(Object o) {
        if (!(o instanceof ColorPoint))
            return false;
        // ColorPoint끼리만 비교!
        // Point가 들어오면 그냥 false
        // 상속관계가 없으니까 비대칭 생길 일 없음

        ColorPoint cp = (ColorPoint) o;
        // 들어온 것을 ColorPoint로 형변환

        return cp.point.equals(point) && cp.color.equals(color);
        // cp.point  = 들어온 cp의 point 필드
        // point     = 나(주체)의 point 필드
        // cp.color  = 들어온 cp의 color 필드
        // color     = 나(주체)의 color 필드
        // 풀어쓰면: cp.point.equals(this.point) && cp.color.equals(this.color)
    }
}
```

### 왜 문제가 해결돼?

```java
Point p        = new Point(1, 2);
ColorPoint cp  = new ColorPoint(1, 2, Color.RED);

p.equals(cp)
// Point의 equals 실행
// "cp가 Point야?" → NO! (이제 extends Point 없음, 상속관계 없음)
// → false

cp.equals(p)
// ColorPoint의 equals 실행
// "p가 ColorPoint야?" → NO
// → false

// 둘 다 false → 대칭성 OK ✅
```

---

## 📌 equals 올바르게 구현하는 단계별 방법

```java
@Override public boolean equals(Object o) {

    // 1단계: 나 자신이야?
    if (o == this) return true;
    // p.equals(p) 이런 경우
    // 당연히 같으니까 바로 true
    // 복잡한 비교 안 해도 되니까 성능상 이득

    // 2단계: 타입이 맞아? (null도 자동 처리)
    if (!(o instanceof 내클래스)) return false;
    // 타입이 아니면 바로 false
    // o가 null이면 instanceof가 자동으로 false 반환
    // → null 검사를 따로 안 해도 됨!

    // 3단계: 형변환
    내클래스 변수 = (내클래스) o;
    // 2단계에서 instanceof 통과했으니까 100% 안전하게 형변환 가능

    // 4단계: 핵심 필드 비교
    return 변수.필드1 == 필드1 && 변수.필드2 == 필드2;
    // 모든 핵심 필드가 같아야 true
}
```

---

## 📌 코드 10-6: PhoneNumber - 전형적인 올바른 예

```java
public final class PhoneNumber {
    private final short areaCode;  // 지역코드 ex) 02
    private final short prefix;    // 프리픽스 ex) 1234
    private final short lineNum;   // 가입자번호 ex) 5678

    public PhoneNumber(int areaCode, int prefix, int lineNum) {
        this.areaCode = rangeCheck(areaCode, 999, "지역코드");
        this.prefix   = rangeCheck(prefix, 999, "프리픽스");
        this.lineNum  = rangeCheck(lineNum, 9999, "가입자 번호");
        // rangeCheck = 범위 벗어나면 에러 발생시키는 메서드
    }

    private static short rangeCheck(int val, int max, String arg) {
        if (val < 0 || val > max)
            throw new IllegalArgumentException(arg + ": " + val);
        return (short) val;
    }

    @Override public boolean equals(Object o) {

        // 1단계: 나 자신이야?
        if (o == this) return true;

        // 2단계: 타입 확인 + null 자동 처리
        if (!(o instanceof PhoneNumber)) return false;

        // 3단계: 형변환
        PhoneNumber pn = (PhoneNumber) o;

        // 4단계: 핵심 필드 비교
        return pn.lineNum   == lineNum
            && pn.prefix    == prefix
            && pn.areaCode  == areaCode;
        // 세 필드 모두 같아야 같은 전화번호!
    }
}
```

---

## 📌 주의사항

### ⚠️ 입력 타입은 반드시 Object여야 해!

**왜 Object여야 해?**
모든 클래스는 Object를 자동으로 상속받아.
Object에 원래 이렇게 생긴 equals가 있어:

```java
// Object 클래스 (Java가 만들어놓은 것)
public boolean equals(Object o) { ... }
//                     ↑ Object 타입을 받음
```

재정의(Override) = 부모꺼랑 완전히 똑같은 형태로 덮어쓰는 것.
부모가 `equals(Object o)` 이니까 나도 `equals(Object o)` 여야 진짜 재정의야!

```java
// ❌ 잘못된 예
public boolean equals(PhoneNumber o) { ... }
// 부모: equals(Object o)
// 나:   equals(PhoneNumber o)
// 이름은 같은데 받는 타입이 달라!
// → Java는 이걸 재정의가 아니라 새로운 메서드 추가로 봄

// 그러면 두 개가 공존하게 됨:
// equals(Object o)      ← Object한테 물려받은 것 (주소 비교)
// equals(PhoneNumber o) ← 내가 새로 만든 것 (내용 비교)

PhoneNumber p1 = new PhoneNumber(02, 1234, 5678);
PhoneNumber p2 = new PhoneNumber(02, 1234, 5678);
Object      o  = p2;

p1.equals(p2)  // equals(PhoneNumber o) 호출 → true ✅
p1.equals(o)   // equals(Object o) 호출 → false ❌
// 같은 번호인데 결과가 달라짐!

// ✅ 올바른 예
@Override
public boolean equals(Object o) { ... }
// 부모꺼를 완전히 덮어씀
// 누가 어떤 타입으로 넣어도 항상 내 코드가 실행됨!
```

> @Override를 붙이면 컴파일러가 잘못된 재정의를 에러로 잡아줘서 실수를 방지할 수 있어!

### ⚠️ equals 재정의하면 hashCode도 반드시 재정의해!
equals를 바꿨는데 hashCode를 안 바꾸면
HashMap, HashSet 같은 컬렉션에서 이상하게 동작함.
→ 아이템 11에서 자세히 다룸.

### ⚠️ 너무 복잡하게 하지 마!
필드들의 동치성만 검사해도 equals 규약을 지킬 수 있어.

---

## 📌 아이템 10 전체 최종 요약

### equals 재정의할 때 지켜야 할 5가지 규약
```
1. 반사성:   a.equals(a) = true
2. 대칭성:   a.equals(b) = true면 b.equals(a)도 true
3. 추이성:   a==b, b==c면 a==c
4. 일관성:   몇 번을 호출해도 결과 같아야 함
5. null-아님: a.equals(null) = 무조건 false
```

### equals 올바르게 짜는 순서
```
1. o == this?           → true  반환 (성능 최적화)
2. instanceof 확인?     → false 반환 (null도 자동 처리)
3. 형변환
4. 핵심 필드 비교
```

### 상속 vs 컴포지션
```
❌ 구체 클래스를 상속해서 새로운 값을 추가하면서
   equals 규약을 지키는 건 불가능해

✅ 상속 대신 컴포지션(필드로 갖기)을 써!
   ColorPoint 안에 Point를 필드로 두면 해결!
```

### 핵심 책 문구
> **꼭 필요한 경우가 아니면 equals를 재정의하지 말자.**
> 많은 경우에 Object의 equals가 여러분이 원하는 비교를 정확히 수행해준다.
> 재정의해야 할 때는 그 클래스의 핵심 필드 모두를 빠짐없이,
> 다섯 가지 규약을 확실히 지켜가며 비교해야 한다.