# I2C Master/Slave (RTL) — Design Spec & Verification Spec

<p align="center">
  <img alt="I2C" src="https://img.shields.io/badge/bus-I2C-blue">
  <img alt="RTL" src="https://img.shields.io/badge/language-SystemVerilog-orange">
  <img alt="ADDR" src="https://img.shields.io/badge/address-7bit-informational">
  <img alt="FSM" src="https://img.shields.io/badge/FSM-START%2FADDR%2FDATA%2FACK%2FSTOP-green">
</p>

> **What this is**  
> 내부 제어 신호(`I2C_En / I2C_start / I2C_stop / length / addr / tx_data`)를 입력으로 받아  
> I2C 프로토콜(START/STOP, Address+R/W, ACK/NACK, Data)을 생성하는 **I2C Master**와,  
> 버스에서 START/STOP을 감지하고 Address match 시 ACK 및 READ/WRITE를 수행하는 **I2C Slave** RTL입니다.

---

## Table of Contents

- [DESIGN SPEC](#design-spec)
  - [1. Overview](#1-overview)
  - [2. Protocol Summary](#2-protocol-summary)
  - [3. Interfaces](#3-interfaces)
    - [3.1 I2C_MASTER Ports](#31-i2c_master-ports)
    - [3.2 I2C_SLAVE Ports](#32-i2c_slave-ports)
  - [4. Master FSM & Behavior](#4-master-fsm--behavior)
    - [4.1 States](#41-states)
    - [4.2 Transaction Flow](#42-transaction-flow)
    - [4.3 Length (Burst) Rule](#43-length-burst-rule)
  - [5. Slave FSM & Behavior](#5-slave-fsm--behavior)
    - [5.1 Address / ACK / READ-WRITE](#51-address--ack--read-write)
    - [5.2 Repeated START / STOP Detection](#52-repeated-start--stop-detection)
  - [6. Key Assumptions / Limitations](#6-key-assumptions--limitations)

- [VERIFICATION SPEC](#verification-spec)
  - [7. Verification Requirements (Shall)](#7-verification-requirements-shall)
  - [8. Assertions (SVA) Checklist](#8-assertions-sva-checklist)
  - [9. Functional Coverage](#9-functional-coverage)
  - [10. Test Plan](#10-test-plan)

- [IMPLEMENTATION](#implementation)
  - [11. RTL Code](#11-rtl-code)

- [DOC / REFERENCES](#doc--references)
  - [12. My Docs (Images)](#12-my-docs-images)
  - [13. Public References](#13-public-references)

---

## DESIGN SPEC

## 1. Overview

**Modules**
- `I2C_MASTER.sv` : I2C Master (SCL 생성 + SDA 구동/해제 + ACK/NACK 처리 + Burst length)
- `I2C_SLAVE.sv`  : I2C Slave  (START/STOP 감지 + 주소 수신 + ACK + READ/WRITE 데이터 처리)

**Bus Signals**
- `SCL` : Master가 생성하는 Clock
- `SDA` : Data line (tri-state 사용)

**Reset**
- 두 모듈 모두 `posedge reset` 기반 (active-high, async 스타일)

---

## 2. Protocol Summary

I2C 기본 규칙(요약):
- **START**: `SCL=HIGH`에서 `SDA: 1→0`
- **STOP** : `SCL=HIGH`에서 `SDA: 0→1`
- **Data Valid**: 일반 데이터는 `SCL=HIGH` 구간에서 SDA 안정(stable)  
  (데이터 변경은 `SCL=LOW`에서, 단 START/STOP 예외)
- **ACK/NACK**: 8비트 전송 후 9번째 클럭에서 ACK(0)/NACK(1)

---

## 3. Interfaces

### 3.1 I2C_MASTER Ports

| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `clk` | in | 1 | system clock |
| `reset` | in | 1 | active-high reset |
| `I2C_En` | in | 1 | 트랜잭션 시작(Enable) |
| `addr` | in | 7 | 7-bit slave address |
| `CR_RW` | in | 1 | 0=WRITE, 1=READ |
| `tx_data` | in | 8 | WRITE data byte |
| `tx_done` | out | 1 | 바이트 단위 전송 완료 pulse |
| `tx_ready` | out | 1 | IDLE에서 1 (시작 가능) |
| `rx_data` | out | 8 | READ data byte |
| `rx_done` | out | 1 | 바이트 단위 수신 완료 pulse |
| `I2C_start` | in | 1 | HOLD/HOLD2에서 다음 동작 트리거(continue / repeated start) |
| `I2C_stop` | in | 1 | HOLD에서 STOP 트리거 |
| `length` | in | `$clog2(D_LENGTH)+1` | Burst length(바이트 개수) |
| `SCL` | out | 1 | I2C clock |
| `SDA` | inout | 1 | I2C data (tri-state) |

**Request Acceptance Rule**
- `I2C_En==1`이 **IDLE에서** 들어오면 START부터 진행
- 주소 프레임은 내부에서 `{addr, CR_RW}`로 자동 구성

---

### 3.2 I2C_SLAVE Ports

| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `clk` | in | 1 | system clock |
| `reset` | in | 1 | active-high reset |
| `tx_data` | in | 8 | READ 요청 시 Slave가 내보낼 데이터 |
| `tx_done` | out | 1 | 바이트 송신 완료 pulse |
| `tx_ready` | out | 1 | (현재 RTL에서 의미 거의 없음/미사용) |
| `rx_data` | out | 8 | WRITE로 수신한 데이터 |
| `rx_done` | out | 1 | 바이트 수신 완료 pulse |
| `scl` | in | 1 | I2C clock |
| `sda` | inout | 1 | I2C data (tri-state) |

**Slave Address**
- 현재 RTL 고정: `7'b1111110 (0x7E)`
- 포트폴리오 버전에서는 parameter화 추천

---

## 4. Master FSM & Behavior

### 4.1 States

`I2C_MASTER`는 아래 상태로 구성됩니다.

- `ST_IDLE`      : `tx_ready=1`, `I2C_En` 대기
- `ST_START1/2`  : START 조건 생성(SDA↓ while SCL=1)
- `ST_WRITE`     : 바이트 송신(ADDR 또는 DATA)
- `ST_ACK`       : Address ACK 샘플링 + 다음 동작 분기
- `ST_WRITE_ACK` : Data ACK 샘플링 + 다음 바이트/종료 분기
- `ST_READ`      : 바이트 수신(SDA 샘플링)
- `ST_READ_ACK`  : Master가 ACK/NACK 구동(마지막 바이트면 NACK)
- `ST_HOLD2`     : 다음 tx_data 로딩 대기 (`I2C_start`로 진행)
- `ST_HOLD`      : STOP 또는 repeated START 선택
- `ST_STOP1/2`   : STOP 생성

> 내부 구현은 `clk_counter(249/499)`와 `data_cnt(0~3)`로 SCL 토글/샘플 구간을 분할해 타이밍을 만듭니다.

---

### 4.2 Transaction Flow

#### (A) WRITE 흐름 (Master → Slave)
1. `IDLE`에서 `I2C_En=1`  
2. START 생성 (`ST_START1/2`)
3. Address frame `{addr, CR_RW=0}` 송신 (`ST_WRITE`)
4. Address ACK 확인 (`ST_ACK`)
5. 데이터 전송은 `ST_HOLD2`에서 `I2C_start`를 받아 `tx_data` 로딩 후 `ST_WRITE`로 진행
6. 매 바이트 후 ACK 확인 (`ST_WRITE_ACK`)
7. `length_reg==0`이면 `ST_HOLD`로 이동 → `I2C_stop`이면 STOP

#### (B) READ 흐름 (Slave → Master)
1. `IDLE`에서 `I2C_En=1`
2. START 생성
3. Address frame `{addr, CR_RW=1}` 송신
4. Address ACK 확인 후 `ST_READ`
5. 8비트 수신 완료 후 `ST_READ_ACK`에서 ACK/NACK 생성  
   - 마지막 바이트(`length_reg==0`)면 **NACK**
6. `ST_HOLD`로 이동 → STOP 또는 repeated START 선택

---

### 4.3 Length (Burst) Rule

- `length = N` : “데이터 바이트 N개” 전송/수신을 의미  
- Address ACK 성공 시점부터 내부 `length_reg`가 감소하며 진행됩니다.
- WRITE: 각 Data ACK 성공마다 `length_reg--`, 0이면 HOLD에서 종료 선택
- READ : 각 byte 수신 후 `ST_READ_ACK`에서 `length_reg==0`이면 NACK 후 HOLD

> 사용 팁: WRITE는 바이트마다 `I2C_start`로 “다음 바이트 진행” 트리거를 주는 구조(`HOLD2`)입니다.

---

## 5. Slave FSM & Behavior

### 5.1 Address / ACK / READ-WRITE

Slave는 START 후 8비트(Address[7:1] + R/W[0])를 수신합니다.

- Address match(`addr_reg[7:1]==7'b1111110`)이면 ACK(SDA=0)
- R/W=0 : `RCV_DATA`로 들어가 WRITE 데이터 수신
- R/W=1 : `SEND_DATA`로 들어가 READ 데이터 송신

WRITE 수신:
- 8비트 수신 후 ACK를 내보내고 다음 바이트 수신 대기

READ 송신:
- `tx_data`를 MSB-first로 송신
- 8비트 송신 후 Master ACK를 샘플링(`RCV_ACK`)
  - ACK(0)이면 다음 바이트 계속(Burst read)
  - NACK(1)이면 종료 방향

---

### 5.2 Repeated START / STOP Detection

Slave는 동작 중에도 아래를 감지합니다.
- `SCL=1 && SDA rising`  → STOP으로 판단
- `SCL=1 && SDA falling` → repeated START로 판단하고 ADDR로 복귀

---

## 6. Key Assumptions / Limitations

- **Open-Drain 정확 모델링 한계**
  - `SDA`는 tri-state를 쓰지만, 일부 구간에서 `O_SDA=1`로 직접 구동도 발생할 수 있음  
  - 권장: 상위 TB에서 `pullup(sda)`를 추가하고, “LOW만 구동 / HIGH는 Z”로 정교화
- **Clock stretching 미지원**, **multi-master arbitration 미지원**
- **10-bit address 미지원** (7-bit only)
- Slave address가 현재 고정(0x7E) → parameter화 추천
- Slave의 burst read 종료 정책은 일부 주석 처리 구간이 있어(ACK 이후 STOP 처리) 개선 여지 있음

---

## VERIFICATION SPEC

## 7. Verification Requirements (Shall)

| Req ID | Requirement (Shall) | Suggested Check Method |
|---|---|---|
| I2C-FR-001 | reset 시 Master/Slave는 IDLE로 복귀하고 내부 카운터/레지스터 초기화 | Directed + SVA |
| I2C-FR-010 | Master는 `I2C_En`이 IDLE에서 1이면 START를 생성해야 함 | SVA + waveform |
| I2C-FR-020 | START/STOP 조건은 SCL=HIGH 구간에서 SDA edge로 정의되어야 함 | SVA(temporal) |
| I2C-FR-030 | Master는 `{addr, CR_RW}` 8비트를 MSB-first로 전송해야 함 | Scoreboard |
| I2C-FR-040 | 9th clock에서 ACK/NACK를 샘플/생성해야 함 | SVA + Scoreboard |
| I2C-FR-050 | WRITE: 각 바이트 전송 후 ACK 성공이면 length가 감소하고 다음 바이트 진행 가능해야 함 | Scoreboard |
| I2C-FR-060 | READ: 각 바이트 수신 후 마지막 바이트면 NACK를 생성해야 함 | Scoreboard |
| I2C-FR-070 | Slave는 address match 시 ACK를 생성해야 하며 mismatch면 transaction을 종료해야 함 | Directed |
| I2C-FR-080 | repeated START 발생 시 Slave는 ADDR 수신으로 복귀해야 함 | Directed |
| I2C-FR-090 | STOP 이후 버스가 idle(SCL=1,SDA=1) 상태로 복귀해야 함 | SVA |

---

## 8. Assertions (SVA) Checklist

추천 SVA 체크리스트(요약):

- **A1**: Master `IDLE`에서 `I2C_En` → START 상태 진입
- **A2**: START 조건: `SCL==1`일 때 `SDA 1->0`
- **A3**: STOP 조건 : `SCL==1`일 때 `SDA 0->1`
- **A4**: 데이터 비트 동안 `SCL==1` 구간에서 SDA 안정(stable) (START/STOP 제외)
- **A5**: Address ACK 샘플은 9th clock에서만 수행
- **A6**: READ 마지막 바이트는 NACK 생성 (`length_reg==0`이면 SDA=1 구동)
- **A7**: Slave는 address mismatch면 ACK를 내지 않아야 함(또는 STOP로 전이)

> 구현 팁: “SCL rising edge 카운트(8/9비트)” 기반 assertion이 제일 깔끔합니다.

---

## 9. Functional Coverage

- **C1**: READ/WRITE 각각 hit
- **C2**: length binning: 1,2,3,4+ 바이트
- **C3**: ACK/NACK 케이스(ACK 성공, NACK/주소불일치)
- **C4**: repeated START hit
- **C5**: STOP hit

---

## 10. Test Plan

- **T1**: reset sanity (SCL/SDA idle 확인)
- **T2**: WRITE 1 byte (ACK 성공)
- **T3**: WRITE N bytes (length>1, HOLD2 + I2C_start 반복)
- **T4**: READ 1 byte (마지막 NACK 확인)
- **T5**: READ N bytes (NACK가 마지막에서만 발생)
- **T6**: address mismatch (Slave가 ACK 안 함/STOP 처리)
- **T7**: repeated START 시나리오 (STOP 없이 START 재생성)
- **T8**: STOP 타이밍/idle 복귀 확인

---

## IMPLEMENTATION

## 11. RTL Code

<details>
  <summary><b>Click to expand RTL (I2C_MASTER / I2C_SLAVE)</b></summary>

> ✅ 아래에 `I2C_MASTER.sv`, `I2C_SLAVE.sv` 원문 코드를 그대로 붙이면 됩니다.

</details>

---

## DOC / REFERENCES

## 12. My Docs (Images)

> 아래 파일명을 맞춰 `docs/` 폴더에 넣으면 README에서 바로 렌더링됩니다.

- `docs/i2c_protocol.png`
- `docs/i2c_timing.png`
- `docs/i2c_master_asm.png`

예시:
```md
![I2C Protocol](docs/i2c_protocol.png)
![I2C Timing](docs/i2c_timing.png)
![I2C Master ASM](docs/i2c_master_asm.png)
