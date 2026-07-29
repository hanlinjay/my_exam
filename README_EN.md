sequenceDiagram
    autonumber
    participant Client as Client Thread (Multi-threaded)
    participant TIFILE as TIFILE Engine
    participant Queue as Simple Queue
    participant Async as Background Async Thread (Worker)
    participant TICache as TI-Cache Server
    participant Cache as Local Cache

    Note over Client, Queue: —— Submit file for detection, fast response ——
    Client->>TIFILE: Submit detection request
    TIFILE->>Cache: Query local cache (Lock protected)
    Cache-->>TIFILE: Cache miss
    TIFILE->>Queue: Lock and enqueue (Deduplicate)
    TIFILE->>Client: Detect via Bit AV
    
    Note over Async, TICache: —— Async TI FILE detection, triggered every 5 seconds ——
    Async->>Async: Timer reached 5 seconds (Wake up)
    rect rgb(230, 245, 255)
        Note right of Async: Lock when reading queue, unlock immediately after extraction
        Async->>Queue: Request queue lock (Lock)
        Async->>Queue: Extract all tasks at once (Queue length up to 2000)
        Async->>Queue: Release queue lock (Unlock)
    end
    
    rect rgb(255, 240, 240)
        Note right of Async: Query network outside lock (To avoid blocking submitting thread)
        Async->>TICache: Send query request to TI Cache server (Batch query of 100 items per group, query all if less than 100)
        TICache-->>Async: Return batch query results
    end

    rect rgb(230, 255, 230)
        Note right of Async: Write to local cache (Lock)
        Async->>Cache: Request cache lock (Lock)
        Async->>Cache: Batch write and update local cache
        Async->>Cache: Release cache lock (Unlock)
    end
    
    Async->>Async: Continue sleeping for 5 seconds
