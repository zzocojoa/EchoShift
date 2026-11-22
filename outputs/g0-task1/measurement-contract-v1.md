# G0 측정 계약 v1

Task Master #1의 앱·PC 공통 계약이다. D01 Galaxy S23+ 한 대의 내장 스피커와 마이크를 대상으로 30개 본 실험 슬롯을 고정한다. 수치와 상태는 [계측 앱 설계](../g0-android-instrumentation-plan.md)를 따른다.

## 30회 배치와 배포

| 파일 | 용도 |
| --- | --- |
| [측정 상수](measurement-constants-v1.json) | 앱과 PC가 공통으로 사용할 프로필, 프레임 수, 장면 코드와 판정 기준 |
| [운영자용 순서](operator-runs-v1.jsonl) | 실행 순서, 중립 `trial_id`, 사용할 `profile_id` |
| [배치 재현 정보](batch-meta-v1.json) | 배치 버전, 재현 시드, 파일 해시. 배치 담당자가 보관 |
| [배치 담당자용 장면표](coordinator/ground-truth-plan-v1.jsonl) | 배치 순서별 판 위치와 반복 번호, 계획 거리. 실제 거리 기록은 현장에서 추가 |
| [D01 실기 환경 기록](coordinator/environment-v1.json) | 제안한 설치 조건과 확인한 실제 값. 현장 확인 전 `observed`는 `null` |

- 원본 배치 파일을 확보할 수 없어 `G0-001`부터 `G0-030`까지의 중립 ID로 재작성했다. 재현 시드와 장면표는 배치 담당자가 보관한다. 앱에 전달하는 일정표는 `operator-runs-v1.jsonl` 한 파일이다. 운영 중 화면과 ZIP에 장면표를 넣지 않는다.
- 진단 `diag-16-22`와 운영 후보 `candidate-20-22` 각각에 `no_plate`, `plate_050`, `plate_100`을 5회씩 배정한다. 반사판을 다시 놓거나 치울 때에도 전화기, 거치대와 다른 주변 물체의 위치를 유지한다. `no_plate`의 판 거리는 `null`이다.
- 예비 확인과 파일럿은 `G0P-001`부터 별도 번호를 쓴다. 본 실험의 누락·중단·무효는 예정 슬롯과 최초 시도 분모에 남긴다. 재시험에는 같은 `trial_id`, 새 `attempt_id`, 바로 앞 시도를 가리키는 `retry_of`를 쓴다.
- 한 시도 디렉터리는 `trial_id/attempt_id/`다. 최초 시도는 `G0-001-A01` 형태로 발급하고 `retry_of: null`이다. 재시험은 `G0-001-A02`처럼 번호를 올린다. ID와 폴더명에 장면·거리·시도 결과를 넣지 않는다.

프로젝트 루트에서 `node .\.taskmaster\scripts\build-g0-batch-v1.mjs`를 실행하면 같은 배치를 생성한다. 기존 v1 파일이 동일하면 그대로 사용하며, 내용이 달라진 v1 파일을 발견하면 오류를 반환한다.

## 파형과 프레임 기준

