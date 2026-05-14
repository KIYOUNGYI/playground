---
name: ocpp
description: >
  EV 충전 인프라 개발을 위한 OCPP 프로토콜 참조 문서.
  OCPP 2.0.1 및 OCPP 1.6J를 다룹니다. OCPP 메시지, 충전소 코드,
  CSMS/중앙 시스템 백엔드, 스마트 충전, 트랜잭션 처리, EV 충전 프로토콜
  관련 작업 시 사용합니다. 다음 키워드에서 활성화됩니다:
  OCPP, 충전소, 충전 포인트, CSMS, 중앙 시스템, EVSE,
  충전 프로파일, BootNotification, TransactionEvent, StartTransaction,
  StopTransaction, SetChargingProfile 또는 기타 OCPP 메시지 이름.
user-invocable: true
allowed-tools: Read, Grep, Glob
argument-hint: "[topic: smart-charging | authorize | transactions | schemas | sequences | 1.6 | ...]"
---

# OCPP — AI 에이전트 참조 문서

이 스킬은 OCPP를 사용하여 EV 충전 인프라를 개발하는 개발자를 지원합니다.
**OCPP 2.0.1**과 **OCPP 1.6J** 양쪽을 모두 다룹니다. 정확한 스키마 참조, 구현 지침을 제공하고 사양이 명시하지 않는 영역을 표시하는 데 사용합니다.

## 버전 감지

개발자의 코드에서 OCPP 버전을 감지합니다:
- **1.6J 지표:** `StartTransaction`, `StopTransaction`, `RemoteStartTransaction`, `RemoteStopTransaction`, `Charge Point`, `Central System`, `idTag` (문자열), `evseId` 없는 `connectorId`, `.req` / `.conf` 네이밍
- **2.0.1 지표:** `TransactionEvent`, `RequestStartTransaction`, `RequestStopTransaction`, `Charging Station`, `CSMS`, `IdTokenType` (객체), `evseId`, `Request` / `Response` 네이밍

불분명한 경우 개발자에게 어느 버전을 사용하는지 확인합니다.

## 빠른 참조

**OCPP란:** Open Charge Point Protocol — WebSocket + JSON을 통한 EV 충전기와 관리 백엔드 간의 통신 프로토콜. 충전기가 연결을 시작하며, 양측 모두 메시지를 전송할 수 있습니다.

### OCPP 2.0.1
- **역할:** 충전기(CS) ↔ CSMS
- **장치 모델:** 충전기 → EVSE(들) → 커넥터(들). `evseId`와 `connectorId`는 1부터 시작. `evseId=0`은 전체 충전기를 의미.
- **64개 메시지**, 기능 블록별 구성

### OCPP 1.6J
- **역할:** 충전기(CP) ↔ 중앙 시스템(CS)
- **장치 모델:** 충전기 → 커넥터(들). `connectorId`는 1부터 시작. `connectorId=0`은 전체 충전기를 의미. EVSE 개념 없음.
- **28개 메시지**, 기능 프로파일별 구성 (Core, Smart Charging, Firmware, Local Auth List, Reservation, Remote Trigger)

**메시지 프레임 (양 버전 공통):** JSON-RPC 유사 형식. 세 가지 유형:
- `CALL` — `[2, messageId, action, payload]`
- `CALLRESULT` — `[3, messageId, payload]`
- `CALLERROR` — `[4, messageId, errorCode, errorDescription, errorDetails]`

## OCPP 2.0.1 전체 64개 메시지

### 프로비저닝 및 라이프사이클
- `BootNotification` (CS→CSMS) — 충전기가 연결 시 등록
- `Heartbeat` (CS→CSMS) — KeepALive
- `StatusNotification` (CS→CSMS) — 커넥터/EVSE 상태 변경
- `GetVariables` (CSMS→CS) — 설정 읽기
- `SetVariables` (CSMS→CS) — 설정 쓰기
- `GetBaseReport` (CSMS→CS) — 전체 변수 목록 요청
- `NotifyReport` (CS→CSMS) — 변수 목록 응답 (페이징)
- `SetNetworkProfile` (CSMS→CS) — 네트워크 연결 프로파일 설정
- `Reset` (CSMS→CS) — 충전기 재부팅

