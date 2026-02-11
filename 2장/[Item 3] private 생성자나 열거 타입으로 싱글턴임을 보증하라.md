#  Item 3: private 생성자나 열거 타입으로 싱글턴임을 보증하라

## 📌 싱글턴(Singleton)이란?

**인스턴스를 오직 하나만 생성할 수 있는 클래스**

엘비스 프레슬리는 세상에 한 명뿐이다. 복제하면 안 된다. 이게 싱글턴의 핵심이다.

---

## 🔑 핵심 개념 정리

### 클라이언트가 뭐야?

**Elvis 클래스를 사용하는 다른 코드**를 말한다.

```java
// 이 코드가 "클라이언트"
Elvis king = Elvis.INSTANCE;
king.leaveTheBuilding();
```

### private 생성자가 뭐야?

생성자 앞에 `private`을 붙이면 **클래스 외부에서 호출할 수 없는 생성자**가 된다.

```java
private Elvis() { ... }  // Elvis 클래스 내부에서만 호출 가능
```

다른 클래스에서 `new Elvis()` 하면 **컴파일 에러**가 난다. 단, **같은 클래스 내부에서는 호출 가능**하다.

### public static final 필드가 뭐야?

- **public**: 어디서든 접근 가능
- **static**: 객체를 만들지 않아도 `Elvis.INSTANCE`로 바로 접근 가능 (클래스 레벨에 속함)
- **final**: 한 번 할당되면 다른 값으로 **재할당 불가**

### 왜 인스턴스가 하나뿐임이 보장돼?

1. **private 생성자** → 외부에서 `new Elvis()` 불가능
2. **클래스 내부에서 딱 한 번만** `new Elvis()` 호출 (static final 필드 초기화 시)
3. **final** → `INSTANCE`에 다른 객체를 다시 넣을 수 없음

```java
// 클라이언트가 시도할 수 있는 모든 방법이 막혀 있음
new Elvis();                  // ❌ 컴파일 에러 (private 생성자)
Elvis.INSTANCE = new Elvis(); // ❌ 컴파일 에러 (final 필드)
```

---

## 🏗️ 싱글턴을 만드는 3가지 방법

### 방법 1: public static final 필드 방식

```java
public class Elvis {
    public static final Elvis INSTANCE = new Elvis();  // 집 안에 있는 진짜 엘비스 (딱 한 명)
    private Elvis() { ... }                             // 문 잠금 (밖에서 새 엘비스 못 만듦)
    public void leaveTheBuilding() { ... }
}
```

```java
// 사용법: 필드에 직접 접근
Elvis elvis = Elvis.INSTANCE;
```

**장점**: 간결하고 싱글턴임이 API에 명백히 드러남

---

### 방법 2: 정적 팩터리 메서드 방식

**객체를 생성해서 반환하는 static 메서드.** `new`를 직접 쓰는 대신, 메서드를 통해 인스턴스를 얻는 방식이다. 공장(factory)처럼 **객체를 만들어서 넘겨주는 역할**을 한다.

```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();     // 집 안에 있는 진짜 엘비스 (이번엔 private!)
    private Elvis() { ... }                                 // 문 잠금
    public static Elvis getInstance() { return INSTANCE; }  // 인터폰 (엘비스를 만나는 유일한 방법)
    public void leaveTheBuilding() { ... }
}
```

```java
// 사용법: 메서드를 통해 접근
Elvis elvis = Elvis.getInstance();
```

방법 1과의 차이: `INSTANCE` 필드가 **private**으로 바뀌었다. 클라이언트는 반드시 `getInstance()` 메서드를 거쳐야 한다.

```
비유 정리:
Elvis 클래스 = 엘비스의 집
INSTANCE = 집 안에 있는 진짜 엘비스 (딱 한 명)
private Elvis() = 문 잠금 (밖에서 새 엘비스 못 만듦)
getInstance() = 인터폰 (밖에서 엘비스를 만나는 유일한 방법)

손님(클라이언트): "엘비스 만나고 싶어요"
인터폰(getInstance): "네, 여기 있습니다" → 항상 같은 엘비스를 보여줌
```

---

### 정적 팩터리 방식의 3가지 장점

#### 장점 1: API를 바꾸지 않고도 싱글턴이 아니게 변경할 수 있다

"API를 바꾸지 않는다" = **클라이언트 코드를 안 바꿔도 된다**

