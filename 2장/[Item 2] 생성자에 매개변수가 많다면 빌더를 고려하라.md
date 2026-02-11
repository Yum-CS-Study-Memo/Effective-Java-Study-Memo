**아래 세 개의 코드에 대한 구체적인 설명**

```java
package effectivejava.chapter2.item2.hierarchicalbuilder;
import java.util.*;
// 코드 2-4 계층적으로 설계된 클래스와 잘 어울리는 빌더 패턴 (19쪽)
// 참고: 여기서 사용한 '시뮬레이트한 셀프 타입(simulated self-type)' 관용구는
// 빌더뿐 아니라 임의의 유동적인 계층구조를 허용한다.
public abstract class Pizza {
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }
    final Set<Topping> toppings;
    abstract static class Builder<T extends Builder<T>> {
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);
        public T addTopping(Topping topping) {
            toppings.add(Objects.requireNonNull(topping));
            return self();
        }
        abstract Pizza build();
        // 하위 클래스는 이 메서드를 재정의(overriding)하여
        // "this"를 반환하도록 해야 한다.
        protected abstract T self();
    }
    
    Pizza(Builder<?> builder) {
        toppings = builder.toppings.clone(); // 아이템 50 참조
    }
}
```
```java
package effectivejava.chapter2.item2.hierarchicalbuilder;
import java.util.Objects;
// 코드 2-5 뉴욕 피자 - 계층적 빌더를 활용한 하위 클래스 (20쪽)
public class NyPizza extends Pizza {
    public enum Size { SMALL, MEDIUM, LARGE }
    private final Size size;
    public static class Builder extends Pizza.Builder<Builder> {
        private final Size size;
        public Builder(Size size) {
            this.size = Objects.requireNonNull(size);
        }
        @Override public NyPizza build() {
            return new NyPizza(this);
        }
        @Override protected Builder self() { return this; }
    }
    private NyPizza(Builder builder) {
        super(builder);
        size = builder.size;
    }
    @Override public String toString() {
        return toppings + "로 토핑한 뉴욕 피자";
    }
}
```
```java
package effectivejava.chapter2.item2.hierarchicalbuilder;
// 코드 2-6 칼초네 피자 - 계층적 빌더를 활용한 하위 클래스 (20~21쪽)
public class Calzone extends Pizza {
    private final boolean sauceInside;
    public static class Builder extends Pizza.Builder<Builder> {
        private boolean sauceInside = false; // 기본값
        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }
        @Override public Calzone build() {
            return new Calzone(this);
        }
        @Override protected Builder self() { return this; }
    }
    private Calzone(Builder builder) {
        super(builder);
        sauceInside = builder.sauceInside;
    }
    @Override public String toString() {
        return String.format("%s로 토핑한 칼초네 피자 (소스는 %s에)",
                toppings, sauceInside ? "안" : "바깥");
    }
}
```
---
**0. 이 설명을 읽기 전에 알아야 할 기초 개념들**

**생성자란 무엇인가**

생성자는 객체를 만들 때 자동으로 실행되는 특별한 메서드입니다. 클래스 이름과 똑같은 이름을 가지고, 반환 타입이 없습니다.

```java
public class Dog {
    String name;
    
    // 이게 생성자입니다
    public Dog(String name) {
        this.name = name;
    }
}
```

new Dog("뽀삐")라고 쓰면, Dog 객체가 만들어지면서 생성자가 실행됩니다. 생성자 안에서 this.name = name이 실행되어 뽀삐라는 이름이 저장됩니다.

this는 "지금 만들어지고 있는 이 객체"를 가리킵니다. this.name은 이 객체의 name 변수이고, 그냥 name은 파라미터로 받은 값입니다.

비유하자면: 생성자는 공장에서 제품을 만들 때 "초기 설정"을 하는 단계입니다. 스마트폰을 만들 때 처음 전원을 켜면 언어 설정, 와이파이 설정 등을 하죠? 생성자가 바로 그 역할입니다. **객체가 태어날 때 필요한 초기값들을 넣어주는 것입니다.**

**상속이란 무엇인가**

상속은 기존 클래스의 내용을 물려받아서 새 클래스를 만드는 것입니다.

```java
public class Animal {
    String name;
    
    public void eat() {
        System.out.println(name + "이(가) 먹습니다");
    }
}

public class Dog extends Animal {
    public void bark() {
        System.out.println("멍멍!");
    }
}
```

extends Animal이라고 쓰면 Dog는 Animal을 상속받습니다. Dog는 Animal의 name 변수와 eat() 메서드를 그대로 가지고 있습니다. 직접 안 썼지만 물려받은 것입니다.

```java
Dog dog = new Dog();
dog.name = "뽀삐";
dog.eat();   // "뽀삐이(가) 먹습니다" 출력
dog.bark();  // "멍멍!" 출력
```

Dog는 Animal이 가진 것(name, eat)도 쓸 수 있고, 자기만의 것(bark)도 가질 수 있습니다.

비유하자면: 상속은 부모님에게 재산을 물려받는 것과 같습니다. 부모님이 가진 집, 차를 자식이 그대로 쓸 수 있습니다. 그러면서 자식은 자기만의 물건(예: 자기가 산 노트북)도 가질 수 있습니다. Animal이 부모이고, Dog가 자식입니다. Dog는 Animal의 것을 다 쓸 수 있으면서, bark()라는 자기만의 기능도 가집니다.

**super란 무엇인가**

super는 "부모 클래스"를 가리킵니다. 자식 클래스에서 부모 클래스의 생성자나 메서드를 호출할 때 씁니다.

```java
public class Animal {
    String name;
    
    public Animal(String name) {
        this.name = name;
    }
}

public class Dog extends Animal {
    String breed;  // 품종
    
    public Dog(String name, String breed) {
        super(name);        // 부모(Animal)의 생성자를 호출합니다
        this.breed = breed;
    }
}
```

super(name)은 부모 클래스인 Animal의 생성자 Animal(String name)을 호출합니다. 부모 생성자가 실행되면서 name이 설정됩니다.

왜 이렇게 할까요? Animal에 name을 설정하는 코드가 이미 있습니다. Dog에서 같은 코드를 또 쓸 필요가 없습니다. 부모가 해놓은 걸 그대로 쓰면 됩니다.

```java
Dog dog = new Dog("뽀삐", "시바견");
// 1. super(name) 실행 → Animal 생성자 실행 → this.name = "뽀삐"
// 2. this.breed = "시바견"
```

비유하자면: super는 "부모님한테 부탁하기"입니다. 자식이 집을 지을 때, 기초 공사는 부모님이 이미 잘 아시니까 부모님한테 맡기는 것입니다. super(name)은 "부모님, name 설정하는 건 부모님이 해주세요"라고 부탁하는 것입니다. 부모가 하던 일을 자식이 또 할 필요가 없습니다.

**제네릭 <>이란 무엇인가**

제네릭은 "타입을 나중에 정하겠다"는 뜻입니다.

```java
public class Box<T> {
    T item;
    
    public void put(T item) {
        this.item = item;
    }
    
    public T get() {
        return item;
    }
}
```

T는 아직 정해지지 않은 타입입니다. 나중에 Box를 사용할 때 정합니다.

```java
Box<String> stringBox = new Box<>();   // T가 String이 됩니다
stringBox.put("안녕");
String s = stringBox.get();            // "안녕" 반환

Box<Integer> intBox = new Box<>();     // T가 Integer가 됩니다
intBox.put(123);
Integer n = intBox.get();              // 123 반환
```

Box<String>이라고 쓰면 T 자리에 String이 들어갑니다. 그래서 put은 String을 받고, get은 String을 반환합니다.

Box<Integer>라고 쓰면 T 자리에 Integer가 들어갑니다. 같은 Box 클래스인데, 어떤 타입을 넣느냐에 따라 다르게 동작합니다.

왜 쓸까요? 하나의 클래스로 여러 타입을 다룰 수 있기 때문입니다. StringBox, IntegerBox를 따로 만들 필요가 없습니다.

