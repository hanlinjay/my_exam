sequenceDiagram
    autonumber
    participant Client as 客户端线程 (x6)
    participant TIFILE as TIFILE 引擎
    participant Queue as 简单队列 (Queue)
    participant Async as 后台异步线程 (Worker)
    participant TICache as TI-Cache 服务器
    participant Cache as 本地缓存 (Cache)

    Note over Client, Queue: —— 投递阶段：不唤醒线程，极速返回 ——
    Client->>TIFILE: 提交检测请求
    TIFILE->>Cache: 查询本地缓存 (锁保护)
    Cache-->>TIFILE: 缓存未命中 (Miss)
    TIFILE->>Queue: 锁定并加入队列 (不进行条件变量唤醒)
    TIFILE->>Client: 临时通过 AV 兜底放行
    
    Note over Async, TICache: —— 消费阶段：定时 10 分钟触发 ——
    Async->>Async: 定时 10 分钟到达 (唤醒)
    rect rgb(230, 245, 255)
        Note right of Async: 读取队列时加锁，提取完毕立即解锁
        Async->>Queue: 申请队列锁 (Lock)
        Async->>Queue: 一次性提取最多 20 条任务 (若不足20则全取)
        Async->>Queue: 释放队列锁 (Unlock)
    end
    
    rect rgb(255, 240, 240)
        Note right of Async: 锁外网络查询 (避免阻塞主链路投递)
        Async->>TICache: 发起云端批量信誉查询
        TICache-->>Async: 返回批量查询结果 (信誉数据)
    end

    rect rgb(230, 255, 230)
        Note right of Async: 写入本地缓存 (加锁)
        Async->>Cache: 申请缓存锁 (Lock)
        Async->>Cache: 批量写入更新本地缓存
        Async->>Cache: 释放缓存锁 (Unlock)
    end
    
    Async->>Async: 继续休眠 10 分钟
