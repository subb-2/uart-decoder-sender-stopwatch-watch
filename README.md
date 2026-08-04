# Verilog Design — UART Stopwatch & Watch Controller

📅 프로젝트 정보

* 진행 기간: 2026.01.20 ~ 2026.02.10
* 설계 대상: UART 양방향 제어 기반 스톱워치 / 시계 시스템 (FPGA 실동작 검증)
* 기술 스택: `Verilog HDL`, `Vivado XSim`, `Basys3 (Artix-7)`, `ComPortMaster`

---

## 📝 프로젝트 개요

PC의 UART 터미널에서 ASCII 문자를 전송해 FPGA의 스톱워치 / 시계를 실시간으로 제어하고,  
FPGA가 현재 시각을 다시 PC로 송신하는 **양방향 UART 제어 시스템**입니다.

기존 물리 버튼/스위치 인터페이스를 UART 인터페이스로 병행 운용할 수 있도록 설계하였으며,  
단순 기능 동작 구현을 넘어 **두 입력 소스 간 충돌 없는 선택 구조 설계**와  
**ASCII Sender의 BCD → ASCII 변환 및 12회 순차 전송 타이밍**을 정확히 검증하는 데 집중했습니다.

---

## 🏗️ 시스템 구조

<p align="center">
<img width="900" alt="Block Diagram" src="./SW_W_UART_Sender.drawio.png" />
</p>

### 기능 인터페이스 맵

| 기능 | 물리 입력 | UART 입력 |
|------|-----------|-----------|
| Run / Stop | `btn_r` | `r` (0x72) |
| Clear | `btn_l` | `l` (0x6C) |
| Up (시간 증가) | `btn_u` | `u` (0x75) |
| Down (시간 감소) | `btn_d` | `d` (0x64) |
| Up / Down 모드 전환 | `sw[0]` | `0` (0x30) — toggle |
| Stopwatch / Watch 전환 | `sw[1]` | `1` (0x31) — toggle |
| Hour:Min / Sec:Msec 표시 전환 | `sw[2]` | `2` (0x32) — toggle |
| Watch 시간 설정 모드 | `sw[3]` | `3` (0x33) — toggle |
| UART 제어 모드 활성화 | `sw[4]` | — |
| 현재 시각 PC 송신 | — | `s` (0x73) |

> `sw[4] = 1`일 때 UART 입력이 물리 스위치를 대체합니다. `sw[4] = 0`이면 물리 스위치가 우선합니다.

---

## 🔑 주요 구현 내용

### 1. UART RX / TX FSM (9600 baud, 8-N-1)

16배 오버샘플링 기반의 Baud Tick(`6.51 µs @ 100 MHz`)으로 비트 중앙을 정확히 샘플링합니다.

```
RX / TX 공통 FSM: IDLE → START → DATA → STOP → IDLE
```

* **RX**: start bit 감지(`b_tick & !rx`) 후 `b_tick_cnt == 7`에서 DATA 진입. 매 `b_tick_cnt == 15`마다 right-shift 수신 (`{rx, buf[7:1]}`)
* **TX**: `tx_start` 펄스로 전송 개시, `tx_data`를 내부 버퍼에 캡처 후 left-shift 출력 (`{1'b0, buf[7:1]}`)

### 2. ASCII Decoder & SW Set

* **`ascii_decoder`**: `rx_done` 펄스 발생 시 수신 데이터를 4비트 원-핫 제어 신호(`ascii_d[3:0]`)로 변환. 매 클럭 기본값 0으로 초기화하여 1클럭 펄스처럼 동작 → 기존 버튼 신호와 OR 연산으로 통합
* **`ascii_sw_set`**: `0`~`3` 문자 수신 시 대응하는 내부 레지스터를 toggle. `sw[4]` 신호로 물리 스위치와 3항 연산자(MUX)로 선택 — OR 대신 MUX를 사용해 두 소스 간 충돌 방지

### 3. ASCII Sender

`s` 문자 수신 시 현재 시각(24비트)을 캡처하고, IDLE → SEND → WAIT FSM으로 12회 반복 전송합니다.

```
IDLE ──(ascii_d_s==1)──► SEND ──(tx_start=1)──► WAIT ──(tx_done & cnt!=12)──► SEND
                                                       ──(tx_done & cnt==12) ──► IDLE
```

각 숫자는 BCD 분리 후 `bcd_sender`로 ASCII 코드(0x30~0x39)로 변환하여 `HH:MM:SS:ms` 포맷으로 순차 전송합니다.

---

## ✅ 검증 시나리오

