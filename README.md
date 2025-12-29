# I2C Master/Slave (RTL) — Design Spec & Verification Spec

<p align="center">
  <img alt="I2C" src="https://img.shields.io/badge/bus-I2C-blue">
  <img alt="RTL" src="https://img.shields.io/badge/language-SystemVerilog-orange">
  <img alt="OpenDrain" src="https://img.shields.io/badge/line-Open--Drain%20%2B%20Pull--Up-green">
  <img alt="FSM" src="https://img.shields.io/badge/FSM-Master%2FSlave%20FSM-purple">
</p>

> **What this is**  
> I2C(2-wire) 버스에서 **Master 1개 + Slave 1개**를 RTL(SystemVerilog)로 구현한 프로젝트입니다.  
> Master는 **START/STOP, Address(7b)+R/W, ACK/NACK, Write/Read, Burst(length)**를 생성하고,  
> Slave는 **START/STOP 검출(동기화+엣지검출), Address match, ACK 응답, Write 수신 / Read 송신**을 수행합니다.

---

## Table of Contents

- [DESIGN SPEC](#design-spec)
  - [1. Overview](#1-overview)
  - [2. I2C Basics (Protocol Summary)](#2-i2c-basics-protocol-summary)
  - [3. Interfaces](#3-interfaces)
  - [4. Timing & Bit Transfer Rule](#4-timing--bit-transfer-rule)
  - [5. Master FSM & Behavior](#5-master-fsm--behavior)
  - [6. Slave FSM & Behavior](#6-slave-fsm--behavior)
  - [7. Key Assumptions / Limitations](#7-key-assumptions--limitations)

- [VERIFICATION SPEC](#verification-spec)
  - [8. Verification Requirements (Shall)](#8-verification-requirements-shall)
  - [9. Assertions (SVA) Checklist](#9-assertions-sva-checklist)
  - [10. Functional Coverage](#10-functional-coverage)
  - [11. Test Plan](#11-test-plan)

- [IMPLEMENTATION](#implementation)
  - [12. RTL Code](#12-rtl-code)
  - [13. Block Diagrams (Optional)](#13-block-diagrams-optional)

- [DOC](#doc)
  - [14. References](#14-references)

---

## DESIGN SPEC

## 1. Overview

### 1.1 Modules
- **I2C Master**: `I2C_MASTER.sv`
- **I2C Slave**:  `I2C_SLAVE.sv`

### 1.2 Feature Summary (based on RTL)
**Master**
- START/STOP 생성
- 7-bit Address + R/W(=CR_RW) 전송
- ACK 샘플링(ACK=0 성공)
- Write: 데이터 전송 + ACK 확인
- Read: 데이터 수신 + ACK/NACK 생성
- Burst(length): length 값 기준으로 연속 byte 처리

**Slave**
- SDA/SCL 입력 동기화(2FF) + 엣지 검출로 START/STOP 감지
- Address 수신 후 match 시 ACK 구동
- Write 시 데이터 수신 + ACK 구동
- Read 시 데이터 송신, Master ACK 기반으로 burst_read 유지(ACK=0이면 연속 송신)

---

## 2. I2C Basics (Protocol Summary)

### 2.1 2-wire bus
- **SCL**: Clock
- **SDA**: Data
- 라인은 기본적으로 **Pull-up**에 의해 HIGH가 되고, 디바이스는 **LOW만 당기는(Open-Drain)** 방식

### 2.2 Start / Stop condition
- **START**: SCL=HIGH 동안 SDA가 **1→0**
- **STOP** : SCL=HIGH 동안 SDA가 **0→1**

### 2.3 Bit transfer rule
- 일반 데이터 비트는 **SCL=HIGH 구간에서 SDA가 안정(stable)** 해야 함
- SDA 변경은 **SCL=LOW 구간**에서만 (START/STOP 예외)

### 2.4 Frame (7-bit Address)
- `ADDR[6:0] + R/W(1bit)` 총 8비트 전송
- 다음 9번째 클럭에서 ACK/NACK
  - **ACK=0**: 수신 성공
  - **NACK=1**: 거부/종료

---

## 3. Interfaces

## 3.1 I2C Master (`I2C_MASTER.sv`)

### Global
- `clk` : system clock
- `reset` : async reset (posedge reset)

### Internal control/data
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `I2C_En` | in | 1 | transaction 시작 요청 |
| `addr` | in | 7 | 7-bit slave address |
| `CR_RW` | in | 1 | 0=WRITE, 1=READ |
| `tx_data` | in | 8 | write data byte |
| `tx_done` | out | 1 | write byte done tick |
| `tx_ready` | out | 1 | IDLE 상태에서 1 (ready) |
| `rx_data` | out | 8 | read data byte |
| `rx_done` | out | 1 | read byte done tick |
| `I2C_start` | in | 1 | HOLD/HOLD2에서 다음 데이터/재시작 제어 |
| `I2C_stop` | in | 1 | HOLD에서 STOP 제어 |
| `length` | in | clog2(D_LENGTH)+1 | burst length (byte count) |

### External
- `SCL` : output clock
- `SDA` : inout data (open-drain 스타일로 tri-state)

---

## 3.2 I2C Slave (`I2C_SLAVE.sv`)

### Global
- `clk`, `reset`

### Internal
| Signal | Dir | Width | Description |
|---|---:|---:|---|
| `tx_data` | in | 8 | slave가 READ 응답으로 내보낼 데이터 |
| `tx_done` | out | 1 | byte 송신 완료 tick |
| `tx_ready` | out | 1 | (현재 RTL에서 실사용/구동 미흡: 개선 여지) |
| `rx_data` | out | 8 | master가 WRITE로 보낸 데이터 수신 |
| `rx_done` | out | 1 | byte 수신 완료 tick |

### External
- `scl` : input SCL
- `sda` : inout SDA

---

## 4. Timing & Bit Transfer Rule

### 4.1 Master SCL generation (based on RTL)
Master는 `clk_counter`와 `data_cnt`로 SCL을 토글합니다.

- `clk_counter`가 0→249까지 카운트될 때마다 sub-phase 진행
- `data_cnt`가 **0 또는 2**일 때 SCL 토글
- 결과적으로 SCL의 1주기 ≈ **1000 * clk_period**

> 즉, SCL 주파수는 대략 `F_SCL ≈ F_CLK / 1000` (코드 구조 기준)

### 4.2 SDA direction (tri-state)
- Master: ACK 수신 / READ 구간에서는 SDA를 `Z`로 두고 샘플
- Slave: ACK 송신 / DATA 송신 구간에서만 SDA drive

---

## 5. Master FSM & Behavior

### 5.1 States (from RTL)
- `ST_IDLE`
- `ST_START1`, `ST_START2`
- `ST_WRITE`
- `ST_ACK` (Address ACK)
- `ST_WRITE_ACK` (Data ACK)
- `ST_READ`
- `ST_READ_ACK` (Master ACK/NACK)
- `ST_HOLD`, `ST_HOLD2` (STOP or RESTART or next byte control)
- `ST_STOP1`, `ST_STOP2`

### 5.2 High-level flow
**Address Phase**
1) IDLE에서 `I2C_En=1` → START → `tx_data_reg={addr,CR_RW}` 전송  
2) `ST_ACK`에서 ACK 샘플 (`ACK=0`이면 성공)

**Write Flow (CR_RW=0)**
- `ST_HOLD2`에서 `I2C_start=1` 시 `tx_data` 전송(`ST_WRITE`)
- `ST_WRITE_ACK`에서 ACK 확인 후
  - `length==0`이면 HOLD로 이동하여 STOP/RESTART 대기
  - `length>0`이면 다음 byte를 위해 HOLD2로 이동

**Read Flow (CR_RW=1)**
- `ST_READ`에서 SDA 샘플링으로 `rx_data_reg` shift-in
- `ST_READ_ACK`에서
  - 남은 length가 있으면 **ACK(0)**, 다음 byte 계속 READ
  - 마지막이면 **NACK(1)** 후 HOLD로 이동

---

## 6. Slave FSM & Behavior

### 6.1 Start/Stop detect
동기화된 SDA/SCL 기준으로:
- SCL=HIGH에서 SDA falling → START
- SCL=HIGH에서 SDA rising → STOP

### 6.2 Address handling
- Address byte(8b) 수신 후 `addr_reg[7:1]` 비교
- RTL 상 slave address는 고정:
  - `slave_addr = 7'b1111110`

### 6.3 Data handling
- WRITE(R/W=0): `RCV_DATA`로 8비트 수신 후 `SEND_ACK_2`로 ACK
- READ(R/W=1): `SEND_DATA`로 8비트 송신 후 `RCV_ACK`
  - Master가 ACK(0)면 `burst_read` 유지하여 다음 byte 송신
  - NACK(1)면 종료 방향(현 RTL은 STOP으로 강제 이동 대신 HOLD 형태: 개선 가능)

---

## 7. Key Assumptions / Limitations

- **Multi-master arbitration 미지원**
- **Clock stretching 미지원**
- **Pull-up 저항/라인 RC 모델은 RTL에서 직접 모델링하지 않음**  
  (실제 HW에서는 외부 Pull-up 필수)
- Slave address가 RTL에서 **고정값(7'b1111110)** (파라미터화 여지)
- Slave의 `tx_ready`는 현재 RTL에서 적극적으로 사용되지 않음(정리/개선 여지)

---

## VERIFICATION SPEC

## 8. Verification Requirements (Shall)

| Req ID | Requirement (Shall) | Suggested Check Method |
|---|---|---|
| I2C-FR-001 | reset 시 Master/Slave는 IDLE로 복귀하고 SDA/SCL을 idle 상태로 유지해야 함 | Directed + SVA |
| I2C-FR-010 | START/STOP 조건을 I2C 정의대로 생성/검출해야 함 | Monitor + SVA |
| I2C-FR-020 | Address(7b)+R/W 전송 후 9th clock에서 ACK 샘플링/응답을 수행해야 함 | Scoreboard + SVA |
| I2C-FR-030 | WRITE: byte 전송 후 ACK 확인, burst length에 따라 연속 전송/종료를 수행해야 함 | Scoreboard |
| I2C-FR-040 | READ: byte 수신 후 Master가 ACK/NACK을 생성하고 length 기반으로 연속 READ/종료를 수행해야 함 | Scoreboard |
| I2C-FR-050 | SCL=HIGH 동안 SDA는 START/STOP을 제외하고 안정(stable)해야 함 | SVA |
| I2C-FR-060 | Slave는 address match 시 ACK를 drive(LOW)하고 mismatch면 응답하지 않아야 함 | Directed + SVA |
| I2C-FR-070 | Burst read에서 Slave는 Master ACK(0)일 때 다음 데이터를 이어서 송신해야 함 | Scoreboard |

---

## 9. Assertions (SVA) Checklist

추천 assertion 아이디어(핵심만):

- **A1 (START)**: `SCL==1 && SDA`가 `1->0`이면 START로 판단
- **A2 (STOP)** : `SCL==1 && SDA`가 `0->1`이면 STOP로 판단
- **A3 (SDA stable when SCL high)**: START/STOP 제외하고 `SCL==1` 동안 `SDA` 변화 금지
- **A4 (ACK timing)**: 9번째 비트에서 ACK는 `SCL==1` 구간에서 샘플되어야 함
- **A5 (Open-drain rule)**: 어떤 디바이스도 SDA를 강제로 '1'로 밀지 않고, drive 시 0만 허용(모델링 정책에 따라)

---

## 10. Functional Coverage

- **C1**: R/W 분기 hit (WRITE/READ)
- **C2**: Address match/mismatch
- **C3**: ACK/NACK 케이스
- **C4**: burst length binning (0, 1, 2, 4+)
- **C5**: STOP vs RESTART 경로 hit

---

## 11. Test Plan

- **T1**: reset sanity (master/slave idle)
- **T2**: address match + 1-byte write
- **T3**: address match + 1-byte read (NACK 종료)
- **T4**: burst write (length=2/4)
- **T5**: burst read (length=2/4) + master ACK 동작 확인
- **T6**: address mismatch (slave ACK 없어야 함)
- **T7**: restart 시나리오 (STOP 없이 repeated start)

---

## IMPLEMENTATION

## 12. RTL Code

<details>
  <summary><b>Click to expand RTL (I2C_MASTER)</b></summary>

```systemverilog
`timescale 1ns / 1ps

module I2C_MASTER #(
    parameter D_LENGTH = 2
) (
    // global ports
    input  logic                      clk,
    input  logic                      reset,
    // internal ports
    input  logic                      I2C_En,
    input  logic [               6:0] addr,
    input  logic                      CR_RW,
    input  logic [               7:0] tx_data,
    output logic                      tx_done,
    output logic                      tx_ready,
    output logic [               7:0] rx_data,
    output logic                      rx_done,
    input  logic                      I2C_start,
    input  logic                      I2C_stop,
    input  logic [$clog2(D_LENGTH):0] length,
    // external ports
    output logic                      SCL,
    inout  logic                      SDA
);

    logic [7:0] tx_data_reg, tx_data_next;
    logic [7:0] rx_data_reg, rx_data_next;
    logic tx_done_next, tx_done_reg;
    logic rx_done_next, rx_done_reg;

    logic [$clog2(D_LENGTH):0] length_next, length_reg;

    logic SDA_EN;
    logic O_SDA;
    logic SCL_REG, SCL_NEXT;
    logic addr_sig_reg, addr_sig_next;
    logic ACK_LOW_REG, ACK_LOW_NEXT;

    logic [$clog2(500)-1:0] clk_counter_reg, clk_counter_next;
    logic [$clog2(4)-1:0] data_cnt_reg, data_cnt_next;
    logic [$clog2(7)-1:0] bit_counter_reg, bit_counter_next;

    assign tx_done = tx_done_reg;
    assign rx_done = rx_done_reg;
    assign rx_data = rx_data_reg;

    assign SCL = SCL_REG;


    typedef enum logic [3:0] {
        ST_IDLE,
        ST_START1,
        ST_START2,
        ST_WRITE,
        ST_READ,  // slave -> master (read DATA)
        ST_ACK,
        ST_WRITE_ACK,
        ST_READ_ACK,  // master -> slave (ACK/NACK)
        ST_HOLD,
        ST_HOLD2,
        ST_STOP1,
        ST_STOP2
    } i2c_state_t;

    i2c_state_t state, next_state;

    assign SDA = (SDA_EN) ? O_SDA : 1'bz;

    always_ff @(posedge clk, posedge reset) begin
        if (reset) begin
            state           <= ST_IDLE;
            tx_data_reg     <= 8'h00;
            rx_data_reg     <= 8'h00;
            clk_counter_reg <= 0;
            bit_counter_reg <= 0;
            data_cnt_reg    <= 0;
            tx_done_reg     <= 0;
            rx_done_reg     <= 0;
            SCL_REG         <= 1;
            addr_sig_reg    <= 1;
            ACK_LOW_REG     <= 0;
            length_reg      <= 0;
        end else begin
            state           <= next_state;
            tx_data_reg     <= tx_data_next;
            rx_data_reg     <= rx_data_next;
            clk_counter_reg <= clk_counter_next;
            bit_counter_reg <= bit_counter_next;
            data_cnt_reg    <= data_cnt_next;
            tx_done_reg     <= tx_done_next;
            rx_done_reg     <= rx_done_next;
            SCL_REG         <= SCL_NEXT;
            addr_sig_reg    <= addr_sig_next;
            ACK_LOW_REG     <= ACK_LOW_NEXT;
            length_reg      <= length_next;
        end

    end


    always_comb begin
        next_state       = state;
        O_SDA            = 1'b1;
        SCL_NEXT         = SCL_REG;
        tx_data_next     = tx_data_reg;
        rx_data_next     = rx_data_reg;
        clk_counter_next = clk_counter_reg;
        bit_counter_next = bit_counter_reg;
        data_cnt_next    = data_cnt_reg;
        tx_done_next     = 1'b0;
        SDA_EN           = 1'b1;
        tx_ready         = 1'b0;
        rx_done_next     = 1'b0;
        addr_sig_next    = addr_sig_reg;
        ACK_LOW_NEXT     = ACK_LOW_REG;
        length_next      = length_reg;
        case (state)
            ST_IDLE: begin
                O_SDA    = 1'b1;
                SCL_NEXT = 1'b1;
                tx_ready = 1'b1;
                if (I2C_En) begin
                    next_state    = ST_START1;
                    addr_sig_next = 1'b1;
                    length_next   = length;
                    tx_data_next  = {addr, CR_RW};
                end
            end
            ST_HOLD2: begin
                if (I2C_start && !I2C_stop) begin
                    next_state   = ST_WRITE;
                    tx_data_next = tx_data;
                end
            end
            ST_HOLD: begin
                if (!I2C_start && I2C_stop) begin
                    SCL_NEXT   = 1;
                    next_state = ST_STOP1;
                end
                if (I2C_start && !I2C_stop) begin
                    SCL_NEXT      = 1;
                    addr_sig_next = 1'b1;
                    length_next   = length;
                    tx_data_next  = {addr, CR_RW};
                    next_state    = ST_START1;
                end
            end
            ST_START1: begin
                O_SDA = 0;
                if (clk_counter_reg == 499) begin
                    clk_counter_next = 0;
                    SCL_NEXT = 0;
                    next_state = ST_START2;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_START2: begin
                O_SDA = 0;
                if (clk_counter_reg == 499) begin
                    clk_counter_next = 0;
                    SCL_NEXT         = 0;
                    next_state       = ST_WRITE;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_WRITE: begin
                O_SDA = tx_data_reg[7];
                if ((bit_counter_reg == 7) && (data_cnt_reg == 3)) begin
                    SDA_EN = 1'b0;
                end
                if (clk_counter_reg == 249) begin
                    SCL_NEXT = (data_cnt_reg == 0 || data_cnt_reg == 2) ? ~SCL_NEXT : SCL_NEXT;
                    clk_counter_next = 0;
                    if (data_cnt_reg == 3) begin
                        data_cnt_next = 0;
                        tx_data_next  = {tx_data_reg[6:0], 1'b0};
                        if (bit_counter_reg == 7) begin
                            bit_counter_next = 0;
                            SCL_NEXT = 0;
                            next_state = (addr_sig_reg) ? ST_ACK : ST_WRITE_ACK;
                        end else begin
                            bit_counter_next = bit_counter_reg + 1;
                        end
                    end else begin
                        data_cnt_next = data_cnt_reg + 1;
                    end
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_READ: begin
                if ((bit_counter_reg == 7) && (data_cnt_reg == 3)) begin
                    SDA_EN = 1'b1;
                end else begin
                    SDA_EN = 1'b0;
                end
                if (clk_counter_reg == 249) begin
                    SCL_NEXT = (data_cnt_reg == 0 || data_cnt_reg == 2) ? ~SCL_NEXT : SCL_NEXT;
                    clk_counter_next = 0;
                    if (data_cnt_reg == 1) begin
                        rx_data_next = {rx_data_reg[6:0], SDA};
                    end
                    if (data_cnt_reg == 3) begin
                        data_cnt_next = 0;
                        if (bit_counter_reg == 7) begin
                            bit_counter_next = 0;
                            SCL_NEXT         = 0;
                            next_state       = ST_READ_ACK;
                        end else begin
                            bit_counter_next = bit_counter_reg + 1;
                        end
                    end else begin
                        data_cnt_next = data_cnt_reg + 1;
                    end
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_ACK: begin
                SDA_EN = 1'b0;
                if (clk_counter_reg == 249) begin
                    SCL_NEXT = (data_cnt_reg == 0 || data_cnt_reg == 2) ? ~SCL_NEXT : SCL_NEXT;
                    clk_counter_next = 0;
                    if (data_cnt_reg == 3) begin
                        data_cnt_next = 0;
                        tx_done_next  = 1;
                        if (!ACK_LOW_REG) begin
                            next_state   = (length_reg == 0) ? ST_STOP1: (CR_RW == 1) ? ST_READ : ST_HOLD2 ;
                            SCL_NEXT     = (length_reg == 0) ? 1 : 0;
                            addr_sig_next = 1'b0;
                            length_next = length_reg - 1;
                        end else begin
                            SCL_NEXT   = 1;
                            next_state = ST_STOP1;
                        end
                    end else begin
                        data_cnt_next = data_cnt_reg + 1;
                    end
                    if (data_cnt_reg == 1) ACK_LOW_NEXT = SDA;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_WRITE_ACK: begin
                if ((data_cnt_reg == 3)) begin
                    SDA_EN = 1'b1;
                end else begin
                    SDA_EN = 1'b0;
                end
                O_SDA  = 1'b1;
                if (clk_counter_reg == 249) begin
                    SCL_NEXT = (data_cnt_reg == 0 || data_cnt_reg == 2) ? ~SCL_NEXT : SCL_NEXT;
                    clk_counter_next = 0;
                    if (data_cnt_reg == 3) begin
                        data_cnt_next = 0;
                        tx_done_next  = 1;
                        if (!ACK_LOW_REG) begin
                            if (length_reg == 0) begin
                                SCL_NEXT   = 1;
                                next_state = ST_HOLD;
                            end else begin
                                tx_data_next = tx_data;
                                SCL_NEXT     = 0;
                                next_state   = ST_HOLD2;
                                length_next  = length_reg - 1;
                            end
                        end else begin
                            SCL_NEXT   = 1;
                            next_state = ST_STOP1;
                        end
                    end else begin
                        data_cnt_next = data_cnt_reg + 1;
                    end
                    if (data_cnt_reg == 1) ACK_LOW_NEXT = SDA;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_READ_ACK: begin
                if (data_cnt_reg == 3) begin
                    SDA_EN = 1'b0;
                end else begin
                    SDA_EN = 1'b1;
                end
                O_SDA  = (length_reg == 0) ? 1'b1 : 1'b0;  // nack : ack
                if (clk_counter_reg == 249) begin
                    SCL_NEXT = (data_cnt_reg == 0 || data_cnt_reg == 2) ? ~SCL_NEXT : SCL_NEXT;
                    clk_counter_next = 0;
                    if (data_cnt_reg == 3) begin
                        data_cnt_next = 0;
                        rx_done_next  = 1;
                        if (length_reg == 0) begin
                            next_state = ST_HOLD;
                        end else begin
                            SCL_NEXT    = 0;
                            next_state  = ST_READ;
                            length_next = length_reg - 1;
                        end
                    end else begin
                        data_cnt_next = data_cnt_reg + 1;
                    end
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_STOP1: begin
                O_SDA = 0;
                if (clk_counter_reg == 499) begin
                    clk_counter_next = 0;
                    next_state = ST_STOP2;
                    SCL_NEXT = 1;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
            ST_STOP2: begin
                O_SDA = 1;
                if (clk_counter_reg == 499) begin
                    clk_counter_next = 0;
                    next_state = ST_IDLE;
                    SCL_NEXT = 1'b1;
                end else begin
                    clk_counter_next = clk_counter_reg + 1;
                end
            end
        endcase
    end
endmodule
```

</details>

<details>
  <summary><b>Click to expand RTL (I2C_SLAVE)</b></summary>

```systemverilog
`timescale 1ns / 1ps

module I2C_SLAVE (
    // global signals
    input  logic       clk,
    input  logic       reset,
    // Internal signals
    input  logic [7:0] tx_data,
    output logic       tx_done,
    output logic       tx_ready,
    output logic [7:0] rx_data,
    output logic       rx_done,
    // External signals
    input  logic       scl,
    inout  logic       sda
);

    logic sda_en;
    logic o_sda;
    logic [7:0] addr_reg, addr_next;
    logic [7:0] rx_data_reg, rx_data_next;
    logic [7:0] tx_data_reg, tx_data_next;
    logic burst_read_reg, burst_read_next;
    
    ///////////////////////synchronizer && edge detector////////////////////////////////
    logic sda_falling, sda_rising;
    logic scl_falling, scl_rising;
    logic sda_sync0, sda_sync1;
    logic scl_sync0, scl_sync1;

    always_ff @(posedge clk, posedge reset) begin
        if (reset) begin
            sda_sync0 <= 0;
            sda_sync1 <= 0;
            scl_sync0 <= 0;
            scl_sync1 <= 0;
        end else begin
            sda_sync0 <= sda;
            sda_sync1 <= sda_sync0;
            scl_sync0 <= scl;
            scl_sync1 <= scl_sync0;
        end
    end

    assign sda_rising  = sda_sync0 && (~sda_sync1);
    assign sda_falling = (~sda_sync0) && sda_sync1;
    assign scl_rising  = scl_sync0 && (~scl_sync1);
    assign scl_falling = (~scl_sync0) && scl_sync1;
    ////////////////////////////////////////////////////////////////////////////////////

    assign sda         = (sda_en) ? o_sda : 1'bz;

    typedef enum {
        IDLE,
        ADDR,
        SEND_ACK,
        WAIT,
        SEND_DATA,
        RCV_DATA,
        RCV_ACK,
        RCV_ACK_HOLD,
        WAIT_2,
        SEND_ACK_2,
        STOP
    } state_e;

    state_e state, state_next;

    logic [2:0] bit_cnt_reg, bit_cnt_next;
    logic rx_done_reg, rx_done_next;
    wire [6:0] slave_addr = 7'b1111110;

    always_ff @(posedge clk, posedge reset) begin
        if (reset) begin
            state          <= IDLE;
            bit_cnt_reg    <= 0;
            addr_reg       <= 0;
            rx_data_reg    <= 0;
            rx_done_reg    <= 0;
            tx_data_reg    <= 0;
            burst_read_reg <= 0;
        end else begin
            state          <= state_next;
            bit_cnt_reg    <= bit_cnt_next;
            addr_reg       <= addr_next;
            rx_data_reg    <= rx_data_next;
            rx_done_reg    <= rx_done_next;
            tx_data_reg    <= tx_data_next;
            burst_read_reg <= burst_read_next;

        end
    end

    always_comb begin
        state_next      = state;
        bit_cnt_next    = bit_cnt_reg;
        addr_next       = addr_reg;
        rx_data_next    = rx_data_reg;
        rx_done_next    = 0;
        o_sda           = 1'b1;
        tx_data_next    = tx_data_reg;
        burst_read_next = burst_read_reg;
        tx_done         = 0;

        if (state != IDLE) begin
            if (scl_sync1 && sda_rising) begin
                state_next = STOP;
            end else if (scl_sync1 && sda_falling) begin
                state_next   = ADDR;
                bit_cnt_next = 0;
            end
        end
        case (state)
            IDLE: begin
                rx_data_next = 0;
                if (scl && sda_falling) begin
                    bit_cnt_next = 0;
                    state_next   = ADDR;
                end
            end
            ADDR: begin
                if (scl_rising) begin
                    addr_next = {addr_reg[6:0], sda};
                    if (bit_cnt_reg == 7) begin
                        state_next   = WAIT;
                        rx_done_next = 1;
                        bit_cnt_next = 0;
                    end else begin
                        bit_cnt_next = bit_cnt_reg + 1;
                    end
                end
            end
            WAIT: begin
                if (scl_falling) begin
                    state_next = SEND_ACK;
                end
            end
            SEND_ACK: begin
                if (addr_reg[7:1] == slave_addr) begin
                    o_sda = 0;
                end
                if (scl_falling) begin
                    if (addr_reg[7:1] == slave_addr) begin
                        addr_next = 0;
                        if (addr_reg[0]) begin
                            state_next   = SEND_DATA;
                            tx_data_next = tx_data;
                        end else begin
                            state_next = RCV_DATA;
                        end
                    end else begin
                        state_next = STOP;
                    end
                end
            end
            RCV_DATA: begin
                if (scl_rising) begin
                    rx_data_next = {rx_data_reg[6:0], sda};
                    if (bit_cnt_reg == 7) begin
                        state_next   = WAIT_2;
                        bit_cnt_next = 0;
                    end else begin
                        bit_cnt_next = bit_cnt_reg + 1;
                    end
                end
            end
            WAIT_2: begin
                if (scl_falling) begin
                    state_next = SEND_ACK_2;
                end
            end
            SEND_ACK_2: begin
                o_sda = 0;
                if (scl_falling) begin
                    rx_done_next = 1'b1;
                    rx_data_next = 0;
                    state_next   = RCV_DATA;
                end
            end
            SEND_DATA: begin
                o_sda = tx_data_reg[7];
                if (scl_falling) begin
                    if (bit_cnt_reg == 7) begin
                        state_next = RCV_ACK;
                        tx_done    = 1;
                    end else begin
                        tx_data_next = {tx_data_reg[6:0], 1'b0};
                        bit_cnt_next = bit_cnt_reg + 1;
                    end
                end
            end
            RCV_ACK: begin
                if (scl_rising) begin
                    state_next = RCV_ACK_HOLD;
                    if (!sda_sync1) begin
                        burst_read_next = 1;
                        bit_cnt_next = 0;
                    end else begin
                        burst_read_next = 0;
                        bit_cnt_next = 0;
                    end
                end
            end
            RCV_ACK_HOLD: begin
                if (scl_falling) begin
                    if (burst_read_reg) begin
                        state_next   = SEND_DATA;
                        tx_data_next = tx_data;
                    end
                end
            end
            STOP: begin
                rx_data_next = 0;
                state_next   = IDLE;
            end
        endcase
    end

    assign sda_en  = (state == SEND_ACK) || (state == SEND_ACK_2) || (state == SEND_DATA);
    assign rx_done = rx_done_reg;
    assign rx_data = rx_data_reg;

endmodule
```

</details>

---

## 13. Block Diagrams (Optional)

> 아래처럼 이미지 넣고 싶으면, repo에 파일 넣고 경로만 맞추면 됨:
>
> - `docs/i2c_protocol.png`
> - `docs/i2c_timing.png`
> - `docs/i2c_master_asm.png`

<p align="center">
  <img src="docs/i2c_protocol.png" width="85%">
</p>
<p align="center">
  <img src="docs/i2c_timing.png" width="85%">
</p>
<p align="center">
  <img src="docs/i2c_master_asm.png" width="85%">
</p>

---

## DOC

## 14. References

### (A) Official I2C Specification
- [NXP UM10204 — I2C-bus specification and user manual (PDF)](https://www.nxp.com/docs/en/user-guide/UM10204.pdf)

### (B) “타사 제품/상용 SoC에 들어가는” I2C Controller 문서 예시 (공개 문서)
> 상용 IP의 “풀 유저가이드”는 보통 NDA라 공개가 제한적이라, 공개 접근 가능한 형태로는 Linux DT binding / driver 문서가 가장 현실적인 레퍼런스임.

- [Synopsys DesignWare I2C (DT binding, text)](https://www.kernel.org/doc/Documentation/devicetree/bindings/i2c/i2c-designware.txt)
- [Synopsys DesignWare I2C (DT binding, YAML)](https://www.kernel.org/doc/Documentation/devicetree/bindings/i2c/snps%2Cdesignware-i2c.yaml)
- [Cadence I2C Controller (DT binding, text)](https://www.kernel.org/doc/Documentation/devicetree/bindings/i2c/i2c-cadence.txt)

### (C) Example: 실제 플랫폼 문서에서 DesignWare 사용 언급(공개)
- [Altera/Intel HPS I2C (DesignWare controller instances 언급)](https://altera-fpga.github.io/rel-24.1/linux-embedded/i2c/i2c/)