비유하자면: 제네릭은 "빈칸 채우기"입니다. 시험지에 "____에 알맞은 답을 쓰시오"라는 문제가 있죠? Box<T>에서 T는 그 빈칸입니다. 나중에 Box<String>이라고 쓰면 빈칸에 String을 채운 것입니다. 같은 상자인데, 뭘 넣느냐에 따라 String 상자가 되기도 하고, Integer 상자가 되기도 합니다.

**build() 메서드란 무엇인가**

build()는 특별한 자바 문법이 아닙니다. 그냥 메서드 이름입니다. "만들다"라는 뜻으로 붙인 이름입니다.

**빌더 패턴에서 build()는 "준비 끝났으니 이제 진짜 객체를 만들어라"라는 신호입니다.**

```java
// Builder로 설정을 모읍니다
NyPizza.Builder builder = new NyPizza.Builder(LARGE);
builder.addTopping(HAM);
builder.addTopping(ONION);

// build()를 호출하면 진짜 피자가 만들어집니다
NyPizza pizza = builder.build();
```

build() 안에서는 `new NyPizza(this)`를 실행합니다. 드디어 NyPizza 생성자가 호출되어 피자 객체가 만들어지는 것입니다.

**비유하자면:** build()는 "주문 완료" 버튼입니다. 배달 앱에서 피자를 고르고, 토핑을 선택하고, 사이즈를 정한 다음에 마지막으로 "주문하기" 버튼을 누르죠? 그 버튼을 누르기 전까지는 장바구니에 담긴 것일 뿐이고, 버튼을 눌러야 진짜 주문이 됩니다. build()가 바로 그 "주문하기" 버튼입니다.

---

```java
// 패키지 선언: 이 파일이 속한 폴더 경로
package effectivejava.chapter2.item2.hierarchicalbuilder;

// java.util 패키지의 모든 클래스를 가져옴 (Set, EnumSet, Objects 등)
import java.util.*;

// Pizza 클래스 선언. abstract라서 직접 new Pizza()로 못 만듦.
// "피자"라는 개념만 정의하고, 구체적인 피자(뉴욕, 칼초네)는 자식이 만듦.
public abstract class Pizza {

    // 가능한 토핑 5가지를 enum으로 고정. 이 5개 외에는 토핑 불가.
    public enum Topping { HAM, MUSHROOM, ONION, PEPPER, SAUSAGE }

    // 이 피자에 올라간 토핑들을 저장하는 변수.
    // final이라 피자가 만들어진 후에는 다른 Set으로 교체 불가.
    final Set<Topping> toppings;

    // ─── 여기서부터 Builder (주문서) ───

    // Pizza 안에 있는 내부 클래스. 피자를 만들기 위한 "주문서" 역할.
    // abstract: 이 주문서만으로는 어떤 피자인지 모르니까 직접 사용 불가.
    // static: 피자 객체 없이도 주문서를 만들 수 있게 함.
    // <T extends Builder<T>>: T는 "이 주문서를 상속한 자식 주문서 타입".
    //   → addTopping 후 자식 타입 그대로 반환하기 위한 장치.
    abstract static class Builder<T extends Builder<T>> {

        // 주문서에 적힌 토핑 목록. 처음엔 빈 상태.
        // EnumSet.noneOf(Topping.class) = "Topping 타입의 빈 집합"
        EnumSet<Topping> toppings = EnumSet.noneOf(Topping.class);

        // 토핑 하나를 추가하는 메서드. 반환 타입이 T(자식 빌더 타입).
        public T addTopping(Topping topping) {

            // null 토핑이 들어오면 즉시 에러. null 방지용 안전장치.
            toppings.add(Objects.requireNonNull(topping));

            // self()를 호출해서 "진짜 자기 자신"을 반환.
            // → 이래야 .addTopping(HAM).addTopping(ONION) 체이닝 가능.
            // → 그리고 반환 타입이 부모가 아닌 자식 타입으로 유지됨.
            return self();
        }

        // "주문 완료" 버튼. 호출하면 진짜 Pizza 객체가 만들어짐.
        // abstract: 어떤 피자를 만들지는 자식(NyPizza.Builder 등)이 결정.
        abstract Pizza build();

        // "너 진짜 누구야?" 메서드.
        // 자식 클래스가 이걸 구현해서 return this 하면,
        // 그때 this는 자식 타입(예: NyPizza.Builder)이 됨.
        // → addTopping이 부모에 한 번만 있어도 자식 타입 반환 가능.
        protected abstract T self();
    }

    // ─── 여기서부터 Pizza 생성자 ───

    // Pizza의 생성자. Builder(주문서)를 받아서 피자를 완성함.
    // Builder<?>: 어떤 자식 빌더든 다 받겠다는 뜻. (? = 와일드카드)
    Pizza(Builder<?> builder) {

        // 빌더의 토핑을 복사해서 저장.
        // clone(): 복사본을 만듦.
        // → 복사 안 하면 빌더와 피자가 같은 토핑 목록을 공유해서,
        //   나중에 빌더를 수정하면 이미 만든 피자도 바뀌는 문제 발생.
        toppings = builder.toppings.clone();
    }
}
```

**1. 계층적 빌더 패턴 설명**

**먼저, 이 코드가 해결하려는 문제부터**

피자 가게를 비유해봅시다. 피자에는 공통점이 있습니다. 모든 피자는 토핑을 올릴 수 있습니다. 하지만 피자 종류마다 고유한 특성도 있습니다. 뉴욕 피자는 크기를 선택해야 하고, 칼초네 피자는 소스를 안에 넣을지 바깥에 넣을지 선택해야 합니다.

이걸 코드로 만들 때 일반적인 생성자로 만들면 문제가 생깁니다.:

```java
NyPizza pizza = new NyPizza(LARGE, HAM, MUSHROOM, ONION);
```

파라미터가 많아지면 어떤 게 크기고 어떤 게 토핑인지 헷갈립니다. 그리고 토핑을 2개만 넣고 싶을 때, 3개 넣고 싶을 때, 5개 넣고 싶을 때마다 생성자를 따로 만들어야 합니다. 이건 비효율적이죠.

그래서 빌더 패턴을 씁니다. 빌더 패턴은 객체를 한 번에 만들지 않고, **단계별로 설정한 다음 마지막에 완성하는 방식**입니다.

---

**2. Pizza 클래스가 하는 일**

Pizza 클래스는 모든 피자의 공통 부분을 정의합니다. 이 클래스는 abstract로 선언되어 있어서 직접 피자 객체를 만들 수 없습니다. 왜냐하면 "그냥 피자"라는 건 존재하지 않기 때문입니다. 뉴욕 피자, 칼초네 피자처럼 구체적인 종류의 피자만 존재합니다.

Pizza 클래스 안에는 Topping이라는 enum이 있습니다. enum은 가능한 값들을 미리 정해놓은 타입입니다. 여기서는 HAM, MUSHROOM, ONION, PEPPER, SAUSAGE 다섯 가지 토핑만 가능하다고 정해놓았습니다. 다른 토핑은 넣을 수 없습니다.

그리고 toppings라는 변수가 있습니다. 이건 Set 타입이고, Set은 중복을 허용하지 않는 집합입니다. 같은 토핑을 두 번 추가해도 하나만 저장됩니다. final로 선언되어 있어서 한 번 피자가 만들어지면 토핑 집합 자체를 다른 걸로 바꿀 수 없습니다.

---

**3. Pizza.Builder가 하는 일**

비유: Builder는 "주문서"입니다. 피자를 바로 만들지 않고, 주문서에 원하는 것들을 적어놓습니다.

static으로 선언된 이유: static이 없으면 주문서를 만들기 전에 피자가 먼저 있어야 합니다. 근데 주문서의 목적이 피자를 만드는 것입니다. 피자가 있어야 주문서를 만들고, 주문서가 있어야 피자를 만든다? 이건 "닭이 먼저냐 달걀이 먼저냐" 같은 모순입니다. static으로 선언하면 피자 없이도 주문서를 만들 수 있습니다.

abstract인 이유: Pizza.Builder만으로는 어떤 피자를 만들지 모릅니다. 뉴욕 피자? 칼초네? 구체적인 종류는 하위 클래스에서 정합니다.
EnumSet.noneOf(Topping.class)는 **"아직 토핑이 하나도 없는 빈 주문서"**를 만드는 것입니다.

---

