<html>
<body>
<!--StartFragment--><html><head></head><body><h1>이펙티브 자바 - 아이템 4</h1>
<h2>인스턴스화를 막으려거든 private 생성자를 사용하라</h2>
<hr>
<h2>🎯 이 아이템의 목적 (한 줄 요약)</h2>
<blockquote>
<p><strong>"객체(인스턴스)를 만들 필요가 없는 클래스는, 아예 못 만들게 막아라."</strong></p>
</blockquote>
<hr>
<h2>📖 왜 이 아이템이 필요한가?</h2>
<h3>유틸리티 클래스란?</h3>
<p>자바에서는 가끔 <strong>정적(static) 메서드만 모아놓은 클래스</strong>를 만들 때가 있다. 이런 클래스를 <strong>유틸리티 클래스</strong>라고 부른다.</p>
<p>예를 들면 이런 것들이다:</p>
<ul>
<li><code>java.lang.Math</code> → <code>Math.abs()</code>, <code>Math.max()</code> 같은 수학 계산 도구 모음</li>
<li><code>java.util.Arrays</code> → <code>Arrays.sort()</code>, <code>Arrays.asList()</code> 같은 배열 도구 모음</li>
<li><code>java.util.Collections</code> → 컬렉션 관련 도구 모음</li>
</ul>
<p>이런 클래스들의 공통점은 <strong>객체를 만들 이유가 전혀 없다</strong>는 것이다. <code>Math math = new Math()</code> 이런 식으로 쓸 일이 없다. 그냥 <code>Math.abs(-5)</code> 이렇게 바로 쓰면 된다.</p>
<hr>
<h3>문제: 자바가 생성자를 자동으로 만들어준다</h3>
<p>자바에는 규칙이 하나 있다:</p>
<blockquote>
<p><strong>개발자가 생성자를 하나도 안 쓰면, 자바 컴파일러가 "빈 public 생성자"를 자동으로 넣어준다.</strong>
이것을 **기본 생성자(default constructor)**라고 부른다.</p>
</blockquote>
<p>이게 무슨 뜻인지 구체적으로 보자.</p>
<h4>내가 작성한 코드:</h4>
<pre><code class="language-java">public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
    // 생성자? 안 썼다. 필요 없으니까.
}
</code></pre>
<h4>자바 컴파일러가 실제로 만들어내는 코드:</h4>
<pre><code class="language-java">public class MathUtils {
    public MathUtils() { }   // ← 컴파일러가 몰래 이걸 추가해버린다!

    public static int add(int a, int b) {
        return a + b;
    }
}
</code></pre>
<p>내가 쓰지도 않았는데, 자바가 알아서 <code>public MathUtils() { }</code> 라는 빈 생성자를 만들어준 것이다. <strong>이건 .java 파일에 안 보이지만, 컴파일된 .class 파일에는 존재한다.</strong> 눈에 안 보이는 생성자가 숨어있는 셈이다.</p>
<h4>그래서 뭐가 문제야?</h4>
<p>이 숨겨진 public 생성자 때문에, 다른 개발자가 이렇게 쓸 수 있게 되어버린다:</p>
<pre><code class="language-java">MathUtils utils = new MathUtils();  // 이게 되어버린다!
utils.add(1, 2);                    // 객체로 호출 (잘못된 사용)

// 원래 의도한 사용법은 이거다:
MathUtils.add(1, 2);               // 클래스명으로 직접 호출 (올바른 사용)
</code></pre>
<p>객체를 만들어서 쓸 필요가 없는 클래스인데, 객체를 만들 수 있으니 <strong>의도치 않은 사용</strong>이 발생하는 것이다. 실제로 공개된 API들에서도 이런 실수가 종종 발견된다고 한다.</p>
<hr>
<h2>✅ 해결책: 이미 있는 클래스에 private 생성자를 "일부러" 추가한다</h2>
<p>핵심 아이디어: <strong>생성자를 쓰기 위해 추가하는 게 아니라, 쓰지 못하게 막기 위해 추가하는 것이다.</strong>
문을 만들어놓고 잠가버리는 느낌이라고 생각하면 된다.</p>
<h3>BEFORE → AFTER 비교</h3>
<h4>❌ BEFORE (문제 상황): 생성자를 안 쓴 유틸리티 클래스</h4>
<pre><code class="language-java">public class MathUtils {
    public static int add(int a, int b) {
        return a + b;
    }
    // 생성자를 안 썼지만...
    // 자바가 몰래 public MathUtils() {} 를 추가해버림!
}
</code></pre>
<p>→ <code>new MathUtils()</code> 가 가능해져버림 (의도하지 않은 사용)</p>
<h4>✅ AFTER (해결): private 생성자를 직접 추가한 유틸리티 클래스</h4>
<pre><code class="language-java">public class MathUtils {

    // ⬇️ 이걸 일부러 추가하는 것이다!
    // 인스턴스화 방지용
    private MathUtils() {
        throw new AssertionError();
    }

    public static int add(int a, int b) {
        return a + b;
    }
}
</code></pre>
<p>→ 개발자가 생성자를 직접 썼으니 자바가 기본 생성자를 안 만듦
→ 그 생성자가 private이니 바깥에서 <code>new MathUtils()</code> 불가능!</p>
<h3>이 코드가 하는 일 (한 줄씩 해석)</h3>

