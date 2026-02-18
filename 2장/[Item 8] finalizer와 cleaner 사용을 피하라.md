# Effective Java Item 8 - finalizer와 cleaner 사용을 피하라

---

## 목차

1. [왜 이게 필요한 건가?](#1-왜-이게-필요한-건가)
2. [Java의 메모리 관리: Garbage Collector (GC)](#2-java의-메모리-관리-garbage-collector-gc)
3. [Finalizer란?](#3-finalizer란)
4. [Cleaner란?](#4-cleaner란)
5. [Finalizer vs Cleaner 차이](#5-finalizer-vs-cleaner-차이)
6. [둘 다 왜 쓰지 말라는 건가?](#6-둘-다-왜-쓰지-말라는-건가)

7. [Finalizer 공격 (Finalizer Attack)](#7-finalizer-공격-finalizer-attack)
8. [Static 변수와 GC Root](#8-static-변수와-gc-root)
9. [Finalizer 공격 방어](#9-finalizer-공격-방어)
10. [대안: AutoCloseable + try-with-resources](#10-대안-autocloseable--try-with-resources)
11. [닫힘 여부를 추적하라](#11-닫힘-여부를-추적하라)

12. [코드 8-1: Cleaner를 안전망으로 활용하는 Room 클래스](#12-코드-8-1-cleaner를-안전망으로-활용하는-room-클래스)
13. [Room 클래스 완전 해부](#13-room-클래스-완전-해부)
14. [Cleaner의 내부 동작 원리: PhantomReference + ReferenceQueue](#14-cleaner의-내부-동작-원리-phantomreference--referencequeue)
15. [코드 8-2, 8-3: Adult와 Teenager](#15-코드-8-2-8-3-adult와-teenager)
16. [두 경로 최종 정리](#16-두-경로-최종-정리)
17. [Cleaner/Finalizer를 쓸 수 있는 경우](#17-cleanerfinalizer를-쓸-수-있는-경우)
18. [역사적 흐름 요약](#18-역사적-흐름-요약)
19. [최종 핵심 정리](#19-최종-핵심-정리)

---

## 1. 왜 이게 필요한 건가?

프로그래밍할 때 **자원(resource)**을 사용하게 된다.

- 파일을 열어서 읽거나 쓰거나
- 데이터베이스에 연결하거나
- 네트워크 소켓을 열거나

이런 자원들은 **다 쓰고 나면 반드시 반납(해제)**해야 한다. 안 그러면 메모리가 새거나, 파일이 잠기거나, 연결이 고갈되는 문제가 생긴다.

---

## 2. Java의 메모리 관리: Garbage Collector (GC)

Java는 C/C++과 달리 **Garbage Collector(GC)**가 알아서 안 쓰는 객체를 메모리에서 치워준다.

```java
void someMethod() {
    String name = new String("Hello");
    // 메서드가 끝나면 name은 아무도 참조 안 함 → GC 대상
}
```

핵심 문제: **GC는 "언제" 동작할지 아무도 모른다.** GC가 객체를 수거하는 타이밍은 JVM이 알아서 정하기 때문에, 개발자가 "지금 당장 이 객체 치워줘!"라고 할 수 없다.

### GC가 할 수 있는 것

- **Java 힙 메모리** 회수 (Java가 `new`로 만든 객체들)

### GC가 할 수 없는 것

- 파일 닫기
- 데이터베이스 연결 끊기
- 네트워크 소켓 닫기
- **네이티브 메모리** 해제 (C/C++로 할당된 메모리)

```java
public class FileReader {
    private int fileDescriptor; // OS가 관리하는 자원 (GC가 모름!)

    public FileReader(String path) {
        this.fileDescriptor = openFile(path); // OS에게 파일 열어달라고 요청
    }

    // GC는 이 객체의 Java 메모리는 회수할 수 있지만,
    // fileDescriptor(OS 자원)는 존재 자체를 모름!
    // → GC만으로는 파일이 영원히 열린 채로 남음
}
```

**GC는 Java 메모리 관리자이지, 외부 자원 관리자가 아니다.**

그래서 Java 설계자들이 GC가 객체를 수거할 때, 개발자가 정리 코드를 끼워넣을 수 있는 **hook(갈고리)**을 제공하기로 했다. 그것이 바로 `finalize()`이다. 의도 자체는 합리적이었지만, 구현이 문제투성이였다.

---

## 3. Finalizer란?

### 개념

`finalizer`는 Java가 **아주 오래전부터(Java 1.0)** 제공한 메커니즘이다. 객체가 GC에 의해 수거되기 **직전에** 자동으로 호출되는 메서드이다.

모든 Java 객체의 부모인 `Object` 클래스에 `finalize()` 메서드가 정의되어 있으며, 이를 **오버라이드(재정의)**하면 GC가 이 객체를 수거하기 전에 자동으로 이 메서드를 호출해준다.

```java
public class MyResource {

    @Override
    protected void finalize() throws Throwable {
        try {
            System.out.println("자원 정리 중...");
            // 파일 닫기, 연결 끊기 등
        } finally {
            super.finalize();
        }
    }
}
```

### 의도된 용도

"개발자가 자원 해제를 깜빡했을 때, 마지막 안전망으로 자원을 정리해주자"는 취지였다.

### 비유

> 카페에서 손님(객체)이 떠날 때(GC 수거) 테이블을 직접 치우면 좋지만(명시적 자원 해제), 깜빡하고 그냥 나가더라도 청소 아르바이트생(finalizer)이 **언젠가** 와서 치워주겠다는 개념이다.
>
> 문제는 이 아르바이트생이 **언제 올지, 올지 안 올지조차 보장이 안 된다**는 것이다.

### ⚠️ 기본 동작이 아님에 주의

**GC에 finalizer가 기본으로 붙는 게 아니다!**

```
일반 객체 (99%):     finalize() 오버라이드 안 함 → GC가 그냥 바로 회수
finalizer 객체 (1%): 개발자가 직접 finalize()를 오버라이드한 경우에만 → GC가 특별 처리
```

개발자가 명시적으로 `finalize()`를 오버라이드해야만 finalizer가 동작한다. 아무것도 안 하면 GC는 깔끔하게 메모리만 회수한다. 대부분의 경우 GC만으로 충분하다. 문제가 되는 건 GC가 모르는 외부 자원을 쓸 때뿐이다.

---

## 4. Cleaner란?

### 배경

Java 9부터 `finalize()`가 **deprecated(사용 자제 권고)**되었다. 그 대안으로 나온 것이 `java.lang.ref.Cleaner`이다.

### 개념

Cleaner는 finalizer와 **목적은 같다** — 객체가 GC 대상이 되었을 때 정리 작업을 수행하는 것. 하지만 구현 방식이 더 체계적이다.

```java
import java.lang.ref.Cleaner;

public class MyResource implements AutoCloseable {

    // 1. Cleaner 인스턴스 생성 (보통 static으로)
    private static final Cleaner cleaner = Cleaner.create();

    // 2. 정리 작업을 담당하는 내부 클래스 (Runnable 구현)
    //    ⚠️ 반드시 static이어야 함! (바깥 객체를 참조하면 안 되니까)
    private static class CleaningAction implements Runnable {
        private int[] data; // 정리해야 할 네이티브 자원 등

        CleaningAction(int[] data) {
            this.data = data;
        }

        @Override
        public void run() {
            System.out.println("Cleaner가 자원 정리 중...");
            data = null;
        }
    }

    // 3. Cleaner에 이 객체와 정리 작업을 등록
    private final Cleaner.Cleanable cleanable;
    private final CleaningAction cleaningAction;

    public MyResource() {
        int[] data = new int[100];
        this.cleaningAction = new CleaningAction(data);
        // 이 객체(this)가 GC 대상이 되면 cleaningAction.run()을 호출해달라고 등록
        this.cleanable = cleaner.register(this, cleaningAction);
    }

    // 4. 명시적으로 정리할 수 있는 close() 메서드
    @Override
    public void close() {
        cleanable.clean(); // 직접 정리 호출
    }
}
```

---

## 5. Finalizer vs Cleaner 차이

### 핵심 차이 한 줄 요약

> **Finalizer**는 "죽어가는 객체 자신이 자기 뒷정리를 하는 것"이고, **Cleaner**는 "제3자가 객체와 독립적으로 뒷정리를 하는 것"이다.

### Finalizer

`Object.finalize()`를 오버라이드하는 방식. **객체 자신 안에** 정리 로직이 들어가 있다.

```java
public class MyResource {
    @Override
    protected void finalize() {
        // 정리 로직이 "이 객체 내부"에 있음
        // → 이 객체(this)에 자유롭게 접근 가능
        // → 이게 문제의 원인!
    }
}
```

`finalize()` 안에서 `this`에 접근할 수 있기 때문에, 정리해야 할 객체를 다시 살릴 수 있다:

```java
public class Zombie {
    static Zombie survivor; // static 변수

    @Override
    protected void finalize() {
        survivor = this; // GC 대상이었던 나 자신을 다시 참조시킴!
        // → 이 객체는 죽지 않고 "부활"함 (객체 부활 문제)
    }
}
```

### Cleaner

정리 로직이 **객체 바깥의 별도 Runnable**에 분리되어 있다.

```java
// 정리 작업이 별도의 static 클래스에 있음
private static class CleaningAction implements Runnable {
    // ⚠️ 바깥 객체(MyResource)에 대한 참조가 없음!
    // → 객체 부활 불가능
    // → this 접근 불가능

    private int[] nativeResource;

    @Override
    public void run() {
        nativeResource = null; // 자원만 정리
    }
}
```

### 상세 비교표

| 관점 | Finalizer | Cleaner |
|------|-----------|---------|
| 도입 시기 | Java 1.0 | Java 9 |
| 사용법 | `finalize()` 오버라이드 | `Cleaner` 객체에 등록 |
| 정리 로직 위치 | 객체 내부 (`this` 접근 가능) | 객체 외부 (별도 Runnable) |
| 객체 부활 가능성 | 있음 (`this`를 다시 참조시킬 수 있음) | 없음 (대상 객체 참조 자체가 없음) |
| 예외 처리 | 예외가 **조용히 무시**됨 | 예외가 정상적으로 처리됨 |
| 실행 스레드 | GC 내부의 Finalizer 스레드 (제어 불가) | 별도 Cleaner 데몬 스레드 (좀 더 예측 가능) |
| 명시적 호출 | `finalize()` 직접 호출 가능하지만 권장 안 됨 | `cleanable.clean()`으로 안전하게 명시적 호출 가능 |
| 현재 상태 | **deprecated** (Java 9~) | finalizer의 대안 |
| **공통 한계** | ⏰ 실행 시점 미보장 | ⏰ 실행 시점 미보장 |

Cleaner는 Finalizer의 **설계적 결함(객체 자기참조, 예외 무시, 보안 취약점)**을 고친 버전이지만, GC 타이밍에 의존한다는 **근본적 한계**는 동일하다. 그래서 둘 다 "피하라"는 것이다.

---

## 6. 둘 다 왜 쓰지 말라는 건가?

이것이 Item 8의 핵심이다.

### 6-1. 실행 시점을 보장할 수 없다

```java
MyResource resource = new MyResource();
resource = null; // 이제 GC 대상이지만...
// finalizer/cleaner가 "언제" 실행될지 아무도 모른다!
// 1초 후? 1분 후? 아예 안 될 수도?
```

GC가 언제 동작할지 모르니까, 정리 작업도 언제 될지 모른다. 파일을 열어놓고 finalizer가 닫아주길 기다리면? 그 사이에 파일이 수백 개 열려서 시스템이 터질 수도 있다.

### 6-2. 실행 자체가 보장되지 않는다

GC의 객체 수거 과정 (finalizer가 있는 경우):

```
일반 객체:      GC 발견 → 즉시 메모리 회수 ✅ (끝)

finalizer 객체: GC 발견 → "아 이거 finalizer 있네"
                        → finalization queue에 넣음
                        → Finalizer 전용 스레드가 큐에서 꺼내서 실행
                        → 실행 완료 후 다음 GC 때 비로소 메모리 회수
```

- **Finalizer 스레드의 우선순위가 낮다.** 애플리케이션이 바쁘면 큐에 쌓인 finalizer들이 계속 대기 상태로 밀린다. 큐에 수천, 수만 개가 쌓이면 메모리가 계속 회수되지 않아서 `OutOfMemoryError`가 발생할 수 있다.
- **JVM 명세 자체가 실행을 보장하지 않는다.** Java 언어 스펙에 명시: *"The Java programming language does not specify which thread will invoke the finalizer for any given object, nor does it guarantee that finalizers will be called at all."* JVM 종료 시점에 큐에 남아있는 finalizer는 실행 안 되고 버려질 수 있다.

### 6-3. 예외가 발생하면 무시됨 (Finalizer만 해당)

```java
@Override
protected void finalize() throws Throwable {
    throw new RuntimeException("정리 중 에러 발생!");
    // 이 예외는 잡히지도, 로그에 남지도 않고
    // 그냥 조용히 무시됨. 정리도 안 된 채로 끝남.
}
```

finalizer 안에서 예외가 발생하면 **경고도 없이 무시**된다. 디버깅이 거의 불가능해진다.

### 6-4. 성능이 심각하게 나빠진다

finalizer가 있는 객체를 만들고 GC하는 데 걸리는 시간이 일반 객체보다 **수십 배** 느리다. finalizer가 있으면 GC가 객체를 즉시 수거하지 못하고 / 별도의 큐에 넣어서 처리해야 하기 때문이다.

### 6-5. Finalizer 공격 (Finalizer Attack)

보안 취약점도 있다. 이 내용은 아래에서 상세히 다룬다.

### 위험 요소 종합

| 위험 요소 | 설명 |
|-----------|------|
| 실행 시점 불확실 | 큐에 쌓여서 한참 뒤에 실행될 수 있음 |
| 실행 자체 미보장 | JVM 종료 시 아예 안 실행될 수 있음 |
| 예외 무시 | 에러가 나도 조용히 삼켜버림 (Finalizer) |
| 메모리 누수 | 객체가 2번의 GC 사이클을 살아남아야 해서 메모리 점유가 길어짐 |
| 성능 저하 | 일반 객체 대비 수십 배 느림 |
| 보안 취약점 | Finalizer Attack 가능 |
| GC 스레드 제어 불가 | 어떤 스레드가 실행할지, 어떤 순서로 실행할지 개발자가 전혀 제어 못함 |

---

## 7. Finalizer 공격 (Finalizer Attack)

> 책 원문: "finalizer를 사용한 클래스는 finalizer 공격에 노출되어 심각한 보안 문제를 일으킬 수도 있다. 생성자나 직렬화 과정에서 예외가 발생하면, 이 생성되다 만 객체에서 악의적인 하위 클래스의 finalizer가 수행될 수 있게 된다. / 이 finalizer는 정적 필드에 자신의 참조를 할당하여 가비지 컬렉터가 수집하지 못하게 막을 수 있다."

### 배경: 생성자에서 검증하는 이유

보안이 중요한 클래스는 생성자에서 "너 자격 있어?"를 검증한다:

```java
public class BankAccount {
    private long balance;

    public BankAccount(String userId) {
        // 🔒 보안 검증: 권한 없으면 객체 생성을 막음
        if (!isAuthorized(userId)) {
            throw new SecurityException("권한 없음!");
        }
        this.balance = 0;
    }

    public void transferMoney(long amount) {
        System.out.println(amount + "원 송금!");
    }
}
```

권한 없는 사람이 `new BankAccount("해커")`를 하면 예외가 터지면서 **객체 생성이 실패**한다. 정상적이라면 이 객체는 사용할 수 없어야 한다.

### Step 1: 해커가 악의적인 하위 클래스를 만든다

```java
public class EvilAccount extends BankAccount {

    // 💀 핵심: 정적 필드 (클래스에 딱 하나 존재, GC 대상 아님)
    static BankAccount stolenAccount;

    public EvilAccount(String userId) {
        super(userId); // 부모 생성자 호출 → 여기서 예외 터짐!
    }

    @Override
    protected void finalize() {
        // 🔥 생성자에서 예외가 터졌는데도 이게 실행됨!
        stolenAccount = this; // 자기 자신을 정적 필드에 저장
    }
}
```

### Step 2: 해커가 객체 생성을 시도한다

```java
try {
    new EvilAccount("해커"); // 생성자에서 SecurityException 발생!
} catch (Exception e) {
    // 예외 잡고 넘어감
}
```

여기서 일어나는 일을 시간 순서로 보면:

```
1. new EvilAccount("해커") 호출
2. super("해커") → BankAccount 생성자 실행
3. isAuthorized("해커") → false → SecurityException 발생! 💥
4. 객체 생성 실패! ... 했지만 JVM 입장에서는?
   → "메모리는 이미 할당했고, 이 객체에 finalize()가 있네"
   → "나중에 GC할 때 finalize() 호출해줘야지"
5. (시간이 지남...)
6. GC 동작 → EvilAccount.finalize() 호출됨
7. finalize() 안에서: stolenAccount = this;
   → 💀 생성 실패한 객체가 정적 필드에 저장됨!
   → 정적 필드가 참조하고 있으니 GC가 수거 못함!
```

### Step 3: 해커가 훔친 객체를 사용한다

```java
// GC가 동작할 시간을 줌
System.gc();
Thread.sleep(1000);

// 💀 생성에 실패했어야 할 객체를 사용!
EvilAccount.stolenAccount.transferMoney(1000000);
// 출력: "1000000원 송금!"
```

**보안 검증을 통과하지 못한 객체로 송금을 해버렸다!**

### 왜 이런 일이 가능한가?

```
일반적인 기대:  생성자 실패 → 객체 사용 불가 → 끝

실제 현실:     생성자 실패 → 메모리는 이미 할당됨
                         → finalize()가 있으면 GC가 호출해줌
                         → finalize() 안에서 this에 접근 가능
                         → this를 정적 필드에 저장하면 객체 부활!
```

`finalize()` 메서드는 객체 내부에서 실행되니까 `this`에 접근할 수 있고, `this`를 GC가 건드릴 수 없는 곳(정적 필드)에 저장하면 **불완전한 객체가 영원히 살아남는** 것이다.

---

## 8. Static 변수와 GC Root

### GC가 객체를 수거하는 기준

GC는 **"아무도 이 객체를 참조하지 않으면"** 수거한다. 반대로 말하면, **누군가 참조하고 있으면 절대 수거하지 않는다.**

여기서 "누군가"가 누구냐가 중요한데, **GC Root**라는 개념이 등장한다:

```
GC Root (절대 수거되지 않는 것들)
├─ 현재 실행 중인 스레드의 지역 변수
├─ static 변수  ← 💀 핵심
├─ JNI 참조
└─ 시스템 클래스 등
```

**static 변수는 클래스가 메모리에 로드되어 있는 한 절대 사라지지 않는다.** 클래스는 보통 JVM이 종료될 때까지 살아있으니까, static 변수도 JVM 종료까지 살아있다.

### Finalizer 공격에서 static이 핵심인 이유

```java
public class EvilAccount extends BankAccount {
    // static = GC Root → 여기에 연결된 건 GC가 수거 못함
    static BankAccount stolenAccount;

    @Override
    protected void finalize() {
        stolenAccount = this;
        // this(이 객체)가 static 변수에 연결됨
        // → GC Root에서 이 객체까지 참조 체인이 생김
        // → GC: "아 누가 참조하고 있네, 수거하면 안 되겠다"
    }
}
```

그림으로 보면:

```
finalize() 실행 전:
  stolenAccount(static) → null
  EvilAccount 객체 → 아무도 참조 안 함 → GC 수거 대상 ✅

finalize() 실행 후:
  stolenAccount(static) → EvilAccount 객체
                          ↑ GC Root에서 참조 체인 존재!
                          → GC 수거 불가 ❌ (부활!)
```

만약 static이 아니라 일반 지역 변수에 저장했다면, 그 지역 변수 자체도 곧 사라지니까 의미가 없다. static이기 때문에 **영구적으로 참조가 유지**되는 것이다.

---

## 9. Finalizer 공격 방어

### 공격의 구조

```
BankAccount (상위 클래스) - 보안 검증 담당
    ↑ 상속
EvilAccount (하위 클래스) - 해커가 만든 악의적 클래스
    → finalize()를 오버라이드해서 공격
```

해커가 `EvilAccount`에서 `finalize()`를 오버라이드하는 것이 문제의 시작이므로, **상위 클래스인 BankAccount에서 오버라이드를 원천 차단**한다.

### 방어법 1: finalize()를 final로 선언

```java
public class BankAccount {
    private long balance;

    public BankAccount(String userId) {
        if (!isAuthorized(userId)) {
            throw new SecurityException("권한 없음!");
        }
        this.balance = 0;
    }

    // ✅ 방어: final이니까 하위 클래스가 오버라이드 불가!
    @Override
    protected final void finalize() {
        // 의도적으로 아무것도 안 함
    }
}
```

이제 해커가 시도하면:

```java
public class EvilAccount extends BankAccount {
    static BankAccount stolenAccount;

    @Override
    protected void finalize() { // ❌ 컴파일 에러!
        stolenAccount = this;   // "Cannot override final method"
    }
}
```

`final` 메서드는 하위 클래스에서 오버라이드할 수 없으므로, **컴파일 자체가 안 되어** 공격이 원천 봉쇄된다.

### 방어법 2: 클래스 자체를 final로

```java
// 클래스 자체를 final로 → 상속 자체가 불가능
public final class BankAccount {
    // EvilAccount extends BankAccount → 컴파일 에러!
}
```

상속 자체를 불가능하게 만들므로 더 강력한 방어이다.

### 방어법 비교

| 방어 방법 | 효과 |
|-----------|------|
| `protected final void finalize()` | finalize() 오버라이드만 차단 |
| `public final class BankAccount` | 상속 자체를 차단 (더 강력) |

두 방법 모두 **보안이 중요한 상위 클래스 쪽에서** 적용해야 한다. 해커가 만드는 하위 클래스를 우리가 통제할 수는 없으므로, 상위 클래스에서 미리 막아두는 것이다.

---

## 10. 대안: AutoCloseable + try-with-resources

> 책 원문: "그렇다면 파일이나 스레드 등 종료해야 할 자원을 담고 있는 객체의 클래스에서 finalizer나 cleaner를 대신해줄 묘안은? 그저 AutoCloseable을 구현해주고, 클라이언트에서 인스턴스를 다 쓰고 나면 close 메서드를 호출하면 된다."

### AutoCloseable이란?

**"나는 다 쓰고 나면 닫아줘야 하는 자원이야"**라고 선언하는 인터페이스이다.

```java
// Java가 제공하는 인터페이스 (우리가 만드는 게 아님)
public interface AutoCloseable {
    void close() throws Exception;
}
```

이게 전부다. 메서드가 `close()` **딱 하나**만 있다. 이걸 구현한다는 건 "이 클래스는 `close()`로 정리해야 하는 자원을 갖고 있다"고 약속하는 것이다.

### 왜 이걸 구현해야 하나?

AutoCloseable을 구현하면 **try-with-resources** 문법을 쓸 수 있기 때문이다:

```java
// AutoCloseable을 구현하지 않은 클래스
public class BadResource {
    public void close() { /* 정리 */ }
}

try (BadResource r = new BadResource()) { // ❌ 컴파일 에러!
    // AutoCloseable 안 구현했으니 try-with-resources 못 씀
}
```

```java
// AutoCloseable을 구현한 클래스
public class GoodResource implements AutoCloseable {
    @Override
    public void close() { /* 정리 */ }
}

try (GoodResource r = new GoodResource()) { // ✅ 정상 동작!
    r.doSomething();
} // ← 여기서 close()가 자동 호출됨!
```

**try-with-resources는 AutoCloseable을 구현한 객체만 받아준다.**

### try-with-resources의 컴파일 규칙

`BadResource r = new BadResource()` 자체는 문제없다. 문제는 **`try ()` 괄호 안에 넣었을 때**이다.

```java
// 이건 완벽하게 정상 코드
BadResource r = new BadResource();
r.close(); // 이것도 정상. 그냥 일반 메서드 호출이니까.
```

`try ()` 괄호는 아무 객체나 받지 않는다. Java 컴파일러가 규칙을 강제한다:

```java
try (BadResource r = new BadResource()) {
    // ...
}

// 컴파일러: "try 블록 끝나면 r.close()를 자동 호출해야 하는데..."
// 컴파일러: "BadResource가 AutoCloseable을 구현했나? 확인해보자"
// 컴파일러: "안 했네. 그러면 이 객체에 close()가 있다는 보장이 없잖아"
// 컴파일러: "컴파일 에러!" ❌
```

**"close()라는 메서드가 우연히 있는 것"과 "AutoCloseable을 구현해서 close()가 있는 것"은 Java 입장에서 완전히 다르다.**

### 비유

```
카페 출입 규칙: "회원증 있는 사람만 입장 가능"

BadResource:  돈도 있고, 옷도 잘 입었지만 회원증이 없음 → 입장 불가 ❌
GoodResource: 회원증 있음 (= implements AutoCloseable)  → 입장 가능 ✅
```

`implements AutoCloseable`이 바로 그 **회원증**이다. close() 메서드가 있든 없든, 회원증이 없으면 try-with-resources에 들어갈 수 없다. 이것이 바로 Java에서 **인터페이스가 "계약(contract)"**이라고 불리는 이유이다.

### 해결은 딱 한 단어 추가

```java
//                          ↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓↓
public class BadResource implements AutoCloseable {
    @Override
    public void close() { /* 정리 */ }
}

try (BadResource r = new BadResource()) { // ✅ 이제 됨!
}
```

### 실제 Java에서 이미 구현하고 있는 것들

자주 쓰는 클래스들이 이미 AutoCloseable을 구현하고 있다:

```java
// 파일 읽기 - AutoCloseable 구현됨
try (FileInputStream fis = new FileInputStream("test.txt")) {
    // 파일 읽기
} // 자동으로 파일 닫힘

// DB 연결 - AutoCloseable 구현됨
try (Connection conn = DriverManager.getConnection(url)) {
    // DB 작업
} // 자동으로 연결 끊김

// 네트워크 - AutoCloseable 구현됨
try (Socket socket = new Socket("localhost", 8080)) {
    // 네트워크 통신
} // 자동으로 소켓 닫힘
```

### try-with-resources의 동작 원리

```java
// 이 코드가
try (Room myRoom = new Room(7)) {
    System.out.println("안녕~");
}

// Java 컴파일러가 내부적으로 이렇게 바꿔주는 거다
Room myRoom = new Room(7);
try {
    System.out.println("안녕~");
} finally {
    myRoom.close(); // ← 컴파일러가 자동으로 넣어줌!
}
```

`finally`는 **무슨 일이 있어도 반드시 실행되는 블록**이다. 그래서 try 블록이 끝나면 `close()`가 확실하게 호출된다.

---

## 11. 닫힘 여부를 추적하라

> 책 원문: "각 인스턴스는 자신이 닫혔는지를 추적하는 것이 좋다. close 메서드에서 이 객체는 더 이상 유효하지 않음을 필드에 기록하고, 다른 메서드는 이 필드를 검사해서 객체가 닫힌 후에 불렀다면 IllegalStateException을 던지는 것이다."

### 문제 상황

```java
public class MyFileReader implements AutoCloseable {
    private String filePath;

    public MyFileReader(String path) {
        this.filePath = path;
        System.out.println(path + " 파일 열음");
    }

    public String read() {
        return "파일 내용입니다";
    }

    @Override
    public void close() {
        System.out.println("파일 닫음");
    }
}
```

이 클래스의 문제:

```java
MyFileReader reader = new MyFileReader("data.txt");
reader.close();      // 파일 닫음

// 💀 이미 닫혔는데 또 읽으려고 함!
String data = reader.read(); // 에러도 안 나고 그냥 실행됨...
                              // 실제로는 닫힌 파일에서 읽는 거라 위험!
```

닫힌 자원을 계속 사용하는 걸 막을 방법이 없다. 조용히 잘못된 동작을 하거나, 예측 불가능한 에러가 나중에 터질 수 있다.

### 해결: 닫힘 여부를 추적하는 필드 추가

```java
public class MyFileReader implements AutoCloseable {
    private String filePath;
    private boolean closed = false; // ← 닫혔는지 추적하는 필드!

    public MyFileReader(String path) {
        this.filePath = path;
        System.out.println(path + " 파일 열음");
    }

    public String read() {
        // ✅ 매 메서드 호출마다 닫혔는지 확인!
        if (closed) {
            throw new IllegalStateException("이미 닫힌 FileReader입니다!");
        }
        return "파일 내용입니다";
    }

    @Override
    public void close() {
        if (!closed) {
            System.out.println("파일 닫음");
            closed = true; // ← 닫혔다고 기록!
        }
    }
}
```

이제 잘못된 사용을 하면:

```java
MyFileReader reader = new MyFileReader("data.txt");
reader.close();

String data = reader.read(); // 💥 IllegalStateException: 이미 닫힌 FileReader입니다!
                              // → 개발자가 즉시 버그를 발견할 수 있음
```

### 왜 이게 중요한가?

```
추적 안 하는 경우:
  close() 호출 → read() 호출 → 조용히 잘못된 결과 반환 또는 알 수 없는 에러
                                → 버그 찾기 매우 어려움 😰

추적하는 경우:
  close() 호출 → closed = true
               → read() 호출 → closed 확인 → 즉시 IllegalStateException!
                                             → 버그 원인이 명확함 ✅
```

### 책의 문장을 코드로 번역

> "각 인스턴스는 자신이 닫혔는지를 추적하는 것이 좋다"

```java
private boolean closed = false; // 추적 필드
```

> "close 메서드에서 이 객체는 더 이상 유효하지 않음을 필드에 기록하고"

```java
@Override
public void close() {
    closed = true; // 유효하지 않음을 기록
}
```

> "다른 메서드는 이 필드를 검사해서 객체가 닫힌 후에 불렀다면 IllegalStateException을 던진다"

```java
public String read() {
    if (closed) throw new IllegalStateException(); // 검사 후 예외
    // ...
}
```

이 세 줄이 책 전체 문장의 정체이다. 결국 **방어적 프로그래밍**을 하라는 것이다.

---

## 12. 코드 8-1: Cleaner를 안전망으로 활용하는 Room 클래스

### 전체 코드 (줄별 상세 주석)

```java
package effectivejava.chapter2.item8;

import java.lang.ref.Cleaner;

// Cleaner를 안전망으로 활용하는 AutoCloseable 클래스
public class Room implements AutoCloseable {

    // ① Cleaner 인스턴스를 하나만 만들어 공유 (static final)
    //    - static: 모든 Room 객체가 같은 Cleaner를 공유
    //    - final: 한번 만들면 바꿀 수 없음
    //    - Cleaner.create(): Cleaner 전용 데몬 스레드가 하나 생성됨
    //      이 스레드가 나중에 등록된 정리 작업들을 실행해줌
    private static final Cleaner cleaner = Cleaner.create();

    // ② 실제 정리해야 할 자원을 담는 내부 클래스
    //    ⚠️ 반드시 static class여야 함!
    //    - static이 아니면(내부 클래스면) 바깥 Room 객체를 암묵적으로 참조함
    //    - Room을 참조하면 GC가 Room을 수거할 수 없음 → Cleaner가 영원히 호출 안 됨
    //    - Runnable 구현: run() 메서드가 실제 정리 작업
    private static class State implements Runnable {

        // 방 안의 쓰레기 더미 수 (정리해야 할 자원을 상징)
        // 실제 프로젝트에서는 네이티브 메모리 포인터, 파일 핸들 등이 올 수 있음
        int numJunkPiles;

        // 생성자: 쓰레기 더미 수를 받아서 저장
        State(int numJunkPiles) {
            this.numJunkPiles = numJunkPiles;
        }

        // ③ 실제 정리 작업을 수행하는 메서드
        //    이 run()은 두 가지 경우에 호출됨:
        //    1) close() → cleanable.clean() → run() (명시적 정리 - 정상 경로)
        //    2) GC가 Room 객체를 수거할 때 → Cleaner가 run() 호출 (안전망 경로)
        @Override
        public void run() {
            System.out.println("Cleaning room");
            numJunkPiles = 0; // 쓰레기 더미를 0으로 → 자원 정리 완료
        }
    }

    // ④ 방의 상태 객체
    //    - Room과 Cleanable이 이 State 객체를 공유함
    //    - Room은 이걸 통해 방의 상태(쓰레기 수)에 접근
    //    - Cleaner는 이걸 통해 정리 작업(run())을 실행
    private final State state;

    // ⑤ Cleaner에 등록한 결과로 받는 Cleanable 객체
    //    - cleanable.clean()을 호출하면 State.run()이 실행됨
    //    - clean()은 최초 1회만 실행되고, 이후 호출은 무시됨 (중복 정리 방지)
    private final Cleaner.Cleanable cleanable;

    // ⑥ Room 생성자
    public Room(int numJunkPiles) {
        // 정리할 자원(State)을 생성
        state = new State(numJunkPiles);

        // Cleaner에 등록:
        // "this(Room 객체)가 GC 대상이 되면, state(State 객체)의 run()을 호출해줘"
        // - 첫 번째 인자 (this): 감시 대상 객체 (이 Room)
        // - 두 번째 인자 (state): 정리 작업 (Runnable)
        // ⚠️ 여기서 state는 Room을 참조하지 않는 static 클래스의 인스턴스!
        //    만약 state가 Room을 참조하면 → Room이 GC 안 됨 → Cleaner 영원히 안 불림
        cleanable = cleaner.register(this, state);
    }

    // ⑦ AutoCloseable의 close() 구현
    //    - 사용자가 try-with-resources로 쓰면 이게 자동 호출됨
    //    - cleanable.clean() → State.run() 실행 → 자원 정리
    //    - clean()은 1회만 동작하므로, 나중에 GC 때 Cleaner가 또 호출해도 중복 실행 안 됨
    @Override
    public void close() {
        cleanable.clean();
    }
}
```

---

## 13. Room 클래스 완전 해부

### 이 코드가 하려는 것

**"방(Room)을 다 쓰면 청소해야 한다"**는 것뿐이다. 그게 전부다.

### 비유로 이해하기

```
Room          = 호텔 방
State         = 방 안의 쓰레기 상태 (쓰레기 몇 개 있는지)
Cleaner       = 호텔 청소 매니저
Cleanable     = 청소 예약 티켓
close()       = 손님이 체크아웃할 때 직접 "청소해주세요" 요청
GC + Cleaner  = 손님이 그냥 도망가면, 매니저가 언젠가 알아서 청소
```

### Runnable이 뭔가?

```java
// Java가 제공하는 인터페이스
public interface Runnable {
    void run(); // "실행할 작업"을 담는 메서드
}
```

Runnable은 **"나중에 실행할 작업을 미리 적어두는 메모지"**이다.

```java
// 예시: "나중에 이거 실행해줘"라고 작업을 미리 적어두는 것
Runnable 메모지 = new Runnable() {
    @Override
    public void run() {
        System.out.println("청소해!"); // ← 이 작업을 나중에 실행해달라는 것
    }
};

// 나중에 누군가가 이걸 실행
메모지.run(); // "청소해!" 출력
```

State는 바로 이 **"청소 메모지"**이다:

```
┌──────────────────────────┐
│ 청소 메모지 (State)        │
│                          │
│ 현재 쓰레기: 7개           │
│                          │
│ 청소 방법 (run):           │
│  1. "Cleaning room" 출력   │
│  2. 쓰레기를 0개로 만들기   │
└──────────────────────────┘
```

### 왜 State가 static class여야 하는가?

Java에서 내부 클래스는 두 종류가 있다:

```java
public class Room {
    // ❌ static 없는 내부 클래스 (non-static inner class)
    class BadState implements Runnable {
        // 이 클래스는 몰래 바깥의 Room을 참조하고 있음!
        // BadState 안에 숨겨진 코드: Room.this (바깥 Room에 대한 참조)
    }

    // ✅ static 내부 클래스 (static nested class)
    static class GoodState implements Runnable {
        // 바깥 Room과 완전히 독립적. Room을 참조하지 않음!
    }
}
```

왜 이게 중요한가:

```
static 안 붙인 경우 (BadState):
  Cleaner → BadState → Room을 참조!
                        ↑ 누군가 참조하고 있음
                        → GC: "수거 못해"
                        → Cleaner: 영원히 안 불림
                        → 💀 메모리 누수!

static 붙인 경우 (GoodState):
  Cleaner → GoodState (Room 참조 없음!)
  Room → 아무도 참조 안 함 → GC: "수거 가능!"
  → Cleaner: "Room이 수거됐네, GoodState.run() 실행!"
  → ✅ 청소 완료!
```

참조 체인으로 정리:

```
Cleaner → BadState → Room ← 참조 체인 때문에 GC 수거 불가 ❌

Cleaner → GoodState    Room ← 참조 체인 없음! GC 수거 가능 ✅
          (Room 참조 없음)
```

**static이 없으면 Cleaner가 영원히 동작하지 않는다.** 그래서 코드에 "절대 Room을 참조해서는 안 된다!"라는 주석이 있는 것이다.

### register()는 어떻게 Cleanable을 반환하는가?

`register()` 메서드는 Java가 미리 만들어놓은 기능이다:

```java
// Java 내부에 이미 이렇게 정의되어 있음 (우리가 만드는 게 아님)
public class Cleaner {
    public Cleanable register(Object obj, Runnable action) {
        // 1. obj(감시 대상)와 action(실행할 작업)을 수첩에 기록
        // 2. Cleanable 객체를 만들어서 돌려줌
        return new Cleanable(/* ... */);
    }
}
```

마치 우리가 ArrayList를 쓸 때 `list.add()`나 `list.size()`가 어떻게 동작하는지 만들 필요 없는 것과 같다.

Cleanable은 아주 단순한 인터페이스이다:

```java
// Java 내부에 정의되어 있음
public interface Cleanable {
    void clean(); // 이것만 있음. 호출하면 등록된 run()이 실행됨
}
```

### clean()을 호출하면 왜 State.run()이 실행되는가?

register()할 때 이미 연결이 된 것이다:

```java
// register 시점에 연결됨
cleanable = cleaner.register(this, state);
//                                 -----
//                                 이 state의 run()을 실행하겠다고 등록한 것
```

비유하면 **리모컨 버튼**이다:

```
register() = 리모컨 만들기
  "이 버튼(clean) 누르면 저 작업(state.run) 실행되게 설정"

cleanable.clean() = 리모컨 버튼 누르기
  → 설정해둔 대로 state.run() 실행!
```

전체 연결 관계:

```
register(this, state) 시점:
  "나중에 clean()이 호출되면, state.run()을 실행해라"라고 연결해둠

            ┌──────────────────────┐
cleanable = │ clean() 호출하면      │
            │ → state.run() 실행    │
            └──────────────────────┘

close() 호출 시점:
  cleanable.clean()
  → 위에서 연결해둔 대로 state.run() 실행!
  → "Cleaning room" 출력, 쓰레기 0개로
```

### Cleaner에 "등록"하는 과정

"안에 정의"가 아니라 "등록"이다:

```java
// 이 순간에 Cleaner에게 알려주는 것이다
cleanable = cleaner.register(this, state);
//                           ----  -----
//                           감시대상  실행할작업
```

실생활 비유:

```
나: "매니저님, 제 이름이랑 청소 메모지 드릴게요"
매니저: "알겠어. 내 수첩에 적어둘게"

┌─── 매니저(Cleaner)의 수첩 ───┐
│                              │
│  Room@abc → State@xyz        │
│  (감시 대상)   (청소 메모지)    │
│                              │
│  Room@def → State@uvw        │
│  (다른 방)     (다른 메모지)    │
│                              │
└──────────────────────────────┘
```

매니저는 이 수첩을 계속 들여다보면서 감시한다:

```
매니저: "Room@abc 아직 살아있나? → 응, 누가 참조 중 → 넘어감"
매니저: "Room@abc 아직 살아있나? → 아, 아무도 참조 안 하네! GC 수거됐네!"
매니저: "수첩 보자... Room@abc에 연결된 건 State@xyz네"
매니저: "State@xyz.run() 실행!" → 청소!
```

그런데 만약 State가 Room을 참조하면:

```
매니저 수첩: Room@abc → BadState@xyz
                        BadState@xyz → Room@abc를 참조! (static 안 붙여서)

GC: "Room@abc 수거해도 되나?"
GC: "잠깐, BadState@xyz가 Room@abc를 참조하고 있어"
GC: "그리고 Cleaner가 BadState@xyz를 참조하고 있어"
GC: "Cleaner → BadState → Room... 참조 체인이 있으니 수거 못해!"

→ Room이 영원히 수거 안 됨
→ Cleaner가 영원히 "아직 살아있네" 판단
→ 청소가 영원히 안 됨 💀
```

---

## 14. Cleaner의 내부 동작 원리: PhantomReference + ReferenceQueue

### 핵심 질문: "close()를 안 불렀는데 어떻게 run()이 실행돼?"

비밀은 `register()`에 있다. 이 한 줄이 내부적으로 하는 일:

```java
cleanable = cleaner.register(this, state);
```

```java
// cleaner.register(this, state)가 내부적으로 하는 일 (간략화)
public Cleanable register(Object obj, Runnable action) {

    // 1. obj(Room)를 "약한 참조(PhantomReference)"로 감싼다
    //    약한 참조 = "GC가 이 객체를 수거해도 괜찮아,
    //               대신 수거되면 나한테 알려줘"
    PhantomReference<Object> ref = new PhantomReference<>(obj, referenceQueue);

    // 2. "이 참조가 수거 알림을 받으면 → action.run()을 실행해"라고 매핑
    cleanerMap.put(ref, action);

    // 3. Cleanable을 돌려줌
    return new CleanableImpl(ref, action);
}
```

그리고 Cleaner 내부에는 감시 스레드가 돌고 있다:

```java
// Cleaner.create()가 내부적으로 만드는 스레드 (간략화)
Thread cleanerThread = new Thread(() -> {
    while (true) {
        // referenceQueue에서 대기
        // → GC가 객체를 수거하면 여기에 알림이 온다!
        Reference<?> ref = referenceQueue.remove(); // 알림 올 때까지 대기

        // 알림 왔다 = 누군가가 GC에 수거됐다!
        Runnable action = cleanerMap.get(ref); // 등록된 작업 찾기
        action.run(); // 실행!
    }
});
cleanerThread.setDaemon(true); // 데몬 스레드 (JVM 종료 시 같이 종료)
cleanerThread.start();
```

### 시간 순서로 보는 close() 미호출 시나리오

```java
// 1단계: Room 생성
Room room = new Room(7);
// 내부에서 일어나는 일:
//   state = new State(7);
//   cleanable = cleaner.register(this, state);
//   → Cleaner가 Room 객체를 PhantomReference로 감시 시작
//   → Cleaner 스레드: referenceQueue.remove()에서 대기 중... 💤

// 2단계: Room 사용
System.out.println("방 사용 중");

// 3단계: 참조를 끊음 (close()는 안 부름!)
room = null;

// 4단계: GC 발생
// GC: "Room 객체를 아무도 참조 안 하네, 수거하자"
// GC: Room 객체 수거!
// GC: "PhantomReference가 있네, referenceQueue에 알림 넣어줄게"
//
//     referenceQueue ← [Room의 PhantomReference]  ← GC가 넣음!

// 5단계: Cleaner 스레드가 깨어남
// Cleaner 스레드: "오! referenceQueue에 뭔가 왔다!"
//   ref = referenceQueue.remove();  ← 대기 끝, 알림 받음!
//   action = cleanerMap.get(ref);   ← 등록된 작업 찾기 → State 객체
//   action.run();                   ← State.run() 실행!
//   → "Cleaning room" 출력
//   → numJunkPiles = 0
```

### 핵심 메커니즘 그림

```
[등록 시점]
  cleaner.register(room, state)
  → room을 PhantomReference로 감싸서 referenceQueue에 연결

[GC 발생 시점]
  GC가 room 수거 → referenceQueue에 "수거됐어!" 알림을 넣음

[Cleaner 스레드]
  referenceQueue를 계속 감시하고 있다가
  → 알림이 오면 → 등록된 state.run()을 실행
```

```
등록 시:
  Cleaner 스레드 ──감시──▶ referenceQueue (비어있음, 대기 중... 💤)
  PhantomReference ──감시──▶ Room 객체

GC 발생 시:
  GC가 Room 수거 → PhantomReference가 referenceQueue에 들어감!

  Cleaner 스레드 ──감시──▶ referenceQueue [알림 도착!]
  Cleaner 스레드: "Room이 수거됐구나! state.run() 실행!"
```

결국 **close()를 안 불러도 run()이 실행되는 원리**는:

> Cleaner가 register() 시점에 Room을 PhantomReference로 감시하고 있다가, GC가 Room을 수거하면 알림을 받고, 그때 등록해둔 state.run()을 실행하는 것이다.

이것이 "안전망"의 정체이다.

---

## 15. 코드 8-2, 8-3: Adult와 Teenager

### Adult (제대로 활용하는 클라이언트)

```java
public class Adult {
    public static void main(String[] args) {
        try (Room myRoom = new Room(7)) {
            System.out.println("안녕~");
        }
    }
}
```

실행 결과:
```
안녕~
Cleaning room
```

흐름:

```
1. new Room(7) → 방 생성
2. "안녕~" 출력
3. try 블록 끝 → close() 자동 호출 (try-with-resources 덕분)
   → cleanable.clean()
   → State.run()
   → "Cleaning room" 출력 ✅
```

**close()를 직접 호출**하는 것이므로 GC와는 아무 관련 없다. 100% 확실하게 동작한다.

### Teenager (제대로 활용하지 못하는 클라이언트)

```java
public class Teenager {
    public static void main(String[] args) {
        new Room(99);
        System.out.println("Peace out");
//      System.gc();
    }
}
```

실행 결과:
```
Peace out
(끝 — "Cleaning room"이 출력되지 않음)
```

### "당연히 안 되는 거 아니야?"에 대한 답

결론부터 말하면 **"안 될 수도 있지만, 될 수도 있다"**가 정확한 표현이다. 이론적으로는 이럴 수도 있다:

```
1. new Room(99) → 방 생성, 변수에 안 담음 → 즉시 GC 대상
2. "Peace out" 출력
3. 프로그램 종료 직전에 GC가 동작할 수도 있음
4. GC가 동작하면 → Cleaner가 State.run() 호출 → "Cleaning room" 출력될 수도!
```

실행 결과가 이럴 수도 있고:
```
Peace out
(끝)
```

이럴 수도 있다:
```
Peace out
Cleaning room
(끝)
```

### 왜 결과가 매번 다를 수 있는가?

```
프로그램 종료 시 JVM이 하는 일:

  "Peace out" 출력 완료
  → main() 끝남
  → JVM 종료 시작

  이 시점에서 JVM이 선택할 수 있는 것:

  선택 A: "GC 한번 돌려볼까?" → Cleaner 동작 → "Cleaning room" 출력 ✅
  선택 B: "그냥 바로 꺼야지" → Cleaner 동작 안 함 → 출력 안 됨 ❌

  어떤 선택을 할지는 JVM 구현마다 다름!
```

Cleaner의 명세에도 이렇게 쓰여있다: **"System.exit을 호출할 때의 cleaner 동작은 구현하기 나름이다. 청소가 이뤄질지는 보장하지 않는다."**

### 책이 말하려는 핵심

```
✅ 확실한 코드 (Adult):
   → 항상 "Cleaning room" 출력. 100%. 어떤 JVM에서든.

❌ 불확실한 코드 (Teenager):
   → 내 컴퓨터에서는 안 나옴
   → 네 컴퓨터에서는 나올 수도 있음
   → 오늘은 안 나왔는데 내일은 나올 수도 있음
   → Java 버전 바꾸면 결과가 달라질 수도 있음
```

문제는 **"안 된다"가 아니라 "될지 안 될지 모른다"**는 것이다. 예측 불가능한 코드는 프로덕션에서 절대 쓰면 안 된다.

### System.gc() 주석 해제하면?

```java
new Room(99);
System.out.println("Peace out");
System.gc(); // GC를 "강제로" 요청
```

```
System.gc()를 부르면:
→ JVM에게 "GC 좀 돌려줘"라고 요청
→ GC가 Room을 수거할 가능성이 높아짐
→ Cleaner가 State.run()을 호출할 가능성이 높아짐
→ "Cleaning room"이 출력될 가능성이 높아짐

하지만! System.gc()도 "요청"일 뿐 "명령"이 아님
→ JVM이 무시할 수도 있음
→ 여전히 100% 보장은 안 됨!
```

그래서 주석에 **"가비지 컬렉터를 강제로 호출하는 이런 방식에 의존해서는 절대 안 된다"**고 써있는 것이다.

---

## 16. 두 경로 최종 정리

State.run()이 실행되는 경로는 두 가지이다:

```
경로 1: close() 직접 호출 (Adult)
  개발자가 close() 호출
  → cleanable.clean()
  → State.run()
  → GC와 무관. 즉시. 확실. ✅

경로 2: GC + Cleaner (Teenager)
  close()를 안 부름
  → 언젠가 GC가 Room 수거
  → Cleaner가 알림 받음
  → State.run()
  → 언제 될지 모름. 안 될 수도 있음. ⚠️
```

**같은 `State.run()`이 실행되는 건 맞는데, "누가 실행시키느냐"가 다르다:**

```
경로 1: 개발자 → close() → clean() → run()
경로 2: GC → Cleaner 스레드 → run()
```

Room 코드의 주석도 이것을 말하고 있다:

```java
// close 메서드나 cleaner가 호출한다.
@Override
public void run() {
    System.out.println("Cleaning room");
    numJunkPiles = 0;
}
```

"close 메서드**나** cleaner**가**" — 둘 중 하나가 호출한다는 뜻이다. 두 갈래 길이 있는 것이다.

### 전체 동작 흐름 요약

```
정상 경로 (try-with-resources 사용 시):
  try (Room room = new Room(7)) {
      // 방 사용
  }
  → close() 호출
  → cleanable.clean() 호출
  → State.run() 실행 → "Cleaning room" 출력, 자원 정리 ✅
  → 나중에 GC가 와도 이미 clean 됐으니 무시

안전망 경로 (close()를 깜빡한 경우):
  Room room = new Room(7);
  room = null; // close() 안 부르고 참조만 끊음
  → (시간이 지남...)
  → GC가 Room을 수거 대상으로 판단
  → Cleaner가 State.run() 호출 → "Cleaning room" 출력, 자원 정리
  → ⚠️ 단, 이게 "언제" 될지, "될지 안 될지"는 보장 안 됨!
```

---

## 17. Cleaner/Finalizer를 쓸 수 있는 경우

Item 8이 "피하라"고 했지 "절대 쓰지 마라"는 아니다. 두 가지 합리적인 용도가 있다:

1. **안전망(safety net)으로서의 역할**: 개발자가 close()를 깜빡 호출하면, 늦더라도 cleaner가 정리해주는 게 안 하는 것보단 낫다.
2. **네이티브 피어(native peer) 정리**: Java 객체가 C/C++ 같은 네이티브 코드로 만들어진 자원과 연결되어 있을 때, GC가 네이티브 자원의 존재를 모르니까 cleaner로 정리해줄 수 있다.

---

## 18. 역사적 흐름 요약

```
Java 1.0 시절 (1996년)
├─ GC: Java 메모리 자동 회수 ✅ 잘 동작함
├─ 외부 자원(파일, DB 등)은 GC가 못 치움
├─ 해결책: "finalize()를 제공하자!"
│   → 개발자가 필요한 경우에만 오버라이드해서
│   → GC 시점에 외부 자원도 정리하게 하자
├─ 현실: finalize()가 위험하고 문제투성이
│
Java 9 (2017년)
├─ finalize() deprecated 선언
├─ Cleaner 도입: 같은 목적이지만 좀 더 안전한 설계
│   → 하지만 근본적 한계(실행 시점 미보장)는 동일
│
현재 권장 방식
├─ 외부 자원 정리 → AutoCloseable + try-with-resources
├─ Cleaner/Finalizer → 안전망으로만 사용
└─ GC → 원래 하던 대로 Java 메모리만 관리
```

---

## 19. 최종 핵심 정리

### 자원 해제 전략

```
자원 해제가 필요한 객체
        │
        ▼
  ┌─────────────────────┐
  │  AutoCloseable +     │  ← ✅ 이걸 써라! (확실하고 빠름)
  │  try-with-resources  │
  └──────────┬──────────┘
             │
             │  만약 close() 호출을 깜빡했다면?
             ▼
  ┌─────────────────────┐
  │  Cleaner (안전망)     │  ← ⚠️ 최후의 보루 (늦더라도 안 하는 것보단 나음)
  └──────────┬──────────┘
             │
             │  이것도 실행 안 될 수 있음
             ▼
  ┌─────────────────────┐
  │    자원 누수 💀       │  ← ❌ 최악의 상황
  └─────────────────────┘
```

### 한 줄 요약

| 방법 | 설명 | 권장도 |
|------|------|--------|
| `try-with-resources` | 개발자가 직접, 즉시, 확실하게 정리 | ✅ 이걸 써라 |
| Cleaner | GC 시점에 자동 정리 (안전망) | ⚠️ 보조적으로만 |
| Finalizer | GC 시점에 자동 정리 (레거시) | ❌ 쓰지 마라 |

대부분의 경우 GC만으로 충분하다. 문제가 되는 건 GC가 모르는 외부 자원을 쓸 때뿐이며, 그 해결책은 `AutoCloseable` + `try-with-resources`이다.