**4. 제네릭 `<T extends Builder<T>>`의 의미**

비유: 주문서에 "토핑 추가"를 적으면, 그 주문서를 다시 돌려받아서 다른 것도 적을 수 있어야 합니다.

근데 문제가 있습니다. 뉴욕 피자 주문서에 토핑을 추가했는데, 돌려받은 게 "뉴욕 피자 주문서가 아니라, **그냥 피자 주문서**"면 어떡할까요? 뉴욕 피자 주문서에만 있는 "사이즈 선택" 칸을 못 씁니다.

```java
// 제네릭이 없으면 이런 문제가 생깁니다
NyPizza.Builder builder = new NyPizza.Builder(LARGE);
builder.addTopping(HAM);  // 반환 타입: Pizza.Builder (부모 타입)
// 이제 NyPizza.Builder의 기능을 쓸 수 없습니다!
```
제네릭 **<T extends Builder<T>>**는 이 문제를 해결합니다. **"T는 Builder를 상속한 타입이어야 하고, 메서드가 T 타입을 반환한다"**는 뜻입니다.

NyPizza.Builder에서 T는 NyPizza.Builder가 됩니다. 그래서 addTopping을 호출해도 NyPizza.Builder가 반환됩니다. 뉴욕 피자 주문서에 토핑을 추가하면, 뉴욕 피자 주문서가 그대로 돌아오는 것입니다.

이렇게 복잡하게 사용하는 이유는 addTopping 메서드 때문입니다. addTopping 메서드를 보면 반환 타입이 T입니다. **토핑을 추가하고 나서 자기 자신을 반환해서 메서드를 연속으로 호출할 수 있게 하려는 것**입니다.

다시 한 번 설명하지만, 만약 제네릭 없이 그냥 Builder를 반환하면 문제가 생깁니다. NyPizza.Builder에서 addTopping을 호출하면 Pizza.Builder 타입이 반환됩니다. 그러면 NyPizza.Builder에만 있는 메서드를 그 다음에 호출할 수 없습니다. 컴파일러가 보기에는 Pizza.Builder 타입이기 때문입니다.

제네릭을 쓰면 NyPizza.Builder에서 T가 NyPizza.Builder가 됩니다. addTopping을 호출해도 NyPizza.Builder 타입이 반환됩니다. 그래서 NyPizza.Builder의 메서드를 계속 호출할 수 있습니다.

---

**5. addTopping 메서드가 하는 일**

addTopping 메서드는 토핑 하나를 받아서 toppings 집합에 추가합니다. Objects.requireNonNull은 null이 들어오면 즉시 에러를 발생시킵니다. **null 토핑은 허용하지 않겠다는 것**입니다.

그리고 return self()를 합니다. self()가 뭔지는 아래에서 설명하겠습니다. 중요한 건 자기 자신을 반환한다는 것입니다. 그래서 아래와 같이 연속으로 호출할 수 있습니다:

```java
builder.addTopping(HAM).addTopping(ONION).addTopping(PEPPER)
```

addTopping이 Builder를 반환하니까, 반환된 Builder에서 다시 addTopping을 호출할 수 있습니다. 이걸 **메서드 체이닝**이라고 합니다.

---

**6. self() 메서드가 필요한 이유**

비유를 더 자세히: 피자 가게에 "기본 주문서"와 "뉴욕 피자 주문서"가 있습니다.

기본 주문서에 "토핑 추가" 기능이 있습니다. 토핑을 추가하면 주문서를 돌려줍니다.

근데 기본 주문서 입장에서는 자기가 어떤 주문서인지 모릅니다. 그냥 "기본 주문서요~"라고 돌려줍니다. 실제로는 뉴욕 피자 주문서였는데 말이죠.

self()는 이 문제를 해결합니다. "너 진짜 뭔지 말해봐"라고 물어보는 것입니다.

뉴욕 피자 주문서의 self()는 return this를 합니다. 여기서 this는 "나, 뉴욕 피자 주문서"입니다.

칼초네 주문서의 self()는 return this를 합니다. 여기서 this는 "나, 칼초네 주문서"입니다.

이렇게 하면 addTopping은 부모(Pizza.Builder)에 한 번만 만들어도, 실제로 반환되는 건 진짜 자기 자신의 타입입니다.

addTopping에서 return this를 하면 안 됩니다. Pizza.Builder 안에서 this는 Pizza.Builder 타입입니다. 그런데 우리가 원하는 건 NyPizza.Builder에서 호출했을 때 NyPizza.Builder 타입이 반환되는 것입니다.

자바에서는 this의 타입이 항상 현재 클래스입니다. 하위 클래스에서 호출해도 this를 쓰면 **선언된 클래스의 타입으로** 인식됩니다.

그래서 self()라는 추상 메서드를 만든 것입니다. **하위 클래스에서 self()를 구현할 때 return this를 하면, 그때의 this는 하위 클래스 타입입니다.** NyPizza.Builder에서 self()를 구현하면 this는 NyPizza.Builder 타입이 되는 것입니다.

이렇게 하면 addTopping은 부모 클래스에 한 번만 작성하면서도, 호출했을 때 정확한 하위 클래스 타입을 반환할 수 있습니다.

---

**7. build() 메서드가 하는 일**

build()는 추상 메서드입니다. 이 메서드가 호출되면 실제 피자 객체가 만들어집니다. 하위 클래스에서 구현해야 합니다.

비유: "주문 완료" 버튼입니다.

주문서에 다 적었으면 build()를 호출합니다. 그러면 진짜 피자가 만들어집니다.

Pizza.Builder에서는 abstract입니다. 왜냐하면 "어떤 피자를 만들지"는 하위 클래스(NyPizza.Builder, Calzone.Builder)에서 정해야 하기 때문입니다.

---

**8. Pizza 생성자가 하는 일**

비유: 주방에서 주문서를 받아서 피자를 만드는 과정입니다.

Builder<?>에서 ?는 "아무 종류의 주문서든 받겠다"는 뜻입니다. 뉴욕 피자 주문서든, 칼초네 주문서든 상관없습니다.

builder.toppings.clone()은 주문서에 적힌 토핑 목록을 복사해서 피자에 저장하는 것입니다.

왜 복사할까요? 복사 안 하면 주문서와 피자가 같은 토핑 목록을 공유합니다. 피자를 만든 후에 주문서의 토핑을 바꾸면, 이미 만들어진 피자의 토핑도 바뀝니다. 이건 이상합니다. 복사해서 각자 독립적인 목록을 가지게 합니다.

---

