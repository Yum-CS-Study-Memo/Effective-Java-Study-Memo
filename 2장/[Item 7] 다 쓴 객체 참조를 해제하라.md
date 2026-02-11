# Effective Java Item 7 - 다 쓴 객체 참조를 해제하라

## 📌 핵심 요약

메모리 누수는 겉으로 잘 드러나지 않는다. 자기 메모리를 직접 관리하는 클래스, 캐시, 리스너/콜백이 메모리 누수의 **3대 주범**이며, 다 쓴 객체 참조는 반드시 해제해야 한다.

---

## 1. 메모리 누수가 숨어있는 Stack 코드

### 문제의 코드

```java
public class Stack {
    private Object[] elements;
    private int size = 0;
    private static final int DEFAULT_INITIAL_CAPACITY = 16;

    public Stack() {
        elements = new Object[DEFAULT_INITIAL_CAPACITY];
    }

    public void push(Object e) {
        ensureCapacity();
        elements[size++] = e;
    }

    public Object pop() {
        if (size == 0)
            throw new EmptyStackException();
        return elements[--size];  // ⚠️ size만 줄이고, 배열에는 값이 그대로!
    }

    private void ensureCapacity() {
        if (elements.length == size)
            elements = Arrays.copyOf(elements, 2 * size + 1);
    }
}
```

### 코드 흐름

이 코드는 **스택(Stack)**을 직접 구현한 것이다. 스택은 **접시 쌓기**와 같다. 접시를 위에 올리고(push), 뺄 때도 맨 위에서부터 빼는(pop) 구조다.

```
    ┌───────┐
    │  포도   │ ← 마지막에 넣은 게 맨 위 (pop하면 이게 나옴)
    ├───────┤
    │  바나나  │
    ├───────┤
    │  사과   │ ← 처음에 넣은 건 맨 아래
    └───────┘
```

#### 필드(변수)들의 역할

```java
private Object[] elements;    // 실제 데이터를 저장하는 배열 (접시를 쌓는 선반)
private int size = 0;         // 현재 쌓인 개수 (지금 접시가 몇 개인지)
private static final int DEFAULT_INITIAL_CAPACITY = 16;  // 처음 선반 크기 = 16칸
```

#### push() — 데이터 넣기

```java
public void push(Object e) {
    ensureCapacity();      // 1단계: 자리 있는지 확인
    elements[size++] = e;  // 2단계: 넣고 size 증가
}
```

`elements[size++] = e`를 풀어쓰면:

```java
elements[size] = e;  // 현재 size 위치에 데이터 넣고
size = size + 1;     // size를 1 증가
```

#### pop() — 데이터 빼기

```java
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();  // 빈 스택이면 에러
    return elements[--size];              // size 줄이고 그 위치의 값 반환
}
```

#### ensureCapacity() — 공간 확보

```java
private void ensureCapacity() {
    if (elements.length == size)
        elements = Arrays.copyOf(elements, 2 * size + 1);
}
```

16칸을 다 채우면 **배열을 약 2배로 늘린다**. 이사하는 것과 같다. 방이 좁아지면 짐을 다 들고 더 큰 집으로 이사하는 것이다.

### 어디서 메모리 누수가 발생하는가?

push 3번 후 pop 2번 하면 이런 상태가 된다:

```
push("사과"), push("바나나"), push("포도") 후:

elements: ["사과", "바나나", "포도", null, null, ...]
                                    ↑ size = 3

pop() 두 번 실행 후:

elements: ["사과", "바나나", "포도", null, null, ...]
            ↑ size = 1
           유효     ↑──────↑
                   이것들은 꺼냈는데
                   배열에 아직 남아있음! 💀
```

"바나나"와 "포도"는 이미 `pop()`으로 꺼냈으니 **더 이상 쓸모가 없는 데이터**다. 그런데 `elements` 배열이 여전히 이 객체들을 **붙잡고(참조하고)** 있다.

### 왜 이게 문제인가? — 가비지 컬렉터(GC)의 한계

Java에는 **가비지 컬렉터(GC)**라는 청소부가 있다. 아무도 안 쓰는 객체를 자동으로 치워주는 녀석이다.

```
🧹 가비지 컬렉터의 판단 기준:
"누군가가 이 객체를 참조하고 있나?"
   → YES: 아직 쓰이는 거구나, 놔두자
   → NO:  아무도 안 쓰네, 치우자!
```

문제는 GC가 **"활성 영역"과 "비활성 영역"을 구분할 수 없다**는 것이다.

```
elements: ["사과", "바나나", "포도", null, null, ...]

           ┌─────┐  ┌──────────────┐
           │활성   │  │ 비활성 (쓸모없음) │
           │영역   │  │              │
           └─────┘  └──────────────┘
           size=1    ↑
                     GC 입장에서는 배열이 참조하고 있으니
                     "아직 쓰이는 객체"로 봄! 😱
```

