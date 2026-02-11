## 🚫 잘못된 방식 1: 정적 유틸리티 클래스

> **정적 유틸리티 클래스**란? 인스턴스를 만들지 않고, `static` 메서드만으로 기능을 제공하는 클래스.  
> 생성자를 `private`으로 막아서 `new`로 객체를 못 만들고, 모든 메서드가 `static`이며, 상태(인스턴스 변수)를 갖지 않는다.  
> 대표적인 예: `Math.abs(-5)`, `Math.max(3, 7)`, `Arrays`, `Collections` 등  
> Item 4에서 배운 "인스턴스화를 막는 클래스"가 바로 이 패턴이다.

```java
// 부적절한 static 유틸리티 사용 예 - 유연하지 않고 테스트할 수 없다.
public class SpellChecker {
    private static final Lexicon dictionary = new KoreanDictionary();

    private SpellChecker() {} // Noninstantiable

    public static boolean isValid(String word) { ... }
    public static List suggestions(String typo) { ... }
}

// 사용법: 객체 없이 클래스에 직접 호출
SpellChecker.isValid("hello");
```

## 🚫 잘못된 방식 2: 싱글턴

```java
// 부적절한 싱글톤 사용 예 - 유연하지 않고 테스트할 수 없다.
public class SpellChecker {
    private final Lexicon dictionary = new KoreanDictionary();

    private SpellChecker() {}
    public static final SpellChecker INSTANCE = new SpellChecker();

    public boolean isValid(String word) { ... }
    public List suggestions(String typo) { ... }
}

// 사용법: 유일한 인스턴스를 통해 호출
SpellChecker.INSTANCE.isValid("hello");
```

### 정적 유틸리티 vs 싱글턴 차이

| | 정적 유틸리티 | 싱글턴 |
|---|---|---|
| 인스턴스 | 객체 자체가 없음 | 객체가 딱 하나 존재 |
| 호출 방식 | `SpellChecker.isValid("hello")` | `SpellChecker.INSTANCE.isValid("hello")` |

### 둘 다 나쁜 이유

두 방식 모두 **dictionary를 하드코딩**하고 있다. 사전을 바꾸려면 클래스 코드를 직접 수정해야 하므로 유연하지 않고, 테스트하기도 어렵다.

> **비유:** 식당에서 메뉴가 김치찌개 하나뿐. 바꾸려면 식당을 리모델링해야 한다.

---

## ✅ 올바른 방식: 의존 객체 주입(Dependency Injection)

클래스가 여러 자원 인스턴스(영어 사전, 중국어 사전, 프랑스어 사전 등)를 지원해야 하며, 클라이언트가 원하는 자원을 사용해야 한다면 → **인스턴스를 생성할 때 생성자에 필요한 자원을 넘겨주는 방식**을 사용한다.

```java
public class SpellChecker {
    private final Lexicon dictionary;

    public SpellChecker(Lexicon dictionary) {
        this.dictionary = Objects.requireNonNull(dictionary);
    }

    public boolean isValid(String word) { ... }
    public List suggestions(String typo) { ... }
}
```

> **비유:** 식당에 주방은 있는데, 재료는 손님이 가져온다. 한식 재료 가져오면 한식, 양식 재료 가져오면 양식. 주방(클래스)은 그대로인데 결과물이 달라진다.

### "자원이 몇 개든 잘 작동한다"

dictionary 하나만이 아니라, 필요한 만큼 주입할 수 있다.

```java
public class SpellChecker {
    private final Lexicon dictionary;
    private final Grammar grammar;
    private final StyleGuide style;

    // 3개든 10개든 필요한 만큼 주입
    public SpellChecker(Lexicon dictionary, Grammar grammar, StyleGuide style) {
        this.dictionary = dictionary;
        this.grammar = grammar;
        this.style = style;
    }
}
```

하드코딩이었다면 자원 하나 바꿀 때마다 클래스 코드를 직접 수정해야 한다. 주입 방식이면 밖에서 넣어주기만 하면 되니까 클래스는 건드릴 필요가 없다.

### "의존 관계가 어떻든 상관없이"

의존 객체끼리 서로 엮여 있어도 괜찮다.

```java
// Grammar가 Lexicon에 의존하는 상황
Lexicon dict = new KoreanDictionary();
Grammar grammar = new KoreanGrammar(dict);  // Grammar가 dict를 필요로 함

// 복잡한 관계여도 밖에서 조립해서 넣어주면 끝
SpellChecker checker = new SpellChecker(dict, grammar);
```

