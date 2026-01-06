# Sequence Diagrams for Ad Network Integration

이 문서는 Naver, Hivestack, Vistar 광고 네트워크의 연동 시퀀스 다이어그램을 포함합니다.

## 1. Naver (`A`) Flow
- **특징**: 트래킹(Tracking)이 비디오 로딩 시점에 즉시 발생하며(`Fire-and-Forget`), 결과와 무관하게 DB에 저장됩니다.

```mermaid
sequenceDiagram
    participant Player
    participant NaverAPI as Naver Ad Server
    participant Tracking as Tracking Server
    participant Cache as Browser Cache
    participant DB as Internal DB

    Player->>NaverAPI: GET metadata (NAVER_URL)
    NaverAPI-->>Player: XML Response (MediaFile, Tracking[])
    
    par Fire-and-Forget Tracking
        loop For each Tracking URL
            Player->>Tracking: GET Tracking URL
            Note right of Player: Log error if fails (No Retry)
        end
    and Pre-loading
        Player->>Player: Extract MediaFile (Video URL)
        Player->>Cache: GET Video File (Pre-load)
    end

    Player->>Player: Play Video (from Cache)
    
    Player->>DB: Save Report (Always)
```

## 2. Hivestack (`Y`) Flow
- **특징**: 재생 완료 후 리포팅하며, 리포트 전송 성공 여부와 상관없이 DB에 저장됩니다.

```mermaid
sequenceDiagram
    participant Player
    participant HS_API as Hivestack Server
    participant Cache as Browser Cache
    participant DB as Internal DB

    Player->>HS_API: GET metadata (API_URL)
    HS_API-->>Player: XML Response (MediaFile, Impression)
    
    Player->>Player: Extract MediaFile & Impression URL
    Player->>Cache: GET Video File (Pre-load)

    Player->>Player: Play Video
    
    Note over Player, DB: On Video End
    Player->>HS_API: GET Impression URL (Report)
    Player->>DB: Save Report (Always, even if API fails)
```

## 3. Vistar (`V`) Flow
- **특징**: 재생 완료 후 리포팅하며, **성공(200 OK)시에만** DB에 저장합니다.

```mermaid
sequenceDiagram
    participant Player
    participant VistarAPI as Vistar Server
    participant Cache as Browser Cache
    participant DB as Internal DB

    Player->>VistarAPI: POST metadata (VISTAR_URL + JSON)
    VistarAPI-->>Player: JSON Response (asset_url, proof_of_play_url)
    
    Player->>Player: Extract asset_url & proof_of_play_url
    Player->>Cache: GET Video File (Pre-load)

    Player->>Player: Play Video
    
    Note over Player, DB: On Video End
    Player->>VistarAPI: GET proof_of_play_url
    
    alt Status == 200 OK
        Player->>DB: Save Report
    else Status != 200
        Player->>Player: Log Error (Do NOT Save)
    end
```