```java
package effectivejava.chapter2.item2.hierarchicalbuilder;
import java.util.Objects;

// NyPizza 클래스 선언. Pizza를 상속받음.
// Pizza가 "모든 피자의 공통 개념"이었다면,
// NyPizza는 "뉴욕 스타일 피자"라는 구체적인 피자.
// Pizza가 abstract라서 직접 못 만들었지만, NyPizza는 만들 수 있음.
public class NyPizza extends Pizza {

    // 뉴욕 피자만의 고유 속성: 크기.
    // Pizza에는 없는 것. 뉴욕 피자만 사이즈를 선택해야 함.
    // enum이라 SMALL, MEDIUM, LARGE 세 가지만 가능. 다른 값은 불가.
    public enum Size { SMALL, MEDIUM, LARGE }

    // 이 뉴욕 피자의 크기를 저장하는 변수.
    // private: 클래스 바깥에서 직접 접근 불가. nyPizza.size 하면 에러.
    // final: 한 번 정해지면 변경 불가. LARGE로 만들면 영원히 LARGE.
    // → Pizza의 toppings도 final이었음. 같은 이유: 불변 객체를 만들기 위해.
    private final Size size;

    // ─── 여기서부터 NyPizza.Builder (뉴욕 피자 전용 주문서) ───

    // Pizza.Builder가 "기본 주문서"였다면,
    // NyPizza.Builder는 "뉴욕 피자 전용 주문서".
    // 기본 주문서의 토핑 추가 기능을 물려받고, 사이즈 선택 기능을 추가함.
    //
    // extends Pizza.Builder<Builder>에서:
    //   Pizza.Builder<T extends Builder<T>>의 T 자리에
    //   NyPizza.Builder(여기서는 그냥 Builder)가 들어감.
    //   → Pizza.Builder의 addTopping이 반환하는 T가 NyPizza.Builder가 됨.
    //   → 토핑 추가 후에도 뉴욕 피자 주문서가 그대로 돌아옴.
    //
    // static인 이유: Pizza.Builder와 동일.
    //   NyPizza 객체 없이도 주문서를 만들 수 있어야 하니까.
    //   주문서의 목적이 NyPizza를 만드는 건데,
    //   NyPizza가 있어야 주문서를 만들 수 있다면 모순.
    public static class Builder extends Pizza.Builder<Builder> {

        // 주문서에 적힌 사이즈. NyPizza의 size와는 다른 변수.
        // NyPizza.size = 완성된 피자의 크기 (최종 결과물)
        // Builder.size = 주문서에 적힌 크기 (임시 저장)
        // → build() 호출 시 Builder.size → NyPizza.size로 전달됨.
        // final이라 주문서 작성 후 사이즈 변경 불가.
        private final Size size;

        // 주문서 생성자. 반드시 사이즈를 지정해야 함.
        // new NyPizza.Builder() → 에러! 사이즈 없는 뉴욕 피자는 존재 불가.
        // new NyPizza.Builder(LARGE) → OK!
        //
        // 비교: Pizza.Builder에는 생성자에 필수 파라미터가 없었음.
        //   → 토핑은 선택사항이니까. (토핑 없는 피자도 가능)
        //   → 하지만 사이즈는 필수. 사이즈 없는 뉴욕 피자는 불가능.
        //   → 그래서 생성자에서 강제하는 것.
        public Builder(Size size) {

            // Pizza.Builder의 addTopping에서 봤던 것과 같은 패턴.
            // null이면 즉시 NullPointerException 발생.
            // null 사이즈가 들어오면 나중에 문제가 커지기 전에 여기서 바로 잡음.
            this.size = Objects.requireNonNull(size);

            // this.size = Builder의 멤버 변수 size
            // size = 파라미터로 받은 size
            // 이름이 같아서 this로 구분함.
        }

        // "주문 완료" 버튼. Pizza.Builder의 abstract Pizza build()를 구현함.
        //
        // 반환 타입 주목: Pizza가 아니라 NyPizza.
        // 부모(Pizza.Builder)에서는 Pizza를 반환한다고 했는데,
        // 자식(NyPizza.Builder)에서는 더 구체적인 NyPizza를 반환.
        // → 이것을 "공변 반환 타이핑(covariant return typing)"이라 함.
        // → NyPizza는 Pizza의 하위 타입이니까 가능.
        //
        // new NyPizza(this)에서 this = 현재 Builder 객체.
        // Builder 안에 저장해둔 size와 toppings를 NyPizza 생성자에 넘김.
        @Override public NyPizza build() {
            return new NyPizza(this);
        }

        // Pizza.Builder의 protected abstract T self()를 구현.
        //
        // 복습: Pizza.Builder의 addTopping은 return self()를 함.
        //   addTopping은 부모(Pizza.Builder)에 딱 한 번만 정의됨.
        //   근데 NyPizza.Builder에서 addTopping을 호출하면
        //   NyPizza.Builder 타입이 반환되어야 함.
        //
        //   addTopping 안에서 return this를 하면?
        //   → this는 Pizza.Builder 타입으로 인식됨. (선언된 클래스 기준)
        //   → NyPizza.Builder의 고유 기능을 이어서 쓸 수 없음.
        //
        //   그래서 self()를 만들어서 하위 클래스에서 return this를 하게 함.
        //   여기서 this는 NyPizza.Builder 타입.
        //   → addTopping이 NyPizza.Builder를 반환하게 됨.
        //   → .addTopping(HAM).addTopping(ONION) 체이닝 가능!
        @Override protected Builder self() { return this; }
    }

    // ─── 여기서부터 NyPizza 생성자 ───

    // private: 클래스 바깥에서 직접 호출 불가.
    //   new NyPizza(builder) → 에러!
    //   오직 NyPizza 내부(= Builder.build())에서만 호출 가능.
    //   → 반드시 Builder를 거쳐야만 피자를 만들 수 있게 강제.
    //   → Builder를 통해야 필수값(size) 검증, 토핑 추가 등의 과정을 거침.
    private NyPizza(Builder builder) {

        // 부모 클래스(Pizza)의 생성자를 호출.
        //
        // Pizza 생성자를 다시 보면:
        //   Pizza(Builder<?> builder) {
        //       toppings = builder.toppings.clone();
        //   }
        //
        // NyPizza.Builder는 Pizza.Builder를 상속받았으므로
        // Builder<?> 타입에 해당함. (? = 와일드카드, 아무 자식이든 OK)
        //
        // 이 호출로 builder에 저장된 toppings가 복사되어 Pizza.toppings에 저장됨.
        // clone()으로 복사하는 이유는 Pizza 코드에서 설명한 것과 동일:
        //   → 주문서와 피자가 같은 토핑 목록을 공유하면,
        //     주문서 수정 시 이미 만든 피자도 바뀌는 문제 발생.
        //
        // 자바에서 자식 생성자는 반드시 부모 생성자를 호출해야 함.
        // Pizza에는 파라미터 없는 생성자가 없으므로
        // 명시적으로 super(builder)를 써야 함. 안 쓰면 컴파일 에러.
        super(builder);

        // 부모 생성자에서 공통 부분(toppings)을 처리한 후,
        // NyPizza만의 고유 속성인 size를 설정.
        // builder.size에 저장되어 있던 값 → NyPizza의 size 변수로.
        //
        // 이 시점에서 NyPizza 완성:
        //   toppings = 부모 생성자에서 설정됨 (final)
        //   size = 여기서 설정됨 (final)
        //   둘 다 final이므로 이후 변경 불가. 완전한 불변 객체.
        size = builder.size;
    }

    // 피자 정보를 문자열로 표현. System.out.println(pizza) 하면 이게 호출됨.
    // toppings는 부모(Pizza)에서 물려받은 변수. 여기서 직접 접근 가능.
    // (Pizza에서 toppings가 private이 아니라 package-private(기본 접근제어자)이기 때문)
    @Override public String toString() {
        return toppings + "로 토핑한 뉴욕 피자";
    }
}
```

---

**9. NyPizza 클래스가 하는 일**

NyPizza는 Pizza를 상속받습니다. `extends Pizza`라고 선언했기 때문입니다. 상속받는다는 것은 부모 클래스인 Pizza가 가진 모든 것을 물려받는다는 뜻입니다. 그래서 NyPizza는 **Pizza의 toppings 변수**를 가지고 있고, **Pizza.Builder의 addTopping 메서드**도 사용할 수 있습니다.

그런데 NyPizza는 Pizza에는 없는 고유한 특성이 있습니다. 바로 **크기(size)**입니다. 뉴욕 피자는 주문할 때 크기를 선택해야 합니다.

`public enum Size { SMALL, MEDIUM, LARGE }`는 NyPizza 안에 정의된 열거형입니다. Pizza 안에 `enum Topping`이 있었던 것과 같은 구조입니다. Pizza.Topping은 "어떤 토핑을 올릴 수 있는가"를 정의했고, NyPizza.Size는 "어떤 크기를 선택할 수 있는가"를 정의합니다. 가능한 값은 SMALL, MEDIUM, LARGE 세 가지뿐입니다. 다른 값은 존재할 수 없습니다. EXTRA_LARGE라는 값을 쓰려고 하면 컴파일 에러가 납니다.

`private final Size size`는 NyPizza 객체가 가지는 크기 정보를 저장하는 변수입니다. **Pizza의 `final Set<Topping> toppings`와 같은 설계 의도**입니다. Pizza에서 toppings를 final로 선언해서 피자가 만들어진 후 토핑 집합을 교체할 수 없게 한 것처럼, NyPizza에서도 size를 final로 선언해서 크기를 바꿀 수 없게 합니다.

추가로 private이라서 NyPizza 클래스 바깥에서는 이 변수에 직접 접근할 수 없습니다. 다른 클래스에서 `nyPizza.size`라고 쓰면 컴파일 에러가 납니다. (참고로 Pizza의 toppings는 private이 아니라 접근제어자가 없는 package-private입니다. 같은 패키지 안에서는 접근할 수 있습니다.)