클라이언트 입장에서 보자. 클라이언트는 `Elvis.getInstance()`를 호출한다. 이 메서드가 내부적으로 뭘 하는지는 모른다. 알 필요도 없다. 그냥 "엘비스를 달라"고 요청하면 된다.

이 구조 덕분에 `getInstance()` 메서드의 **내부 구현만 바꾸면** 동작을 완전히 바꿀 수 있다. 클라이언트 코드는 한 글자도 안 바꿔도 된다.

**현재: 싱글턴 (엘비스 한 명)**

```java
public static Elvis getInstance() {
    return INSTANCE;  // 항상 같은 엘비스
}
```

```java
Elvis a = Elvis.getInstance();  // 엘비스 1호
Elvis b = Elvis.getInstance();  // 엘비스 1호 (같은 애)
System.out.println(a == b);     // true
```

**변경 후: 매번 새 엘비스**

```java
// getInstance() 내부만 바꿈
public static Elvis getInstance() {
    return new Elvis();  // 매번 새 엘비스를 만들어서 줌!
}
```

```java
// 클라이언트 코드는 똑같아! 안 바꿔도 됨!
Elvis a = Elvis.getInstance();  // 엘비스 1호
Elvis b = Elvis.getInstance();  // 엘비스 2호 (다른 애!)
System.out.println(a == b);     // false
```

**스레드별로 다른 인스턴스를 주는 경우:**

스레드 = 동시에 일하는 직원. ThreadLocal = 직원마다 개인 사물함.

```java
// getInstance() 내부만 또 바꿈
private static final ThreadLocal THREAD_ELVIS =
    ThreadLocal.withInitial(Elvis::new);
    // 각 직원(스레드)이 처음 열면 새 엘비스가 하나씩 들어있음

public static Elvis getInstance() {
    return THREAD_ELVIS.get();  // 자기 사물함에서 자기 엘비스를 꺼냄
}
```

```java
// 직원1(스레드1)이 호출 → 직원1의 엘비스
// 직원2(스레드2)가 호출 → 직원2의 엘비스 (다른 엘비스!)
// 직원1이 다시 호출   → 직원1의 엘비스 (처음과 같은 엘비스)
```

**핵심: 클라이언트는 항상 `Elvis.getInstance()`만 호출.** 인터폰(getInstance) 뒤에서 무슨 일이 일어나든, 손님은 인터폰만 누르면 된다.

만약 **필드 방식(방법 1)**이었다면? `Elvis.INSTANCE`는 변수니까 항상 같은 객체를 가리킨다. 바꿀 방법이 없어서 클라이언트 코드를 전부 찾아 고쳐야 한다.

정리하면 이런 흐름이다:

```
[필드 방식] 클라이언트 → Elvis.INSTANCE (변수) → 항상 같은 객체
  → 변경하려면? 클라이언트 코드 전부 수정 필요 😰

[팩터리 방식] 클라이언트 → getInstance() (메서드) → 내부 구현에 따라 달라짐
  → 변경하려면? getInstance() 내부만 수정 ✅
  → 클라이언트 코드 수정 불필요!
```

이것이 소프트웨어 설계에서 말하는 **캡슐화(encapsulation)**의 장점이다. 메서드라는 껍데기 뒤에 구현을 숨겨두면, 구현을 바꿔도 바깥은 영향을 안 받는다.

---

#### 장점 2: 제네릭 싱글턴 팩터리로 만들 수 있다

**제네릭이란?** "타입을 나중에 정할게"

먼저 제네릭이 왜 필요한지부터 보자. 제네릭이 없으면 타입별로 클래스를 따로 만들어야 한다.

```java
// 제네릭 없이: 타입별로 따로 만들어야 함
public class BoxForString { private String item; }
public class BoxForInteger { private Integer item; }
// String용, Integer용, Double용... 끝이 없다!

// 제네릭으로: 하나만 만들면 됨!
public class Box { private T item; }   // T = "나중에 정할 타입"

Box box1 = new Box<>();   // T가 String이 됨
Box box2 = new Box<>();  // T가 Integer가 됨
// 클래스는 하나인데 여러 타입에 쓸 수 있다!
```

제네릭의 핵심: `<T>`는 "이 자리에 어떤 타입이든 들어올 수 있다"는 표시다. 실제로 사용할 때 `<String>`, `<Integer>` 등 구체적인 타입을 지정한다.

**이걸 싱글턴에 적용하면:**