SpellChecker 입장에서는 "나한테 완성된 걸 넣어줘"라고만 하면 되니까, 내부에서 누가 누구에 의존하는지 알 필요가 없다.

### "불변을 보장하여 안심하고 공유"

`final` 키워드가 포인트다.

```java
private final Lexicon dictionary;  // 한번 설정되면 절대 변경 불가
```

```java
Lexicon korDict = new KoreanDictionary();

// 여러 곳에서 같은 사전을 공유
SpellChecker checker1 = new SpellChecker(korDict);
SpellChecker checker2 = new SpellChecker(korDict);
SpellChecker checker3 = new SpellChecker(korDict);
```

만약 `final`이 없다면?

```java
// final 없는 경우 - 위험!
private Lexicon dictionary;

// checker1이 사전을 몰래 바꿔버리면?
checker1.setDictionary(new EnglishDictionary());
// checker2, checker3도 영향받을 수 있음 → 버그!
```

`final`이니까 "한번 넣으면 절대 안 바뀜"이 보장된다. 그래서 여러 클라이언트가 같은 객체를 공유해도 안전하다.

### "생성자, 정적 팩터리, 빌더 모두에 응용"

주입하는 통로만 다를 뿐, "외부에서 넣어준다"는 본질은 동일하다.

> **정적 팩터리 메서드**란? 객체를 만들어주는 `static` 메서드. `new` 대신 메서드를 호출해서 객체를 얻는 방식이다.  
> 예: `List.of(1, 2, 3)`, `Optional.empty()`, `Collections.emptyList()`

```java
// 방법 1: 생성자
SpellChecker checker = new SpellChecker(korDict);

// 방법 2: 정적 팩터리 메서드 (Item 1)
SpellChecker checker = SpellChecker.of(korDict);

// 방법 3: 빌더 (Item 2)
SpellChecker checker = new SpellChecker.Builder()
    .dictionary(korDict)
    .grammar(korGrammar)
    .style(formalStyle)
    .build();
```

---

## 🔧 변형: 생성자에 자원 팩터리를 넘겨주는 방식

### 팩터리란?

호출할 때마다 특정 타입의 인스턴스를 반복해서 만들어주는 객체. 즉, 팩터리 메서드 패턴을 구현한 것이다.

> **비유:**
> - **객체를 직접 넘기기** = "여기 커피 한 잔 줄게" (끝. 더 필요하면? 못 줌)
> - **팩터리를 넘기기** = "여기 커피머신 줄게. 버튼 누르면 커피 나와" (1잔이든 100잔이든 OK)

### 왜 팩터리가 필요한가?

기존 DI는 "이미 만들어진 객체 하나"를 넘기는 것이다. 대부분은 이걸로 충분하다.

```java
Lexicon dict = new KoreanDictionary();          // 먼저 만들고
SpellChecker checker = new SpellChecker(dict);  // 넣어줌
```

하지만 **호출할 때마다 새 객체가 필요한 상황**이라면?

```java
// 문서가 100개고, 각 문서마다 새 사전이 필요하다면?
for (Document doc : documents) {
    // 매번 새 사전이 필요한데... 사전 100개를 미리 만들어서 넘길 수는 없다.
}
```

이럴 때 완성된 객체가 아니라 **"만드는 방법"**을 넘긴다.

### Supplier\<T\>

자바 8에서 소개한 `Supplier<T>` 인터페이스가 팩터리를 표현한 완벽한 예다.

```java
// Supplier는 자바가 미리 만들어놓은 인터페이스다.
// 우리가 만드는 게 아니라, java.util.function 패키지에 이미 있다.
@FunctionalInterface  // 이 인터페이스는 메서드가 딱 1개만 있다는 표시
public interface Supplier {
    // T는 "아무 타입이나 들어올 수 있다"는 뜻 (제네릭)
    // 예: Supplier이면 T 자리에 Lexicon이 들어감
    // 예: Supplier이면 T 자리에 String이 들어감
    
    T get();
    // ↑ 이게 전부다. 메서드가 이거 딱 하나.
    // 의미: "나를 호출하면(.get()), T 타입 객체를 하나 만들어서 돌려줄게"
    // 매개변수 없음 → 그냥 호출만 하면 됨
    // 반환값 T → 우리가 정한 타입의 객체가 나옴
}
```

> **쉽게 말하면:** `Supplier<Lexicon>`은 "`.get()` 하면 `Lexicon`을 하나 줄게"라는 약속이다.

### 코드 비교: 기존 방식 vs 팩터리 방식

**기존 방식: 완성된 객체를 직접 넘김**