왜 private final로 선언할까요? 피자가 한 번 만들어지면 그 피자의 크기가 바뀌면 안 되기 때문입니다. 큰 피자를 주문했는데 나중에 누군가 코드에서 작은 피자로 바꿔버리면 문제가 생깁니다. **불변(immutable) 객체**로 만들어서 이런 문제를 원천 차단하는 것입니다. Pizza의 toppings도 같은 이유로 final이었습니다.

---

**10. NyPizza.Builder가 하는 일**

비유: NyPizza.Builder는 **"뉴욕 피자 전용 주문서"**입니다. Pizza.Builder가 "기본 주문서"였다면, 이것은 기본 주문서의 토핑 추가 기능을 물려받고, **사이즈 선택 기능을 추가한 확장 주문서**입니다.

**클래스 선언부 분석**

`public static class Builder extends Pizza.Builder<Builder>`

이 한 줄에 많은 정보가 담겨 있습니다. 하나씩 뜯어보겠습니다.

`public`은 어디서든 접근 가능하다는 뜻입니다. 다른 패키지에서도 NyPizza.Builder를 사용할 수 있습니다.

`static`은 NyPizza 객체 없이도 Builder를 만들 수 있다는 뜻입니다. **Pizza.Builder가 static인 이유와 동일합니다.** static이 없으면 Builder를 만들기 전에 NyPizza 객체가 먼저 있어야 합니다. 그런데 Builder의 목적이 NyPizza를 만드는 것입니다. NyPizza가 있어야 Builder를 만들고, Builder가 있어야 NyPizza를 만든다면 서로가 서로를 필요로 하는 모순이 생깁니다. static으로 선언해서 이 모순을 해결합니다.

`class Builder`는 이 클래스의 이름이 Builder라는 뜻입니다. 전체 이름은 NyPizza.Builder입니다. NyPizza 클래스 안에 정의되어 있기 때문입니다.

`extends Pizza.Builder<Builder>`는 **Pizza.Builder를 상속받는다는 뜻**입니다. 여기가 핵심입니다. Pizza.Builder의 제네릭 선언을 다시 보면:

```java
// Pizza.java에서:
abstract static class Builder<T extends Builder<T>> { ... }
```

`T extends Builder<T>`에서 T 자리에 NyPizza.Builder(여기서는 그냥 Builder라고 씀)가 들어갑니다. 그러면 Pizza.Builder의 addTopping 메서드에서:

```java
// Pizza.Builder의 addTopping:
public T addTopping(Topping topping) {
    toppings.add(Objects.requireNonNull(topping));
    return self();  // T 타입 반환
}
```

T가 NyPizza.Builder가 되므로, **addTopping의 반환 타입이 NyPizza.Builder**가 됩니다. 그래서 뉴욕 피자 주문서에 토핑을 추가해도, 돌아오는 건 여전히 뉴욕 피자 주문서입니다. 사이즈 선택 등 뉴욕 피자 주문서만의 기능을 이어서 쓸 수 있습니다.

**size 변수**

`private final Size size`

NyPizza.Builder 안에도 size 변수가 있습니다. NyPizza 클래스에도 size가 있고, NyPizza.Builder에도 size가 있습니다. **이 둘은 서로 다른 변수**입니다.

이것은 Pizza.Builder의 toppings와 Pizza의 toppings 관계와 같습니다:
- **Pizza.Builder.toppings** = 주문서에 적힌 토핑 목록 (임시 저장)
- **Pizza.toppings** = 완성된 피자의 토핑 (최종 결과물)
- build() 시 Builder.toppings → Pizza.toppings로 **복사(clone)**되어 전달

마찬가지로:
- **NyPizza.Builder.size** = 주문서에 적힌 크기 (임시 저장)
- **NyPizza.size** = 완성된 피자의 크기 (최종 결과물)
- build() 시 Builder.size → NyPizza.size로 전달

**생성자**

```java
public Builder(Size size) {
    this.size = Objects.requireNonNull(size);
}
```

Builder를 만들 때 **반드시 Size를 지정해야 합니다.** 파라미터가 있는 생성자만 있기 때문입니다. `new NyPizza.Builder()`라고 쓰면 컴파일 에러가 납니다. 반드시 `new NyPizza.Builder(LARGE)`처럼 크기를 지정해야 합니다.

이것은 **Pizza.Builder와의 차이점**입니다. Pizza.Builder는 생성자에 필수 파라미터가 없었습니다. 토핑은 선택사항이니까요. 토핑 없는 피자도 가능합니다. 하지만 **사이즈 없는 뉴욕 피자는 존재할 수 없습니다.** 그래서 생성자에서 강제하는 것입니다. 이것이 빌더 패턴에서 **필수값과 선택값을 구분하는 방법**입니다. 필수값은 Builder 생성자에 넣고, 선택값은 addXxx() 메서드로 추가합니다.

`Objects.requireNonNull(size)`는 **Pizza.Builder의 addTopping에서 봤던 것과 같은 패턴**입니다. null이 들어오면 즉시 NullPointerException을 던집니다. null이 들어오면 나중에 문제가 생기는 것보다 여기서 바로 에러를 내는 것이 낫습니다. 문제가 발생한 지점에서 바로 에러가 나야 디버깅이 쉽습니다.

`this.size = ...`에서 this.size는 Builder 클래스의 멤버 변수 size를 가리킵니다. 그냥 size는 파라미터로 받은 size입니다. 이름이 같아서 this로 구분하는 것입니다.

**build() 메서드**

```java
@Override public NyPizza build() {
    return new NyPizza(this);
}
```

`@Override`는 **부모 클래스의 메서드를 재정의한다는 표시**입니다. Pizza.Builder에 이렇게 선언되어 있었습니다:

```java
// Pizza.Builder에서:
abstract Pizza build();
```

NyPizza.Builder는 이 추상 메서드를 구현해야 합니다.

반환 타입이 Pizza가 아니라 **NyPizza**입니다. 부모에서는 Pizza를 반환한다고 했는데 자식에서는 NyPizza를 반환합니다. 이것이 가능한 이유는 NyPizza가 Pizza의 하위 타입이기 때문입니다. 자바에서는 오버라이드할 때 반환 타입을 더 구체적인 하위 타입으로 바꿀 수 있습니다. 이것을 **공변 반환 타이핑(covariant return typing)**이라고 합니다.

`return new NyPizza(this)`에서 this는 **현재 Builder 객체**입니다. Builder 자신을 NyPizza 생성자에 넘겨서 피자를 만듭니다. Builder 안에 저장해둔 size와 toppings(Pizza.Builder에서 물려받은) 정보가 NyPizza로 전달됩니다.

**self() 메서드**

```java
@Override protected Builder self() { return this; }
```

**Pizza.Builder의 self()를 구현하는 부분**입니다. Pizza.Builder에 이렇게 선언되어 있었습니다:

```java
// Pizza.Builder에서:
protected abstract T self();
```

Pizza.Builder의 addTopping에서 `return self()`를 했던 것을 기억하세요. addTopping은 Pizza.Builder에 딱 한 번만 정의되어 있습니다. 이 메서드 안에서 `return this`를 하면, this는 Pizza.Builder 타입으로 인식됩니다. 그러면 NyPizza.Builder에서 addTopping을 호출했을 때, 반환되는 게 Pizza.Builder 타입이 되어서 NyPizza.Builder의 고유 기능을 이어서 쓸 수 없습니다.

그래서 self()라는 우회 장치를 만든 것입니다. NyPizza.Builder에서 self()를 구현할 때 `return this`를 하면, **여기서의 this는 NyPizza.Builder 타입**입니다. 그러면 addTopping이 NyPizza.Builder 타입을 반환하게 되고, 아래처럼 체이닝이 가능해집니다:

```java
new NyPizza.Builder(LARGE)
    .addTopping(HAM)       // addTopping → self() → NyPizza.Builder 반환
    .addTopping(ONION)     // 여전히 NyPizza.Builder이므로 계속 체이닝 가능
    .build();              // NyPizza.Builder의 build() 호출 → NyPizza 완성
```

---

**11. NyPizza 생성자가 하는 일**