```java
public class Elvis {
    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { }

    // : "타입은 호출할 때 정할게"
    // Elvis: "그 타입에 맞는 Elvis를 돌려줄게"
    @SuppressWarnings("unchecked")  // 타입 변환 경고 끄기
    public static  Elvis getInstance() {
        return (Elvis) INSTANCE;
        // 같은 엘비스인데, 옷(타입)만 갈아입혀서 줌
    }
}
```

```java
Elvis a = Elvis.getInstance();   // 캐주얼 엘비스 (String 옷)
Elvis b = Elvis.getInstance();  // 정장 엘비스 (Integer 옷)
// 실제로는 같은 사람! 옷(타입)만 다를 뿐
System.out.println(a == b);  // true (같은 객체!)
```

왜 같은 객체인데 다른 타입으로 쓸 수 있을까? 자바의 제네릭은 **컴파일 시점에만 타입을 검사하고, 실행 시점에는 타입 정보가 지워진다** (이것을 타입 소거(type erasure)라고 한다). 그래서 실행 시점에서 `Elvis<String>`과 `Elvis<Integer>`는 사실 그냥 `Elvis`다. 같은 객체를 공유해도 문제가 없는 것이다.

**실제 자바 예시: `Collections.reverseOrder()`**

"역순 비교" 로직은 숫자든 문자열이든 똑같다. 두 값을 받아서 순서를 뒤집는 것뿐이니까. 그래서 객체 하나로 여러 타입에 쓸 수 있다.

```java
// Collections 내부 구조 (간략화)
private static final ReverseComparator INSTANCE = new ReverseComparator();

public static  Comparator reverseOrder() {
    return (Comparator) INSTANCE;  // Elvis의 getInstance()와 구조 동일!
}
```

```java
Comparator intReverse = Collections.reverseOrder();  // 숫자 역순 비교기
Comparator strReverse = Collections.reverseOrder();   // 문자열 역순 비교기
System.out.println(intReverse == strReverse);  // true (같은 객체!)
// 역순 비교 로직은 타입과 무관하게 동일하므로, 하나의 객체를 공유할 수 있다.
```

**왜 필드 방식에서는 이게 안 될까?**

필드 방식은 그냥 변수다. 변수에 접근하면 저장된 값이 그대로 나온다. 중간에 타입 변환 같은 처리를 끼워넣을 수가 없다.

```java
// 필드 방식: 변수를 직접 꺼냄
Elvis a = Elvis.INSTANCE;   // 이때 INSTANCE의 타입은 이미 고정됨
Elvis b = Elvis.INSTANCE;  // 타입 불일치 에러 가능

// 팩터리 방식: 메서드를 거침
Elvis a = Elvis.getInstance();   // 메서드가 (Elvis)으로 캐스팅해서 줌
Elvis b = Elvis.getInstance();  // 메서드가 (Elvis)로 캐스팅해서 줌
```

**메서드**가 있어야 `<T>`를 선언하고 호출 시점에 타입을 유연하게 바꿔줄 수 있다. 필드에는 `<T>`를 붙일 수 없다.

---

#### 장점 3: 메서드 참조를 공급자(Supplier)로 사용할 수 있다

**Supplier란?** "내가 부르면 뭔가를 갖다주는 배달부"

```java
// Supplier 인터페이스 (자바에 이미 있음)
public interface Supplier {
    T get();  // "줘!" 하면 T 타입의 뭔가를 줌
}
```

Supplier의 `get()` 메서드를 보자. **파라미터 없이, 뭔가를 반환**한다. `getInstance()`도 마찬가지다. 파라미터 없이 Elvis를 반환한다. 메서드 모양(시그니처)이 똑같다!

자바에서는 메서드 모양이 같으면, 그 메서드를 인터페이스 구현체처럼 쓸 수 있다. 이것을 **메서드 참조(method reference)**라고 한다.

```java
// Elvis::getInstance = "getInstance 메서드를 가리켜!"
// getInstance()는 파라미터 없이 Elvis를 반환 → Supplier의 get()과 모양이 같음
Supplier elvisSupplier = Elvis::getInstance;

// 배달부한테 "줘!" 하면 내부적으로 getInstance()가 실행됨
Elvis elvis = elvisSupplier.get();  // == Elvis.getInstance()
```

**이게 왜 유용해?**

