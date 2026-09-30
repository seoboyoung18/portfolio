<h1 align="center">📜 서보영 포트폴리오</h1>
<p align="center">
  서보영(Boyoung Seo) · <a href="https://github.com/seoboyoung18">seoboyoung18</a>
</p>

---

## 👋 Intro

안녕하세요! **추측 대신 측정으로 판단하는** 백엔드 개발자 서보영입니다.

성능 문제를 만나면 코드부터 고치는 대신 구간을 나눠 재고, 원인을 찾은 뒤에도 그것이 정말 원인인지 검증하고 나서 손을 댑니다. 그 과정에서 **왜 그렇게 판단했는지를 코드 주석과 문서에 남기는 것**을 습관으로 삼고 있습니다.

주로 다룬 문제는 **동시성과 멱등성**입니다. 같은 요청이 두 번 들어와도 결과가 하나여야 하는 상황 — 소셜 가입, AI 결과 콜백, 견적 재산정 — 을 DB 제약과 조건부 UPDATE로 풀어 왔습니다.

**소셜 로그인은 두 프로젝트에서 구현하면서 방식을 다르게 골랐습니다.** 냉큼은 JWT, 노카는 세션 + Redis입니다. 노카에서 세션을 고른 이유는 **탈퇴·강제 로그아웃 때 그 회원의 세션을 찾아 끊어야 했기 때문**입니다 — JWT는 발급 후 서버가 무효화할 수단이 없습니다.

---

## 📝 Projects

### 1. 🚗 노카(NOCA) — AI 자동차 수리비 견적 서비스

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

### 2. 🏠 방방봐(BangBangBwa) — 원격 매물 검수 서비스

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

### 3. 🥬 냉큼(Naeng-Keum) — AI 스마트 냉장고 서비스

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

**Database**

![PostgreSQL](https://img.shields.io/badge/PostgreSQL-4169E1?style=for-the-badge&logo=postgresql&logoColor=white)
![MySQL](https://img.shields.io/badge/MySQL-4479A1?style=for-the-badge&logo=mysql&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)

**Infra & DevOps**

![AWS](https://img.shields.io/badge/AWS_EC2_S3-232F3E?style=for-the-badge&logo=amazonwebservices&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![Nginx](https://img.shields.io/badge/Nginx-009639?style=for-the-badge&logo=nginx&logoColor=white)
![GitLab CI](https://img.shields.io/badge/GitLab_CI-FC6D26?style=for-the-badge&logo=gitlab&logoColor=white)
![JUnit5](https://img.shields.io/badge/JUnit5-25A162?style=for-the-badge&logo=junit5&logoColor=white)

**Tools**

![Git](https://img.shields.io/badge/Git-F05032?style=for-the-badge&logo=git&logoColor=white)
![IntelliJ IDEA](https://img.shields.io/badge/IntelliJ_IDEA-000000?style=for-the-badge&logo=intellijidea&logoColor=white)
![Notion](https://img.shields.io/badge/Notion-000000?style=for-the-badge&logo=notion&logoColor=white)
![Claude](https://img.shields.io/badge/Claude-D97757?style=for-the-badge&logo=claude&logoColor=white)

---

## 📞 Contact

[![Email](https://img.shields.io/badge/tjqhdud2580@gmail.com-EA4335?style=for-the-badge&logo=gmail&logoColor=white)](mailto:tjqhdud2580@gmail.com)
[![GitHub](https://img.shields.io/badge/seoboyoung18-181717?style=for-the-badge&logo=github&logoColor=white)](https://github.com/seoboyoung18)