```java
private NyPizza(Builder builder) {
    super(builder);
    size = builder.size;
}
```

**private 생성자**

생성자가 private입니다. private 생성자는 클래스 바깥에서 호출할 수 없습니다.

```java
// 이렇게 쓰면 컴파일 에러
NyPizza pizza = new NyPizza(builder);
```

오직 NyPizza 클래스 안에서만 이 생성자를 호출할 수 있습니다. NyPizza.Builder의 build() 메서드가 NyPizza 클래스 안에 있으므로 거기서는 호출할 수 있습니다.

왜 이렇게 할까요? **피자를 만들 때 반드시 Builder를 거치도록 강제하기 위해서**입니다. Builder를 통하지 않고 직접 피자를 만드는 것을 막는 것입니다. Builder를 통해야만 필수값(size) 검증, null 검사, 토핑 추가 등의 과정을 거칠 수 있습니다.

**super(builder) 호출**

`super(builder)`는 **부모 클래스인 Pizza의 생성자를 호출**합니다. Pizza 생성자를 다시 보면:

```java
// Pizza.java에서:
Pizza(Builder<?> builder) {
    toppings = builder.toppings.clone();
}
```

NyPizza.Builder도 Pizza.Builder를 상속받았으므로 `Builder<?>` 타입에 해당합니다. (`?`는 와일드카드로, "아무 종류의 자식 빌더든 받겠다"는 뜻입니다. Pizza 코드에서 설명한 것과 동일합니다.)

Pizza 생성자가 실행되면 **builder에 저장된 toppings를 clone()으로 복사**해서 Pizza의 toppings에 저장합니다. clone()으로 복사하는 이유는 Pizza 코드에서 설명한 것과 같습니다. 복사 안 하면 주문서와 피자가 같은 토핑 목록을 공유해서, 나중에 주문서를 수정하면 이미 만든 피자도 바뀌는 문제가 발생합니다.

비유: 뉴욕 피자를 만들 때, 먼저 "기본 피자 만들기"를 합니다. 도우를 펴고, 토핑을 올리는 건 모든 피자가 다 하는 일입니다. 이 부분은 부모(Pizza)에게 맡깁니다. `super(builder)`가 "부모님, 기본 피자 부분은 맡아주세요"라고 부탁하는 것입니다.

자바에서 자식 클래스의 생성자는 반드시 부모 클래스의 생성자를 호출해야 합니다. 명시적으로 super()를 쓰지 않으면 컴파일러가 자동으로 `super()`(파라미터 없는 버전)를 넣습니다. 그런데 Pizza에는 파라미터 없는 생성자가 없습니다. `Pizza(Builder<?> builder)`만 있습니다. 그래서 명시적으로 `super(builder)`를 써야 합니다. 안 쓰면 컴파일 에러입니다.

**size 설정**

`size = builder.size`

부모 생성자 호출이 끝난 후, **NyPizza 고유의 속성인 size를 설정**합니다. builder.size에 저장되어 있던 값을 NyPizza의 size 변수에 저장합니다.

이 시점에서 NyPizza 객체는 완성됩니다:
- **toppings** = 부모 생성자(super)에서 설정됨 (final)
- **size** = 여기서 설정됨 (final)
- 둘 다 final이므로 이후로는 절대 바뀌지 않습니다. **완전한 불변 객체**입니다.

---

**전체 흐름 정리: Pizza 코드 → NyPizza 코드가 연결되는 과정**

```java
NyPizza pizza = new NyPizza.Builder(LARGE)  // 1단계
    .addTopping(HAM)                         // 2단계
    .addTopping(ONION)                       // 3단계
    .build();                                // 4단계
```

**1단계: `new NyPizza.Builder(LARGE)`**
- NyPizza.Builder 생성자 호출
- `Objects.requireNonNull(LARGE)` → null 아니므로 통과
- Builder.size = LARGE로 설정
- 동시에 부모(Pizza.Builder)의 toppings = 빈 EnumSet으로 초기화됨

**2단계: `.addTopping(HAM)`**
- **Pizza.Builder에 정의된** addTopping 호출 (NyPizza.Builder에는 없음. 상속받은 것)
- `Objects.requireNonNull(HAM)` → null 아니므로 통과
- toppings에 HAM 추가
- `return self()` → NyPizza.Builder의 self() 호출 → `return this` → **NyPizza.Builder 반환**

**3단계: `.addTopping(ONION)`**
- 2단계와 동일한 과정. toppings에 ONION 추가
- 여전히 NyPizza.Builder 타입이 반환됨

**4단계: `.build()`**
- NyPizza.Builder의 build() 호출
- `return new NyPizza(this)` → NyPizza 생성자 호출
  - `super(builder)` → **Pizza 생성자 호출** → `toppings = builder.toppings.clone()` → {HAM, ONION} 복사
  - `size = builder.size` → LARGE 설정
- 완성된 NyPizza 반환: toppings = {HAM, ONION}, size = LARGE

---