실제 프로그래밍에서는 "지금 당장 객체가 필요한 게 아니라, 나중에 필요할 때 만들어줄 방법"을 넘기는 경우가 많다. Supplier가 바로 그 역할을 한다.

```java
// "배달부(Supplier)"를 받도록 설계된 메서드
public static void printStar(Supplier supplier) {
    // 여기서 필요할 때 꺼냄. 어떤 방식으로 Elvis를 만드는지는 모르고 신경 안 씀.
    Elvis elvis = supplier.get();  // 배달부한테 엘비스를 받아서
    elvis.leaveTheBuilding();       // 뭔가를 시킴
}

// getInstance 메서드를 배달부로 넘김
printStar(Elvis::getInstance);
```

`printStar` 메서드는 Elvis가 어떻게 만들어지는지 모른다. 그냥 Supplier에게 "줘"라고 하면 된다. 이것이 **의존성 주입(dependency injection)**의 기본 원리다. "내가 직접 만들지 않고, 누군가 갖다주는 걸 쓴다."

**필드 방식이었다면?**

```java
Supplier supplier = Elvis::INSTANCE;       // ❌ 컴파일 에러! (필드는 메서드가 아님)
// :: 는 메서드를 가리키는 문법이다. 필드는 메서드가 아니라서 ::로 가리킬 수 없다.

Supplier supplier = () -> Elvis.INSTANCE;  // ⭕ 되긴 하지만 깔끔하지 않음
// 람다식으로 감싸야 해서 한 단계 더 거치는 셈이다.
```

---

### 3가지 장점 비유 총정리

```
인터폰(메서드)이 있어서 가능한 것들:

장점 1: 인터폰 뒤에서 일어나는 일을 자유롭게 바꿀 수 있음 (캡슐화)
        → 한 명을 보여주다가, 스레드별로 다른 사람을 보여줄 수도 있음
        → 손님은 인터폰만 누르면 되니까 모름

장점 2: 인터폰이 타입에 맞게 옷을 갈아입혀서 보여줄 수 있음 (제네릭)
        → 같은 엘비스를 String용, Integer용으로 줄 수 있음
        → 필드는 옷을 갈아입힐 방법이 없음

장점 3: 인터폰 자체를 다른 곳에 연결할 수 있음 (Supplier)
        → "엘비스가 필요하면 이 인터폰 눌러" 하고 넘겨줄 수 있음
        → 필드는 메서드가 아니라서 ::로 가리킬 수 없음

만약 인터폰 없이 문을 열어놓은 거였다면 (필드 방식):
→ 위 세 가지 다 못함!
```

---

## ⚠️ 직렬화 문제: 가짜 엘비스 탄생

방법 1, 2 모두 해당되는 문제.

### 직렬화/역직렬화란?

```java
// 직렬화 = 객체를 파일로 저장 (엘비스를 냉동보관 🧊)
ObjectOutputStream out = new ObjectOutputStream(new FileOutputStream("elvis.dat"));
out.writeObject(elvis);

// 역직렬화 = 파일에서 객체를 꺼냄 (냉동된 엘비스를 해동 🔥)
ObjectInputStream in = new ObjectInputStream(new FileInputStream("elvis.dat"));
Elvis loadedElvis = (Elvis) in.readObject();
```

직렬화는 객체의 상태(필드 값들)를 바이트 스트림으로 변환해서 파일에 저장하는 것이다. 역직렬화는 그 반대로, 파일에서 바이트를 읽어서 다시 객체로 만드는 것이다.

문제는 역직렬화할 때 자바가 **생성자를 호출하지 않고 새 객체를 만든다**는 점이다. private 생성자로 막아놨지만, 역직렬화는 이 제한을 우회한다.

### 문제: 해동하면 가짜 엘비스가 나온다!

```java
Elvis original = Elvis.getInstance();      // 진짜 엘비스
Elvis loaded = (Elvis) in.readObject();    // 해동된 엘비스

System.out.println(original == loaded);    // false!! 😱
// 역직렬화는 private 생성자를 우회해서 몰래 새 객체를 만들어버림!
```

```
진짜 엘비스 (INSTANCE) 👤
가짜 엘비스 (역직렬화로 생긴 놈) 👤  ← 싱글턴이 깨졌다!
```

싱글턴인데 인스턴스가 2개가 되어버렸다. 싱글턴의 존재 이유가 사라진 것이다.

### 해결: transient + readResolve