### 인증
- `Authorize` (CS→CSMS) — ID 토큰 검증
- `ClearCache` (CSMS→CS) — 인증 캐시 초기화
- `SendLocalList` (CSMS→CS) — 로컬 인증 목록 푸시
- `GetLocalListVersion` (CSMS→CS) — 로컬 인증 목록 버전 조회

### 트랜잭션
- `TransactionEvent` (CS→CSMS) — 시작/업데이트/종료 이벤트
- `RequestStartTransaction` (CSMS→CS) — 원격 시작
- `RequestStopTransaction` (CSMS→CS) — 원격 중지
- `GetTransactionStatus` (CSMS→CS) — 미완료 트랜잭션 메시지 조회
- `MeterValues` (CS→CSMS) — 트랜잭션 외부 맥락에서 계량기 값 전송

### 원격 제어
- `TriggerMessage` (CSMS→CS) — CS에 특정 메시지 전송 요청
- `UnlockConnector` (CSMS→CS) — 커넥터 물리적 잠금 해제
- `ChangeAvailability` (CSMS→CS) — EVSE 운영/비운영 설정

### 스마트 충전
- `SetChargingProfile` (CSMS→CS) — 충전 프로파일 설치
- `GetChargingProfiles` (CSMS→CS) — 설치된 프로파일 조회
- `ClearChargingProfile` (CSMS→CS) — 프로파일 제거
- `ClearedChargingLimit` (CS→CSMS) — 외부 제한 해제
- `NotifyChargingLimit` (CS→CSMS) — 외부 제한 알림
- `ReportChargingProfiles` (CS→CSMS) — 프로파일 조회 응답
- `GetCompositeSchedule` (CSMS→CS) — 유효 스케줄 계산
- `NotifyEVChargingSchedule` (CS→CSMS) — EV 제안 스케줄 (ISO 15118)
- `NotifyEVChargingNeeds` (CS→CSMS) — EV 충전 필요사항 보고 (ISO 15118)

### 펌웨어
- `UpdateFirmware` (CSMS→CS) — 펌웨어 업데이트 트리거
- `FirmwareStatusNotification` (CS→CSMS) — 업데이트 진행 상황
- `PublishFirmware` (CSMS→CS) — 로컬 네트워크에 펌웨어 배포
- `PublishFirmwareStatusNotification` (CS→CSMS) — 배포 진행 상황
- `UnpublishFirmware` (CSMS→CS) — 배포된 펌웨어 제거

### 보안 및 인증서
- `Get15118EVCertificate` (CS→CSMS) — EV 인증서 요청
- `GetCertificateStatus` (CS→CSMS) — OCSP 상태 확인
- `SignCertificate` (CS→CSMS) — 충전소 인증서용 CSR
- `CertificateSigned` (CSMS→CS) — 서명된 인증서 전달
- `InstallCertificate` (CSMS→CS) — CA 인증서 설치
- `DeleteCertificate` (CSMS→CS) — 인증서 제거
- `GetInstalledCertificateIds` (CSMS→CS) — 설치된 인증서 목록
- `SecurityEventNotification` (CS→CSMS) — 보안 관련 이벤트 보고

### 진단 및 모니터링
- `GetLog` (CSMS→CS) — 로그 업로드 요청
- `LogStatusNotification` (CS→CSMS) — 로그 업로드 진행 상황
- `NotifyEvent` (CS→CSMS) — 컴포넌트/변수 이벤트
- `SetMonitoringBase` (CSMS→CS) — 모니터링 기준선 설정
- `SetVariableMonitoring` (CSMS→CS) — 변수 모니터 설정
- `SetMonitoringLevel` (CSMS→CS) — 모니터링 심각도 레벨 설정
- `GetMonitoringReport` (CSMS→CS) — 활성 모니터 조회
- `ClearVariableMonitoring` (CSMS→CS) — 모니터 제거
- `NotifyMonitoringReport` (CS→CSMS) — 모니터 조회 응답
- `CustomerInformation` (CSMS→CS) — 고객 데이터 요청
- `NotifyCustomerInformation` (CS→CSMS) — 고객 데이터 응답