```java
public class SpellChecker {
    private final Lexicon dictionary;

    // 생성자: Lexicon 타입 객체를 받아서 저장
    public SpellChecker(Lexicon dictionary) {
        this.dictionary = dictionary;
        // ↑ 받은 사전을 그대로 저장. 끝.
    }
}

// === 사용하는 쪽 ===
// 1단계: 사전 객체를 먼저 만든다
KoreanDictionary korDict = new KoreanDictionary();
// ↑ 이 시점에 이미 사전이 만들어짐 (메모리에 올라감)

// 2단계: 만들어진 사전을 SpellChecker에 넘긴다
SpellChecker checker = new SpellChecker(korDict);
// ↑ korDict라는 "이미 완성된 사전"을 전달

// 비유: 붕어빵을 미리 구워서 건네주는 것
```

**팩터리 방식: "만드는 방법"을 넘김**

```java
public class SpellChecker {
    private final Lexicon dictionary;

    // 생성자: Lexicon이 아니라 Supplier을 받는다!
    // Supplier = "Lexicon을 만들어주는 공장"
    public SpellChecker(Supplier dictionaryFactory) {
        // dictionaryFactory는 아직 사전을 만든 게 아니다.
        // "사전을 만드는 방법"만 들고 있는 상태.

        this.dictionary = dictionaryFactory.get();
        //                                 ↑ .get()을 호출하는 이 순간에 비로소 사전이 만들어진다!
        //                                   "공장아, 사전 하나 만들어줘" → 사전이 생성됨
    }
}

// === 사용하는 쪽 ===
SpellChecker checker = new SpellChecker(() -> new KoreanDictionary());
//                                      ↑ 이건 뭐지? 아래에서 자세히 설명!

// 비유: 붕어빵이 아니라 "붕어빵 틀(레시피)"을 건네주는 것
```

### `() -> new KoreanDictionary()` 이게 대체 뭔데?

이 문법이 제일 낯설 텐데, 단계별로 쪼개보자.

**Step 1. 원래 코드 (풀어쓴 버전)**

```java
// Supplier을 구현하는 "이름 없는 클래스"를 만든 것
Supplier factory = new Supplier() {
    //                       ↑ Supplier 인터페이스를 구현하는 객체를 만든다

    @Override
    public Lexicon get() {
        // ↑ Supplier의 유일한 메서드 get()을 구현한다
        // "get()이 호출되면 KoreanDictionary를 새로 만들어서 돌려줘"라는 뜻

        return new KoreanDictionary();
        // ↑ 이 코드가 실행되는 건 get()이 호출될 때!
        //   factory를 만드는 시점이 아니다!
    }
};

// 이 시점에서 KoreanDictionary는 아직 안 만들어졌다.
// factory는 "만드는 방법"만 알고 있는 상태.

Lexicon dict = factory.get();
//                     ↑ 이제야 비로소 KoreanDictionary가 만들어진다!
```

**Step 2. 람다식 (줄인 버전)**

```java
// 위의 긴 코드를 자바 8 람다식으로 한 줄로 줄인 것:
Supplier factory = () -> new KoreanDictionary();
//                          ^^    ^^^^^^^^^^^^^^^^^^^^^^^
//                          ①              ②
//
// ① () → get() 메서드의 매개변수 (없으니까 빈 괄호)
// ② new KoreanDictionary() → get()이 호출될 때 실행할 코드 (= return할 것)
//
// 풀어서 읽으면:
// "매개변수 없이() 호출하면(→) new KoreanDictionary()를 만들어서 돌려줄게"
```

**Step 3. 왜 같은 건지 나란히 비교**

```java
// === 이 둘은 완전히 같은 코드다 ===

// 긴 버전 (익명 클래스)
Supplier factory = new Supplier() {
    @Override
    public Lexicon get() {        // () ← 매개변수 없음
        return new KoreanDictionary();  // ← 이걸 리턴
    }
};

// 짧은 버전 (람다식)
Supplier factory = () -> new KoreanDictionary();
//                          ()    new KoreanDictionary()
//                     매개변수 없음    이걸 리턴

// 자바가 알아서 "아, Supplier이니까 get() 메서드를 구현하는 거구나" 추론함
```

### 팩터리가 진짜로 동작하는 모습