```java
public class Elvis implements Serializable {
    // transient = "냉동할 때 이 값은 저장하지 마!"
    // 가짜 엘비스에 진짜의 개인정보(필드)가 들어가면 위험하니까 🔒
    private transient String song = "Love Me Tender";

    private static final Elvis INSTANCE = new Elvis();
    private Elvis() { }
    public static Elvis getInstance() { return INSTANCE; }

    // "해동할 때 새 객체 만들지 말고, 진짜 엘비스를 돌려줘!"
    private Object readResolve() {
        return INSTANCE;  // 가짜 말고 진짜를 반환!
    }
}
```

`readResolve()`가 하는 일: 자바가 역직렬화로 새 객체를 만든 직후, 이 메서드가 있는지 확인한다. 있으면 새로 만든 객체 대신 **이 메서드가 반환하는 객체를 사용한다.** 여기서는 `INSTANCE`를 반환하니까, 결국 항상 진짜 엘비스가 반환된다.

`transient`가 하는 일: 직렬화할 때 이 필드는 저장하지 않는다. 역직렬화 과정에서 가짜 객체가 잠깐이라도 만들어지는데, 이때 진짜 엘비스의 필드 값이 가짜에게 복사되는 것을 방지한다. 모든 non-static, non-transient 필드에 transient를 붙여야 안전하다.

```
냉동 해동 과정:
1. 자바가 해동하면서 가짜 엘비스를 만들려고 함
2. readResolve가 가로챔: "잠깐! 이 가짜 버려!"
3. 대신 진짜 INSTANCE를 돌려줌
4. 가짜는 쓰레기 수거됨 🗑️
```

---

## 방법 3: 열거 타입 (enum) 싱글턴 ⭐ 가장 추천!

```java
// 이게 끝이다. 진짜로.
public enum Elvis {
    INSTANCE;

    public void leaveTheBuilding() {
        System.out.println("Whoa baby, I'm outta here!");
    }
}
```

```java
// 사용법
Elvis elvis = Elvis.INSTANCE;
elvis.leaveTheBuilding();
```

### 왜 가장 좋아?

방법 1, 2에서 고민했던 문제들을 자바가 알아서 처리해준다.

**리플렉션 공격**: 리플렉션은 자바의 기능으로, private 생성자에도 접근할 수 있게 해준다. 방법 1, 2에서는 이를 막기 위해 "두 번째 객체 생성 시 예외를 던지는" 방어 코드를 직접 작성해야 한다. enum은 자바 자체가 리플렉션으로 enum을 생성하는 것을 차단한다.

**직렬화 가짜 객체**: 방법 1, 2에서는 readResolve를 직접 작성하고 모든 필드에 transient를 붙여야 했다. enum은 자바의 직렬화 메커니즘이 enum을 특별 취급해서, 역직렬화 시 항상 같은 인스턴스를 반환한다.

| 문제 | 방법 1, 2 | 방법 3 (enum) |
|---|---|---|
| 리플렉션 공격 | 방어 코드 직접 작성 필요 | 자바가 알아서 차단 |
| 직렬화 가짜 객체 | readResolve + transient 직접 작성 필요 | 자바가 알아서 처리 |
| 코드 양 | 길다 | 짧고 간결 |

### 단, enum 방식을 쓸 수 없는 경우

만들려는 싱글턴이 **enum 외의 클래스를 상속해야 한다면** enum 방식을 쓸 수 없다. 자바에서 enum은 이미 `java.lang.Enum`을 상속하고 있고, 자바는 다중 상속을 허용하지 않기 때문이다.

```java
// 이런 경우 enum을 못 씀
public class SpecialElvis extends SomeBaseClass { ... }
// → SomeBaseClass를 상속해야 하니까 enum으로 만들 수 없음
// → 방법 1 또는 2를 사용해야 함
```

단, enum도 **인터페이스는 구현(implements)할 수 있다.** 상속과 구현은 다르다.

```java
// 이건 됨! enum도 인터페이스 구현 가능
public enum Elvis implements Singer {
    INSTANCE;

    @Override
    public void sing() { System.out.println("Love Me Tender~"); }
}
```

> **결론: 대부분의 상황에서는 원소가 하나뿐인 열거 타입(enum)이 싱글턴을 만드는 가장 좋은 방법이다.** 단, 만들려는 싱글턴이 enum 외의 클래스를 상속해야 한다면 방법 1 또는 2를 사용하자.