### 디스플레이 메시지
- `CostUpdated` (CSMS→CS) — 표시 비용 업데이트
- `SetDisplayMessage` (CSMS→CS) — 디스플레이에 메시지 표시
- `GetDisplayMessages` (CSMS→CS) — 표시 중인 메시지 조회
- `ClearDisplayMessage` (CSMS→CS) — 표시 메시지 제거
- `NotifyDisplayMessages` (CS→CSMS) — 디스플레이 메시지 조회 응답

### 예약
- `ReserveNow` (CSMS→CS) — 예약 생성
- `CancelReservation` (CSMS→CS) — 예약 취소
- `ReservationStatusUpdate` (CS→CSMS) — 예약 만료/제거

### 데이터 전송
- `DataTransfer` (CS↔CSMS) — 양방향 벤더 확장

## OCPP 1.6J 전체 28개 메시지

### Core (필수 프로파일)
- `Authorize` (CP→CS) — idTag 검증
- `BootNotification` (CP→CS) — 충전기가 (재)부팅 후 등록
- `ChangeAvailability` (CS→CP) — 커넥터 운영/비운영 설정
- `ChangeConfiguration` (CS→CP) — 설정 키 값 설정
- `ClearCache` (CS→CP) — 인증 캐시 초기화
- `DataTransfer` (CP↔CS) — 벤더 고유 데이터 교환
- `GetConfiguration` (CS→CP) — 설정 키 읽기
- `Heartbeat` (CP→CS) — KeepAlive
- `MeterValues` (CP→CS) — 주기적 계량기 읽기
- `RemoteStartTransaction` (CS→CP) — 원격 시작
- `RemoteStopTransaction` (CS→CP) — 원격 중지
- `Reset` (CS→CP) — 충전기 재부팅
- `StartTransaction` (CP→CS) — 트랜잭션 시작
- `StatusNotification` (CP→CS) — 커넥터 상태/오류 변경
- `StopTransaction` (CP→CS) — 트랜잭션 종료
- `UnlockConnector` (CS→CP) — 커넥터 물리적 잠금 해제

### 스마트 충전
- `SetChargingProfile` (CS→CP) — 충전 프로파일 설치
- `ClearChargingProfile` (CS→CP) — 프로파일 제거
- `GetCompositeSchedule` (CS→CP) — 유효 스케줄 조회

### 펌웨어 관리
- `GetDiagnostics` (CS→CP) — 진단 로그 업로드 요청
- `DiagnosticsStatusNotification` (CP→CS) — 업로드 진행 상황
- `UpdateFirmware` (CS→CP) — 펌웨어 업데이트 트리거
- `FirmwareStatusNotification` (CP→CS) — 업데이트 진행 상황

### 로컬 인증 목록 관리
- `SendLocalList` (CS→CP) — 로컬 인증 목록 푸시
- `GetLocalListVersion` (CS→CP) — 목록 버전 조회

### 예약
- `ReserveNow` (CS→CP) — 커넥터 예약
- `CancelReservation` (CS→CP) — 예약 취소

### 원격 트리거
- `TriggerMessage` (CS→CP) — CP에 특정 메시지 전송 요청

## 주요 데이터 타입 (OCPP 2.0.1)

- **IdTokenType** — 사용자 식별 정보 (eMAID, RFID 등), 선택적 groupIdToken 포함
- **ChargingProfileType** — 충전 제한: id, stackLevel, purpose, kind, chargingSchedule
- **MeterValueType** — 타임스탬프가 포함된 SampledValue 배열 (에너지, 전력, 전류, 전압, SoC)
- **EVSEType** — EVSE 식별자 (id + 선택적 connectorId)
- **StatusInfoType** — 상태 응답의 이유 코드 + 추가 정보
- **TransactionType** — 트랜잭션 상태: transactionId, chargingState, stoppedReason
- **ChargingScheduleType** — 기간별 시간 기반 전력/전류 제한
- **IdTokenInfoType** — 인증 결과: status, cacheExpiryDateTime, groupIdToken

## 에스컬레이션 모델

OCPP 동작을 구현할 때 사양이 완전히 정의하지 않는 영역을 만나게 됩니다. 이를 다음과 같이 분류합니다:

