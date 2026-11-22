# G0 Android 계측 앱 설계

Galaxy S23+ 한 대의 내장 스피커와 마이크를 동시에 가동해, 기준 반사판의 위치에 따라 반사 지연이 이동하는지 검사한다. 앱은 PCM과 계측 로그를 수집한다. PC 검증기는 파일 무결성, 기록 유효율, 반사 지연과 배경 대비 전력을 계산한다.

## 1. 실험 및 파형 계약

| 항목 | 첫 배치 사양 |
|---|---|
| 검사 기기 | D01 Galaxy S23+. 실제 모델 코드, OS 빌드, 앱 빌드, 케이스, 폰 방향과 거치 위치 기록 |
| 30회 배치 | 진단 16–22 kHz와 운영 후보 20–22 kHz × 반사판 없음·0.5 m·1.0 m × 독립 5회 |
| 본 실험 구간 | 앞 2초 무송신, 중간 6초 송신, 뒤 2초 무송신. TX 480,000프레임 기준 |
| PCM | 앱 입출력 모두 48,000 Hz, mono, PCM16. 스트림 형식이 다르면 준비 단계에서 실행 차단 |
| Chirp | 20 ms, 100 ms 간격, 상승·하강 교대, 회당 60개. `k=0…59`의 시작은 `96,000 + 4,800k`이고 길이는 960프레임 |
| 시험 환경 | 같은 폰·거치대·방·출력 볼륨·반사판을 사용. 거리 기준점, 판 재질·크기·각도, 실측 거리, 실온과 배경 소음 메모를 기록 |

본 실험 TX의 반개구간은 `[0,96000)` 무음, `[96000,384000)` 송신, `[384000,480000)` 무음이다. 상승부터 시작하는 선형 sweep에 짧은 양끝 taper를 적용한다. taper 길이, 진폭, PCM 변환 방식, 프로필 버전과 SHA-256을 파일럿에서 확정한다. 마지막 chirp의 끝은 프레임 `380160`(제외)이다. 디지털 0인 준비·종료 여유 구간은 이 타임라인 밖에 둔다.

기존 배치표를 확보하면 `trial_id`에 해당 `run_id`를 그대로 쓴다. 재작성 시에는 6개 조합이 각각 5개인 목록을 고정 시드로 섞고, 목록 버전과 시드를 남긴다. 계측 화면에는 중립 ID와 파형 코드만 표시한다. 배치 담당자가 별도 `ground_truth.jsonl`에 장면, 설정 거리, 실측 거리와 설치 정보를 기록한다. `attempt_id`는 매 시도마다 발급하며 재시험은 같은 `trial_id`와 `retry_of`로 연결한다. 파일럿에는 별도 ID를 쓴다.

## 2. Android 동시 송수신

