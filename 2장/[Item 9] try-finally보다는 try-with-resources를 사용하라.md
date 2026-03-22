# Effective Java Item 9: try-finally vs try-with-resources 복습 노트

---

## 1. throws 핵심 개념

### throws의 본질: "나는 이 예외 처리 안 할게, 나를 부른 쪽이 처리해"

```java
// throws가 없는 경우 → 내가 직접 try-catch로 처리
static String firstLineOfFile(String path) {
    try {
        // ...
    } catch (IOException e) {
        // 내가 직접 처리
    }
}

// throws가 있는 경우 → "나는 모르겠고, 나를 호출한 main이 처리해"
static String firstLineOfFile(String path) throws IOException {
    // IOException이 발생하면 그냥 위로 전달
}
```

- `throws IOException` = "이 메서드는 IOException을 일으킬 수 있고, 내가 처리하지 않으니 호출한 쪽에서 처리하라"는 선언
- `main`도 `throws IOException`이면 → JVM으로 던져서 프로그램 종료 + 에러 출력
- `catch`로 직접 처리하면 → `throws` 불필요, 메서드 시그니처에 선언할 필요 없음

---

## 2. try-finally 방식 (구식, 문제 있음)

### 코드 9-1: 단일 자원 (TopLine)

```java
static String firstLineOfFile(String path) throws IOException {

    // FileReader로 파일 열고 BufferedReader로 감싸서 줄 단위 읽기 가능하게 함
    BufferedReader br = new BufferedReader(new FileReader(path));

    try {
        return br.readLine(); // 파일의 첫 번째 줄을 읽어서 반환
    } finally {
        br.close(); // return이 실행되든, 예외가 나든 무조건 실행되어 파일을 닫음
                    // ⚠️ 단, br.close()도 IOException을 던질 수 있어서 예외가 묻히는 문제 발생
    }
}
```

### 코드 9-2: 복수 자원 (Copy)

```java
static void copy(String src, String dst) throws IOException {

    InputStream in = new FileInputStream(src);   // src 파일을 바이트 단위로 읽기 위해 열기

    try {
        OutputStream out = new FileOutputStream(dst); // dst 파일을 쓰기 위해 열기

        try {
            byte[] buf = new byte[BUFFER_SIZE]; // 8KB 임시 버퍼 생성
            int n;

            // in.read(buf): 버퍼에 최대 8KB만큼 읽어들이고 읽은 바이트 수를 n에 저장
            // n이 -1이면 파일 끝(EOF) → 반복 종료
            while ((n = in.read(buf)) >= 0)
                out.write(buf, 0, n); // 실제로 읽힌 n바이트만큼만 dst에 씀

        } finally {
            out.close(); // 쓰기 스트림 닫기
        }
    } finally {
        in.close(); // 읽기 스트림 닫기
    }
}
```

### try-finally의 문제점 2가지

**문제 1: 예외가 묻힌다 (가장 치명적)**

```
readLine() 에서 예외 A 발생
    ↓
finally에서 close() 실행
    ↓
close()에서 예외 B 발생
    ↓
예외 A가 예외 B에 덮어씌워져서 증발!
결과: 예외 B만 전달됨 (진짜 원인인 A를 잃어버림)
```

→ 디버깅할 때 진짜 원인을 못 찾게 됨

**문제 2: 자원이 2개 이상이면 코드가 지저분해진다**

→ try-finally가 중첩되어야 해서 가독성이 나빠짐

---

## 3. try-with-resources 방식 (현재 권장)

### 핵심 구조

```java
// try(...) 괄호 안에 자원을 선언
// 블록이 끝나는 순간 자동으로 close() 호출
try (BufferedReader br = new BufferedReader(new FileReader(path))) {
    return br.readLine();
}
// ← 이 지점에서 br.close() 자동 호출
```

**왜 가능하냐?**  
`AutoCloseable` 인터페이스를 구현한 클래스만 `try(...)` 안에 쓸 수 있음.  
`BufferedReader`, `InputStream`, `OutputStream` 등 I/O 클래스들은 이미 다 구현되어 있음.

---

### 코드 9-3: 단일 자원 (TopLine)

```java
static String firstLineOfFile(String path) throws IOException {

    // try(...) 안에 자원 선언 → 블록 끝나면 br.close() 자동 호출
    // 이전 코드의 finally { br.close(); } 가 완전히 사라짐
    try (BufferedReader br = new BufferedReader(
            new FileReader(path))) {

        return br.readLine(); // 첫 번째 줄 읽어서 반환
                              // 여기서 예외가 나도 br.close()는 자동으로 실행됨
    }
}
```

---