### SPEC-SILENT (사양 미정의)
OCPP 사양이 이 케이스에 대한 동작을 정의하지 않습니다. 반드시 개발자에게 이를 알려야 합니다. 기본값을 묵시적으로 선택하면 안 됩니다.

### VENDOR-DEPENDENT (벤더 의존)
동작이 충전기 하드웨어나 펌웨어에 따라 다릅니다. 어떤 하드웨어/펌웨어를 대상으로 하는지 확인합니다.

### POLICY-DEPENDENT (정책 의존)
동작이 비즈니스 규칙, 사이트 설정, 또는 그리드 운영자 요구사항에 따라 다릅니다. 비즈니스/운영 맥락을 확인합니다.

### 에스컬레이션 엄격성

개발자 프로젝트의 `CLAUDE.md`에서 에스컬레이션 기본 설정을 확인합니다. 다음과 같이 작성될 수 있습니다:

> OCPP의 경우: 실용적 에스컬레이션 모드를 사용합니다.

또는:

> OCPP 에스컬레이션: 엄격 — 사양 미정의 동작을 가정하기 전에 항상 질문합니다.

두 가지 모드:

- **strict (기본값):** 진행 전 개발자에게 중단하고 질문합니다. 구체적인 옵션을 제시합니다. 답변을 받을 때까지 모호한 영역의 코드를 작성하지 않습니다.
- **pragmatic:** 모호함을 표시하되 합리적인 기본값을 선택합니다. 다음과 같은 가시적 주석을 남깁니다:
  ```
  // OCPP SPEC-SILENT: [가정에 대한 설명]. 요구사항과 일치하는지 확인하세요.
  ```

`CLAUDE.md`에 에스컬레이션 기본 설정이 없으면 **strict**를 기본값으로 사용합니다.

## 문서 파일 맵

필드 레벨 스키마, 시퀀스 다이어그램, 또는 실제 예시가 필요한 경우 플러그인의 `docs/` 디렉토리에서 관련 파일을 읽습니다. `${CLAUDE_PLUGIN_ROOT}`를 사용하여 경로를 확인합니다.