```java
package effectivejava.chapter2.item2.hierarchicalbuilder;

// Calzone 클래스 선언. Pizza를 상속받음.
// NyPizza가 "뉴욕 스타일 피자"였다면,
// Calzone은 "반으로 접은 이탈리아 피자".
// Pizza를 상속받았으므로 toppings(토핑)는 공통으로 가지고 있음.
// 고유 속성: 소스를 안에 넣을지 바깥에 넣을지 (sauceInside).
// → NyPizza의 고유 속성이 size(크기)였던 것과 대응됨.
public class Calzone extends Pizza {

    // 칼초네 피자만의 고유 속성: 소스 위치.
    // true = 소스가 피자 안에 있음, false = 소스가 바깥에 있음.
    // boolean이라 두 가지 값만 가능. (NyPizza.Size는 3가지였음)
    //
    // private: 클래스 바깥에서 직접 접근 불가.
    // final: 한 번 정해지면 변경 불가.
    // → NyPizza의 `private final Size size`와 같은 설계 의도.
    // → Pizza의 `final Set toppings`와도 같은 의도.
    // → 모두 불변(immutable) 객체를 만들기 위한 것.
    private final boolean sauceInside;

    // ─── 여기서부터 Calzone.Builder (칼초네 전용 주문서) ───

    // Pizza.Builder가 "기본 주문서",
    // NyPizza.Builder가 "뉴욕 피자 전용 주문서"였다면,
    // Calzone.Builder는 "칼초네 전용 주문서".
    //
    // extends Pizza.Builder:
    //   NyPizza.Builder와 동일한 구조.
    //   Pizza.Builder>의 T 자리에
    //   Calzone.Builder(여기서는 그냥 Builder)가 들어감.
    //   → Pizza.Builder의 addTopping 반환 타입이 Calzone.Builder가 됨.
    //   → 토핑 추가 후에도 칼초네 주문서가 그대로 돌아옴.
    //
    // static인 이유: Pizza.Builder, NyPizza.Builder와 동일.
    //   Calzone 객체 없이도 주문서를 만들 수 있어야 하니까.
    public static class Builder extends Pizza.Builder {

        // 주문서에 적힌 소스 위치. 기본값 = false (소스가 바깥).
        //
        // ★ NyPizza.Builder와의 핵심 차이점 ★
        // NyPizza.Builder: `private final Size size;` → final, 기본값 없음
        //   → 생성자에서 반드시 값을 받아야 함 (필수값)
        // Calzone.Builder: `private boolean sauceInside = false;` → final 아님, 기본값 있음
        //   → 설정 안 해도 됨 (선택값). 안 하면 false(바깥)가 기본.
        //
        // 왜 final이 아닐까?
        //   NyPizza.Builder의 size는 생성자에서 한 번 정해지면 안 바뀜 → final.
        //   Calzone.Builder의 sauceInside는 나중에 sauceInside() 메서드로 바꿀 수 있음 → final 아님.
        private boolean sauceInside = false;

        // ★ NyPizza.Builder와의 또 다른 차이점 ★
        //
        // NyPizza.Builder에는 명시적 생성자가 있었음:
        //   public Builder(Size size) { this.size = Objects.requireNonNull(size); }
        //   → 사이즈는 필수. 안 넣으면 컴파일 에러.
        //
        // Calzone.Builder에는 명시적 생성자가 없음.
        //   → 자바가 자동으로 파라미터 없는 기본 생성자를 만들어줌.
        //   → new Calzone.Builder()로 만들 수 있음. 아무것도 안 넘겨도 됨.
        //   → sauceInside는 선택값이기 때문. 기본값(false)이 있으니까.
        //
        // 이것이 빌더 패턴에서 필수값과 선택값을 구분하는 방법:
        //   필수값 → Builder 생성자의 파라미터로 (NyPizza의 size)
        //   선택값 → 별도 메서드로 설정 (Calzone의 sauceInside, Pizza의 addTopping)

        // 소스를 안에 넣겠다고 설정하는 메서드.
        // 호출하면 sauceInside = true가 됨. 호출 안 하면 기본값 false(바깥) 유지.
        //
        // 파라미터를 받지 않음. sauceInside(true/false)가 아님.
        // 메서드 이름 자체가 의도를 나타냄:
        //   .sauceInside() = "소스를 안에 넣어주세요"
        //   호출 안 함 = "기본대로 바깥에 주세요"
        //
        // return this를 해서 메서드 체이닝 가능.
        // 여기서는 return self()가 아니라 return this를 직접 사용함.
        // 왜? 이 메서드는 Calzone.Builder 안에 직접 정의된 메서드이므로,
        //   여기서 this는 이미 Calzone.Builder 타입.
        //   self()를 거칠 필요가 없음.
        //
        // 비교: Pizza.Builder의 addTopping은 return self()를 했음.
        //   왜? addTopping은 부모(Pizza.Builder)에 정의되어 있어서,
        //   거기서 this를 쓰면 Pizza.Builder 타입으로 인식되기 때문.
        //   self()를 통해 실제 하위 클래스 타입을 반환받아야 했음.
        //
        //   sauceInside()는 Calzone.Builder에 직접 정의되어 있으므로
        //   this가 이미 Calzone.Builder 타입. self() 불필요.
        public Builder sauceInside() {
            sauceInside = true;
            return this;
        }

        // "주문 완료" 버튼. Pizza.Builder의 abstract Pizza build()를 구현.
        //
        // NyPizza.Builder의 build()와 동일한 구조:
        //   NyPizza.Builder: return new NyPizza(this);
        //   Calzone.Builder: return new Calzone(this);
        //
        // 반환 타입이 Pizza가 아니라 Calzone.
        // → 공변 반환 타이핑(covariant return typing). NyPizza.Builder와 동일.
        // → Calzone이 Pizza의 하위 타입이므로 가능.
        //
        // this = 현재 Calzone.Builder 객체.
        // Builder 안에 저장해둔 sauceInside와 toppings를 Calzone 생성자에 넘김.
        @Override public Calzone build() {
            return new Calzone(this);
        }

        // Pizza.Builder의 protected abstract T self()를 구현.
        // NyPizza.Builder의 self()와 완전히 같은 역할.
        //
        // Pizza.Builder의 addTopping에서 return self()를 호출하면,
        // 여기로 와서 this(= Calzone.Builder)를 반환함.
        // → addTopping 후에도 Calzone.Builder 타입이 유지됨.
        // → .addTopping(HAM).sauceInside() 체이닝 가능.
        @Override protected Builder self() { return this; }
    }

    // ─── 여기서부터 Calzone 생성자 ───

    // private: NyPizza 생성자와 동일한 이유.
    //   클래스 바깥에서 직접 호출 불가.
    //   반드시 Builder.build()를 통해서만 Calzone을 만들 수 있음.
    //   → Builder를 거쳐야 기본값 설정, 토핑 추가 등의 과정을 거침.
    private Calzone(Builder builder) {

        // 부모 클래스(Pizza)의 생성자를 호출.
        // NyPizza 생성자에서 super(builder)를 했던 것과 동일.
        //
        // Pizza 생성자를 다시 보면:
        //   Pizza(Builder builder) {
        //       toppings = builder.toppings.clone();
        //   }
        //
        // Calzone.Builder도 Pizza.Builder를 상속받았으므로
        // Builder 타입에 해당함.
        // → builder에 저장된 toppings가 clone()으로 복사되어 Pizza.toppings에 저장됨.
        // → 복사하는 이유: Pizza 코드에서 설명한 것과 동일.
        //   (주문서와 피자가 같은 목록을 공유하면 나중에 문제 발생)
        super(builder);

        // 부모 생성자에서 공통 부분(toppings)을 처리한 후,
        // Calzone만의 고유 속성인 sauceInside를 설정.
        //
        // NyPizza 생성자에서 `size = builder.size`를 했던 것과 대응:
        //   NyPizza: size = builder.size (크기 설정)
        //   Calzone: sauceInside = builder.sauceInside (소스 위치 설정)
        //
        // 이 시점에서 Calzone 완성:
        //   toppings = 부모 생성자(super)에서 설정됨 (final)
        //   sauceInside = 여기서 설정됨 (final)
        //   둘 다 final이므로 이후 변경 불가. 완전한 불변 객체.
        sauceInside = builder.sauceInside;
    }

    // 피자 정보를 문자열로 표현.
    // NyPizza의 toString()이 toppings + "로 토핑한 뉴욕 피자"였던 것처럼,
    // Calzone도 토핑 + 소스 위치 정보를 함께 보여줌.
    //
    // String.format은 포맷 문자열을 사용하는 방법.
    // %s 자리에 변수 값이 순서대로 들어감.
    // 첫 번째 %s = toppings (부모 Pizza에서 물려받은 변수)
    // 두 번째 %s = sauceInside ? "안" : "바깥"
    //   → 삼항 연산자: sauceInside가 true면 "안", false면 "바깥"
    //
    // 예시: sauceInside=true, toppings={HAM, ONION}이면
    //   → "[HAM, ONION]로 토핑한 칼초네 피자 (소스는 안에)"
    @Override public String toString() {
        return String.format("%s로 토핑한 칼초네 피자 (소스는 %s에)",
                toppings, sauceInside ? "안" : "바깥");
    }
}
```

---

**12. Calzone 클래스가 하는 일**

Calzone은 Pizza를 상속받습니다. NyPizza도 Pizza를 상속받았습니다. **둘 다 Pizza의 자식**이지만 각자 다른 고유 속성을 가집니다.

| | Pizza (부모) | NyPizza (자식) | Calzone (자식) |
|---|---|---|---|
| **공통 속성** | toppings | toppings (상속) | toppings (상속) |
| **고유 속성** | 없음 (abstract) | size (크기) | sauceInside (소스 위치) |
| **고유 속성 타입** | - | enum (SMALL/MEDIUM/LARGE) | boolean (true/false) |

`private final boolean sauceInside`는 소스가 안에 있는지를 나타내는 변수입니다. true면 소스가 피자 안에 있고, false면 소스가 바깥에 있습니다. boolean 타입이므로 true 아니면 false, 두 가지 값만 가능합니다.

**private final로 선언한 이유는 NyPizza의 size, Pizza의 toppings와 동일합니다.** 피자가 한 번 만들어지면 소스 위치가 바뀌면 안 됩니다. 소스를 안에 넣은 칼초네를 주문했는데 나중에 누군가 코드에서 바깥으로 바꿔버리면 문제가 됩니다. 불변 객체로 만들어서 원천 차단합니다.

---

**13. Calzone.Builder가 하는 일**

비유: Calzone.Builder는 **"칼초네 전용 주문서"**입니다. NyPizza.Builder가 "뉴욕 피자 전용 주문서"였던 것처럼요. 기본 주문서(Pizza.Builder)의 토핑 추가 기능을 물려받고, **소스 위치 설정 기능을 추가한 확장 주문서**입니다.

**클래스 선언부: NyPizza.Builder와 동일한 구조**

`public static class Builder extends Pizza.Builder<Builder>`

NyPizza.Builder의 선언과 완전히 같은 구조입니다. `Pizza.Builder<T extends Builder<T>>`의 T 자리에 Calzone.Builder가 들어갑니다. 그래서 **Pizza.Builder의 addTopping 반환 타입이 Calzone.Builder**가 됩니다. 토핑 추가 후에도 칼초네 주문서가 그대로 돌아옵니다.