프로그래머는 "size 이후는 안 쓰는 거야"라고 알고 있지만, **GC는 이걸 모른다**. GC가 보기에는 배열이 여전히 "바나나"와 "포도"를 참조하고 있으니 유효한 객체라고 판단한다.

### 해결 방법: null 처리로 GC에게 알려주기

```java
// ❌ 기존 코드 (메모리 누수)
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    return elements[--size];
}

// ✅ 수정된 코드 (null 처리 추가)
public Object pop() {
    if (size == 0)
        throw new EmptyStackException();
    Object result = elements[--size];
    elements[size] = null;  // 🔑 다 쓴 참조 해제!
    return result;
}
```

```
null 처리 후:

elements: ["사과", null, null, null, null, ...]
            ↑ size = 1

"바나나"와 "포도"를 가리키던 참조가 null로 끊어짐!
→ GC가 "아, 아무도 안 쓰는구나" 하고 치워줌 🧹✨
```

> 비유: **퇴실할 때 방 열쇠를 반납하는 것**과 같다. 열쇠를 안 반납하면(참조를 안 끊으면) 호텔은 "아직 투숙 중"이라고 생각하고 그 방을 다른 사람에게 못 준다.

### 왜 Stack이 특히 메모리 누수에 취약할까?

> **스택이 자기 메모리를 직접 관리하기 때문이다.**

이 Stack은 `elements` 배열로 **저장소 풀(pool)**을 만들어서 원소들을 직접 관리한다. 배열 안에서 어디까지가 "활성"이고 어디부터가 "비활성"인지는 **프로그래머만 알고 GC는 모른다**.

> **자기 메모리를 직접 관리하는 클래스라면, 프로그래머는 항시 메모리 누수에 주의해야 한다.** 원소를 다 사용한 즉시 그 원소가 참조한 객체들을 다 null 처리해줘야 한다.

---

## 2. 메모리 누수 주범 ② — 캐시

### 용어 정리

**캐시(Cache)**: 자주 쓰는 데이터를 미리 꺼내놓는 **임시 보관소**

```
캐시 = 빠르게 꺼내 쓰려고 임시 저장해둔 것
비유: 냉장고 문에 붙이는 자주 쓰는 전화번호 메모
예: 웹 브라우저가 한번 본 이미지를 저장해두는 것
```

**엔트리(Entry)**: 캐시 안에 저장된 **데이터 한 건**(Map에서 "키-값 한 쌍")

```
캐시(냉장고 문):
  엔트리 1: "엄마" → "010-1234-5678"
  엔트리 2: "피자집" → "02-999-8888"
  엔트리 3: "이사한 친구" → "010-0000-0000"  ← 더 이상 안 쓰는데 남아있음 💀
```

### 문제 상황

```
1. 객체 참조를 캐시에 넣는다
2. 이 사실을 까맣게 잊는다
3. 그 객체를 다 쓴 뒤로도 한참을 그냥 놔둔다
→ 메모리 누수! 💀
```

> 비유: 냉장고에 반찬을 넣어놓고 **존재를 잊어버려서** 한참 뒤에 상한 채로 발견하는 것

### 해결 방법 1: WeakHashMap 사용

캐시 **외부에서** 키를 참조하는 동안만 엔트리가 살아 있으면 되는 상황이라면 `WeakHashMap`을 사용한다. 키를 아무도 안 쓰면 자동 삭제된다.

```java
WeakHashMap<Key, Value> cache = new WeakHashMap<>();
```

> 단, WeakHashMap은 이러한 상황에서만 유용하다는 사실을 기억하자.

### 해결 방법 2: 시간 기반 정리

캐시 엔트리의 유효 기간을 정확히 정의하기 어렵기 때문에, **시간이 지날수록 엔트리의 가치를 떨어뜨리는 방식**을 흔히 사용한다.

```
방법 A: 백그라운드 스레드(ScheduledThreadPoolExecutor)로 주기적 청소
  → 뒤에서 주기적으로 유통기한 지난 엔트리를 치우는 것

방법 B: 새 엔트리 추가할 때 부수 작업으로 오래된 것 삭제
  → LinkedHashMap의 removeEldestEntry() 메서드 활용
  → "새 반찬 넣을 때 가장 오래된 반찬 버리기"
```

**백그라운드 스레드**: 뒤에서 몰래 돌아가는 작업 (식당의 주방 직원처럼 손님이 안 보는 곳에서 알아서 청소)

**LinkedHashMap**: HashMap인데 **넣은 순서를 기억**하는 Map → "가장 오래된 것"을 알 수 있어서 `removeEldestEntry()`로 자동 삭제 가능

---

## 3. 메모리 누수 주범 ③ — 리스너(Listener) / 콜백(Callback)

### 용어 정리

**리스너/콜백**: "어떤 일이 생기면 나한테 알려줘!"라고 등록해놓는 것

```
비유: 유튜브 구독 알림

유튜브 채널(이벤트 발생자) ← "구독+알림 설정"(리스너 등록) ← 나(리스너)

새 영상 올라옴! → 유튜브가 나한테 알림 보냄 → 내가 반응
```

