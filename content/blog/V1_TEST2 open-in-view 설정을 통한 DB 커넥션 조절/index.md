## ✅  `open-in-view`가 무엇인가?

> 📌 "JPA가 DB 연결을 언제까지 열어놓을까?" 를 정하는 설정
>
- `true` (기본값)

  → **HTTP 응답을 브라우저에 보낼 때까지** DB 연결을 유지함.

- `false`

  → **서비스 메서드(@Transactional) 끝나면 바로** DB 연결을 닫음.


즉, `false`면 **좀 더 빨리** DB 자원을 정리하지만  **지연 로딩**이 깨질 수 있음.

### 비유로 설명하면…

> “마트에서 물건 담기 vs 계산하기”
>
- `true`: 계산대에 가서도 물건을 더 담을 수 있음. (편함)
- `false`: 물건 담기는 상품 진열대 앞에서만 끝내야 하고, 계산대 가면 더 이상 담을 수 없음. (빨리 정리됨, 하지만 불편)

이제 `false`일 때는 **물건을 미리 다 담아야 하니까**,

**서둘러 이것저것 미리 처리**해야 함.

→ 이게 **오히려 느려지게** 만들 수 있음.

예를 들어, 연관 데이터(User, Ticket, UserTicket 등)를 **미리 다 불러와야** 해서 **DB 부하가 커지거나 락이 오래 유지될 수 있다.**

대규모 요청이 발생하는 상황에서는 “하나라도 더 빨리 DB 커넥션을 회수”하는 게 중요하기 때문에 open-in-view : false로 둔다.

| 설정 | 특징 | 언제 추천? |
| --- | --- | --- |
| `open-in-view: true` | 느슨함. 늦게까지 DB 열어둠. | 개발 편하게 할 때 |
| `open-in-view: false` | 엄격함. 트랜잭션 끝나면 DB 닫음. | 대규모 성능 최적화할 때 |

# 🔗 Fetch Join + DTO 조합

- 코드 보기

  ### **Fetch Join 사용**

    ```java
    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @Query("""
        select t from TicketV1 t
        left join fetch t.userTickets ut
        left join fetch ut.user
        where t.ticketId = :ticketId
        """)
    Optional<TicketV1> findByIdForUpdateWithUsers(Long ticketId);
    ```

  ### **서비스 레이어**

    ```java
        @Transactional
        public Ticket reserveTicket(Long userId, Long ticketId) {
    
            log.info(">>> reserveTicket called: userId = {}, ticketId = {}", userId, ticketId);
    
            // user는 fetch join 하지 않았으므로 별도로 조회
            User user = userRepository.findById(userId)
                    .orElseThrow(() -> new IllegalArgumentException("User not found"));
    
            // fetch join으로 모든 필요한 정보 로딩
            Ticket ticket = ticketRepositoryV1.findByIdForUpdateNative(ticketId);
    
            if (userTicketRepository.existsByUser_UserIdAndTicket_TicketId(userId, ticketId)) {
                log.warn("이미 예약한 유저입니다. userId={}, ticketId={}", userId, ticketId);
                return ticket; // 중복이면 insert 안 하고 그냥 리턴
            }
    
            if (ticket.getQuantity() <= 0) {
                throw new IllegalStateException("Ticket out of stock");
            }
    
            ticket.setQuantity(ticket.getQuantity() - 1);
            userTicketRepository.save(new UserTicket(user, ticket));
            return ticket;
        }
    ```

  ### **응답 객체 DTO**

    ```java
    public record TicketReserveResponse(
        Long ticketId,
        String ticketName,
        int remainingQuantity,
        String reservedByUsername
    ) {
        public static TicketReserveResponse of(TicketV1 ticket, UserV1 user) {
            return new TicketReserveResponse(
                ticket.getTicketId(),
                ticket.getName(),
                ticket.getQuantity(),
                user.getUsername()
            );
        }
    }
    
    ```

  ### **컨트롤러**

    ```java
    @PostMapping
    public ResponseEntity<TicketReserveResponse> reserve(@RequestParam Long userId, @RequestParam Long ticketId) {
        try {
            TicketV1 ticket = reserveServiceV1.reserveTicket(userId, ticketId);
            UserV1 user = userRepositoryV1.findById(userId).orElseThrow();
            return ResponseEntity.ok(TicketReserveResponse.of(ticket, user));
        } catch (Exception e) {
            log.error("예약 실패: {}", e.getMessage());
            return ResponseEntity.status(HttpStatus.BAD_REQUEST).body(null);
        }
    }
    
    ```


**`open-in-view: false`** 에서는 **트랜잭션 안에서 모든 걸 끝내야 하므로**
→ **Fetch Join**과 **DTO 변환**을 사용해 미리 필요한 데이터를 다 불러와야 한다.

- **결과** :

  **Average**: 69,005ms (← 이전보다 훨씬 더 느림), **Throughput**: 64.9/sec


- **이유**
    1. **비관적 락 자체의 비용이 큼**
        - 트랜잭션이 길어질수록 락 보유 시간도 증가 → 지연이 누적됨.
        - **동시에 10000명이 몰리면**, 락이 걸린 다른 요청은 **대기하게 되므로 전체 응답 속도가 늘어나고 Throughput 감소**

    2. **Fetch Join은 결과적으로 더 많은 데이터를 불러오고 락을 지연시킴**
        - `userTickets`, `users`까지 `JOIN FETCH`하면 → 데이터양 × 조인카디널리티 증가.
        - DB 입장에선 **락을 유지한 채 더 많은 데이터를 처리** → 락 보유 시간 증가 → 전체 요청 지연됨.

    3. **open-in-view: false 설정은 잘못 사용하면 Lazy 예외는 방지하나 트랜잭션 범위가 커지기 쉬움**
        - DTO로 만드는 과정도 트랜잭션 내에서 끝내야 하므로 → **트랜잭션 종료 지점이 늦어짐**.

          → **락이 오래 유지**되고, 다음 요청들이 더 많이 밀리게 됨.

- **해결 방법**

  ### **✅ 불필요한 Fetch Join 제거**

  → Ticket만 락 걸고, 연관 객체는 Lazy로 분리하거나 별도로 필요 시 조회.


```java
@Lock(PESSIMISTIC_WRITE)
@Query("select t from TicketV1 t where t.ticketId = :ticketId")
Optional<TicketV1> findByIdForUpdate(Long ticketId);
```