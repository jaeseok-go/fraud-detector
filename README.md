# 🛡 FDS-System Modules Overview

본 프로젝트는 Kafka 기반의 이상 거래 탐지 시스템(Fraud Detection System)을 목표로 구성된 토이 프로젝트입니다.  
각 모듈은 도메인 책임을 기준으로 분리되어 있으며, 이벤트 기반 아키텍처를 통해 느슨한 결합을 유지합니다.

---

## 📦 1. simulator

- **역할**: 외부 시스템을 모사하여 로그 이벤트를 HTTP API로 전송
- **실행 위치**: Docker network 바깥
- **기능**
    - 테스트용 고객 로그 (거래, 로그인 등) 생성
    - `ingestor` 모듈로 HTTP POST 요청 발송

---

## 📦 2. ingestor

- **역할**: 외부 요청을 수신하고, 거래 처리 흐름을 제어
- **실행 위치**: Docker network 내부
- **기능**
  - REST API로 로그 수신 (POST /logs)
  - rule-service에 blocking 여부 질의
  - non-blocking: Kafka topic(`fds.transactions`)에 메시지 publish
  - blocking: profile-service 및 detect-service 연계 호출 후 결과 응답 반환

---

## 📦 3. profile-service

- **역할**: Kafka 메시지를 기반으로 고객 프로파일 상태 갱신
- **실행 위치**: Docker network 내부
- **기능**
  - `fds.transactions` 토픽 구독 
  - 고객별 최근 행동 로그/통계 집계
  - 탐지에 용이한 구조로 상태 정형화 (ProfileSnapshot)
  - 룰 기반 탐지용 상태 조회 API 제공 (GET /profiles/{customerId})
  - 프로파일 갱신 후 Kafka topic(`profile.updated`)에 이벤트 발행

---

## 📦 4. rule-service

- **역할**: 룰 정의 및 평가 로직 담당
- **실행 위치**: Docker network 내부
- **기능**
  - 거래의 blocking 여부 판단
  - SubRule 구성 및 평가
  - 탐지 서비스가 평가 요청 시 `evaluate(profile, log)` 수행

---

## 📦 5. detect-service

- **역할**: 룰 평가를 통해 이상 거래 탐지 및 조치
- **실행 위치**: Docker network 내부
- **기능**
  - `profile.updated` 토픽 구독
  - rule-service를 통해 룰 평가
  - 이상 탐지 시 Detect 생성 및 저장
  - blocking 흐름에서는 Ingestor에 탐지 결과 응답 반환

---

## 🔗 Kafka Topic 구조

| Topic 이름               | 설명                 |
|------------------------|--------------------|
| `fds.transactions`     | 거래/로그인 등 고객 로그 이벤트 |
| `fds.profiles.updated` | 프로파일 업데이트 이벤트      |
---

## 🧭 시스템 흐름 요약

```plaintext
[simulator] ─▶ [ingestor] ────────────────────────────────────────────▶ [rule-service]
                   └─▶ Kafka(fds.transactions)
                            │
                            └─▶ [profile-service] ─▶ Kafka(fds.profiles.updated)
                                                          │
                                                          └─▶[detect-service]
                                                                    │
                                                                    └─▶ [rule-service]
```

---

## ⚖ Blocking / Non-Blocking 처리
- Ingestor는 거래 수신 후 rule-service에 blocking 여부 질의
- 룰에 따라 `고위험 거래는 blocking`, `일반 거래는 non-blocking`으로 분기
- Blocking: profile-service → detect-service 순차 호출 후 응답 반환
- Non-Blocking: Kafka(`fds.transactions`) 발행 후 비동기 처리
- 탐지 정확도와 처리 효율성 균형을 위한 구조