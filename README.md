# 🛍️ 갈팡질팡 — 커머스 플랫폼 (팀 프로젝트)

> 회원이 상품을 검색하고 장바구니에 담아 **주문 · 결제 · 부분 환불**까지 진행하고, **실시간 채팅**으로 문의할 수 있는 커머스 서비스입니다.  
> 저는 **프로젝트 초기 세팅, 공통 응답 · 예외 처리, JWT 인증/인가, 회원 · 관리자 API, Redis 캐싱, 배포 환경 구성**을 맡았고, 통합 단계에서 **환불 · 채팅 버그 수정과 문서화**를 담당했습니다.

🔗 팀 원본 저장소: [TeamOJOSAMA/Commerce](https://github.com/TeamOJOSAMA/Commerce) · 프론트엔드: [commerce-frontend](https://github.com/TeamOJOSAMA/commerce-frontend)

<br>

## 📌 프로젝트 정보

| 항목 | 내용 |
| --- | --- |
| 기간 | 2026.09.07 ~ 2026.09.17 |
| 인원 | 6명 |
| 내 역할 | 프로젝트 세팅, 공통 모듈, 인증/인가, 회원 · 관리자, 캐싱, 배포 환경, 통합 버그 수정, 문서화 |
| 협업 | GitHub (Feature 브랜치 + PR, `dev` 통합 브랜치), Notion |

<br>

## 🛠 Tech Stack

<img src="https://img.shields.io/badge/Java%2017-007396?style=for-the-badge&logo=openjdk&logoColor=white">
<img src="https://img.shields.io/badge/Spring%20Boot%204-6DB33F?style=for-the-badge&logo=springboot&logoColor=white">
<img src="https://img.shields.io/badge/Spring%20Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white">
<img src="https://img.shields.io/badge/JWT-000000?style=for-the-badge&logo=jsonwebtokens&logoColor=white">
<img src="https://img.shields.io/badge/Spring%20Data%20JPA-6DB33F?style=for-the-badge&logo=spring&logoColor=white">
<img src="https://img.shields.io/badge/QueryDSL-0769AD?style=for-the-badge&logoColor=white">
<br>
<img src="https://img.shields.io/badge/MySQL%208-4479A1?style=for-the-badge&logo=mysql&logoColor=white">
<img src="https://img.shields.io/badge/Redis%207-DC382D?style=for-the-badge&logo=redis&logoColor=white">
<img src="https://img.shields.io/badge/WebSocket%20(STOMP)-010101?style=for-the-badge&logo=socketdotio&logoColor=white">
<img src="https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white">
<img src="https://img.shields.io/badge/React%2019-61DAFB?style=for-the-badge&logo=react&logoColor=black">

<br>

## ✨ 전체 기능

| 도메인 | 주요 기능 |
| --- | --- |
| **인증 · 회원** ⭐ | JWT 회원가입 · 로그인, 내 정보 조회, 관리자 회원 검색 · 역할 변경 (USER / SELLER / ADMIN) |
| 상품 | QueryDSL 동적 검색, 판매자 상품 관리, **인기 상품 Redis 캐싱** ⭐, Redis 조회수 버퍼링 |
| 이벤트 · 쿠폰 | 기간 한정 할인 이벤트, 쿠폰 정책 · 사용자별 발급 이력 분리, 주문 시 쿠폰 예약 |
| 장바구니 · 주문 | 장바구니 CRUD, 재고 선차감 주문, 주문 상품 스냅샷 |
| 결제 | 결제 금액 재검증 후 결제 · 쿠폰 사용 · 주문 확정을 한 트랜잭션으로 처리 |
| 환불 | 전액 / **여러 번의 부분 환불**, 항목별 잔여 수량 검증 |
| 채팅 | STOMP 실시간 1:1 문의, 챗봇 자동 응답, 미응답 상담 자동 종료 |

⭐ 표시가 제가 직접 구현한 기능입니다.

<br>

## 🙋 담당 기능 상세

### 1. 프로젝트 초기 세팅 · 공통 모듈
- 도메인형 패키지 구조(`common` / `domain`) 설계 및 Spring Initializr 기반 프로젝트 생성
- 모든 응답을 감싸는 `ApiResponse`, 예외를 표준화하는 `BusinessException` + `ErrorCode` + `GlobalExceptionHandler` 구성
- 컨트롤러별로 달랐던 URL prefix와 응답 타입(`ResponseEntity<ApiResponse<T>>`)을 팀 전체 기준으로 통일

### 2. JWT 인증 / 인가
- Spring Security + `OncePerRequestFilter` 기반 JWT 필터 구현, 세션을 쓰지 않는 `STATELESS` 정책
- 토큰에 회원 ID · 이메일 · 역할을 담고, 필터에서 `SecurityContext`에 인증 정보 저장
- URL 단위 권한 분리: `/api/admin/**`는 ADMIN, `/api/seller/**`는 SELLER · ADMIN, 상품 · 이벤트 **GET 조회만** 비로그인 허용
- 비밀번호는 `BCryptPasswordEncoder`로 해싱

### 3. 회원가입 동시성 방어
이메일 중복 검사(`existsByEmail`)와 저장 사이에는 시간차가 있어, 같은 이메일로 **동시에** 가입 요청이 오면 둘 다 검사를 통과할 수 있습니다.  
1차로 애플리케이션에서 중복을 검사하고, 2차로 DB 유니크 제약 위반을 잡아 같은 에러 코드로 변환했습니다.

```java
if (userRepository.existsByEmail(request.email())) {
    throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);   // 1차: 대부분의 중복
}
try {
    userRepository.saveAndFlush(user);                        // flush로 제약 위반을 즉시 확인
} catch (DataIntegrityViolationException e) {
    throw new BusinessException(ErrorCode.DUPLICATE_EMAIL);   // 2차: 동시 요청
}
```

### 4. 관리자 회원 관리
- **QueryDSL**로 역할 조건 동적 검색 + 페이징 (`UserRepositoryCustom` / `Impl`)
- 마지막 남은 ADMIN은 강등할 수 없도록 검증해, 관리자가 한 명도 없는 상태를 방지

### 5. 인기 상품 Redis 캐싱
실시간성이 필요 없는 인기 상품 목록에 `@Cacheable`을 적용하고 **TTL 5분**으로 설정해, 반복 요청 시 DB 조회를 줄였습니다.  
캐시 키는 페이지 번호와 크기를 조합해 페이지별로 따로 저장합니다.

### 6. 배포 환경 구성
- **Dockerfile(멀티 스테이지 빌드)** + **docker-compose**(애플리케이션 · MySQL · Redis)
- DB 계정, 호스트, JWT 키, CORS 허용 도메인을 모두 **환경 변수로 분리**해 코드 수정 없이 환경 전환
- `prod` 프로필에서 `ddl-auto: validate`, SQL 로그 비활성화

<br>

## 🔧 트러블슈팅

### 1. 채팅방 자동 종료 스케줄러가 같은 방을 무한히 다시 닫던 문제

**문제**  
5분간 응답이 없는 상담을 자동 종료하는 스케줄러가, 이미 종료 메시지를 보낸 방을 매 주기마다 계속 다시 종료하고 있었습니다.

**원인**  
스케줄러 메서드가 같은 클래스의 `@Transactional` 메서드를 `this::closeAndNotify`로 직접 호출하고 있었습니다.  
Spring의 `@Transactional`은 **프록시를 거쳐 호출될 때만** 동작하기 때문에, 내부 호출(self-invocation)에서는 트랜잭션이 열리지 않았고 채팅방의 `COMPLETED` 상태 변경이 DB에 반영되지 않았습니다.

**해결**  
스케줄러가 호출하는 바깥 메서드 `closeIdleChatRooms()`에 `@Transactional`을 붙여, 프록시를 통해 트랜잭션이 열린 상태에서 내부 메서드가 실행되도록 수정했습니다.

```java
@Transactional                                      // 프록시를 거치는 진입점에서 트랜잭션 시작
@Scheduled(fixedDelayString = "${chat.cleanup-interval-ms}")
public void closeIdleChatRooms() {
    chatRoomRepository.findIdleChatRoomIds(threshold)
            .forEach(this::closeAndNotify);         // 이미 열린 트랜잭션에 참여
}
```

**배운 점**  
`@Transactional`은 어노테이션을 붙이는 것만으로 동작하는 게 아니라 **프록시 기반으로 동작한다**는 것을 체감했습니다.  
또한 이 방식은 유휴 방 전체가 하나의 트랜잭션으로 묶여, 한 방에서 예외가 나면 나머지도 롤백된다는 트레이드오프가 있다는 것도 함께 정리했습니다.

<br>

### 2. 부분 환불이 한 번만 가능하던 문제

**문제**  
한 주문의 상품 일부를 환불한 뒤, 남은 상품을 다시 환불하려 하면 항상 실패했습니다.

**원인**  
원인이 두 겹으로 겹쳐 있었습니다.
1. `refunds.payment_id`에 **유니크 제약**이 걸려 있어, 한 결제에 환불 기록이 하나만 생성될 수 있었습니다.
2. 1번을 고친 뒤에도, 환불 완료 메서드가 **환불 타입과 상관없이 결제를 취소(`CANCELED`)** 하고 있어서, 다음 환불 요청이 "결제 완료 건만 환불 가능" 검증에 막혔습니다.

**해결**
- 유니크 제약을 제거하고, **주문 항목별 잔여 수량**(원래 수량 − 이미 환불된 수량)을 집계해 중복 · 초과 환불을 막도록 변경
- 결제 취소는 **전액 환불일 때만** 수행하고, 부분 환불은 결제를 `PAID`로 유지
- 주문 상세 응답에 항목별 환불 수량을 추가해, 화면에서 추가 환불 가능 수량을 계산할 수 있도록 함
- 실제 DB로 **부분 환불 3회 연속 처리 · 초과 요청 거부**를 검증하는 통합 테스트 추가

**배운 점**  
첫 번째 원인을 고쳤을 때 테스트를 한 번만 해봤다면 두 번째 원인을 놓쳤을 것입니다. **"여러 번 반복되는 시나리오"는 반복 자체를 테스트해야 한다**는 것을 배웠습니다.

<br>

### 3. Redis 캐시 직렬화 오류

**문제**  
인기 상품 목록에 Redis 캐시를 적용하자 직렬화 단계에서 오류가 발생했습니다.

**원인**  
Spring Boot 4로 넘어가며 Jackson 3를 사용하는데, `spring-data-redis`의 Jackson 직렬화기와 버전이 맞지 않았습니다.

**해결**  
커스텀 ObjectMapper 대신 기본 직렬화 방식을 사용하고, 캐시 대상인 `ProductResponse`가 `Serializable`을 구현하도록 수정했습니다.

<br>

## 🗂 패키지 구조

```
com.example.commerce
├── common
│   ├── config      # Security, Redis, Cache, QueryDSL, CORS, Swagger, Scheduling
│   ├── entity      # BaseEntity (JPA Auditing)
│   ├── exception   # BusinessException, ErrorCode, GlobalExceptionHandler
│   ├── filter      # JwtAuthFilter
│   ├── jwt         # JwtProvider
│   └── response    # ApiResponse, PageResponse
└── domain
    ├── auth · user · product · event · coupon
    ├── cart · order · payment · refund
    └── chat
```

`controller → (facade) → service → repository` 계층을 따르며, 여러 도메인에 걸친 트랜잭션은 `facade`에서 조립합니다.  
도메인 간에는 엔티티를 직접 참조하지 않고 ID로만 연결합니다.

<br>

## 📡 담당 API

| 기능 | Method | URL | 권한 |
| --- | --- | --- | --- |
| 회원가입 | POST | `/api/auth/signup` | 전체 |
| 로그인 | POST | `/api/auth/login` | 전체 |
| 내 정보 조회 | GET | `/api/users/me` | 로그인 |
| 회원 목록 검색 | GET | `/api/admin/users?role=` | ADMIN |
| 회원 역할 변경 | PATCH | `/api/admin/users/{userId}/role` | ADMIN |
| 인기 상품 조회 | GET | `/api/products/popular` | 전체 |

전체 API는 애플리케이션 실행 후 `/swagger-ui/index.html`에서 확인할 수 있습니다.

<br>

## 🚀 실행 방법

```bash
git clone https://github.com/tls123qwe/Commerce.git
cd Commerce
git checkout dev

# MySQL, Redis 실행
docker compose up -d mysql redis

# 환경 변수 설정 후 실행
export DB_USERNAME=root
export DB_PASSWORD=your_password
export JWT_SECRET=your_base64_secret
./gradlew bootRun
```

시연용 더미 데이터는 `db/seed/` 스크립트를 순서대로 실행합니다. 자세한 순서는 [팀 README](https://github.com/TeamOJOSAMA/Commerce)를 참고해주세요.

<br>

## 📈 개선하고 싶은 점

- [ ] JWT 필터에서 서명 위조 예외(`io.jsonwebtoken.security.SignatureException`)까지 처리하고, 인증 실패 응답을 공통 `ApiResponse` 형식으로 통일
- [ ] 토큰이 없을 때 400이 아닌 **401** 반환
- [ ] JWT 필터의 인증 제외 경로와 `SecurityConfig`의 `permitAll` 목록을 한 곳에서 관리
- [ ] 운영 환경에서는 `JWT_SECRET`, `CORS_ALLOWED_ORIGINS`의 기본값 없이 반드시 주입받도록 변경
- [ ] 마지막 관리자 강등 방지 로직의 동시 요청 대비
- [ ] Refresh Token 도입

<br>

## 📝 회고

이전 프로젝트에서 받은 피드백(DB 비밀번호 하드코딩, 예외가 500으로 나가는 문제, 동시성 미고려)을 이번에는 처음부터 반영하려고 했습니다.  
특히 팀 전체가 사용하는 **공통 모듈과 인증**을 맡으면서, 내가 정한 규칙이 다른 팀원 모두의 코드 방향을 결정한다는 책임감을 느꼈습니다.  
통합 단계에서는 남이 만든 코드의 버그를 추적하면서, **프레임워크가 내부에서 어떻게 동작하는지 알아야 고칠 수 있는 버그**가 있다는 것을 배웠습니다.
