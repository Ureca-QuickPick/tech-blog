# 🎟️ 동시성 환경에서 DB 락을 이용한 티켓 예약 시스템 테스트

## ✅ 테스트 목표

> 100개의 티켓을 대상으로 수천 명의 유저가 동시에 예약 요청을 보냈을 때,
>
>
> 동시성 이슈 없이 정확히 100명까지만 예약되도록 제어되는지 검증한다.
>

## 1️⃣ 락을 사용하지 않은 경우

🔗 [코드 보기](https://www.notion.so/1e6ec761786c80e79875eb0214e770e8?pvs=21)

- **설명**:

  동시성 제어 없이 `reserveTicket()`을 호출한 기본 구현.

- **시나리오**:

  `1000명의 사용자`가 동시에 100개의 티켓을 예약 시도.

- **결과**:

  ❌ **테스트 실패** – 동시성 제어 부재로 **티켓 수량 초과 예약 발생**.


## 2️⃣-1 비관적 락(Pessimistic Lock)을 적용한 경우

🔗 [코드 보기](https://www.notion.so/1e6ec761786c8071b72aed507a6744b0?pvs=21)

- **설명**:

  `@Lock(LockModeType.PESSIMISTIC_WRITE)`를 적용해 동시에 하나의 티켓만 수정 가능하도록 제한.

- **시나리오**:

  `10000명의 사용자`가 동시에 100개의 티켓을 예약 시도.

- **결과**:

  ✅ **테스트 성공** – 정확히 `100명만 예약`됨.

- 문제점:
    - 사용자 10,000명을 `개별적으로 저장`해 테스트 시간이 매우 길어짐.
        - **해결 방법** : **벌크 Insert (`saveAll()`) 사용**

## 2️⃣-2 saveAll() 활용한 벌크 Insert 시도

🔗 [코드 보기](https://www.notion.so/1e7ec761786c8029b92ec2956086b59f?pvs=21)

- **설명**:

  `saveAll()`로 사용자 10,000명을 한 번에 저장.

- **문제점**:
    - 내부적으로 여전히 `persist()`를 루프 돌며 호출 → `쿼리 N번 발생`.
    - 즉, **실제 벌크 성능은 기대보다 낮음**.
- **해결 방안**:

  Hibernate의 **Batch Insert 설정** + `EntityManager` 직접 사용.


## 2️⃣-3 EntityManager + @Transactional 방식 적용

🔗 [코드 보기](https://www.notion.so/1e7ec761786c801f9749e72c3c3e3177?pvs=21)

- **설명**:

  `@PersistenceContext`로 주입한 `EntityManager`를 통해 `flush()` & `clear()`를 주기적으로 호출하며 batch 처리.

- **문제 발생**:
  ❌ **`@Transactional`이 적용되지 않음**

    ```
    
    @Transactional self-invocation (in effect, a method within the target object calling another method of the target object) does not lead to an actual transaction at runtime
    ```

- **이유**:
    - Spring은 **AOP 기반으로 트랜잭션 처리**
    - **같은 클래스 내에서 메서드 호출(this)** → **프록시 우회** → 트랜잭션 적용 안 됨

## 2️⃣-4 Self-Invocation 문제 해결: Self 주입 구조 도입

🔗 [코드 보기](https://www.notion.so/1e7ec761786c803aa370e19889828cbb?pvs=21)

- **설명**:

  `this.insertUsersInBulk()` 대신, **Bean으로 주입된 자신(Self)** 을 통해 호출

- **문제 발생**:
  ❌ 테스트 클래스에 `@Autowired TicketReservationServiceTest` →

  `Could not autowire. No beans of 'TicketReservationServiceTest' type found.`

- **이유**:
- `@Transactional`이 붙은 `insertUsersInBulk()`를 같은 클래스 내에서 호출하고 있어서 **트랜잭션이 적용되지 않고**, 그 결과 `EntityManager`가 트랜잭션 바깥에서 실행되며 오류가 발생
- 테스트 클래스는 `@SpringBootTest`로 실행되지만, 일반적인 **Spring Bean으로 등록되지 않음**

## 2️⃣-5 최종 해결 – 벌크 Insert 전용 서비스 분리

🔗 [코드 보기](https://www.notion.so/1e7ec761786c801c8bade22af1bdabc2?pvs=21)

- **설명**:
  `@Service` 클래스로 `InsertUsersInBulkService`를 분리하고,

  여기에 `@Transactional`과 `EntityManager`를 적용해 올바른 트랜잭션 처리 구현.

- **결과**:
  **트랜잭션 정상 동작 + 빠른 대용량 insert 처리**

  **테스트 성공**



| 테스트 방식 | 결과 | 동시성 제어 | 성능 문제 | 트랜잭션 문제 |
| --- | --- | --- | --- | --- |
| 단순 로직 | ❌ 실패 | ❌ 없음 | ❌ 느림 | - |
| Pessimistic Lock | ✅ 성공 | ✅ 있음 | ❌ 느림 | - |
| saveAll() | ✅ 성공 | ✅ 있음 | ⚠️ 기대 이하 | - |
| EntityManager | ❌ 실패 | ✅ 있음 | ✅ 빠름 | ❌ 적용 안 됨 |
| Self 주입 구조 | ❌ 실패 | ✅ 있음 | ✅ 빠름 | ❌ 테스트 클래스 불가 |
| 서비스 분리 | ✅ 성공 | ✅ 있음 | ✅ 빠름 | ✅ 정상 동작 |


---
🖋️ Written by [홍승민](https://github.com/tmdals1207)