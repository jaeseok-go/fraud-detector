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

- **역할**: 외부 API 요청을 받아 Kafka에 메시지를 발행
- **실행 위치**: Docker network 내부
- **기능**
    - REST API로 로그 수신 (`POST /logs`)
    - Kafka topic(`fds.logs`)으로 메시지 publish

---

## 📦 3. profile-service

- **역할**: Kafka 로그 메시지를 소비하여 고객 상태(Profile)를 갱신
- **실행 위치**: Docker network 내부
- **기능**
    - `fds.logs` 토픽 구독
    - 고객별 최근 행동 로그/통계 집계
    - log를 해석하여 탐지에 용이한 구조로 정형화
    - ProfileSnapshot은 룰 평가를 위한 View 구조 (룰 추가에 따라 점진적 확장)
    - 탐지 서비스용 상태 조회 API 제공 (`GET /profiles/{customerId}`)

---

## 📦 4. rule-service

- **역할**: 룰 정의 및 평가 로직 담당
- **실행 위치**: Docker network 내부
- **기능**
    - SubRule 구성 및 평가
    - 탐지 서비스가 평가 요청 시 `evaluate(profile, log)` 수행

---

## 📦 5. detect-service

- **역할**: Kafka 로그를 소비하여 룰 평가 → 탐지 이벤트 생성
- **실행 위치**: Docker network 내부
- **기능**
    - `fds.profiles.updated` 토픽 구독
    - `profile-service`에서 상태 조회
    - `rule-service`를 통해 룰 평가
    - 이상 탐지 시 `Detect` 생성 및 저장

---

## 🔗 Kafka Topic 구조

| Topic 이름               | 설명                 |
|------------------------|--------------------|
| `fds.logs`             | 거래/로그인 등 고객 로그 이벤트 |
| `fds.profiles.updated` | 프로파일 업데이트 이벤트      |
---

## 🧭 시스템 흐름 요약

```plaintext
[simulator] ─▶ [ingestor] ─▶ Kafka(fds.logs)
                                   │
                                   └─▶ [profile-service] ─▶ Kafka(fds.profiles.updated)
                                                                 │
                                                                 └─▶[detect-service]
                                                                           │
                                                                           └─▶ [rule-service]