```java
// "KoreanDictionary를 만드는 레시피"를 factory에 저장
Supplier factory = () -> new KoreanDictionary();

// 아직 사전은 하나도 안 만들어졌다!
// factory는 "만드는 방법"만 들고 있는 상태.

// .get()을 호출할 때마다 새 객체가 만들어진다:
Lexicon dict1 = factory.get();  // 이 순간 new KoreanDictionary() 실행 → 사전 1 탄생
Lexicon dict2 = factory.get();  // 이 순간 new KoreanDictionary() 실행 → 사전 2 탄생
Lexicon dict3 = factory.get();  // 이 순간 new KoreanDictionary() 실행 → 사전 3 탄생

// dict1, dict2, dict3은 각각 다른 별개의 객체다!
// dict1 == dict2 → false
// dict1 == dict3 → false

// 비유: 커피머신 버튼을 누를 때마다 새 커피가 나오는 것처럼!
// factory = 커피머신
// .get()  = 버튼 누르기
// dict    = 나온 커피
```

### 그래서 SpellChecker에 적용하면?

```java
// === 기존 방식 ===
// "완성된 커피 한 잔"을 넘김
Lexicon dict = new KoreanDictionary();         // 사전을 미리 만듦
SpellChecker checker = new SpellChecker(dict); // 만든 걸 넘김
// → SpellChecker는 이 사전 하나만 쓸 수 있음


// === 팩터리 방식 ===
// "커피머신"을 넘김
SpellChecker checker = new SpellChecker(() -> new KoreanDictionary());
//                                      ↑ 사전을 만드는 "방법"을 넘김
//                                        아직 사전은 안 만들어졌음
//                                        SpellChecker 내부에서 .get() 할 때 만들어짐

// → SpellChecker가 필요할 때마다 새 사전을 만들 수 있음
// → 사전을 1개 쓸지, 100개 쓸지는 SpellChecker가 결정
```

---

## 🔒 한정적 와일드카드 타입으로 팩터리 제한

`Supplier<T>`를 입력으로 받는 메서드는 일반적으로 **한정적 와일드카드 타입**을 사용해 팩터리의 타입 매개변수를 제한해야 한다. 이 방식을 사용해 클라이언트는 자신이 명시한 타입의 하위 타입이라면 무엇이든 생성할 수 있는 팩터리를 넘길 수 있다.

### 상황 설정

```java
class Tile { }                          // 부모: 기본 타일
class GlassTile extends Tile { }        // 자식: 유리 타일
class StoneTile extends Tile { }        // 자식: 돌 타일
class MarbleTile extends Tile { }       // 자식: 대리석 타일
```

### 와일드카드 없이 쓰면?

```java
Mosaic create(Supplier tileFactory) { ... }
```

**정확히 `Tile`만** 받을 수 있다. 자바 제네릭의 특성 때문에 자식 타입은 거부된다.

```java
create(() -> new Tile());        // ✅ OK
create(() -> new GlassTile());   // ❌ 컴파일 에러!
create(() -> new StoneTile());   // ❌ 컴파일 에러!
```

### `? extends Tile`을 쓰면?

```java
Mosaic create(Supplier tileFactory) { ... }
```

**"Tile이거나 Tile의 자식이면 뭐든 OK"**

```java
create(() -> new Tile());        // ✅
create(() -> new GlassTile());   // ✅
create(() -> new StoneTile());   // ✅
create(() -> new MarbleTile());  // ✅
```

### 실제 동작

```java
Mosaic create(Supplier tileFactory) {
    List tiles = new ArrayList<>();
    for (int i = 0; i < 100; i++) {
        tiles.add(tileFactory.get());  // 팩터리에서 타일을 100개 찍어냄
    }
    return new Mosaic(tiles);
}

// 유리 모자이크 만들기
Mosaic glassMosaic = create(() -> new GlassTile());
// → GlassTile 100개로 구성된 모자이크

// 대리석 모자이크 만들기
Mosaic marbleMosaic = create(() -> new MarbleTile());
// → MarbleTile 100개로 구성된 모자이크
```

> **비유:**
> - `Supplier<Tile>` → "정확히 '기본타일' 기계만 받겠습니다" (유리타일 기계, 돌타일 기계는 거부)
> - `Supplier<? extends Tile>` → "타일 종류면 어떤 기계든 받겠습니다" (유리든 돌이든 대리석이든 OK)

---

## 📝 핵심 정리

- 클래스가 내부적으로 하나 이상의 자원에 의존하고, 그 자원이 클래스 동작에 영향을 준다면 → **정적 유틸리티 클래스나 싱글턴으로 구현하지 마라.**
- 이 자원들을 클래스가 직접 만들게 해서도 안 된다.
- 대신 필요한 자원을 (혹은 그 자원을 만들어주는 팩터리를) **생성자에 넘겨줘라.**
- 의존 객체 주입은 클래스의 **유연성, 재사용성, 테스트 용이성**을 크게 개선해준다.