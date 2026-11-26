<h1 align="center">📜 서보영 포트폴리오</h1>
<p align="center">
  서보영(Boyoung Seo) · <a href="https://github.com/seoboyoung18">seoboyoung18</a>
</p>

---

## 👋 Intro

안녕하세요! **추측 대신 측정으로 판단하는** 백엔드 개발자 서보영입니다.

성능 문제를 만나면 코드부터 고치는 대신 구간을 나눠 재고, 원인을 찾은 뒤에도 그것이 정말 원인인지 검증하고 나서 손을 댑니다. 그 과정에서 **왜 그렇게 판단했는지를 코드 주석과 문서에 남기는 것**을 습관으로 삼고 있습니다.

주로 다룬 문제는 **동시성과 멱등성**입니다. 같은 요청이 두 번 들어와도 결과가 하나여야 하는 상황 — 소셜 가입, AI 결과 콜백, 견적 재산정, 발자취 저장, AR 팁 등록 — 을 DB 제약과 조건부 UPDATE, 행 잠금, 멱등키로 풀어 왔습니다. 여기록에서는 잠금을 뺀 비교용 빌드를 따로 만들어 **잠금이 정말 필요한지까지 실험으로 확인**했습니다.

**소셜 로그인은 두 프로젝트에서 구현하면서 방식을 다르게 골랐습니다.** 냉큼은 JWT, 노카는 세션 + Redis입니다. 노카에서 세션을 고른 이유는 **탈퇴·강제 로그아웃 때 그 회원의 세션을 찾아 끊어야 했기 때문**입니다 — JWT는 발급 후 서버가 무효화할 수단이 없습니다.

---

## 📝 Projects

### 1. 🧭 여기록(Yeogirok) — 여행 코스 공유 · 기록 Android 앱

> 다른 여행자의 코스에서 가고 싶은 곳만 골라 일행과 일정을 짜고, 다녀온 뒤 사진과 영수증으로 여행기를 만들어 다시 코스로 공개하는 앱

- **개발기간** : 2026.10 ~ 2026.11 (SSAFY 자율 프로젝트)
- **팀 구성** : 5명 (FE 2 · BE 2 · AI 1, 인프라는 FE · BE에서 1명씩 겸임)
- **핵심 역할** : 백엔드(여행 기록 · 코스 공개 · 공개 코스 조회 · 발자취 · AR 팁 · 파일 업로드), 인프라(S3 파일 저장소 · 외부 API 연동)

**Back-end**

- Language : Java 21
- Skill : Spring Boot 4.0, Spring Security(JWT), Spring Data JPA, PostgreSQL 16, Flyway, Redis, AWS S3(Presigned URL)
- 외부 연동 : 카카오 로컬 API(역지오코딩 · 장소 검색), 한국관광공사 TourAPI, 영수증 OCR 서버(FastAPI · PaddleOCR)
- Infra : AWS EC2, Docker Compose, Nginx, Jenkins, GitHub Actions
- Test : JUnit5, Testcontainers(PostgreSQL · Redis), k6

**담당 구현**

| 영역 | 내용 |
|---|---|
| 여행 기록 | 사진의 촬영 시각 · 위치(EXIF), 영수증 OCR 결과, 계획 일정을 합쳐 **방문 초안 생성** — 시간 · 거리 기준으로 사진과 영수증을 묶고, 영수증 주소로 카카오 장소 후보를 찾음 |
| 코스 공개 | 완성한 여행기를 코스로 공개(사진 공개 여부 선택 · 공개 미리보기), 관광공사 TourAPI 코스를 공식 계정 코스로 적재 |
| 발자취 | 같은 날 500m 안 중복 저장 거부 — **회원 행 `FOR UPDATE` 잠금**, 역지오코딩 같은 외부 호출은 트랜잭션 · 잠금 밖에서 실행 |
| AR 팁 | 접수 즉시 202를 주고 커밋 뒤 **비동기 게시**, 중간에 끊긴 팁을 찾아 재시도 가능 상태로 돌리는 복구 작업 |
| 멱등 처리 | Redis `SET NX` + 요청 해시로 같은 요청의 재전송을 한 번만 처리 (컨트롤러 12개 · 28곳) |
| 파일 | Presigned URL 업로드 세션 · 완료 확인 · 조회 URL, 사진과 영수증 버킷 분리 |

**성과 — 측정으로 확인한 것**

PC 한 대에서 앱 · PostgreSQL · Redis · k6 를 함께 돌린 **같은 조건의 전후 비교**입니다. 운영 서버의 절대 수치가 아닙니다.