```java
button.addActionListener(myListener);  // "버튼 클릭되면 나한테 알려줘!"

// 문제: 더 이상 알림 안 받고 싶은데 해지를 안 하면?
// → myListener 객체가 계속 메모리에 남아있음! 💀
```

### 문제 상황

```
1. 클라이언트가 콜백을 등록한다
2. 명확히 해지하지 않는다
3. 뭔가 조치해주지 않으면 콜백은 계속 쌓여간다
→ 메모리 누수! 💀
```

> 비유: 유튜브 채널을 구독만 하고 절대 구독 취소를 안 하는 것

### 해결 방법: 약한 참조(Weak Reference)로 저장

콜백을 약한 참조로 저장하면, 해당 콜백을 더 이상 아무도 안 쓸 때 GC가 자동으로 수거해간다. 예를 들어 WeakHashMap에 키로 저장하면 된다.

```java
WeakHashMap<Callback, Boolean> callbacks = new WeakHashMap<>();
```

---

## 4. 강한 참조 vs 약한 참조 — GC 수거 기준의 차이

### GC의 기본 원리

```
도달 가능(reachable) → 살림
도달 불가능(unreachable) → 수거
```

### 강한 참조 (HashMap)

```java
HashMap<Key, Value> cache = new HashMap<>();
Key myKey = new Key("데이터");
cache.put(myKey, someValue);

myKey = null;  // 내가 키 참조를 끊었음!

// 하지만... HashMap 내부에서 여전히 이 Key를 강하게 참조 중!
// GC: "HashMap이 잡고 있네 → 못 치운다" ❌ 수거 안 됨!
```

```
[나(myKey)] ----X---→ Key("데이터")    ← 내가 끊었지만
[HashMap 내부] ------→ Key("데이터")    ← 얘가 꽉 잡고 있음!
                                        GC: "누가 잡고 있네, 패스~"
```

### 약한 참조 (WeakHashMap)

```java
WeakHashMap<Key, Value> cache = new WeakHashMap<>();
Key myKey = new Key("데이터");
cache.put(myKey, someValue);

myKey = null;  // 내가 키 참조를 끊었음!

// WeakHashMap 내부는 이 Key를 "약하게" 참조 중
// GC: "강하게 잡고 있는 사람이 아무도 없네 → 치운다!" ✅ 수거됨!
```

```
[나(myKey)] ----X---→ Key("데이터")    ← 내가 끊었고
[WeakHashMap] --~~→  Key("데이터")    ← 약하게만 잡고 있음 (옷자락)
                                        GC: "꽉 잡는 사람 없네, 치운다!"
```

### 비유: 놀이공원 퇴장 규칙

```
강한 참조 = 입장 팔찌 🎫
  → 팔찌가 하나라도 남아있으면 퇴장 불가
  → HashMap이 팔찌를 채워주기 때문에,
    내가 내 팔찌를 풀어도(null) HashMap의 팔찌가 남아있어서 못 나감

약한 참조 = 그냥 같이 온 친구가 "저 사람 알아요" 하는 것 👋
  → 이건 팔찌가 아님!
  → 내가 팔찌를 풀면(null), 진짜 팔찌 가진 사람이 없으므로 퇴장됨
```

### 정리: "언제" 수거되는가?

| 상황 | HashMap (강한 참조) | WeakHashMap (약한 참조) |
|---|---|---|
| 내가 `key = null` 했을 때 | ❌ **수거 안 됨** (Map 내부가 강하게 잡고 있으니까) | ✅ **수거됨** (강하게 잡는 사람이 아무도 없으니까) |
| Map에서 `remove(key)` 했을 때 | ✅ 수거됨 | ✅ 수거됨 |
| 아무것도 안 했을 때 | ❌ 영원히 남아있음 → **메모리 누수!** | GC가 알아서 치움 |

> 약한 참조의 핵심: **"Map 바깥에서 아무도 그 키를 강하게 참조하지 않으면"** → GC가 자동 수거. HashMap은 자기가 강하게 붙잡아서 GC가 못 치우고, WeakHashMap은 약하게만 잡고 있어서 바깥에서 놓는 순간 GC가 치울 수 있다.

---

## 📝 전체 요약: 메모리 누수 3대 주범

| 주범 | 원인 | 해결 방법 |
|---|---|---|
| **① 자기 메모리 직접 관리** | Stack처럼 배열로 직접 관리하는 경우 pop 후에도 배열이 다 쓴 객체를 참조 | 다 쓴 참조를 **null 처리** |
| **② 캐시** | 객체 참조를 캐시에 넣고 잊어버리는 경우 | **WeakHashMap** 또는 백그라운드 스레드/LinkedHashMap으로 주기적 청소 |
| **③ 리스너/콜백** | 등록만 하고 명확히 해지하지 않는 경우 | **약한 참조(WeakHashMap)**로 저장하여 자동 수거 |