| 주제 | 읽을 파일 |
|------|-----------|
| **공유 데이터 타입 전체 (열거형 + 복합형)** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-DataTypes.md` |
| **인증 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Authorization.md` |
| **가용성 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Availability.md` |
| **진단 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Diagnostics.md` |
| **디스플레이 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Display.md` |
| **펌웨어 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Firmware.md` |
| **프로비저닝 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Provisioning.md` |
| **예약 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Reservation.md` |
| **보안 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Security.md` |
| **스마트 충전 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-SmartCharging.md` |
| **트랜잭션 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Schemas/OCPP-2.0.1-Schemas-Transactions.md` |
| **부팅, 인증, 트랜잭션 시퀀스** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Sequences/OCPP-2.0.1-Sequences.md` |
| **오프라인, 펌웨어, 진단 시퀀스** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-Sequences/OCPP-2.0.1-Sequences-Operational.md` |
| **스마트 충전 심층 분석** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-SmartCharging/OCPP-2.0.1-SmartCharging.md` |
| **스마트 충전 실제 예시** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-SmartCharging/OCPP-2.0.1-SmartCharging-Examples.md` |
| **ISO 15118 + 스마트 충전** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1-SmartCharging/OCPP-2.0.1-SmartCharging-ISO15118.md` |
| **OCPP 2.0.1 개요 + 마이그레이션 가이드** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-2.0.1.md` |
| **문서 방법론 + 신뢰 모델** | `${CLAUDE_PLUGIN_ROOT}/docs/METHODOLOGY.md` |
| | |
| **OCPP 1.6J 개요 + 설정 키** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J.md` |
| **1.6J Core 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-Core.md` |
| **1.6J 스마트 충전 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-SmartCharging.md` |
| **1.6J 펌웨어 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-Firmware.md` |
| **1.6J 로컬 인증 목록 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-LocalAuthList.md` |
| **1.6J 예약 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-Reservation.md` |
| **1.6J 원격 트리거 스키마** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Schemas/OCPP-1.6J-Schemas-RemoteTrigger.md` |
| **1.6J 메시지 시퀀스** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-Sequences/OCPP-1.6J-Sequences.md` |
| **1.6J 스마트 충전 심층 분석** | `${CLAUDE_PLUGIN_ROOT}/docs/OCPP-1.6J-SmartCharging/OCPP-1.6J-SmartCharging.md` |

### 파일 맵 사용 방법

1. 개발자의 질문에서 주제 파악
2. Read 도구를 사용하여 관련 파일 읽기
3. 문서에서 특정 필드, 제약 조건, 열거값 인용
4. 문서에서 발견한 ESCALATE 마커 처리

### 주제 인수 라우팅

`/ocpp <topic>`으로 호출 시 관련 파일을 즉시 읽습니다:

**OCPP 1.6J 주제:**
- `/ocpp 1.6` 또는 `/ocpp 1.6j` → OCPP-1.6J.md 개요 읽기
- `/ocpp 1.6 smart-charging` → 1.6J SmartCharging 스키마 + 심층 분석 읽기
- `/ocpp 1.6 schemas` → 모든 1.6J 스키마 파일 읽기
- `/ocpp 1.6 sequences` → 1.6J 시퀀스 파일 읽기
- `/ocpp 1.6 core` → 1.6J Core 스키마 읽기
- `/ocpp 1.6 firmware` → 1.6J 펌웨어 스키마 읽기
- `/ocpp 1.6 auth-list` → 1.6J LocalAuthList 스키마 읽기
- `/ocpp 1.6 reservation` → 1.6J 예약 스키마 읽기

**OCPP 2.0.1 주제 (기본값):**
- `/ocpp smart-charging` → 3개의 SmartCharging 파일 모두 읽기
- `/ocpp authorize` 또는 `/ocpp authorization` → 인증 스키마 읽기
- `/ocpp transactions` → 트랜잭션 스키마 + 시퀀스 읽기
- `/ocpp provisioning` 또는 `/ocpp boot` → 프로비저닝 스키마 + 시퀀스 읽기
- `/ocpp schemas` → 모든 스키마 파일 읽기
- `/ocpp sequences` → 두 개의 시퀀스 파일 모두 읽기
- `/ocpp types` 또는 `/ocpp data-types` → DataTypes 읽기
- `/ocpp firmware` → 펌웨어 스키마 + 운영 시퀀스 읽기
- `/ocpp diagnostics` → 진단 스키마 + 운영 시퀀스 읽기
- `/ocpp reservation` → 예약 스키마 읽기
- `/ocpp display` → 디스플레이 스키마 읽기
- `/ocpp security` 또는 `/ocpp certificates` → 보안 스키마 읽기
- `/ocpp availability` → 가용성 스키마 읽기
- 기타 주제 → grep을 사용하여 모든 문서에서 검색

## 행동 지침

1. **항상 출처를 인용합니다.** 필드, 타입, 또는 제약 조건을 참조할 때 어느 문서에서 왔는지 언급합니다. 스키마 기반 사실(높은 신뢰도)과 AI 해석(낮은 신뢰도)을 구분합니다.

2. **에스컬레이션 모델을 준수합니다.** 문서에서 `> **ESCALATE:**` 마커를 만나면 위의 에스컬레이션 엄격성 규칙을 따릅니다.

3. **컨텍스트에서 버전을 감지합니다.** 위의 버전 감지 규칙을 사용합니다. 코드에서 `StartTransaction`/`StopTransaction`을 사용하면 1.6J — 1.6J 문서를 읽습니다. `TransactionEvent`를 사용하면 2.0.1입니다. 버전 지표가 없으면 2.0.1로 가정하고 가정을 언급합니다.

4. **프로토콜 동작을 임의로 만들지 않습니다.** 무언가가 사양에 정의되어 있는지 확실하지 않으면 먼저 문서를 확인합니다. 문서에서 다루지 않으면 추측하지 않고 명시적으로 그렇게 말합니다.

5. **스키마를 검증에 사용합니다.** 개발자가 OCPP 메시지 페이로드를 작성할 때 스키마 문서와 대조하여 필드 이름, 타입, 필수/선택 상태, 제약 조건을 검증합니다.