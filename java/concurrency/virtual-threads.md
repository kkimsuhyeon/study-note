# 가상 스레드 (Virtual Threads) — Java 21 JEP 444

> **한 줄 요약**: 가상 스레드는 **JDK가 관리하는 초경량 스레드** — `java.lang.Thread`지만 OS 스레드에 1:1로 묶이지 않고, 소수의 **캐리어(OS) 스레드** 위에 **수백만 개**를 얹어 돌린다(M:N). 블로킹 I/O를 만나면 캐리어에서 **내려와(unmount)** OS 스레드를 양보 → "요청당 스레드" 블로킹 코드가 리액티브 없이도 확장된다. **Java 21 정식 기능**(플래그 불필요). 단 **확장성만 좋아질 뿐 정합성은 별개 — 락은 그대로 필요.**

관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념](./locks.md)

---

## 1. 왜 나왔나 — 플랫폼 스레드의 벽

[플랫폼 스레드](./threads.md)는 OS 스레드와 1:1이라 비싸서(스택 ~1MB, 수천 개 한계), "요청당 스레드 1개" 방식이 동시 접속 폭증 시 무너진다. 두 가지 길이 있었다:
- **리액티브**(WebFlux 등): 적은 스레드로 비동기 처리 → 확장은 되지만 **코드가 콜백·체인으로 복잡**, 디버깅·스택트레이스 어려움.
- **가상 스레드**(Java 21): **블로킹 코드 그대로 쓰면서** 확장 → "동기 코드의 단순함 + 비동기의 확장성".

---

## 2. 핵심 구조 — 캐리어 위에 M:N으로 얹힘

- 가상 스레드는 **JDK(JVM)가 스케줄**하는 경량 스레드. 스택이 **힙에** 저장돼 (고정 1MB 아님) 매우 쌈 → **수백만 개** 생성 가능.
- 실행되려면 진짜 OS 스레드가 필요한데, 그게 **캐리어 스레드**(소수의 플랫폼 스레드 풀, 보통 CPU 코어 수).
- **M개 가상 스레드를 N개 캐리어에 멀티플렉싱**(M:N) — Go의 goroutine, Erlang 프로세스와 같은 발상.

### mount / unmount (핵심 메커니즘)
```
가상스레드 V가 캐리어 C 위에서 실행(mount)
   ↓ V가 블로킹 I/O(DB·HTTP) 만남
V는 C에서 내려옴(unmount) → C는 풀려서 다른 가상스레드 실행
   ↓ I/O 완료
V가 (아무) 캐리어에 다시 mount되어 이어서 실행
```
→ **블로킹 동안 OS 스레드를 점유하지 않는 게 핵심.** 그래서 수천 개가 동시에 DB를 기다려도 캐리어(OS 스레드)는 몇 개면 충분.

---

## 3. 플랫폼 vs 가상 스레드

| | 플랫폼 스레드 | 가상 스레드 |
|---|---|---|
| OS 스레드 매핑 | **1:1** | **M:N** (캐리어에 다중 적재) |
| 관리 주체 | OS | **JDK** |
| 스택 | OS 스택 ~1MB 고정 | **힙**에 가변(작게 시작) |
| 개수 한계 | 수천 | **수백만** |
| 생성 비용 | 비쌈 → **풀링** | 쌈 → **풀링 금지**(작업마다 새로) |
| 적합 | CPU 바운드 | **I/O 바운드(블로킹 대기 多)** |
| 정체 | `java.lang.Thread` | `java.lang.Thread` (둘 다 같은 타입!) |

> 둘 다 `Thread` 인스턴스다 — API가 같아서 기존 코드가 거의 그대로 돈다.

---

## 4. 쓰는 법 — Java 21에서 플래그 불필요(정식)

```java
// (1) 하나 띄우기
Thread.startVirtualThread(() -> { ... });
Thread.ofVirtual().name("worker").start(runnable);

// (2) Executor — ★작업마다 새 가상 스레드 (풀이 아님)
try (var executor = Executors.newVirtualThreadPerTaskExecutor()) {
    executor.submit(task);   // 제출할 때마다 새 가상 스레드 생성
}
```
> ⚠️ **가상 스레드는 풀링하지 않는다.** 워낙 싸서 "작업마다 새로"가 정석 → `newFixedThreadPool`이 아니라 `newVirtualThreadPerTaskExecutor`. 가상 스레드를 고정 풀에 가두면 이점이 사라진다.

