# 스마트 광고 콘텐츠 관리 시스템 (YA-SmartSolution)

> 산업단지 내 IoT 연동 스마트 광고 디스플레이 관리 시스템입니다.  
> 광고 콘텐츠 등록·배포부터 외부 기관 배너 자동 갱신, IoT 데이터 수집까지 전 과정을 단독으로 설계하고 개발했습니다.

**개발 기간**: 2025.06 ~ 2025.07  
**개발 규모**: 1인 (단독 개발)  
**운영 여부**: 실서비스 운영 중

---

## 기술 스택

| 구분 | 기술 |
|---|---|
| Language | Java 17 |
| Framework | Spring Boot 3.5.3, Spring Security, Spring WebFlux |
| ORM | MyBatis 3.0.3 + Spring Data JPA (혼용) |
| DB | PostgreSQL |
| View | Thymeleaf |
| API 문서 | SpringDoc OpenAPI (Swagger UI) |
| HTML 파싱 | Jsoup 1.17.2 |
| 비동기 HTTP | WebClient (WebFlux) |
| 빌드 | Gradle |
| 인프라 | dev/prod 환경 분리 (Spring Profiles) |

---

## 담당 역할 및 기여 범위

- 전체 시스템 아키텍처 설계 (도메인 구조, 계층 분리)
- 홍보물 관리 전 기능 구현 (등록·수정·삭제·배포·순서관리)
- IoT 장비 데이터 수집 스케줄러 구현
- 외부 기관(영암군청) 배너 크롤링 자동화 구현
- Spring Security 기반 로그인/인증 구현
- Swagger API 문서화 전체

---

## 시스템 구조

### 홍보물 3계층 도메인

광고 디스플레이 장치에 콘텐츠를 표시하는 구조를 도메인으로 설계했습니다.

```
PromotionObject (홍보물 패키지)
└── PromotionZone (화면 영역)
    └── ZoneFile (영역에 속한 이미지/영상 파일)
```

하나의 홍보물 패키지는 여러 개의 화면 영역(존)을 가지고, 각 영역은 여러 파일을 순서대로 담습니다.  
디스플레이 장치는 시설(Facility)에 연결되며, 관리자가 패키지를 선택해 시설에 배포합니다.

### 패키지 구조

```
com.daehoeng.basic
├── common
│   ├── config          # SecurityConfig, WebConfig, HttpClientConfig
│   ├── scheduler       # CommonScheduler (IoT/크롤링 자동화)
│   ├── security        # CustomUserDetailsService
│   └── util            # FileUploadUtil
├── db
│   ├── dto             # Create/Update/Response DTO 분리
│   ├── mapper          # MyBatis Mapper 인터페이스
│   └── service         # 서비스 인터페이스 + 구현체
└── www                 # Controller (화면 라우팅 + REST API)
```

---

## 기술적 도전과 해결 과정

### 1. 서비스 계층 Interface 분리 — DIP 원칙 적용

**문제**  
초기에는 서비스 구현체를 컨트롤러에서 직접 참조했습니다.  
이 방식은 구현 변경 시 컨트롤러 코드도 함께 수정해야 해서 변경에 취약했습니다.

**해결**  
모든 서비스 클래스를 인터페이스로 분리하고, 컨트롤러는 인터페이스만 의존하도록 변경했습니다.  
`ZoneFileService`, `PromotionObjectService`, `PromotionZoneService` 모두 인터페이스 기반으로 설계했습니다.

```java
// 컨트롤러는 인터페이스에만 의존
public class PromotionController {
    private final ZoneFileService zoneFileService;        // 인터페이스
    private final PromotionObjectService promotionObjectService;  // 인터페이스
}

// 실제 구현은 구현체가 담당 (교체 가능)
@Service
public class ZoneFileServiceImpl implements ZoneFileService {
    ...
}
```

이 구조 덕분에 파일 저장 방식을 로컬 → 외부 URL 기반으로 확장할 때 컨트롤러 수정 없이 서비스 구현체만 추가하면 됐습니다.

---

### 2. 파일 순서 관리 — 드래그앤드롭 정렬 구현

**문제**  
광고 영역 내 파일은 순서가 중요합니다. 기존에는 파일 등록 순서가 표시 순서로 고정되어 있어,  
운영 중 순서 변경이 불가능한 구조였습니다.

**해결**  
프론트에서 드래그앤드롭으로 순서를 변경하면, 변경된 순서 정보를 배열로 백엔드에 전송하는 구조를 설계했습니다.  
서비스 레이어에서 파일 ID별 순서를 일괄 업데이트하는 벌크 처리로 처리했습니다.

```java
// 드래그앤드롭 후 순서 일괄 업데이트
@PutMapping("/api/promotions/files/orders")
@Transactional
public ResponseEntity<Map<String, Object>> updateFileOrders(HttpServletRequest request) {
    List<ZoneFileOrderUpdateDto> orderUpdates = parseOrderUpdates(request);
    int updatedCount = zoneFileService.updateFileOrders(orderUpdates);
    return ResponseEntity.ok(Map.of("updatedCount", updatedCount));
}
```

순서 변경 후 디스플레이 화면에 즉시 반영되는 흐름까지 포함해 구현했습니다.

---

### 3. WebClient 비동기 파일 프록시 — 폐쇄망 환경 대응