- **조회 인덱스** : 발자취 · AR 팁 조회가 전부 전체 스캔이던 것을 찾아, 인덱스 후보 4종을 1천 ~ 100만 건에서 비교한 뒤 덮개 인덱스를 선택
  → 10만 건 · 같은 유입률에서 **주변 팁 P95 약 26ms → 5.4ms**, 내 발자취 · 내 팁 목록 P95 15 ~ 17ms → 6 ~ 7ms, 방문 분포(서울 화면) P95 101 ~ 243ms → 50 ~ 55ms
  → 전국 단위 집계는 인덱스로 줄지 않는다는 것도 함께 확인해 한계로 남김
- **동시성** : 같은 사용자의 발자취 저장 5 ~ 500건을 한꺼번에 보내는 실험 18회에서 **모두 1건만 저장**. 잠금 한 줄을 뺀 비교용 빌드는 18회 모두 규칙이 깨짐(4 ~ 10건 저장)
- **멱등 장애 재현** : Redis 완료 기록이 사라지면 같은 멱등키로 AR 팁이 2건 생기는 것을 재현하고, DB 유일 제약으로 막음
- **종료 · 복구** : 강제 종료로 끊긴 팁 1,230건이 모두 복구 대상으로 잡혀 재시도 뒤 게시됨(유실 · 중복 0). 정상 종료에서도 게시 작업이 끊기는 원인(스프링 종료 순서)을 찾아 **끊긴 팁 142 ~ 272건 → 0건**
- **장시간 부하** : 초당 40건을 60분(144,027건) 보내는 동안 실패 0건, 지연 · 힙 · 커넥션이 시간에 따라 늘지 않음
- 테스트 **2,120건** (동시성 · 멱등 · 마이그레이션 포함)

