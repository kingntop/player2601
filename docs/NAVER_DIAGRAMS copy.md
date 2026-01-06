# Naver (`A`) Sequence Diagram

이 문서는 Naver VAST 연동(`HIVESTACK_YN: 'A'`)에 대한 상세 시퀀스 다이어그램을 포함합니다.

## Naver Integration Flow
- **특징**: 트래킹(Tracking)이 비디오 로딩 시점에 즉시 발생하며(`Fire-and-Forget`), 결과와 무관하게 DB에 저장됩니다.

### 호출 시점 (Trigger Points)
Naver 광고 요청(`getUrlFromNaver`)은 다음 두 가지 시점에 발생합니다.
1.  **스케줄링 (Cron)**: 단일 아이템 재생 시, 시작 시간 **2분 전**에 호출 (`cronVideo`).
2.  **연속 재생 (Pre-load)**: 현재 비디오 로딩(`loadeddata`) 직후, **다음 아이템**이 Naver 광고일 때 호출.

```mermaid
sequenceDiagram
    participant Scheduler as Cron / Event
    participant Player
    participant NaverAPI as Naver Ad Server
    participant Tracking as Tracking Server
    participant Cache as Browser Cache
    participant DB as Internal DB

    Note over Scheduler, Player: Trigger: 1) 2min Before Schedule OR 2) Prev Video Loaded
    Scheduler->>Player: Trigger Ad Request

    loop Retry Strategy (Max 3 attempts)
        Note over Player, NaverAPI: Check `NAVER_URL` (e.g., ...&adt=)
        alt Ends with 'adt='
            Player->>NaverAPI: axios.get(NAVER_URL + Timestamp)
        else Normal URL
            Player->>NaverAPI: axios.get(NAVER_URL)
        end

        NaverAPI-->>Player: VAST XML Response

        Player->>Player: Validate XML
        alt Invalid XML or No <MediaFile type="video/mp4">
            Note right of Player: Retry (Count++)
        else Valid
            Note right of Player: Break Retry Loop
        end
    end
    Note right of NaverAPI: <Ad id="..."> <Tracking>URL</Tracking> <MediaFile>URL.mp4</MediaFile>
    
    Player->>Player: Parse XML
    
    par Fire-and-Forget Tracking
        loop For each <Tracking> URL
            Player->>Tracking: axios.get(Tracking_URL)
            Note right of Player: "Tracking event=start"<br/>Log error if fails (No Retry)
        end
    and Pre-loading
        Player->>Player: Extract <MediaFile> (Video URL)
        Player->>Cache: axios.get(Video_URL)
        Note right of Player: Pre-loads video to Service Worker/Browser Cache
    end

    Player->>Player: Play Video (from Cache)
    
    Player->>DB: Save Report (Always)
```