### 코드 9-4: 복수 자원 (Copy)

```java
static void copy(String src, String dst) throws IOException {

    // 자원이 2개: 세미콜론(;)으로 구분해서 나란히 선언
    // 이전 코드의 중첩 try-finally 2개가 이 한 줄로 대체됨
    // 닫힐 때는 선언 역순: out 먼저 닫히고 in 나중에 닫힘
    try (InputStream  in  = new FileInputStream(src);
         OutputStream out = new FileOutputStream(dst)) {

        byte[] buf = new byte[BUFFER_SIZE];
        int n;
        while ((n = in.read(buf)) >= 0)
            out.write(buf, 0, n);
    }
    // 블록 끝: out.close() → in.close() 순서로 자동 호출
}
```

---

### 코드 9-5: catch 절과 함께 사용 (TopLineWithDefault)

```java
// throws IOException이 없다! → catch에서 직접 처리하기 때문
static String firstLineOfFile(String path, String defaultVal) {

    try (BufferedReader br = new BufferedReader(
            new FileReader(path))) {

        return br.readLine(); // 성공하면 첫 줄 반환

    } catch (IOException e) {  // IOException 발생 시 여기서 직접 처리
        return defaultVal;     // 예외 대신 기본값 반환
                               // "파일 못 읽으면 그냥 기본값 줄게"
    }
    // finally 없이도 br.close()는 catch 이후에도 자동 호출됨
}

public static void main(String[] args) throws IOException {
    String path = args[0];
    System.out.println(firstLineOfFile(path, "Toppy McTopFace"));
    // 파일 못 읽으면 "Toppy McTopFace" 출력
}
```

---

## 4. close() 자동 호출 시 예외가 나면?

> "자동으로 close()가 호출되어도 예외가 뜰 수 있지 않아?"

맞다. 날 수 있다. 그런데 try-with-resources는 그 상황을 영리하게 처리한다.

### 상황별 정리

| 상황 | 결과 |
|------|------|
| 본문에서만 예외 A 발생 | 예외 A 전달, close()는 정상 실행 |
| close()에서만 예외 B 발생 | 예외 B 전달 |
| 둘 다 예외 발생 (핵심!) | **예외 A 전달 + 예외 B는 억제된 예외로 보관** |

### 핵심: 억제된 예외 (Suppressed Exception)

```
readLine()에서 예외 A 발생
    ↓
close() 자동 실행
    ↓
close()에서 예외 B 발생
    ↓
예외 B를 예외 A에 "억제된 예외"로 붙여서 보관
결과: 예외 A가 전달됨 + B도 안에 저장됨
```

```java
// 억제된 예외도 꺼내볼 수 있음
try {
    firstLineOfFile("없는파일.txt");
} catch (IOException e) {
    System.out.println(e);                     // 예외 A 출력 (진짜 원인)
    System.out.println(e.getSuppressed()[0]);  // 예외 B도 꺼내볼 수 있음!
}
```

**try-finally와의 차이:**
- try-finally: 예외 A가 예외 B에 덮어씌워져 **증발** → 진짜 원인 못 찾음
- try-with-resources: 예외 A **보존** + 예외 B는 억제된 예외로 **따로 보관** → 둘 다 잃지 않음

---

## 5. 세 코드의 throws 비교 정리

| 코드 | throws | 이유 |
|------|--------|------|
| TopLine (9-3) | `throws IOException` | 예외를 catch 안 하고 호출자에게 넘김 |
| Copy (9-4) | `throws IOException` | 예외를 catch 안 하고 호출자에게 넘김 |
| TopLineWithDefault (9-5) | **없음** | catch에서 직접 처리하고 기본값 반환 |

---

## 6. 전체 요약

| 항목 | try-finally | try-with-resources |
|------|-------------|-------------------|
| 자원 닫기 | 직접 finally에서 close() 호출 | 자동 호출 |
| 예외 2개 발생 시 | 진짜 원인 예외가 덮어씌워져 사라짐 | 진짜 원인 보존, 나머지는 억제된 예외로 보관 |
| 자원 2개 이상 | 중첩 try-finally로 코드 복잡 | 세미콜론으로 나열, 코드 깔끔 |
| 닫는 순서 | 직접 관리 (실수 가능) | 선언 역순으로 자동 처리 |
| 권장 여부 | ❌ 더 이상 권장 안 함 | ✅ 현재 권장 방식 |

> **핵심 한 줄 요약:**  
> `try (자원 선언)` 형태로 쓰면 자원이 자동으로 닫히고, 예외가 묻히는 문제도 해결된다.  
> 자원을 다루는 코드는 항상 try-with-resources를 써라.