**★ NyPizza.Builder와의 핵심 차이점: 필수값 vs 선택값**

여기가 Calzone.Builder를 이해하는 핵심입니다. NyPizza.Builder와 비교하면 차이가 명확하게 보입니다.

```java
// NyPizza.Builder:
private final Size size;                    // final, 기본값 없음
public Builder(Size size) {                 // 생성자에서 강제
    this.size = Objects.requireNonNull(size);
}
// → new NyPizza.Builder()        ← 에러! size 필수
// → new NyPizza.Builder(LARGE)   ← OK

// Calzone.Builder:
private boolean sauceInside = false;        // final 아님, 기본값 있음
// 명시적 생성자 없음 → 자바가 기본 생성자 자동 생성
// → new Calzone.Builder()        ← OK! 기본값(false) 사용
```

NyPizza.Builder는 **생성자에서 size를 강제로 받습니다.** 사이즈 없는 뉴욕 피자는 존재할 수 없기 때문입니다. 이것이 **필수값**을 표현하는 방법입니다.

Calzone.Builder는 **생성자에 파라미터가 없습니다.** sauceInside는 기본값(false)이 있어서, 아무것도 안 해도 "소스가 바깥에 있는 칼초네"가 됩니다. 이것이 **선택값**을 표현하는 방법입니다.

그리고 Calzone.Builder의 sauceInside는 **final이 아닙니다.** NyPizza.Builder의 size는 final이었습니다. 왜 차이가 날까요? NyPizza.Builder의 size는 생성자에서 한 번 정해지면 바꿀 필요가 없습니다. 하지만 Calzone.Builder의 sauceInside는 처음에 false(기본값)로 시작하고, 나중에 sauceInside() 메서드를 호출하면 true로 바뀌어야 합니다. 값이 변경될 수 있으므로 final을 붙일 수 없습니다.

빌더 패턴에서 값을 다루는 세 가지 방식을 정리하면:

| 방식 | 예시 | 특징 |
|---|---|---|
| **필수값: Builder 생성자** | NyPizza.Builder(Size size) | 안 넘기면 컴파일 에러 |
| **선택값: 설정 메서드** | Calzone.Builder.sauceInside() | 호출 안 하면 기본값 |
| **선택값: 추가 메서드** | Pizza.Builder.addTopping() | 여러 번 호출 가능 |

**sauceInside() 메서드: return this vs return self()**

```java
public Builder sauceInside() {
    sauceInside = true;
    return this;
}
```

이 메서드를 호출하면 sauceInside가 true로 바뀝니다. "소스를 안에 넣어주세요"라는 뜻입니다.

파라미터를 받지 않습니다. `sauceInside(true)` 또는 `sauceInside(false)`로 만들지 않은 이유는 **메서드 이름 자체가 의도를 나타내기 때문**입니다. `.sauceInside()`라고 쓰면 "소스를 안에"라는 뜻이 명확합니다. 호출 안 하면 기본값(바깥)이고, 호출하면 안에 넣는 것입니다.

여기서 주목할 점: **`return this`를 직접 사용**합니다. `return self()`가 아닙니다.

Pizza.Builder의 addTopping은 `return self()`를 했습니다. 왜? addTopping은 **부모(Pizza.Builder)에 정의**되어 있어서, 거기서 this를 쓰면 Pizza.Builder 타입으로 인식되기 때문입니다. self()를 통해 실제 하위 클래스 타입을 반환받아야 했습니다.

sauceInside()는 **Calzone.Builder에 직접 정의**되어 있습니다. 여기서 this를 쓰면 이미 Calzone.Builder 타입입니다. self()를 거칠 필요가 없습니다.

```java
// Pizza.Builder (부모)에 정의된 addTopping:
//   return self();  ← this가 Pizza.Builder로 인식되니까 self() 필요

// Calzone.Builder (자식)에 직접 정의된 sauceInside:
//   return this;    ← this가 이미 Calzone.Builder이니까 self() 불필요
```

**build()와 self(): NyPizza.Builder와 동일**

```java
@Override public Calzone build() { return new Calzone(this); }
@Override protected Builder self() { return this; }
```

NyPizza.Builder의 build(), self()와 완전히 같은 구조입니다.

build()는 Pizza.Builder의 `abstract Pizza build()`를 구현합니다. 반환 타입이 Pizza가 아니라 Calzone입니다. **공변 반환 타이핑** — NyPizza.Builder에서 설명한 것과 동일합니다.

self()는 Pizza.Builder의 `protected abstract T self()`를 구현합니다. addTopping에서 `return self()`를 호출하면 여기로 와서 this(= Calzone.Builder)를 반환합니다. 그래서 `.addTopping(HAM).sauceInside()` 체이닝이 가능합니다.

---

**14. Calzone 생성자가 하는 일**

```java
private Calzone(Builder builder) {
    super(builder);
    sauceInside = builder.sauceInside;
}
```

**NyPizza 생성자와 완전히 같은 구조**입니다.

private 생성자: 클래스 바깥에서 직접 호출 불가. **반드시 Builder.build()를 통해서만** Calzone을 만들 수 있음. NyPizza 생성자와 같은 이유입니다.

`super(builder)`: **부모(Pizza)의 생성자를 호출**합니다. Pizza 생성자에서 `toppings = builder.toppings.clone()`이 실행되어, builder에 저장된 토핑이 복사됩니다. clone()으로 복사하는 이유는 Pizza 코드에서 설명한 것과 동일합니다.

`sauceInside = builder.sauceInside`: 부모 생성자에서 공통 부분(toppings)을 처리한 후, **Calzone만의 고유 속성을 설정**합니다. NyPizza에서 `size = builder.size`를 했던 것과 대응됩니다.

| | NyPizza 생성자 | Calzone 생성자 |
|---|---|---|
| **부모 호출** | super(builder) → toppings 복사 | super(builder) → toppings 복사 |
| **고유 속성 설정** | size = builder.size | sauceInside = builder.sauceInside |
| **결과** | 불변 객체 완성 | 불변 객체 완성 |

이 시점에서 Calzone 객체는 완성됩니다:
- **toppings** = 부모 생성자(super)에서 설정됨 (final)
- **sauceInside** = 여기서 설정됨 (final)
- 둘 다 final이므로 이후 변경 불가. **완전한 불변 객체**입니다.

---

**전체 흐름 정리: Pizza → Calzone이 연결되는 과정**

```java
Calzone calzone = new Calzone.Builder()    // 1단계
    .addTopping(HAM)                        // 2단계
    .sauceInside()                          // 3단계
    .build();                               // 4단계
```

**1단계: `new Calzone.Builder()`**
- Calzone.Builder 기본 생성자 호출 (파라미터 없음)
- sauceInside = false (기본값)
- 동시에 부모(Pizza.Builder)의 toppings = 빈 EnumSet으로 초기화됨
- ★ NyPizza와의 차이: `new NyPizza.Builder(LARGE)`는 size 필수였음

**2단계: `.addTopping(HAM)`**
- **Pizza.Builder에 정의된** addTopping 호출 (Calzone.Builder에는 없음. 상속받은 것)
- `Objects.requireNonNull(HAM)` → null 아니므로 통과
- toppings에 HAM 추가
- `return self()` → Calzone.Builder의 self() 호출 → `return this` → **Calzone.Builder 반환**

**3단계: `.sauceInside()`**
- **Calzone.Builder에 직접 정의된** sauceInside() 호출
- sauceInside = true로 변경
- `return this` → Calzone.Builder 반환 (self() 거치지 않음)
- ★ 2단계의 addTopping이 Calzone.Builder를 반환했기에 이 호출이 가능한 것!
  만약 Pizza.Builder를 반환했다면 sauceInside()를 호출할 수 없었음.

**4단계: `.build()`**
- Calzone.Builder의 build() 호출
- `return new Calzone(this)` → Calzone 생성자 호출
  - `super(builder)` → **Pizza 생성자 호출** → `toppings = builder.toppings.clone()` → {HAM} 복사
  - `sauceInside = builder.sauceInside` → true 설정
- 완성된 Calzone 반환: toppings = {HAM}, sauceInside = true
- toString() 출력: "[HAM]로 토핑한 칼초네 피자 (소스는 안에)"