* **Baud Tick**: 카운터 650 주기에서 `b_tick` 발생, 6.51 µs 간격 확인
* **UART RX**: `r` 문자 수신 → `rx_data = 0x72`, `rx_done` 펄스 발생 확인
* **UART TX**: `tx_start` 인가 → bit-by-bit 직렬 출력 및 `tx_done` 확인
* **Run & Stop**: `r` 입력 시 스톱워치 증가 / 재입력 시 정지 확인
* **Clear**: `l` 입력 시 스톱워치 카운터 0으로 초기화 확인
* **Down mode**: `sw[4]=1` 설정 후 `0` 입력 → 카운트 감소(99→98→…) 확인
* **Watch 시간 설정**: `1` 입력으로 Watch 모드 전환 후 `u`/`d` 입력 시 min/hour 증감 확인
* **ASCII Sender (Stopwatch)**: `s` 입력 → `send_cnt_reg` 0→12 순환, `tx_data` 시퀀스 `00:00:00:54` 포맷 확인
* **ASCII Sender (Watch)**: `s` 입력 → `03:06:00:65` 포맷 TX 출력 확인

---

## 🚀 문제 해결 (Troubleshooting)

### 1. sw와 UART 신호 OR 결합 문제

* **문제**: 물리 스위치와 UART toggle 신호를 OR로 결합하면, 어느 한쪽이 1로 고정된 경우 다른 소스가 동작하지 않는 현상 발생.
* **원인**: OR 구조는 두 소스를 상호 배타적으로 제어할 수 없음.
* **해결**: `sw[4]`를 UART 제어 모드 선택 비트로 지정하고 3항 연산자(MUX)로 두 소스를 상호 배타적으로 선택.

```verilog
// Before (문제)
assign w_up_down_or = w_ascii_up_down | sw[0];

// After (수정)
assign w_up_down_mux = w_uart_sw_sel ? w_ascii_up_down : sw[0];
```

### 2. 시뮬레이션 시나리오 오류

* **문제**: 테스트벤치에서 `sw[4]` 설정 없이 UART 값 전송 시 물리 스위치와 UART 제어 신호가 동시에 active되어 Short 발생.
* **원인**: 시뮬레이션 시나리오에 `sw[4] = 1` 선행 설정 누락.
* **해결**: `sw[4] = 1` 설정 후 일정 시간 대기(`#1000`) → UART 전송 순서로 시나리오 수정.

---

## 🎥 Demo Video

### 🎥 Demo Video - stopwatch_watch_uart
https://github.com/user-attachments/assets/b8882699-035a-4da2-a82b-294bbe4bd5b1

### 🎥 Demo Video - stopwatch_watch_sender
https://github.com/user-attachments/assets/d8001410-4d9c-43e0-addc-2a6f1ad7dbc6

---

## 📚 배운 점

* **입력 소스 중재 설계**: 물리 입력과 UART 입력을 OR로 단순 합산하면 level-sensitive 신호 간 충돌이 생긴다는 것을 직접 경험. 두 소스를 상호 배타적으로 운용하려면 MUX 기반의 명시적 선택 구조가 필수임을 이해.
* **오버샘플링의 의미**: 16배 오버샘플링이 단순한 구현 관례가 아니라, 비동기 신호의 비트 중앙을 정확히 포착하기 위한 정량적 전략임을 Baud Tick 시뮬레이션을 통해 이해.
* **순차 전송 FSM 설계**: ASCII Sender의 SEND → WAIT → SEND 반복 구조에서 `tx_done` 타이밍과 `send_cnt_reg` 경계 조건을 정확히 처리하지 않으면 마지막 바이트가 누락되거나 중복 전송된다는 것을 시뮬레이션으로 확인.

---

## 🖥️ 개발 환경

| 항목 | 내용 |
|------|------|
| HDL | Verilog HDL |
| EDA Tool | Xilinx Vivado |
| 시뮬레이터 | Vivado Simulator (XSim) |
| 타겟 보드 | Digilent Basys3 (Xilinx Artix-7) |
| UART 터미널 | ComPortMaster (9600 baud, 8-N-1) |

---

## 📁 파일 구성

```text
├── uart_top.v                   # UART TX/RX 및 Baud Tick Generator 통합 모듈
├── ascii_decoder.v              # UART 수신 ASCII 명령을 스톱워치 제어 신호로 변환
├── control_unit.v               # 스톱워치 상태 제어 FSM (STOP/RUN/CLEAR/SET_WATCH)
├── stopwatch_watch.v            # Top Module - UART, FSM, FND 연결
├── fnd_controller.v             # FND 스캔 및 시간 데이터 표시
└── btn_all.v                    # 버튼 디바운싱 및 One-Pulse 생성
