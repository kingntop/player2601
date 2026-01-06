# CRADS (일반 광고) 편성 로직

"CRADS"(Commercial/Regular Ads로 추정) 시스템은 포도 플레이어의 주요 재생 목록 편성을 관리합니다. 이 시스템은 시간 기반의 "Items"와 콘텐츠 기반의 "Slots"를 조합하여 특정 시간에 어떤 비디오 콘텐츠를 재생할지 결정합니다.

## API 엔드포인트

- **URL**: `/crads` (기본 URL 기준 상대 경로)
- **Method**: `GET`
- **Headers**:
  - `device_id`: 현재 기기의 ID.
  - `auth`: 회사/기기 인증 토큰.

## 데이터 구조

응답 데이터는 크게 두 가지 배열인 `items`와 `slots`로 구성됩니다.

### 1. Items (`items`)
특정 광고 카테고리가 **언제** 재생되어야 하는지를 정의합니다.
- **CATEGORY_ID**: `slots`의 콘텐츠와 연결되는 ID입니다.
- **START_DT**: 해당 스케줄 블록의 시작 일시입니다.
- **END_DT**: 해당 스케줄 블록의 종료 일시입니다.

### 2. Slots (`slots`)
특정 카테고리에 대해 **어떤** 콘텐츠를 재생할지를 정의합니다.
- **CATEGORY_ID**: `items`의 ID와 일치합니다.
- **slots**: 여러 개의 서브 슬롯(sub-slots) 배열이며, 각 서브 슬롯은 `files`(비디오/광고) 목록을 포함합니다.

## 재생 목록 생성 로직

플레이어는 이 원시 데이터를 사용하여 다음과 같은 로직으로 재생 가능한 목록을 생성합니다.

### 슬롯 평탄화 (LCM 알고리즘)
하나의 카테고리는 파일 개수가 서로 다른 여러 서브 슬롯을 포함할 수 있으므로, 시스템은 **최소공배수(LCM)** 알고리즘을 사용하여 모든 서브 슬롯이 조화를 이루는 완벽한 루프 시퀀스를 생성합니다.

**과정:**
1.  **길이 계산**: 각 서브 슬롯의 파일 개수를 확인합니다.
2.  **LCM 찾기**: 이 길이들의 최소공배수(LCM)를 계산합니다.
3.  **인터리빙 (Interleave)**: `0`부터 `LCM`까지 반복합니다. 각 단계에서 모듈로 연산(`i % length`)을 사용하여 각 서브 슬롯에서 파일을 하나씩 선택합니다.
    - *결과*: 모든 서브 슬롯의 콘텐츠가 고르게 섞인 하나의 평탄화된 파일 목록이 생성됩니다.
    
    *예시:*
    - 슬롯 A (비디오 2개): [A1, A2]
    - 슬롯 B (비디오 3개): [B1, B2, B3]
    - LCM(2, 3) = 6
    - 결과 시퀀스: A1, B1, A2, B2, A1, B3, A2, B1, A1, B2, A2, B3 ... (구현 순서에 따라 A가 먼저 올 수도, B가 먼저 올 수도 있음).
    - *실제 코드 구현:*
      ```javascript
      // js/api.js
      for (let i = 0; i < lengths.reduce(lcm); i++) {
        originSlot.slots.forEach(slot => {
          // 순환적으로 파일 선택
          const src = fileToPlaylistSrc(slot.files[i % slot.files.length]);
          formattedSlot.files.push(src);
        });
      }
      ```

### 스케줄링 (Scheduling)
1.  **Items 매핑**: 시스템이 `items` 배열을 순회합니다.
2.  **카테고리 매칭**: `CATEGORY_ID`에 해당하는 평탄화된 슬롯 데이터를 찾습니다.
3.  **시간 변환**: `START_DT`와 `END_DT`를 표준 Date 객체로 변환합니다.
4.  **Cron 등록**: `Croner` 라이브러리를 통해 Cron 작업을 등록합니다.
    -   `START_DT`에 해당 재생 목록 재생을 시작합니다.
    -   `END_DT`에 재생을 중지하거나 다른 작업으로 전환합니다.

## 클라이언트 측 스케줄링 코드

- **함수**: `js/api.js`의 `cradsToPlaylists(crads)`
- **함수**: `js/api.js`의 `schedulePlaylists(playlists, currentTime)` (실제 타이머 등록 처리)

## 예시 데이터 (Example Data)

다음은 `/crads` API의 실제 응답 예시입니다. `items`는 스케줄링 정보를, `slots`는 파일 및 광고 정보를 포함합니다.

```json
{
    "code": "R001",
    "message": "success",
    "device_id": 10401,
    "device_code": "10401",
    "device_name": "서울역",
    "call_time": "03:53:44",
    "start_dt": "2026-01-06",
    "end_dt": "2026-01-06",
    "items": [
        {
            "DEVICE_ID": 10401,
            "CATEGORY_ID": 21,
            "CATEGORY_NAME": "서울역-202504",
            "START_DT": "20260106 06:00:00",
            "END_DT": "20260106 23:53:44",
            "RN": 1
        }
    ],
    "slots": [
        {
            "CATEGORY_ID": 21,
            "CATEGORY_NAME": "서울역-202504",
            "_x0024_self": 21,
            "slots": [
                {
                    "CATEGORY_ID": 21,
                    "SLOT_ID": 21,
                    "SLOT_ORDERED": 1,
                    "SLOT_NAME": 1,
                    "_x0024_self": 21,
                    "files": [
                        {
                            "RN": 1,
                            "FILE_SEQ": 21,
                            "SLOT_ID": 21,
                            "TYP": "일반",
                            "HIVESTACK_YN": "N",
                            "URL_YN": "N",
                            "RUNNING_TIME": 11.28,
                            "FILE_NAME": "741.mp4",
                            "D_FILE_NAME": "분수",
                            "FILE_ID": 741,
                            "ORDERED": 0,
                            "VIDEO_URL": "...",
                            "API_URL": "...",
                            "VISTAR_URL": "...",
                            "NAVER_URL": "..."
                        },
                        {
                            "RN": 2,
                            "FILE_SEQ": 22,
                            "SLOT_ID": 21,
                            "TYP": "일반",
                            "HIVESTACK_YN": "A",
                            "URL_YN": "N",
                            "ORDERED": 1,
                            "VIDEO_URL": "...",
                            "API_URL": "...",
                            "VISTAR_URL": "...",
                            "NAVER_URL": "https://nam.veta.naver.com/dooh/vast/3.0?ddid=460-S128540897&adt="
                        }
                    ]
                }
            ]
        }
    ]
}
```

*참고: 위 JSON에서 `HIVESTACK_YN` 값이 'A'인 경우 `NAVER_URL`을 사용하여 광고를 요청합니다.*
