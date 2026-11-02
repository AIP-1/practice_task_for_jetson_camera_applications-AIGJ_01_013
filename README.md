# practice_task_for_jetson_camera_applications — Jetson 카메라 응용 실습 (T0~T4, B1·B2, A1·A4)

AIGJ_01_013 이진행

Jetson Orin Nano Super에 USB 웹캠 2대와 IP 카메라를 연결해 카메라 입력·지연(T0), 물체 인식(T1), 재실 인원(T2), 행동 패턴(T3), 스테레오 거리(T4)를 차례로 만들고, 심화 B1·B2(하드웨어 복원, RTSP 송출)와 응용 A1(재실 인원)+A4(근접 경고)를 한 서비스로 묶어 12시간 운영한 실습입니다. 실습 키트(카메라 도구)가 보드에 없어 같은 역할의 도구를 직접 구현했습니다(`src/t0`).

- **최종 보고서:** [report/최종보고서.md](report/최종보고서.md)
- **기능 시험 기록:** [docs/T0-T4_기능시험기록.md](docs/T0-T4_기능시험기록.md)

## 핵심 결과

| 과제 | 결과 |
|---|---|
| P 준비 점검 | 전력 모드·MQTT OK, 카메라 2대 480M, 키트 없음 → 도구 직접 구현 |
| T0 연결·뷰어·지연 | 웹캠 30fps 끊김 0, 최신 1장 리더 지연 46ms, 재연결 자동, glass-to-glass 웹캠 180ms·IP 514ms |
| T1 물체 인식 | TensorRT FP16 엔진 4.0ms(PyTorch의 6.3배), 선 통과 IN 5/OUT 5, 개방 어휘 YOLOE-11l 4/5 |
| T2 재실 인원 | MAE 1.23~1.77, 빈 방 오판 0%, 사람이 많을수록 적게 셈 → 기준(MAE ≤ 1.0) 미달 |
| T3 행동 패턴 | 만세 깜빡임 9 → 0, WAVE 오판 0/10, 악수 오류 1, 행동 판정 10/12 |
| T4 스테레오 거리 | 재투영 0.144px, 0.9~4.05m 6개 거리 모두 허용 오차 안, 사람별 거리 ±1.3%(2.7m 이상) |
| B1·B2 심화 | NVDEC은 CPU 복원의 1/4~1/5 CPU, 웹캠 → RTSP → 보드 검출, 프레임 나이 약 50ms |
| A1+A4 응용 | 12시간 health ok 100%(4,320/4,320), 재시작 0, 카메라 재연결 0, GPU 평균 41% |

T2는 카메라 한 대가 강의실 앞쪽에서 뒷줄을 비스듬히 보는 구도라 모니터·앞사람에 가린 뒷줄 착석자를 놓치는 것이 주원인이었습니다(자세한 분석은 보고서 4절).

## 폴더 구성

```
report/           최종 보고서 (Markdown)
docs/             과제별 결과 기록 (P, T0~T4, B1·B2, A1·A4 설계·운영 메모)
  results/        측정 원자료(csv/json): 지연, 신뢰도, 선 통과, 재실 인원, 거리, 12시간 운영
src/              최종 코드 (폴더 이름은 보드의 ~/eg2-lab/applab/ 그대로)
  t0/             공용 카메라 도구 camlib(최신 1장 리더·재연결), 뷰어, 지연 측정
  t2/ ~ t6/       과제 2~6 = 보고서의 T1~T4, A1+A4 (아래 표)
  b1/, b2/        심화 B1·B2
  archive/        중간 버전 (line_count v1, occupancy v1·v2, pose_app ROI 전 등)
ipcam.env.example IP 카메라 계정 견본
```

## 코드와 실행 순서

모든 스크립트는 젯슨의 `~/eg2-lab/applab/`에서 실행했습니다. 실행 전 가상환경을 켭니다: `source ~/eg2/bin/activate`

| 순서 | 파일 | 과제 | 하는 일 |
|---|---|---|---|
| 1 | `t0/camlib.py`, `probe.py`, `view.py`, `latency.py`, `reconn.py` | T0 | 카메라 열기(웹캠·RTSP·파일), 최신 1장 리더, 웹 뷰어, 지연·재연결 시험 |
| 2 | `t2/export_one.py`, `bench.py`, `run_bench.sh` | T1-1 | YOLO11n/s TensorRT FP16·INT8 변환, 속도·mAP 비교 |
| 3 | `t2/conf_probe.py`, `line_count.py`, `items.py` | T1-4·5·8 | 조명×거리 신뢰도, 가상선 IN/OUT + MQTT, COCO 밖 물체 3가지 방법 |
| 4 | `t3/occupancy.py`, `label_t3.py`, `t3_eval_final.py`, `t3_fixeval_final.py` | T2 | 구역별 재실 인원(MQTT retain), 정답 라벨링, 정확도 평가 |
| 5 | `t4/pose_app.py`, `flicker.py`, `summ.py`, `t31sum.py` | T3 | 관절 검출, 동작 판정 안정화, 악수, 행동 패턴(`--behavior`) |
| 6 | `t5/make_board.py`, `capture_pairs.py`, `calib_stereo.py` | T4-1~3 | 체커보드, 좌우 쌍 촬영, 스테레오 보정 |
| 7 | `t5/measure_dist.py`, `fail_offline.py`, `person_dist.py`, `synth_stereo.py` | T4-4~9 | 지점 거리, 실패 조건, 사람별 거리, 합성 영상 검증 |
| 8 | `b1/run_all.sh`, `gst_run.py`, `cv_run.py` | B1 | NVDEC(GStreamer) vs CPU 복원 비교 |
| 9 | `b2/fake_camera_linux.sh`, `rtsp_infer.py` | B2 | 웹캠 → RTMP → MediaMTX → RTSP → 사람 검출 |
| 10 | `t6/service.py`, `usb_setup.sh`, `t6-vision.service`, `mon12h.py`, `summ12h.py` | A1+A4 | 재실 인원+근접 경고 통합 서비스(REST :8030, MQTT), systemd 사용자 서비스, 12시간 감시·요약 |

통합 서비스 설치·운영 방법은 [docs/A1A4_운영메모.md](docs/A1A4_운영메모.md)에 있습니다.

## 환경

- Jetson Orin Nano Super 8GB, 전력 모드 MAXN_SUPER
- Python 3.12, torch 2.14.1+cu132, ultralytics 8.4.184, TensorRT 10.16.2, OpenCV 5.0.0 (FFmpeg 백엔드)
- 카메라: Logitech C920 2대(MJPG), IP 카메라 TP-Link VIGI C540-W(H.265, 2560x1440 / 848x480, 25fps)

## 올리지 않은 것

- 모델 가중치와 TensorRT 엔진(`*.pt`, `*.engine`, `*.onnx`), 스테레오 보정 행렬(`*.npz`, 값은 `docs/results/t5_calib2/calib_25mm.json`에 있음)
- 사람이 찍힌 스냅샷·화면 캡처(`runs/*.jpg`, `t5_fail/`, `t5_person/`)와 시험 영상: 얼굴이 포함되어 보드 밖으로 내보내지 않음
- 원시 프레임 기록(`*.jsonl`, `t4_trace.csv` 등 큰 로그)
- 접속 정보: IP 카메라 계정은 `ipcam.env.example` 형식으로 각자 채움, 카메라 주소는 `<camera-ip>`로 바꿔 둠
- 이 실습은 팀(team4) 공용 보드에서 팀원 정지원 님 계정(`jiwon934`)으로 함께 진행했으며, 코드와 기록은 그 작업 폴더(`applab`)에서 가져왔습니다.
