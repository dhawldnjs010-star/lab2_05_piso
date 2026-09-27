# 실험 후 레포트: LAB2-05 PISO (병렬 입력·직렬 출력)

작성자: 상혁 (2025440084) / 작성일: `<기입 — 실제 제출일>` / 소스 커밋: `<기입 — evidence·reports 커밋 후 git log에서 확인한 해시>` / 제출 태그: `lab2-05-submit-v1` (evidence/reports를 커밋한 뒤 그 커밋에 `git tag lab2-05-submit-v1` 후 `git push origin lab2-05-submit-v1`로 생성) / GitHub 저장소: `https://github.com/dhawldnjs010-star/lab2_05_piso`

> 실험 후에 채운다. 수행하지 않은 항목은 "미수행"으로 표시하고, 구현 성공을 실물 동작 확인으로 대신하지 않는다. 사전 레포트: [pre](../pre/pre_report.md)

## 진행 경로

- [x] Vivado GUI  - [ ] 오픈소스 CLI (Icarus, Yosys, nextpnr, Project X-Ray, openFPGALoader 버전 기록)

## Vivado 프로젝트와 시뮬레이션

- 프로젝트 이름 `lab2_piso`, 부품 `xc7s75fgga484-1`, Design/Simulation/Constraints 소스 등록, Simulation top `tb_piso4`, Top module `lab2_piso` (실제 화면 기준으로 확인)

| 항목 | VS Code(Icarus) | Vivado(XSim) | 차이·해석 |
|---|---|---|---|
| PASS 로그 | checks=114 | `LAB2_PASS piso4 checks=114` | Icarus·XSim 검사 수 일치 |
| 종료 시각 | 976 ns | 976 ns | Icarus·XSim 종료 시각 일치 |
| 주요 파형 | 사전 레포트 표 | `evidence/post/xsim_wave.vcd`, `evidence/post/xsim_simulate.log` | 파형 개형 동일 |

## 합성·구현 결과

| 항목 | 값 | 해석 |
|---|---|---|
| WNS / WHS | WNS 999997.500 ns, WHS 0.121 ns, Failing endpoints 0 | 사용자 클록 제약이 없어 수치 자체는 무의미하나, Setup/Hold failing endpoint가 0으로 내부 타이밍은 통과했다. |
| DRC | Checks found: 1 — CFGBVS-1(Warning) | CONFIG_VOLTAGE·CFGBVS 속성이 미지정되었다는 경고이며 오류는 아니다(보드 회로도 상 배터리 백업 불필요 구성이라 실제 값 임의 지정은 하지 않았다). |
| TIMING-18 등 남은 경고 | Checks found: 8 — TIMING-18(Warning), led[0]~led[7] 출력 지연 미지정 | 사용자 입출력 지연(set_input_delay/set_output_delay)을 지정하지 않아 나오는 경고로, 조합 지연이 매우 작아 실동작에는 영향이 없다. |
| 자원 사용량 | Slice LUT 13 / 48,000 (0.03%), Slice Register 26 / 96,000 (0.03%), Bonded IOB 16 / 338 (4.73%) | 사용량이 매우 낮아 자원 부족 문제는 없다. |

## bit 파일

- 경로: `vivado/piso.runs/impl_1/lab2_piso.bit`
- SHA-256: `718c671177411c4eae67acaf6d6b54f2119f5c3299235e33986852140ec29ef8`
- 보고서 원본: `evidence/post/`의 `synth_runme.log`, `impl_runme.log`, `drc_routed.rpt`, `methodology_drc_routed.rpt`, `timing_summary_routed.rpt`, `utilization_placed.rpt`.

## 실제 장치 기록과 관찰

- Hardware Manager 콘솔에서 `open_hw_target` → `program_hw_devices`가 실행되었다(작성자 PC(djawl)에서 직접 프로그래밍). 콘솔 로그: `evidence/post/vivado_console.txt`.
- Program Device 화면, 보드 전체 사진, 조작 영상은 `evidence/post/`에 추가한다. 아래 표는 실험 중 이미 확인·통과된 결과를 기록한다. 시연 영상: [Google Drive 폴더](https://drive.google.com/drive/folders/1bcvEsSmA-Qr2RFuTtRCmkes4qBkjJrJG?hl=ko).

| 조작 | 예상 (사전 레포트) | 실제 관찰 | 비고 |
|---|---|---|---|
| SW1..4=1010, SW8=1, N8 한 번 | A1 (load, serial_out=1) | A1 (load, serial_out=1) | 예상과 일치 |
| SW8=0, N8 한 번 | 40 | 40 | 예상과 일치 |
| N8 한 번 더 | 81 | 81 | 예상과 일치 |
| N8 한 번 더 | 00 | 00 | 예상과 일치 |
| N8 한 번 더 | 00 | 00 | 예상과 일치 |

## 예상과 실제의 차이·문제 해결

차이 없음 — 시뮬레이션(Icarus/XSim)과 실물 보드 동작 모두 사전 레포트의 예상과 일치했다.

## 링크

- 소스 커밋 / 제출 태그: `<기입 — 커밋 해시> / lab2-05-submit-v1`
- 사전 레포트: [pre_report.md](../pre/pre_report.md)
- 실행 로그·파형·사진·영상: `evidence/`
- GitHub 검증 기록(날짜): `<기입>`
