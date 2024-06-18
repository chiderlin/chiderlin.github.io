---
title: python_async
date: 2024-06-12 16:14:55
tags:
categories:
---

### Fastapi 的 BackgroundTasks 方式做異步

```python
def cron_tasks():
    redis_conn = get_redis()
    logger.info("Cron task started")
    process_google_news()
    logger.info("Cron task finished")
    redis_conn.set("task_running", "0")


@news_api.get("/tasks/")
def trigger_cron(background_tasks: BackgroundTasks, redis=Depends(get_redis)):
    """
        Trigger Cron task APIs
    """
    try:
        task_running = redis.get("task_running")
        if task_running is not None:
            int_value = int(task_running)
            bool_value = bool(int_value)
            if bool_value:
                raise HTTPException(status_code=400, detail="Task is already running")

        redis.set("task_running", "1")
        background_tasks.add_task(cron_tasks)
        return { "message": "Cron task triggered" }
    except HTTPException as e:
        return JSONResponse(status_code=e.status_code, content=e.detail)
```

#### 注意：

沒有一個 async & await 在 function 中，連 process_google_news()裡的也沒

```python
def process_google_news():
    logger.info(f"process_google_news() called")
    with Session() as db_session:
        fetch_and_store_news(db_session)
```

### asyncio 的方式做異步

##### 仍有問題

在於當我打了 cron task 之後，再打一次應該要回傳 已經在執行，但這邊會卡住
也不能打其他的 API... (其他 API 也是會卡住的狀態)

async & await 還要再琢磨..

```python
import asyncio


async def cron_tasks():
    redis_conn = get_redis()
    logger.info("Cron task started")
    await process_google_news()
    logger.info("Cron task finished")
    redis_conn.set("task_running", "0")


@news_api.get("/tasks/")
async def trigger_cron(redis=Depends(get_redis)):
    """
    Trigger Cron task APIs
    """
    try:
        task_running = redis.get("task_running")
        if task_running is not None:
            int_value = int(task_running)
            bool_value = bool(int_value)
            if bool_value:
                raise HTTPException(status_code=400, detail="Task is already running")

        redis.set("task_running", "1")
        asyncio.create_task(cron_tasks())
        return { "message": "Cron task triggered"}
    except HTTPException as e:
        return JSONResponse(status_code=e.status_code, content=e.detail)
```

## 兩者差異

### Asyncio

asyncio 是 Python 的標準庫，專門用於寫異步代碼。它提供了事件循環、協程和其他異步原語，可以用來處理複雜的並發任務。asyncio 非常強大，但也相對複雜，適合需要精細控制異步行為的場景。

#### 使用情境：

- 需要精細控制任務的啟動、停止和協調。
- 需要並行處理多個異步任務。
- 需要長時間運行的任務或計算密集型任務。
- 需要與其他異步代碼或庫進行整合。

### BackgroundTasks

BackgroundTasks 是 FastAPI 提供的一種簡化的方式，用於將任務放到後台運行。它不需要手動管理事件循環或協程，使得代碼更簡潔和易於維護。適合簡單的背景任務，比如發送電子郵件、記錄日誌等。
使用情境：

#### 使用情境：

- 需要簡單地將任務放到後台運行。
- 不需要精細控制任務的執行。
- 任務相對簡單且運行時間較短。
- 希望保持代碼簡潔、易於維護。

## 一次結合兩個套件

### 結論：

選擇哪一個取決於你的應用需求和任務的複雜性。如果你的任務需要更多的控制和並發處理，asyncio 可能是更好的選擇；如果你只是需要簡單地將任務放到後台運行，BackgroundTasks 則更加適合。
