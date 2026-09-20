# 실험 전 레포트: LAB2-05 PISO (병렬 입력·직렬 출력)

작성자: 상혁 (2025440084) / 작성일: 2026-09-20 / 소스 커밋: `17d0481` / workspace: `LAB1.code-workspace` (템플릿 v2.0.1) / OS: `Windows 11 Home 10.0.26200` / Python: `Python 3.14.7` / 시뮬레이터: Icarus Verilog `Icarus Verilog version 12.0 (devel) (s20150603-1539-g2693dd32b)`

> 이 레포트는 VS Code(Icarus) 시뮬레이션까지의 사전 검증이다. Vivado GUI와 실물 보드 결과는 실험 후 레포트([post](../post/post_report.md))에서 다룬다. 시각은 clk 상승 에지(5, 15, 25, … ns)의 1 ns 뒤, 즉 TB가 비교하는 시각으로 적었다.

## 목적과 예상 동작

4비트를 병렬로 load한 뒤 MSB부터 한 비트씩 직렬로 내보내는 PISO를 설계한다. load·enable·rst의 우선순위와 직렬 출력 순서를 계산한다.

### 포트 (`piso4.v`의 `piso4`)

| 포트 | 방향 | 비트 폭 | 설명 |
|---|---|---|---|
| clk | in | 1 | 상승 에지 기준 클록 |
| rst | in | 1 | 동기 active-high 리셋. 최우선 |
| load | in | 1 | 1이면 value←data_in. enable보다 우선 |
| enable | in | 1 | 1이면 왼쪽으로 1비트 shift, 0을 채움 |
| data_in | in | 4 | 병렬 입력 |
| serial_out | out (wire) | 1 | = value[3], 항상 MSB |
| value | out (reg) | 4 | 저장값 |

최상위(`lab2_piso.v`, `lab2_piso`): `load = press && switches[0]`, `enable = press && !switches[0]`, `data_in = switches[7:4]`, `led = {value, 3'b000, serial_out}`.

### 동작 규칙과 경계 입력

