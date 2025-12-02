# WebTest

## 프로젝트 개요  
WebTest는 웹사이트의 성능·보안 지표를 자동 분석하고, 수집된 데이터를 기반으로 AI 개선안을 생성하는 웹 진단 플랫폼입니다. 크롬 확장 프로그램으로 웹사이트를 검사하면, 백엔드의 비동기 파이프라인이 지표 수집·저장·AI 분석을 순차적으로 수행하며 결과를 전달합니다. 이 프로젝트에서는 실서비스 수준의 롱폴링 기반 비동기 구조와 트랜잭션 시점 제어(TxAfterCommit), 메트릭 기반 관측 환경을 직접 설계했습니다.

## 주요 기능  
- 웹 성능(Web Vitals)·보안(Security Headers) 자동 수집  
- 크롬 확장 프로그램 기반 원클릭 진단  
- CORE_READY / AI_READY 롱폴링 비동기 파이프라인  
- OpenAI API 기반 AI 개선안 생성  
- 지표 및 결과 대시보드 시각화  
- 메트릭 기반 장애 진단 및 상태 추적  

## 기술 스택  

### 백엔드  
- Spring Boot  
- PostgreSQL  
- JPA  
- Long Polling, DeferredResult, TxAfterCommit  
- Actuator, Micrometer  
- Prometheus, Grafana  
- Docker, Nginx  

### 프론트엔드  
- React  
- Chrome Extension  

### 인프라  
- AWS EC2  
- RDS (PostgreSQL)  
- Docker Compose  
- GitHub Actions  
- Nginx Reverse Proxy  

---

## 담당 역할  

### 1. 백엔드 아키텍처 설계 및 핵심 로직 구현  
- Long Polling 기반 비동기 파이프라인 전체 설계  
- CORE_READY / AI_READY 상태 전달 로직 개발  
- TxAfterCommit 기반 트랜잭션 후 이벤트 발행 구조 구현  
- JPA 도메인 모델링 및 API·DB 스키마 설계  

### 2. 장애 대응 및 관측 환경 구축  
- traceId 기반 요청 로깅 설계 및 병목 구간 추적  
- Actuator·Micrometer 기반 커스텀 메트릭(webtest.pipeline.*) 설계  
- Prometheus·Grafana 기반 대시보드 구성  
- READY 성공/실패·타임아웃 비율 시각화  

### 3. 협업 및 일정 관리 (백엔드 팀장)  
- 기능 단위 이슈 분리·스토리 포인트 산정  
- GitHub Projects 기반 스프린트·칸반 운영  
- 코드 리뷰 체계 정립 및 팀 내 개발 흐름 관리  

---

## 트러블슈팅  

WebTest에서 가장 큰 문제는 **롱폴링 요청이 반복적으로 타임아웃되는 현상**이었습니다. 프론트에서는 CORE_READY와 AI_READY가 제때 응답되지 않으며 30초 대기 시간 후 타임아웃이 발생했습니다.

### 1. traceId 기반 로깅으로 문제 흐름 추적  
- complete 이벤트는 정상 발생  
- 그러나 이벤트 발생 시 대기 중 요청이 없어 응답이 소실  
- 이후 들어온 요청은 READY 상태를 알 수 없어 계속 대기 → 타임아웃 반복  

### 2. 원인 ① 레이스 컨디션  
이벤트 시점과 요청 도착 시점이 어긋나는 구조적 문제  
→ **사전 상태 체크(isAlreadyReady)** 추가  
→ 뒤늦게 도착한 요청도 즉시 응답  

### 3. 원인 ② 트랜잭션 커밋 전 이벤트 발행  
DB에 결과가 저장되기 전 READY 신호가 발행되는 문제  
→ **TxAfterCommit 도입**  
→ 이벤트는 “DB 커밋 이후”에만 발행하도록 보장  

### 4. 개선 결과  
| 항목 | 개선 전 | 개선 후 |
|------|---------|---------|
| 롱폴링 타임아웃 | 빈번히 발생 | 거의 0% |
| AI_READY 누락 | 반복 발생 | 완전 해소 |
| 이벤트-DB 정합성 | 불일치 발생 | 일관성 확보 |
| 문제 파악 난이도 | 높은 편 | traceId로 전체 흐름 가시화 |

---

## 핵심 개선 요약  
- **사전 상태 체크** → 늦게 도착한 요청 즉시 응답  
- **TxAfterCommit** → 비동기 이벤트 순서·정합성 보장  
- **커스텀 메트릭(webtest.pipeline.*)** → 파이프라인 상태 진단  
- **Grafana 대시보드** → 장애 패턴 시각화 및 모니터링  
