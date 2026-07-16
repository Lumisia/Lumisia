# Backend Developer

> TPS 7.0 → 117.4 — 개선을 말로 하지 않고 **측정으로 증명**합니다.
> 개선 전·후는 물론, 같은 코드로 Redis와 Valkey 대안까지 nGrinder로 비교 검증했습니다.

![Java 17](https://img.shields.io/badge/Java_17-007396?style=flat-square&logo=openjdk&logoColor=white)
![Spring Boot 3](https://img.shields.io/badge/Spring_Boot_3-6DB33F?style=flat-square&logo=springboot&logoColor=white)
![Spring Data JPA](https://img.shields.io/badge/Spring_Data_JPA-6DB33F?style=flat-square&logo=spring&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=flat-square&logo=redis&logoColor=white)
![Valkey](https://img.shields.io/badge/Valkey-5D67E8?style=flat-square)
![MariaDB](https://img.shields.io/badge/MariaDB-003545?style=flat-square&logo=mariadb&logoColor=white)
![Kubernetes](https://img.shields.io/badge/Kubernetes-326CE5?style=flat-square&logo=kubernetes&logoColor=white)
![nGrinder](https://img.shields.io/badge/nGrinder-555555?style=flat-square)

[![Portfolio](https://img.shields.io/badge/Portfolio-lumisia.github.io-2e2e33?style=flat-square&logo=githubpages&logoColor=white)](https://lumisia.github.io)
[![Blog](https://img.shields.io/badge/Blog-Tistory-EB531F?style=flat-square&logo=tistory&logoColor=white)](https://lumisia.tistory.com)
[![Email](https://img.shields.io/badge/Email-zh2171%40gmail.com-EA4335?style=flat-square&logo=gmail&logoColor=white)](mailto:zh2171@gmail.com)

---

## 🔨 Now — 수료 후에도 계속 만들고 있습니다

### ✅ FileinNOut 단독 운영 고도화 (2026.06)

수료 직후, 팀 프로젝트 **FileinNOut**을 개인 리포로 이어받아 실제 운영 수준으로 끌어올렸습니다.

- **SSE 연결 안정화** — 화면별 중복 연결을 단일 연결로 통합하고 heartbeat를 추가해, 배포·절체 시 재연결 폭주 문제를 해결. 동작은 회귀 테스트(`SseHeartbeatTest` 등)로 고정
- **OAuth 쿠키 트러블슈팅** — 서브도메인 콜백 인증 실패를 상위 도메인 쿠키 공유로 해결, Tomcat RFC6265 거부로 인한 500 오류 교정
- **보안·운영 정비** — DB·MinIO 자격증명을 ConfigMap에서 Secret으로 분리하고, 회전 시 소비 Pod가 함께 재기동되도록 checksum 적용
- **CI/CD·인프라** — GitHub Actions 멀티아치(amd64+arm64) buildx 빌드로 전환, OCI ARM64 노드에 Helm 배포
- **관측 스택 공개 운영** — Prometheus·Grafana·Jaeger·Kiali·k8s Dashboard·Swagger를 방문자 읽기 전용으로 공개 → [포트폴리오에서 바로 접속](https://lumisia.github.io/projects/fileinnout/)

### 🔄 Callog 알림 파이프라인 — 이벤트 기반 재설계 (2026.07 ~)

Redis Pub/Sub 기반 알림을 Kafka 중심 구조로 전환합니다. **Kafka는 이번 고도화로 처음 도입하는 스택**이라, 학습과 병행해 설계 결정(ADR)과 함께 단계별로 기록합니다.

- [ ] 알림 이벤트 발행을 Redis Pub/Sub → **Kafka + Outbox 패턴**으로 전환 — DB 트랜잭션과 발행의 정합성 보장
- [ ] 컨슈머 장애 대응 — **재시도 + DLT + 멱등 컨슈머** 설계, 장애 시나리오 직접 재현
- [ ] **Testcontainers** 기반 Kafka·Redis 통합 테스트 구축
- [ ] 전환 전·후 nGrinder 부하 비교 측정

<!-- TODO: 고도화 리포에 첫 커밋이 올라간 뒤 아래 줄의 주석을 해제하고 실제 리포 링크로 교체
→ 진행 기록: [고도화 리포](https://github.com/Lumisia/Callog-advanced) · [블로그](https://lumisia.tistory.com)
-->

---

## 🧩 제가 해결해 본 문제들

### 1. 대시보드 조회 병목 — 전수 조회 제거와 캐시 설계

**Callog** (캠페인 협업 플랫폼) · 담당: 캠페인·대시보드·KPI·Redis

- **문제** — 한 화면에서 5개 도메인을 집계하는 API가 `findAll()` 후 JVM 필터링 + 캠페인별 반복 조회로 구성되어, 100 VU 기준 평균 응답 11초, 에러 44건 발생
- **접근** — 조회 범위를 `IN`·`COUNT`·`GROUP BY` 쿼리로 DB에 위임 → 통합 응답을 사용자·분기·버전 키로 Redis 캐시(`sync=true`로 Cache Stampede 방지) → 변경 사용자 범위만 버전 증가시키는 선택적 무효화
- **검증** — Redis 프로토콜 호환성을 이용해 동일 코드로 Valkey까지 비교 측정

| 구분 | 평균 TPS | 평균 응답 | 에러 |
|---|---|---|---|
| 개선 전 | 7.0 | 11,033ms | 44 |
| JPA 최적화 + Redis | 97.3 | 995ms | 16 |
| 동일 코드 + Valkey | **117.4** | **787ms** | **0** |

→ [트러블슈팅 전문 보기](https://lumisia.github.io/projects/callog/)

### 2. 실시간 동시 편집 정합성 — 병합 단위 재설계

**FileinNOut** (실시간 협업 문서 플랫폼) · 담당: 워크스페이스 도메인·WebSocket 백엔드

- **문제** — 문서 전체를 하나의 String으로 저장해, 마지막 저장자가 다른 사용자의 편집을 덮어쓰고 사용자마다 화면이 달라짐
- **접근** — 편집 잠금 / LWW / OT / CRDT를 비교 후 **CRDT(Yjs)** 채택. 병합 단위를 문서 전체에서 **블록 단위 Y.Map**으로 재설계하고, id 기준 diff로 변경 블록만 감지·반영
- **결과** — 동시 편집 유실 0, 변경 블록만 렌더링, Redis Pub/Sub으로 다중 WebSocket 서버에서도 동일 문서 상태 공유
- **+ 목록 조회 개선** — 30초 폴링을 SSE 푸시로, 전체 엔티티 조회를 Projection + 페이지네이션으로 전환 → TPS 32.7 → 192.4 (5.9배), 평균 응답 89% 감소

→ [트러블슈팅 전문 보기](https://lumisia.github.io/projects/fileinnout/)

### 3. 다중 Pod 상태 조정 — 분산 락과 이벤트 전파

**Callog** · Kubernetes 다중 Pod 환경

- **문제** — 모든 Pod가 같은 `@Scheduled`를 실행해 KPI 스냅샷이 중복 생성되고, SSE 연결이 Pod 로컬 메모리에만 있어 알림이 누락됨
- **접근** — UUID 토큰 + Lua 스크립트 검증 방식의 **Redis 분산 락**으로 스케줄러 단일 실행 보장(TTL 만료 후 타 Pod 락 오해제 방지), **Redis Pub/Sub**으로 Pod 간 SSE 이벤트 중계, 발행 실패 시 로컬 전달 fallback 유지

→ [트러블슈팅 전문 보기](https://lumisia.github.io/projects/callog/)

---

## 🚀 Projects

| 프로젝트 | 소개 | 담당 | 링크 |
|---|---|---|---|
| **Callog** | 캠페인 기획·KPI·검수를 잇는 B2B 협업 플랫폼 (MSA·K8s) | 캠페인 · 대시보드 · KPI · Redis | [Repo](https://github.com/Lumisia/Callog) · [Live](https://www.magamcallog.kro.kr/) · [상세](https://lumisia.github.io/projects/callog/) |
| **FileinNOut** | 실시간 협업 문서·파일 워크스페이스 플랫폼 (수료 후 단독 고도화 중) | 워크스페이스 · WebSocket 백엔드 | [Repo](https://github.com/Lumisia/FileinNOut) · [Live](https://lumisia.fileinnout.com/) · [상세](https://lumisia.github.io/projects/fileinnout/) |

---

## 🎓 Education & Certification

- **한화시스템 BEYOND SW 캠프 24기** (2025.12 – 2026.06) — Java · Spring Boot · MSA · Kubernetes 백엔드 심화
- **서울디지털대학교 컴퓨터공학과** 졸업 (2025.02)
- **정보처리기사** (2025.12)

---

## 📫 Contact

커피챗과 채용 제안 모두 환영합니다.

**Portfolio** [lumisia.github.io](https://lumisia.github.io) · **Blog** [lumisia.tistory.com](https://lumisia.tistory.com) · **Email** [zh2171@gmail.com](mailto:zh2171@gmail.com) · **Algorithm** [solved.ac/zh2171](https://solved.ac/profile/zh2171)

<!--
운영 규칙 (본인용 메모, 렌더링 안 됨)
1. Now-완료(✅) 섹션은 실제 커밋이 존재하는 항목만 기재. Now-진행(🔄) 체크리스트는 커밋과 동기화하고, 완료 항목은 [x] + 결과 수치 한 줄 추가.
2. Kafka는 Callog 고도화에서 Outbox·DLT까지 완료된 뒤에만 상단 스택 라인에 추가. 그 전에는 "처음 도입·학습 병행" 표현 유지 (면접 정직성).
3. 케이스에 새 항목을 추가할 땐 "문제 유형" 기준 유지 (프로젝트 나열 금지).
4. 분기마다 링크 생존 확인 (Live 데모 죽으면 스크린샷 대체).
-->
