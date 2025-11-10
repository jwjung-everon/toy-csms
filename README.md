# 🧩 Toy-CSMS MSA 구조 개요

## 1. 서비스 구성 요약

| 서비스 | 주요 역할 | 핵심 책임 | 주요 이벤트 or 인터페이스 |
|:--|:--|:--|:--|
| **CSMS (Central System Management Service)** | 충전기와의 OCPP 프로토콜 통신 담당 | - 충전기 세션 관리<br>- 충전 시작/종료 요청 처리<br>- MeterValue 수신 및 전달 | - `StartTransaction`<br>- `MeterValue`<br>- `StopTransaction` |
| **Charging-Session Service** | 충전 세션 상태 및 요금 산출 관리 | - 세션별 누적 충전량 관리<br>- 실시간 요금 계산<br>- 세션 종료 시 Billing 요청 발행 | - Kafka Topic: `charging.session` |
| **Billing Service** | 결제 및 포인트 처리 담당 | - 결제 금액 산정<br>- 결제 로그 기록 | - Kafka Topic: `billing.payment` |
| **Notification Service** *(Optional)* | 사용자 알림 발송 | - 충전/결제 완료 시 알림톡 or 이메일 발송<br>- Billing 이벤트 수신 | - Kafka Topic: `csms.transaction` |

---

## 2. 이벤트 기반 흐름 (요약)

1. **충전 시작**
    - CSMS가 충전기로부터 `StartTransaction` 수신
    - → Charging-Session 에 세션 생성 이벤트 전송

2. **충전 진행 중**
    - 충전기에서 `MeterValue` 반복 송신
    - CSMS → Charging-Session 으로 전달
    - Session Repository(Redis)에 충전량/요금 누적

3. **충전 종료**
    - CSMS가 `StopTransaction` 수신
    - Charging-Session 이 세션 종료 처리 후 Billing 서비스로 이벤트 발행
        - Topic: `charging.session`
        - payload: { transactionId, memberUuid, totalEnergy, totalFee }

4. **결제 처리**
    - Billing 서비스가 `charging.session` 이벤트 소비
    - 포인트 차감 및 결제 트랜잭션 생성
    - 결과 이벤트(`billing.payment`) 발행

5. **결제 결과 전파**
    - CSMS 또는 Notification 서비스가 `billing.payment` 이벤트 소비
    - 충전 세션 종료 상태 업데이트 / 알림 발송

---

## 3. 데이터 도메인 경계

| 데이터 종류 | 책임 서비스 | 저장소 | 설명                |
|:--|:--|:--|:------------------|
| 충전 세션 상태 | Charging-Session | Redis (TTL) | 실시간 세션 캐싱 및 요금 누적 |
| 결제 트랜잭션 | Billing | PostgreSQL | 결제 로그 및 포인트 내역    |
| 사용자 및 차량 정보 | CSMS | PostgreSQL | 회원 & 차량 인증 정보     |
| 이벤트 로그 | 각 서비스별 | Kafka | 서비스 간 비동기 통신 채널   |

---

## 4. 서비스 간 의존 관계 요약
```text
[CSMS]
│
├──→ charging.session → [Charging-Session]
│                           │
│                           ├──→ billing.payment → [Billing]
│                           │
│                           └──(StopTransaction 응답)
│
└──← billing.payment ←───────┘
```

---
## 5. 구조도 이미지
<img width="1322" height="1056" alt="image" src="https://github.com/user-attachments/assets/ebed23d8-1ce7-433d-abb8-29b4b6c0dec3" />