**문제**  
운영 환경이 내부망으로 구성되어 있어, 외부 기관(영암군청) 이미지를 디스플레이에 직접 표시할 수 없었습니다.  
클라이언트가 외부 URL에 직접 접근하지 못하는 환경이었습니다.

**해결**  
내부 서버가 외부 이미지를 WebClient로 다운로드한 뒤, 클라이언트에 바이트 스트림으로 전달하는 프록시 구조를 구현했습니다.  
Spring WebFlux의 `DataBuffer`를 활용해 메모리 누수 없이 스트리밍 처리했습니다.

```java
public Mono<byte[]> proxySaveFile(String url) {
    return webClient.get()
            .uri(url)
            .retrieve()
            .bodyToFlux(DataBuffer.class)
            .reduce(new ByteArrayOutputStream(), (baos, buffer) -> {
                try {
                    byte[] bytes = new byte[buffer.readableByteCount()];
                    buffer.read(bytes);
                    baos.write(bytes);
                } catch (IOException e) {
                    throw new RuntimeException("파일 다운로드 중 오류", e);
                } finally {
                    DataBufferUtils.release(buffer); // 메모리 누수 방지
                }
                return baos;
            })
            .map(ByteArrayOutputStream::toByteArray);
}
```

`DataBufferUtils.release()`를 통해 DataBuffer 메모리를 명시적으로 해제하는 것이 포인트였습니다.  
이를 누락하면 WebFlux 환경에서 메모리 누수가 발생한다는 것을 디버깅 과정에서 학습했습니다.

---

### 4. Jsoup 크롤링 + 스케줄러 자동화 — 배너 자동 갱신

**배경**  
영암군청의 메인 배너 이미지가 수시로 변경되는데, 디스플레이에 항상 최신 배너가 표시되어야 했습니다.  
매번 수동으로 이미지를 갱신하는 것은 비효율적이었습니다.

**구현**  
Jsoup으로 영암군청 홈페이지를 파싱해 메인 배너 이미지 URL을 추출하고,  
이를 서버에 저장한 뒤 DB에 등록하는 전 과정을 스케줄러로 자동화했습니다.

```
매일 오전 6시
    ↓
1. 기존 배너 파일 DB에서 삭제
    ↓
2. Jsoup으로 영암군청 메인 배너 img 태그 파싱
    ↓
3. WebClient로 이미지 다운로드 (프록시 경유)
    ↓
4. 로컬 서버에 파일 저장
    ↓
5. ZoneFile DB에 순서대로 등록
```

중복 이미지 제거를 위해 `Set<String>`으로 파일명 기준 중복 검증을 추가했고,  
서버 재시작 시 당일 실행을 놓쳤을 경우를 대비해 `ApplicationListener`로 기동 시 1회 보정 실행하도록 했습니다.

---

### 5. IoT 장비 데이터 수집 스케줄러

**구현**  
산업단지 내 IoT 쉘터 장비들을 주기적으로 폴링해 데이터를 수집하는 스케줄러를 구현했습니다.

| 스케줄러 | 주기 | 역할 |
|---|---|---|
| `getShelterData` | 1분 | 쉘터 장비 센서 데이터 수집 |
| `getPeopleCount` | 1시간 | 피플카운팅 데이터 수집 |
| `getFloatingPopulationData` | 15분 | 유동인구 데이터 수집 |
| `updateYeongamBannerDaily` | 매일 06:00 | 영암군청 배너 자동 갱신 |

장비별 수집 성공/실패 건수를 로그로 남겨 운영 중 이상 징후를 조기에 감지할 수 있도록 했습니다.

```java
for (Map<String, Object> info : infoList) {
    try {
        // 장비 API 호출 및 DB 저장
        success++;
    } catch (Exception e) {
        log.error("[ERROR] 시설 명: {} 데이터 수집 중 오류", info.get("facilityname"), e);
        fail++;
    }
}
log.debug("정상처리: {}개, 오류처리: {}개", success, fail);
```

---

## API 문서화

SpringDoc OpenAPI(Swagger)를 통해 전체 API 명세를 문서화했습니다.  
각 엔드포인트에 `@Operation`, `@Parameter`, `@Tag`를 적용해 파라미터와 응답 구조를 명시했습니다.

---

## 배운 점

이 프로젝트에서 처음으로 도메인 구조를 직접 설계하고 구현해봤습니다.  
홍보물 → 존 → 파일로 이어지는 3계층 구조를 처음에는 단순하게 테이블 3개로 생각했는데,  
실제 구현하다 보니 "어떤 서비스가 어떤 레이어를 책임지는가"를 결정하는 일이 생각보다 복잡했습니다.

한 서비스에 모든 로직을 몰아넣으면 편하지만, 나중에 수정할 때 어디를 건드려야 할지 파악이 안 되는 문제가 생겼습니다.  
이 경험이 SRP와 Interface 분리가 왜 필요한지를 코드 레벨에서 이해하게 해준 계기가 됐습니다.

WebFlux는 기존 MVC 방식과 메모리 처리 방식이 다르다는 것도 이번에 처음 알았습니다.  
`DataBuffer`를 release하지 않으면 메모리 누수가 생기는 문제를 직접 디버깅하면서,  
비동기 스트리밍 환경에서는 리소스 해제를 명시적으로 관리해야 한다는 것을 체감했습니다.