### Spring Boot — 프로퍼티 한 줄 (3.2+)
```yaml
spring:
  threads:
    virtual:
      enabled: true   # Tomcat이 요청마다 가상 스레드로 처리, @Async 등도 적용
```

---

## 5. ⚠️ 함정 — Pinning (캐리어 고정)

가상 스레드가 블로킹 시 캐리어에서 못 내려오고 **고정(pinned)**되면, 그 캐리어(OS 스레드)가 묶여 확장성이 깎인다 — "비싼 플랫폼 스레드"로 되돌아간 셈.

- **Java 21**: `synchronized` 블록 안에서 블로킹하면 **pinning 발생**. → 당시 권고: 블로킹 구간엔 `synchronized` 대신 **`ReentrantLock`** 사용.
- **Java 24 (JEP 491)**: JVM이 모니터 소유를 **가상 스레드 단위로 추적**하도록 바뀌어 **`synchronized` pinning 해소**. → 이제 `synchronized`를 굳이 `ReentrantLock`으로 바꿀 필요 **없음**(JEP 491 저자도 더는 권장 안 함).
- 단 JEP 491 후에도 **다른 pinning 원인은 남음**: JNI 호출, 클래스 초기화, 일부 `Object.wait` 경로 등. `jdk.VirtualThreadPinned` JFR 이벤트로 감지.

> ⚠️ **이 프로젝트는 Java 21**이라 `synchronized`+블로킹 pinning이 *아직 실재*하는 버전. 가상 스레드를 적극 쓸 거면 블로킹 임계구역은 `ReentrantLock` 고려, 또는 JDK 24+로 올리면 신경 덜 써도 됨.

---

## 6. ⚠️ 오해 금지 — 가상 스레드 ≠ 락 불필요

가상 스레드는 **"스레드를 싸게 많이"** 만드는 기능이지, **동시성 정합성을 공짜로 주지 않는다.**
- 여러 가상 스레드가 같은 데이터를 고치면 **race condition은 똑같이 발생** → `@Version`·비관락·`synchronized`·Atomic 다 **그대로 필요**.
- 바뀌는 건 **확장성(얼마나 많이 동시에)**, 안 바뀌는 건 **정합성(올바름)**.
- CPU 바운드 작업엔 이득 없음 — 코어 수만큼만 진짜 병렬이라, 그건 플랫폼 스레드 풀이 맞다.

---

## 7. 💡 판단 기준

| 상황 | 선택 |
|---|---|
| 블로킹 I/O 많고 동시성 높음(웹 요청·외부 API 호출) | **가상 스레드** (Spring `virtual.enabled=true`) |
| CPU 집약 계산 | 플랫폼 스레드 풀(코어 수) |
| 동시 수 제한·자원 풀 | 여전히 Semaphore 등 (가상 스레드여도) |
| 정합성(race) | 가상이든 아니든 **락 필요** |

> 한 줄: **가상 스레드 = 블로킹 I/O를 값싸게 많이 굴리는 도구(M:N, unmount). Java 21 정식이라 플래그 불필요, 직접은 `startVirtualThread`/`newVirtualThreadPerTaskExecutor`(풀링 금지), Spring은 프로퍼티 한 줄.** 단 ① I/O 바운드에만 이득 ② Java 21은 `synchronized` pinning 주의(JDK24서 해결) ③ 락은 그대로 필요(확장성↑ ≠ 정합성).

---

## 8. 참고
- [JEP 444: Virtual Threads](https://openjdk.org/jeps/444)
- [JEP 491: Synchronize Virtual Threads without Pinning (JDK 24)](https://openjdk.org/jeps/491)
- [Oracle - Virtual Threads (Java 21)](https://docs.oracle.com/en/java/javase/21/core/virtual-threads.html)
- 관련 노트: [스레드 기초](./threads.md) · [JVM 동시성 도구](./jvm-concurrency-tools.md) · [락 개념](./locks.md)

---

**학습 날짜**: 2026-06-16
**계기**: Java 21 가상 스레드를 쓰려면 뭘 해야 하나 궁금해서 검색 정리 — M:N/캐리어/unmount, 플랫폼 vs 가상, 사용법(풀링 금지·Spring 프로퍼티), pinning(21 vs JEP491/24), "락은 그대로 필요"까지.