Kotlin 전면 Activity 하나에 준비 확인, 기록 시작/중단, 시도 결과, ZIP 내보내기를 둔다. API 29 이상을 설계 기준으로 잡아 녹음 구성 콜백을 사용할 수 있게 한다. `RECORD_AUDIO`를 선언하고 런타임에 요청한다. 화면을 켠 상태로 실행하며, 화면이 백그라운드로 가거나 권한을 잃으면 해당 시도를 중단하고 보존한다. [Android 녹음 가이드](https://developer.android.com/media/platform/mediarecorder) · [오디오 입력 공유와 콜백](https://developer.android.com/media/platform/sharing-audio-input#config)

| 구성 | 동작 |
|---|---|
| `AudioRecord` | 내장 마이크. `PROPERTY_SUPPORT_AUDIO_SOURCE_UNPROCESSED`가 참이면 `UNPROCESSED`를 시도한다. 미지원 때는 `VOICE_RECOGNITION`을 파일럿에서 확인하고 배치 전체에 고정한다. 클라이언트 포맷과 조회 가능한 활성 캡처 포맷·효과를 모두 남긴다. |
| `AudioTrack` | `MODE_STREAM`과 `USAGE_MEDIA`를 사용한다. 내장 스피커를 선호 경로로 요청하고 시스템 미디어 볼륨·앱 레벨을 기록한다. 내장 마이크와 스피커가 실제 라우팅되었는지 활성 상태에서 확인한다. |
| 작업 스레드 | RX blocking read, TX blocking write, 파일 writer를 분리한다. 시작값으로 RX 480프레임 블록을 쓰고 AudioRecord 내부 버퍼는 API 최소 바이트와 4블록 크기 중 큰 값 이상으로 지정한다. TX 버퍼와 큐 깊이는 실기 부하 검증 후 고정한다. |
| 경로 감시 | RX/TX 경로 변경 콜백과 녹음 구성 콜백을 사용한다. 변경된 실제 경로, `isClientSilenced`, 소스·포맷·효과 변경을 시도 로그에 남긴다. 신호에 영향을 주는 변경은 시도 중단 사유다. |

[AudioRecord 및 버퍼 계약](https://developer.android.com/reference/android/media/AudioRecord) · [AudioTrack 및 프라임](https://developer.android.com/reference/android/media/AudioTrack#play()) · [AudioRouting](https://developer.android.com/reference/android/media/AudioRouting) · [입력 소스 선택](https://developer.android.com/media/platform/mediarecorder#creating-and-running-a-mediarecorder)

1. 권한, 저장공간, 프로필, 볼륨을 확인하고 시도 매니페스트를 `running`으로 만든다. 출력 전에 PCM 타임라인과 로그 버퍼를 준비한다.
2. RX를 먼저 시작해 첫 실제 PCM 블록을 저장한다. TX 시작 가드의 무음 일부를 프라임하고 재생을 시작한다. 출발점으로 TX 앞뒤 각 2초의 무음 가드를 사용하며, 별도 파일럿에서 길이를 고정한다. 양쪽 활성 경로와 시작 앵커를 확인한 뒤 첫 chirp가 위치한 블록을 보낸다. 무음이 끝나기 전에 확인할 수 없으면 시도를 중단한다.
3. 두 스트림을 유지하며 RX는 읽은 샘플만 writer 큐에 넘긴다. TX는 쓰기가 수락한 샘플 수만큼만 커서를 이동한다. 부분 전송은 나머지를 이어서 처리한다. 연속 무진전, API 오류, 큐 포화 또는 정책 변화 시 중단한다. `ERROR_DEAD_OBJECT` 이후에는 새 시도에서 새 스트림을 연다.
4. 정상 종료 시 TX 재생 위치가 본 실험 구간 끝에 이르렀는지 활성 상태에서 확인한다. 종료 가드가 버퍼에 남아 있는 동안 양쪽 최종 앵커와 출력 underrun 카운터를 읽고, RX에 분석 창 뒤 샘플까지 들어오는 시간을 확보한다. 중단 시 TX를 `pause()`와 `flush()`로 멈춘다. 스트림 해제 후 writer를 마감하고 검증한 결과를 화면에 표시한다.

앱 형식 48 kHz는 하드웨어 경로의 주파수 응답을 보장하지 않는다. Android 근초음파 지원 속성의 범위는 18–21 kHz다. 실제 실험 경로에서 두 파형의 스펙트럼, 수신 여유, 포화·왜곡·클릭음을 별도 파일럿으로 확인한다. [AudioManager 근초음파 속성](https://developer.android.com/reference/android/media/AudioManager#PROPERTY_SUPPORT_MIC_NEAR_ULTRASOUND) · [입력 형식 변환](https://developer.android.com/media/platform/sharing-audio-input#config)

## 3. 프레임과 시각

RX는 `AudioRecord.getTimestamp(ts, TIMEBASE_MONOTONIC)`으로 캡처 프레임과 시각을 얻는다. TX는 `AudioTrack.getTimestamp(ts)`로 출력 프레임의 추정 표시 시각을 얻는다. 둘 다 같은 기기의 monotonic ns로 기록한다. RX 프레임 위치는 64비트다. TX는 하위 32비트 값을 unsigned로 보존하고 같은 스트림 안에서 unwrap한다. 스트림을 새로 열거나 재시작할 때는 `stream_epoch`도 새로 부여한다. [AudioRecord 타임스탬프](https://developer.android.com/reference/android/media/AudioRecord#getTimestamp(android.media.AudioTimestamp,%20int)) · [AudioTrack 타임스탬프](https://developer.android.com/reference/android/media/AudioTrack#getTimestamp(android.media.AudioTimestamp)) · [AudioTimestamp 단위와 시계](https://developer.android.com/reference/android/media/AudioTimestamp)

- 시작 때 프레임이 전진할 때까지 주기적으로 조회한다. 본 실험 시작 전 앵커와 끝 지점 부근의 앵커가 잡히도록 가드를 배치한다. TX가 안정화된 뒤에는 공식 권장 간격에 맞춰 마지막 앵커를 약 10초 간격으로 조회하되 본 실험 끝과 종료 가드에 맞춘다. API 실패와 중복·역행·정지 앵커, 프레임 랩 처리 결과도 보존한다.
- TX의 API 스트림 프레임 0은 시작 가드의 첫 프레임이다. `tx.wav`는 본 실험용 프레임만 내보내므로 `tx_trial_start_stream_frame`을 기록한다. RX는 최초 읽기부터 `rx.wav`의 파일 프레임 0으로 기록한다. 각 블록의 누적 읽기와 저장 카운터를 남겨 연속성 및 파일 오프셋을 확인한다. API가 보고한 캡처 위치와 읽기 카운터는 별개 열이다.
- 진행하는 유효 앵커의 `t(f)=t0+(f-f0)×(t1-t0)/(f1-f0)`로 TX 본 실험 시작·끝을 공통 시각에 놓고 RX 파일의 `[start,end)` 프레임으로 변환한다. 예상 RX 파일 매핑은 시도 내부 연속성을 전제로 한다. 실제 녹음 시작 오프셋은 파일럿의 직접 경로 정렬로 확인한다. 앵커 범위, 계산된 표본화율, 오프셋, 모델 버전, 잔차 또는 관측 부족 상태를 기록한다. 앵커 부족·시각 역행·허용 범위 밖의 외삽·RX 밖의 분석 창은 `null`과 이유로 표시한다.
- 송신 시각의 추정값은 각 chirp의 직접 전달 파형을 찾을 RX 탐색 범위를 정하는 데 쓴다. 반사 지연은 같은 RX 시간축의 `반사 피크 − 직접 경로 피크`로 계산한다. API 조회·read·write 호출 시각은 진단용이며 프레임 시각에는 API 앵커를 쓴다. [AOSP 음향 루프백 측정](https://source.android.com/docs/core/audio/latency/measure#technique)

## 4. 시도 디렉터리와 품질

| 파일 | 계약 |
|---|---|
| `rx.wav` | 준비와 종료 가드를 포함한 실제 연속 수신 PCM. 본 실험 10초는 `manifest.rx_trial_window`로 표시한다. 시계가 달라 RX 프레임 수가 TX와 조금 달라도 실제 프레임 수를 보존한다. |
| `tx.wav` | `AudioTrack.write()`로 수락된 본 실험 구간의 PCM. 중단 시 가능한 수락 부분만 보존한다. 송신 의도와 파일 해석을 위한 디지털 기준 파형이다. |
| `manifest.json` | 스키마·앱·프로필 버전, `trial_id`·`attempt_id`·`retry_of`, 모델·OS·거치, 출처와 실제 경로/포맷, 볼륨, 파일·스트림 오프셋, 분석 창, 프레임/바이트 수, 파일별 SHA-256, `capture_status`와 이유. |
| `audio_clock.jsonl` | RX/TX 앵커, 반환 코드와 시계 종류, `stream_epoch`, 앱 카운터와 조회 시각. ns는 JSON에서 손실 없이 읽도록 10진 문자열로 저장한다. |
| `chirps.jsonl` | 시도마다 예정된 60행. `chirp_idx`, `direction`, 시작 TX 파일·스트림 프레임, `planned_frames`, 쓰기 수락 범위, 예측 시각을 기록한다. PC는 결과를 별도 파일에 `attempt_id`와 `chirp_idx`로 결합한다. |
| `quality.jsonl` | `partial_read`, `partial_write`, `stall`, `queue_overflow`, read/write 오류 코드, `route_changed`, `input_silenced`, `tx_underrun`, 앵커/정렬 실패, 포화 및 파일 오류. 영향 프레임 범위가 알려진 경우 `[start,end)`를 함께 남긴다. |

앵커와 품질 행의 예시는 다음과 같다. 숫자는 설명용이다.

```jsonl
{"stream":"rx","stream_epoch":1,"timebase":"MONOTONIC","frame_position":96000,"audio_time_ns":"580023000000","query_ns":"580028000000","app_frames_delivered":95520,"status":"ok"}
{"stream":"rx","stream_epoch":1,"event":"queue_overflow","frame_domain":"rx_app","start_frame":144000,"end_frame":144480,"action":"invalidate_attempt"}
```

- RX writer는 `filesDir`의 `rx.wav.part`를 시작부터 채운다. 사전 할당 풀과 유한 큐를 사용하고 RX 루프에서 파일·ZIP 작업을 피한다. 포화 이벤트는 writer 큐 밖의 상태에도 남긴다. 수신 손실 지점은 0으로 채우지 않고 앞부분을 보존한 뒤 해당 시도를 무효화한다. 짧은 양수 I/O 자체는 손실과 구분한다.
- PCM WAV는 `RIFF/WAVE`, PCM `fmt `, `data` 순으로 쓴다. mono PCM16 LE에서 `blockAlign=2`, `dataBytes=실제 저장 프레임×2`다. 앱이 쓰는 44바이트 헤더에서 `riffSize=36+dataBytes`와 실제 파일 길이를 검증한다. 최종화는 큐 비우기 → 길이 패치 → 동기화/닫기 → 파일명 확정 → 해시 확인 → 완료 매니페스트 커밋 순서다. 재기동 시 완료 표식이 없는 시도는 중단본으로 분류한다. 복구 가능한 PCM 부분의 전체 프레임과 원본 `.part`를 보존한다. [Microsoft RIFF 구조](https://learn.microsoft.com/en-us/windows/win32/xaudio2/resource-interchange-file-format--riff-) · [Android AtomicFile](https://developer.android.com/reference/android/util/AtomicFile)
- `getUnderrunCount()`의 시작 대비 증가는 TX 앱 버퍼의 underrun 신호다. RX는 전달 오류, 큐 손실, 읽기 정체와 앵커 불연속을 기록한다. 하드웨어 RX 손실을 API로 확인할 수 없으면 `unknown`으로 남긴다. PCM 레일 도달 수와 peak/RMS는 저장 후 계산하고, 아날로그 압축·클리핑 가능성은 파일럿의 레벨 비교로 본다. 시도 전체의 영향을 알 수 없는 오류는 전체 `invalid` 처리한다. [AudioTrack underrun 정의](https://developer.android.com/reference/android/media/AudioTrack#getUnderrunCount())
- 배치 ZIP은 기록 종료 후 SAF의 `ACTION_CREATE_DOCUMENT`로 내보낸다. 정상 및 실패 시도, 설정, 배치 인덱스를 포함한다. 같은 ID의 실패 기록을 덮어쓰지 않는다. 순수 PCM은 본 실험 양방향만 계산해도 배치 약 57.6 MB다. 실제 준비 시 가드·로그·일시 ZIP·예상 재시험까지 계산해 여유 공간을 확인한다. 외부 정답표는 PC에서 별도로 유지한다. [Android 앱 내부 저장소](https://developer.android.com/training/data-storage/app-specific#internal) · [SAF 파일 생성](https://developer.android.com/training/data-storage/shared/documents-files#create-file)

## 5. 파일럿과 30회 판정

**파일럿.** 기록 권한, 내장 경로, 48 kHz 초기화, 2/6/2초 프레임, 양쪽 clock anchor, 직접 경로와 두 대역의 수신 신호를 확인한다. 별도 ID로 각 파형에서 무반사판·0.5 m·1.0 m를 시험한다. 실제 응답, 볼륨, 장면 고정성, 운용 후보의 방해음, 직접 경로 탐색과 반사 창이 재현되는지 확인한다. 그런 뒤 프로필/파일 해시, 시계·품질 허용치, 거리 창과 시간 이동 허용 오차, 배경 계산법을 버전으로 동결한다.

**품질.** PC에서 본 실험의 30개 예정 슬롯과 슬롯당 60개 chirp가 모두 집계에 드는지 확인한다. 유효 chirp에는 수락된 TX 파형, 직접 경로 정렬, 예상 반사 창 전체의 RX 샘플, 검증된 파일·시계가 필요하다. 해당 구간의 PCM 레일 도달·누락·경로 변경·음소거·송신 오류는 무효 사유다. 정상적으로 기록된 무반사판 chirp는 품질 유효에 포함한다. 분모는 첫 시도 1,800개이며 전체 `≥1,710/1,800`을 요구한다. 별도로 `≥57/60`을 반사 재현 집계에 포함할 시행 기준으로 제안한다. 무효한 첫 시도와 미수집 슬롯은 분모에 남기고 재시험 결과를 별도 표시한다.

**반사.** PC에서 실제 `tx.wav`의 상승·하강별 PCM을 정합 필터로 사용한다. 시계 추정으로 직접 경로 탐색을 한 chirp 주기 안에 제한하고, 주기 차이로 오정렬되거나 직접 성분이 모호한 경우를 기록한다. 직접 피크를 기준으로 거리별 고정 시간창에서 `P_peak = max(|상호상관|² / 템플릿 에너지)`와 그 지연을 구한다. 같은 파형·방향·시간창의 무반사판 데이터를 시행별 피크 중앙값으로 요약한 뒤 그 시행 값들의 중앙값을 배경으로 삼는다. 반사판 시행마다 유효 chirp의 `10·log10(P_peak/P_background)` 중앙값과 지연 중앙값을 구한다. 기록 품질을 충족한 무반사판 최초 시도가 4회 미만이거나 배경값이 0이면 해당 파형 판정을 보류한다. [SciPy 상호상관](https://docs.scipy.org/doc/scipy/reference/generated/scipy.signal.correlate.html) · [AOSP 루프백 정렬](https://source.android.com/docs/core/audio/latency/measure#technique)

스피커와 마이크가 같은 위치에 있다고 근사하고 음속을 약 343 m/s로 두면 0.5 m와 1.0 m의 지연은 약 2.92 ms와 5.83 ms다. 48 kHz에서 약 140과 280프레임이다. 실제 창 중심과 시간 이동 예측은 `(스피커→판→마이크 경로 길이 − 직접 경로 길이)/음속`을 쓰고 거치 기하와 실온을 반영한다. 파일럿에서 확정한 시간창과 이동 허용 오차를 본 실험 내내 적용한다. [NASA Glenn 음속 관계식](https://www.grc.nasa.gov/WWW/BGH/sound.html)

**진행.** 각 파형의 각 반사판 거리 셀에서 품질 기준을 충족하고 중앙값이 ≥6 dB인 첫 시도가 5회 중 4회 이상이어야 한다. 같은 파형에서 재현 기준을 충족한 시행들의 지연 중앙값을 거리별로 비교한다. 1.0 m가 0.5 m보다 늦고 지연 차이가 사전 고정한 허용 범위 안이어야 한다. 운영 후보 파형의 방해음 확인과 전체 기록 품질 기준도 충족하면 L1으로 진행한다. 95%, 6 dB, 4/5는 연구 계획의 제안 기준이다. 최초 시도 분모, 시행별 품질 기준, 무반사판 최소 시행 수, 파형별 집계는 이 설계가 택한 판정 규칙이다. 시험 결과의 적용 범위는 D01의 고정 설정과 기준 반사판까지다.

## 6. 구현 검증 및 Task Master 순서

| 검증 | 관측할 결과 |
|---|---|
| 파형과 WAV | 480,000 TX 프레임, 정확한 chirp 시작 60개 및 방향 교대, 양(+)의 부분 전송 카운트, 완결/중단 WAV의 길이·해시 일치 |
| 시계와 분석 | 합성 앵커에서 랩·부족·역행·drift 검증. 합성 PCM에서 알려진 지연, 6 dB 경계, 직접 성분 모호성, ±100 ms 오정렬, 무반사판 대조와 재시험 집계 검증 |
| 실패 복구 | 큐 포화, 저장공간 부족, 경로 변화, 음소거, 강제 종료를 유발. 실패 이유·범위·중단 원본 보존, 다음 시도 진행, 배치 ZIP 재내보내기 확인 |
| D01 파일럿 | 두 파형과 세 장면에서 RX/TX, 앵커, 분석 창, 품질 로그, ZIP 검증. 방해음 또는 경로 문제가 있으면 설정을 고친 뒤 새 파일럿 수행 |

Task Master #1에서 측정 계약과 배치를 먼저 확정한다. #2 파형과 #4 저장을 병행할 수 있다. #3 엔진은 #2 이후, #5 시계·품질 통합은 #3·#4 이후다. #6 PC 검증기는 #1·#2가 끝나면 진행할 수 있다. #5·#6이 끝나면 #7 D01 프리플라이트와 파일럿을 거쳐 #8 본 실험을 시작한다. [Task Master 공식 사용법](https://github.com/eyaltoledano/claude-task-master#common-commands)