- 앱 입출력은 48,000 Hz, mono, signed PCM16 little-endian이다. 한 프레임은 2바이트다. 인덱스는 0부터 시작하고 범위는 `[start_frame,end_frame)`으로 기록한다.
- TX 본 실험은 480,000프레임이다. 시작 무음 `[0,96000)`, 송신 구간 `[96000,384000)`, 종료 무음 `[384000,480000)`을 사용한다. 준비·종료 가드의 디지털 무음은 본 실험 범위 밖에 둔다.
- chirp `k=0…59`는 `96000 + 4800 × k`에서 시작하며 길이는 960프레임이다. `k`가 짝수면 상승, 홀수면 하강이다. 마지막 chirp의 끝은 380,160프레임이다. 생성에 사용할 선형 sweep과 양끝 taper는 기존 설계를 따른다. #2에서 파형 생성과 해시를 검증하고 #7에서 실기에서 사용할 진폭·볼륨과 가드 길이를 확정한다.
- `rx.wav`의 0번은 앱이 최초로 저장한 RX 프레임이다. 시작·종료 가드를 포함해 연속 PCM을 보존한다. `tx.wav`의 0번은 TX 본 실험 시작 프레임이다. `AudioTrack.write()`가 수락한 해당 구간의 PCM만 보존하며 물리적 방사 여부는 실기로 확인한다.
- 하드웨어 프레임 위치와 read/write가 수락·전달한 앱 프레임 수는 별개다. 비교 가능한 시각은 같은 기기의 `MONOTONIC`으로 기록한다. 필드 이름이 `_ns`인 값은 `0` 이상의 10진 문자열이다. 실패·미관측은 `null`과 `reason_code`를 함께 남긴다. [Android AudioTimestamp 계약](https://developer.android.com/reference/android/media/AudioTimestamp)

## 원시 파일 계약

WAV는 PCM RIFF/WAVE, JSON·JSONL은 UTF-8이다. JSONL은 한 줄에 객체 하나를 기록한다. 모든 시도에서 저장한 파일은 수정하지 않고 PC 판정 결과를 별도 위치에 쓴다. 앱과 PC는 다음 필드명을 공통으로 사용한다.

| 파일 | v1 필드와 해석 |
| --- | --- |
| `manifest.json` | `schema_version`(정수 1), `contract_id`, `trial_id`, `attempt_id`, `retry_of`, `environment_id`, `profile_id`, `profile_sha256`, `device_model_code`, `android_release`, `android_build_display`, `app_build`, `input_source`, `input_route`, `output_route`, `media_volume_index`, `sample_rate_hz`, `channels`, `sample_format`, `tx_trial_start_stream_frame`, `rx_trial_window`({`start_frame`,`end_frame`} 또는 `null`), `capture_status`, `reason_code`, `files`(파일명→`bytes`, `sha256`, WAV의 `frames`). 부분 시도에서 얻을 수 없는 값은 `null`로 보존한다. |
| `audio_clock.jsonl` | `stream`(`rx`/`tx`), `stream_epoch`, `timebase`(`MONOTONIC`), `frame_position`(랩을 해제한 시계 프레임), `frame_position_raw`(API 원값), `audio_time_ns`, `query_ns`, `app_frames_delivered`(RX만), `app_frames_accepted`(TX만), `status`(`ok`/`unavailable`/`invalid`), `api_code`, `reason_code`. 시계가 실패하면 대응 프레임·시각 값은 `null`이다. |
| `chirps.jsonl` | `chirp_idx`, `direction`(`up`/`down`), `tx_file_start_frame`, `tx_stream_start_frame`, `planned_frames`, `write_accepted_frames`(0–960 또는 `null`), `estimated_tx_time_ns`, `reason_code`. 중단 시도에도 예정 60행을 보존한다. 캡처·예측 항목은 관측한 행만 채운다. |
| `quality.jsonl` | `event`, `stream`, `stream_epoch`, `observed_mono_ns`, `frame_domain`, `start_frame`, `end_frame`, `api_code`, `reason_code`, `action`. 프레임의 기준은 `rx_app`, `rx_file`, `tx_stream`, `tx_file` 중 해당 항목을 쓰고 영향 범위를 확정할 수 없으면 `null`로 둔다. |

`capture_status`는 수집 중 `running`, 정상 파일 마감 `complete`, 품질 또는 경로 문제 `invalid`, 마감 전 종료 `interrupted` 중 하나다. `complete`인 시도도 PC의 chirp 품질과 반사 판정이 남는다. 유효 파일은 확정된 파일명·프레임 수·SHA-256과 대조한다. 중단본은 보존하며 RX 손실을 0 샘플로 메우지 않는다. `quality`는 `partial_read`, `partial_write`, `stall`, `queue_overflow`, `route_changed`, `input_silenced`, `tx_underrun`, `anchor_invalid`, `alignment_failed`, `clipping`, `file_error`와 API 오류를 기록한다. 확인하지 못한 RX 하드웨어 손실 건수는 `null`이다.

정답표와 분석 결과는 원시 파일과 분리한다. 배치 담당자는 `trial_id`로 장면을 확인하고 시도마다 `attempt_id`, `observed_distance_m`, 온도·소음·판 이동 등 배치 이탈을 별도 기록한다. PC는 `attempt_id`와 `chirp_idx`로 유효성, 직접·반사 피크의 RX 파일 프레임, 상대 지연, 배경 대비 dB와 무효 이유를 연결한다.

## 실기 환경과 종료 기준

| 구분 | 결정 및 현재 상태 |
| --- | --- |
| 실험 환경 | D01 Galaxy S23+를 한 거치대·한 방에서 같은 자세로 고정하고, 우선 케이스를 제거한다. 폰 기준점과 판의 앞면에 기준 표시를 한다. 0.5 m·1.0 m는 폰 기준점에서 판 앞면까지 수직으로 잰다. 판이 없는 조건에서는 판만 시험 구역 밖으로 치운다. |
| #1 실물 확인 | 실제 모델 코드, Android 버전·빌드, 케이스와 방향·거치 ID, 방과 온도·배경 소음, 판 규격과 각도, 0.5 m·1.0 m 설치 실측, 확인 가능한 초기 볼륨 인덱스를 기록한다. 이 Windows 호스트의 표준 SDK 경로와 PATH에서 ADB를 찾지 못했고, 연결 장치 검색에서도 D01을 확인하지 못했다. [Android 실제 기기 확인](https://developer.android.com/studio/run/device) |
| #7 실기 파일럿 | 준비된 APK로 실제 내장 입력·출력 경로, 48 kHz와 연속 PCM, 앵커, 직접 피크와 반사 시간창, 방해음을 검증한다. 시험용 최종 볼륨·TX 레벨·허용 오차·프로필 해시는 별도 파일럿 후 고정한다. |
| #8 본 실험 | G0 배치 30회를 순서대로 수행한다. 최초 1,800개 chirp 중 1,710개 이상, 시행당 chirp 57개 이상, 같은 파형·거리별 5회 중 4회 이상 및 6 dB 기준을 적용한다. 기준 통과는 실제 WAV와 PC 판정으로 기록한다. |

**#1 종료 확인표**

- [x] 측정 상수와 파일·프레임·시도 ID 계약을 기록했다.
- [x] 운영자용 중립 일정과 담당자용 장면표를 분리했다.
- [ ] D01 실물과 거치·반사판·시험실 조건을 확인하고 환경 기록에 실제 값을 채운다.

실물 확인까지 Task Master #1은 `in-progress`다. 이 패키지는 30회의 **계획**이며 실제 녹음이나 통과 판정을 포함하지 않는다.