- 규칙: value(k+1) = rst ? 0 : (load ? data_in : (enable ? {value[2:0], 1'b0} : value)). serial_out = value[3].
- 정상·경계: 1010을 load하면 shift 전 serial_out=1, 이후 0, 1, 0 순서로 나온다. 4번(또는 3번) shift 뒤 저장값은 0000. load와 enable이 동시에 1이면 load가 우선하고 rst는 둘보다 우선.
- 시간 기준: TB 클록은 10 ns 주기(상승 에지 5, 15, 25, … ns)이고 입력은 에지 1 ns 뒤에 바꾼다. 레지스터는 다음 상승 에지에서 갱신된다. 실제 보드 클록(1 kHz)은 시뮬레이션의 10 ns와 별개다.

## 소스와 테스트벤치

- 설계 top: `lab2_piso` / 시뮬레이션 top: `tb_piso4`
- 소스: [`src/piso4.v`](../../src/piso4.v), [`src/input_frontend.v`](../../src/input_frontend.v), [`src/lab2_piso.v`](../../src/lab2_piso.v)
- 테스트벤치: [`sim/tb_piso4.sv`](../../sim/tb_piso4.sv)
- 제약: [`constraints/lab2_piso.xdc`](../../constraints/lab2_piso.xdc)
- 설정: [`simulation.json`](../../simulation.json) (sources 3개, testbench `sim/tb_piso4.sv`, simulation_top `tb_piso4`)

| 파일 | 역할 |
|---|---|
| `src/piso4.v` | 핵심 동작을 담은 코어 `piso4`. TB가 이 모듈을 직접 검사한다. |
| `src/input_frontend.v` | 버튼·스위치를 클록에 맞추는 입력 회로. 리셋 2단 해제 동기화, 버튼·스위치 2단 동기화 플립플롭, STABLE_CYCLES=20(1 kHz에서 20 ms) 안정 확인 뒤 한 클록짜리 `press` 펄스를 만든다. |
| `src/lab2_piso.v` | 보드 top. 프런트엔드와 코어를 연결하고 LED로 출력한다. |
| `sim/tb_piso4.sv` | 입력 자극, 기대값 계산, 자동 비교(`check`), PASS/FAIL 출력, VCD 생성, watchdog. |
| `constraints/lab2_piso.xdc` | 핀 번호·전압과 1 kHz 클록 정의. Icarus는 XDC를 읽지 않으므로 이 사전 시뮬레이션에는 사용되지 않는다. |
| `simulation.json` | VS Code 시뮬레이션 작업이 읽는 소스 목록·테스트벤치·시뮬레이션 top. |

### 테스트벤치 동작

- 자극 순서: 리셋 후 word=0~15에 대해 (load=enable=1로 load 우선 확인 → load=enable=0 유지 → enable=1로 비트3부터 0까지 serial_out을 shift 전에 검사하고 4번 shift → 값 0 확인)를 반복한다. 끝으로 rst=1, load=1, data_in=15에서 리셋 우선.
- 검사 횟수: 1(reset)+16×7(load beats shift, hold, MSB first 4회, four shifts zero fill)+1(reset beats load) = 114
- 종료·watchdog: 마지막 검사 뒤 `finish` task가 `LAB2_PASS piso4 checks=N`을 출력하고 `$finish`한다. 별도로 100000 ns(100 µs) 뒤에 `watchdog timeout`으로 `$fatal` 처리한다. 예상 종료 시각은 976 ns이다.
- 이 TB는 코어 `piso4`만 시험한다. 입력 동기화·디바운스와 실제 핀·타이밍이 통과했다는 뜻은 아니다.

### XDC 설명

`lab2_piso.xdc`은 포트 이름을 `lab2_piso.v`과 맞춰 핀을 지정한다. 모든 I/O는 `LVCMOS33`이고 `create_clock -name trainer_1khz -period 1000000.000 [get_ports clk]`로 주 클록을 1 kHz(주기 1,000,000 ns)로 정의하며 `set_false_path -from [get_ports {rst button sw[*]}]`로 비동기 입력을 타이밍 경로에서 제외한다.

| 포트 | 핀 | 보드 대응 |
|---|---|---|
| clk | B6 | 1 kHz 주 클록 |
| rst | K4 | 리셋 (active-high) |
| button | N8 | 스텝 버튼 |
| sw[7:0] | U4(sw[0]), V4, W1, W4, T1, U2, W3, Y1(sw[7]) | DIPSW8..DIPSW1 (DIPSW1..8 = sw[7]..sw[0]) |
| led[7:0] | N5(led[0]), M1, M3, M7, N7, M2, M4, L4(led[7]) | LED0..LED7 |

## VS Code 실행 과정

1. File → New Window → File → Open Workspace from File...로 `LAB1.code-workspace`를 연다. 확장(slang, VaporView, vscode-pdf)을 설치한다.
2. RTL·TB·XDC·`simulation.json`을 직접 입력하고 File → Save All.
3. Terminal → Run Task... → `01 Check tools`로 Git·Python·iverilog·vvp 버전을 확인한다.
4. `02 Simulate`를 실행해 `LAB2_PASS`와 종료 시각을 확인한다.
5. `03 Open waveform`으로 `build/sim/wave.vcd`를 VaporView로 연다. 신호: clk, rst, load, enable, data_in, value, serial_out (value는 2진수).

정상 실행 로그(본인 로그로 교체하고 `evidence/pre/`에 복사):

```text
LAB2_PASS piso4 checks=114
sim/tb_piso4.sv:19: $finish called at 976000 (1ps)
```

- 본인 실행 로그: [`../../evidence/pre/lab2_05_normal.log`](../../evidence/pre/lab2_05_normal.log)
- VCD: [`../../evidence/pre/lab2_05_wave_normal.vcd`](../../evidence/pre/lab2_05_wave_normal.vcd)
- 파형 캡처(VaporView): `evidence/pre/lab2_05_wave_full.png`(전체 Zoom Fit), `evidence/pre/lab2_05_wave_zoom.png`(상승 에지 확대)
- 오류: 첫 실행에서 발생한 오류가 있으면 첫 오류 → 수정 → 재실행 로그 순서로 기록한다. (없으면 "없음", Python 실행 경로를 고쳤다면 그 내용 기입)

### 사전 파형 해석

| 시간 구간 | 입력 | 예상 | 실제 파형 | 해석 |
|---|---|---|---|---|
| 6 ns | rst=1 | value=0000, serial_out=0 | value=0000, serial_out=0 | 동기 리셋. |
| 616 ns | word=A(1010): load=1, enable=1 · 에지 615 ns | value=1010, serial_out=1 | value=1010, serial_out=1 | load가 shift보다 우선. shift 전 serial_out은 MSB=1. |
| 626 ns | load=0, enable=0 · 에지 625 ns | value=1010, serial_out=1 | value=1010, serial_out=1 | hold: 값과 MSB 유지. |
| 636 ns | enable=1, 첫 shift · 에지 635 ns | value=0100, serial_out=0 | value=0100, serial_out=0 | 왼쪽 shift. 다음 비트(0)가 MSB로. |
| 646 ns | 둘째 shift · 에지 645 ns | value=1000, serial_out=1 | value=1000, serial_out=1 | 셋째 비트 1이 MSB로. |
| 656 ns | 셋째 shift · 에지 655 ns | value=0000, serial_out=0 | value=0000, serial_out=0 | 마지막 비트 0. 이제 저장값은 0000. |
| 666 ns | 넷째 shift · 에지 665 ns | value=0000, serial_out=0 | value=0000, serial_out=0 | 네 번 shift 뒤 0으로 채워져 0000 유지. |
| 976 ns | rst=1, load=1, data_in=15 · 에지 975 ns | value=0000, serial_out=0 | value=0000, serial_out=0 | 리셋이 load보다 우선. |

표의 값은 상승 에지 직후 안정된 값이다. `LAB2_PASS`만 적지 않고 각 행에서 입력, 이전 상태, 다음 상태를 비교한다. 파형 캡처에서 위 시각을 확대해 본인 화면으로 확인한다.

## 코드 수정·실패·복구 실험

- 변경: 직렬 출력을 MSB 대신 LSB로 바꾼다(`assign serial_out = value[3];` → `value[0]`).
- 변경한 파일과 위치: `src/piso4.v` 8행
- 테스트벤치 기대값은 바꾸지 않는다.

```diff
-    assign serial_out = value[3];
+    assign serial_out = value[0];
```

실행 전 계산: word=1(0001)의 load 다음 hold(86 ns)에서 TB는 MSB(`(1>>3)&1`=0)를 기대하지만 변경 회로는 LSB 1을 낸다. 86 ns가 첫 실패이고, word=0에서는 모든 비트가 0이라 통과한다.

| 단계 | 소스 커밋 또는 해시 | 실행 폴더·로그 링크 | 입력·기대값·실제값 | 해석 |
|---|---|---|---|---|
| 정상 코드 | `<커밋/해시 기입>` | [normal.log](../../evidence/pre/lab2_05_normal.log) | 86 ns 기대 serial_out=0, 실제 serial_out=0. `LAB2_PASS piso4 checks=114` | 모든 검사 통과, 976 ns 종료. |
| 지정한 RTL 변경 | `<커밋/해시 기입>` | [mod.log](../../evidence/pre/lab2_05_mod.log) | 86 ns 기대 serial_out=0, 실제 serial_out=1. `LAB2_FAIL MSB first before edge time=86000`, `FATAL: sim/tb_piso4.sv:12: check failed` | `MSB first before edge` 검사가 변경을 발견했다(로그의 time은 ps 단위, 86000 ps = 86 ns). |
| 원래 코드로 복구 | `<커밋/해시 기입>` | [recover.log](../../evidence/pre/lab2_05_recover.log) | 복구 후 전체 검사 재실행. `LAB2_PASS piso4 checks=114`, `$finish called at 976000 (1ps)` | PASS와 종료 시각이 정상 실행과 같고 새 VCD를 확인한다. |

- 첫 실패 이후에는 `$fatal`로 시뮬레이션이 끝나므로 뒤의 검사는 실행되지 않는다. 변경 전후 파형은 각각 별도 폴더에 보관한다.
- 문법 오류를 경험했다면 오류 위치로 이동한 화면, 원인, 수정 내용과 재실행 로그도 이 절에 연결한다.

## 보드 실험 계획

- 부품·프로젝트: Vivado 2026.1, RTL Project `lab2_piso`, 부품 `xc7s75fgga484-1`(정확히 -1). Design Sources: `piso4.v`, `input_frontend.v`, `lab2_piso.v`(Copy sources 해제). Simulation Sources: `tb_piso4.sv`(Set as Top: `tb_piso4`). Constraints: `lab2_piso.xdc`. Project Summary의 Top module name은 `lab2_piso`이다.
- 장비 클록·제약: Combo II-DLD S75 주 클록 B6을 1 kHz로 맞추고 XDC의 `trainer_1khz`(1,000,000 ns)와 일치시킨다.
- 입력: DIPSW1..4=`sw[7:4]`=data_in, DIPSW8=`sw[0]`(1이면 load, 0이면 shift), N8=press, K4=리셋. 주 클록은 1 kHz.
- 출력: LED[7:4]=value, LED[3:1]=0, LED[0]=serial_out.

| 조작 | 예상 LED / 동작 |
|---|---|
| SW1..4=1010, SW8=1, N8 한 번 | A1 (load, serial_out=1) |
| SW8=0, N8 한 번 | 40 |
| N8 한 번 더 | 81 |
| N8 한 번 더 | 00 |
| N8 한 번 더 | 00 |

load 후 shift 전에 LED0=1을 먼저 읽고 이후 0, 1, 0을 관찰한다.

촬영할 장면: 보드 전체(배선·입력·출력이 함께 보이는 사진)와 위 표의 조작별 LED 상태 사진·영상. 예상되는 차이: 버튼·스위치는 동기화·안정 확인 지연이 있고, 빠른 신호는 눈으로 구분되지 않을 수 있다.

이 단계에서는 Vivado GUI와 실물 보드의 결과를 수행한 것처럼 기록하지 않는다. 합성·구현·bit 생성과 실제 장치 기록은 실험 후 레포트에서 다룬다.