[프로젝트 상세 설명](https://github.com/seoboyoung18/Yeogirok) · [측정 · 검증 문서](https://github.com/seoboyoung18/Yeogirok/tree/main/docs/engineering)

---

### 2. 🚗 노카(NOCA) — AI 자동차 수리비 견적 서비스

> 사진 한 장으로 예상 수리비를 확인하고, 정비소에서 무엇을 물어봐야 하는지 알려주는 모바일 웹 서비스

- **개발기간** : 2026.08.25 ~ 2026.09.29 (5주)
- **팀 구성** : 6명 (BE 2 · FE 2 · AI 2)
- **핵심 역할** : 견적 도메인 전반, AI 분석 비동기 파이프라인, 소셜 인증, 유사 사례 근거, LLM 리포트 요약

**Back-end**

- Language : Java 21
- Skill : Spring Boot 4.1, Spring Security(OAuth2), Spring Data JPA, PostgreSQL, Redis(Spring Session), AWS S3, Thymeleaf + openhtmltopdf, Gemini(LLM)
- Infra : Docker, GitLab CI/CD, AWS EC2, Nginx

**담당 구현**

| 영역 | 내용 |
|---|---|
| 견적 | 재산정 시 과거 견적을 덮지 않는 **버전 관리**, 항목별 산정 근거 JSONB 스냅샷, 부분 견적 안내, PDF 생성 |
| AI 연동 | 접수 후 즉시 응답하고 폴링 워커가 AI를 호출하는 **비동기 파이프라인**, 멱등 콜백 수신, 재시도·타임아웃 종결 |
| 인증 | 카카오·구글 소셜 로그인, 약관 동의 후 가입, **동시 가입 중복 방어**, 탈퇴 시 즉시 익명화 |
| 유사 사례 | AI가 참조한 사례 ID를 스냅샷으로 보존해 **근거가 나중에 바뀌지 않게** 설계 |
| LLM | 견적 리포트 요약 — 원문에 없던 숫자가 섞이면 버리는 **환각 방어** |

**성과**

- 분석 요청 지연의 원인을 **구간 분리 측정**으로 스케줄러 스레드 경합으로 특정하고, 통제 실험으로 인과를 검증한 뒤 개선
  → `접수→전송` **중앙값 6.13초 → 1.56초 (3.9배)**, 구조상 상한인 폴링 주기 3초를 넘던 값이 전부 주기 이내로
- 테스트 **1,727건** 유지 (동시성·보안·포트 계약·스키마 대조 포함)
- `@Lob` 매핑 문제로 팀 전체가 스키마 검증을 끄고 있던 원인을 찾아 해결 — **테스트 204건 중 87건 실패 → 전체 통과**

[프로젝트 상세 설명](https://github.com/seoboyoung18/Noca)

---

### 3. 🏠 방방봐(BangBangBwa) — 원격 매물 검수 서비스

> 영상통화를 검수 프로세스로 바꿔, 라이브 중 구조화된 점검과 AI 보조·자동 리포트를 제공하는 서비스

- **개발기간** : 2026.07.15 ~ 2026.08.09 (4주)
- **팀 구성** : 6명 (BE 3 · FE 3)
- **핵심 역할** : 매물(property) · 관심 매물(favorite) · 서류 AI 분석(document) 도메인

**Back-end**

- Language : Java 21
- Skill : Spring Boot 4.1, Spring Data JPA, PostgreSQL, Spring Security(OAuth2), AWS S3, Spring WebSocket, openhtmltopdf
- Infra : AWS EC2, Docker Compose, Nginx, coturn(WebRTC TURN), Claude API, RunPod(YOLO)

**주요 기능**

- 실시간 영상통화 중 양쪽이 공유하는 체크리스트
- 하자(곰팡이·균열·누수) AI 탐지 보조
- 서류 AI 분석과 위험도 신호(안전 / 주의 / 위험)
- 공간별 확인 상태·캡처·AI 결과를 담은 자동 검수 리포트

[프로젝트 상세 설명](https://github.com/seoboyoung18/BangBangBwa) · [PORTFOLIO.md](https://github.com/seoboyoung18/BangBangBwa/blob/main/PORTFOLIO.md)

---

### 4. 🥬 냉큼(Naeng-Keum) — AI 스마트 냉장고 서비스

> "냉장고 속 남은 재료로 큼지막한 행복을!" — 1인 가구를 위한 식재료 관리와 AI 레시피 큐레이션

- **개발기간** : 2026.05.25 ~ 2026.06.26 (약 5주)
- **팀 구성** : 2명
- **핵심 역할** : 풀스택 — 소셜 인증, AI 레시피 추천, 레시피·냉장고·관리자 도메인

**Full-stack**

- Language : Java 21, JavaScript
- Back-end : Spring Boot 4, MyBatis, MySQL 8, Flyway, JWT, OAuth2 Client
- Front-end : Vue 3(Vite), Pinia, Vue Router, Axios
- Infra : AWS EC2, Docker Compose, Nginx, Let's Encrypt, DuckDNS
- AI : SSAFY GMS(OpenAI 호환), SSE 스트리밍

**담당 구현**

| 영역 | 내용 |
|---|---|
| 인증 | **OAuth2 소셜 로그인**(구글·카카오) 백엔드 — 핸들러·회원 연동·**JWT 발급**, 회원 탈퇴 |
| 레시피 | 공개 시 사용 재료 **냉장고 재고 자동 차감**, 작성자 프로필 카드 |
| AI | 레시피 추천 장르 확장 (한식 → 양식·일식·중식) |
| 냉장고 | 조미료 보유관리, 단위(g/ml/개) 선택 |
| 관리자 | 회원 삭제·역할 변경, 운영자 권한 재적용 |
| 운영 | **Flyway 마이그레이션** 관리(V4 락 이슈 수정·V14 권한 적용), API 명세·ERD 문서화 |

2인 팀에서 **백엔드 전반을 맡고 프론트엔드도 함께** 작업했습니다.

[프로젝트 상세 설명](https://github.com/seoboyoung18/naengkeum)

---

## 🛠 Tech Stack

**Backend**

![Java](https://img.shields.io/badge/Java_21-007396?style=for-the-badge&logo=openjdk&logoColor=white)
![Spring Boot](https://img.shields.io/badge/Spring_Boot-6DB33F?style=for-the-badge&logo=springboot&logoColor=white)
![Spring Security](https://img.shields.io/badge/Spring_Security-6DB33F?style=for-the-badge&logo=springsecurity&logoColor=white)
![JPA](https://img.shields.io/badge/JPA_Hibernate-59666C?style=for-the-badge&logo=hibernate&logoColor=white)
![MyBatis](https://img.shields.io/badge/MyBatis-C74634?style=for-the-badge)
![Flyway](https://img.shields.io/badge/Flyway-CC0200?style=for-the-badge&logo=flyway&logoColor=white)

**Database**

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

**Infra & DevOps**

![AWS](https://img.shields.io/badge/AWS_EC2_S3-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![GitLab CI](https://img.shields.io/badge/GitLab_CI-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white)
![Jenkins](https://img.shields.io/badge/Jenkins-D24939?style=for-the-badge&logo=jenkins&logoColor=white)
![JUnit5](https://img.shields.io/badge/JUnit5-25A162?style=for-the-badge&logo=junit5&logoColor=white)
![k6](https://img.shields.io/badge/k6-7D64FF?style=for-the-badge&logo=k6&logoColor=white)

**Tools**

![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![IntelliJ IDEA](https://img.shields.io/badge/IntelliJ_IDEA-000000?style=for-the-badge&logo=intellijidea&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-D97757?style=for-the-badge&logo=claude&logoColor=white)

---

## 📞 Contact

[![Email](https://img.shields.io/badge/tjqhdud2580@gmail.com-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:tjqhdud2580@gmail.com)
[![GitHub](https://img.shields.io/badge/seoboyoung18-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/seoboyoung18)