코드 | 의미
-- | --
private MathUtils() | 생성자를 private으로 선언 → 클래스 바깥에서 절대 호출 불가
throw new AssertionError() | 혹시 클래스 안에서 실수로 호출해도 에러 발생 → 이중 안전장치


<h3>왜 이렇게까지 해야 하나?</h3>
<p>자바 컴파일러의 규칙을 역이용하는 것이다:</p>
<ol>
<li>개발자가 생성자를 <strong>하나라도 직접 쓰면</strong> → 컴파일러는 기본 생성자를 만들지 않는다.</li>
<li>그 생성자를 <strong>private으로 만들면</strong> → 바깥에서 접근 불가능.</li>
<li>안에서 <strong>AssertionError를 던지면</strong> → 클래스 내부에서도 실수로 호출 불가능.</li>
</ol>
<p><strong>결과: 이 클래스로는 절대 객체를 만들 수 없다!</strong></p>
<hr>
<h2>🎁 보너스 효과: 상속도 불가능해진다</h2>
<p>private 생성자를 쓰면 <strong>상속(extends)도 자동으로 막힌다.</strong></p>
<p>왜냐하면 자바에서 자식 클래스를 만들면, 자식 클래스의 생성자가 반드시 부모 클래스의 생성자를 호출해야 한다. 그런데 부모의 생성자가 private이면 자식이 접근할 수 없으므로 <strong>컴파일 자체가 안 된다.</strong></p>
<pre><code class="language-java">// 이런 게 불가능해진다!
public class MyUtils extends MathUtils {  // ❌ 컴파일 에러!
    // ...
}
</code></pre>
<p>유틸리티 클래스는 상속해서 쓸 이유도 없으니, 이건 오히려 좋은 부수 효과다.</p>
<hr>
<h2>⚠️ 주의사항</h2>
<ol>
<li>
<p><strong>추상 클래스(abstract class)로는 인스턴스화를 막을 수 없다.</strong></p>
<ul>
<li><code>abstract class</code>로 선언하면 직접 <code>new</code>는 못 하지만, 자식 클래스를 만들면 인스턴스화가 가능하다.</li>
<li>게다가 "상속해서 쓰라는 건가?" 하고 오해할 수 있다.</li>
<li>따라서 <strong>abstract가 아니라 private 생성자가 정답</strong>이다.</li>
</ul>
</li>
<li>
<p><strong>AssertionError는 필수는 아니지만 권장된다.</strong></p>
<ul>
<li>생성자가 private이면 바깥에서는 이미 호출 불가능하다.</li>
<li>하지만 클래스 내부에서 실수로 호출하는 것까지 막으려면 넣어주는 게 좋다.</li>
</ul>
</li>
<li>
<p><strong>주석을 꼭 달아두자.</strong></p>
<ul>
<li>생성자가 있는데 호출하면 안 된다니, 코드만 보면 직관적이지 않다.</li>
<li><code>// 인스턴스화 방지용</code> 같은 주석을 달아서 의도를 명확히 하자.</li>
</ul>
</li>
</ol>
<hr>
<h2>📝 최종 결론 (복습용 핵심 정리)</h2>
<blockquote>
<p><strong>static 메서드만 담은 유틸리티 클래스는 인스턴스를 만들 이유가 없다.</strong></p>
<p>하지만 생성자를 안 만들면 자바 컴파일러가 알아서 public 기본 생성자를 만들어버린다.
(눈에 안 보이지만 .class 파일에 존재한다!)</p>
<p><strong>→ 해결: private 생성자를 "일부러" 추가하고, 안에서 AssertionError를 던져라.</strong></p>
<p>생성자를 쓰기 위해 만드는 게 아니라, 못 쓰게 막기 위해 만드는 것이다.</p>
<p>이렇게 하면:</p>
<ul>
<li>✅ 컴파일러가 기본 생성자를 안 만듦</li>
<li>✅ private이라 외부에서 인스턴스 생성 불가</li>
<li>✅ AssertionError로 내부 실수도 방지</li>
<li>✅ 상속까지 차단</li>
<li>✅ "이 클래스는 객체를 만들면 안 된다"는 의도가 코드에 명확히 드러남</li>
</ul>
</blockquote>
<hr>
<h2>🔑 외워야 할 패턴 (템플릿)</h2>
<pre><code class="language-java">public class 유틸리티클래스이름 {

    // 인스턴스화 방지
    private 유틸리티클래스이름() {
        throw new AssertionError();
    }

    public static 반환타입 메서드이름(매개변수) {
        // 구현
    }
}
</code></pre>
</body></html><!--EndFragment-->
</body>
</html>