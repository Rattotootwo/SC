# Core 模块说明（中文）

`core` 是本项目的“状态中台”，负责定义统一数据结构、线程安全状态管理与异步持久化。

## 1. 设计目标

- 提供稳定的内存数据结构，供不同算法模块共享
- 通过统一 API 降低模块间耦合
- 保证多线程读写安全
- 支持将状态异步写入 CSV，便于回放与分析

## 2. 主要文件

- `core/state.py`：核心类型与状态管理实现
- `core/__init__.py`：对外导出稳定 API

## 3. 核心数据结构

- `Team`：队伍枚举（`HOME` / `AWAY` / `REFEREE` / `UNKNOWN`）
- `PlayerState`：球员单帧状态
- `BallState`：足球单帧状态
- `FrameState`：一帧完整快照（球员集合、球、来源等）
- `FoulEvent`：犯规事件
- `OffsideQuery`：越位查询与结果

这些结构均使用 `dataclass`，便于扩展和序列化。

## 4. GameStateManager（内存状态管理）

`GameStateManager` 提供线程安全读写能力，内部使用 `RLock` 保护。

常用写入接口：
- `update_frame(frame)`
- `add_foul_event(event)`
- `add_offside_query(query)`

常用读取接口：
- `get_current_frame()`
- `get_recent_frames(n)`
- `get_frame_by_id(frame_id)`
- `get_frames_range(start_id, end_id)`
- `get_player_trajectory(player_id, n_frames)`
- `get_stats()`

事件回调：
- `on_frame(callback)`
- `on_foul(callback)`

## 5. AsyncPersistence（异步持久化）

`AsyncPersistence` 继承 `threading.Thread`，周期性从 `GameStateManager` 拉取新帧并追加写入 CSV。

特点：
- 不阻塞主推理流程
- 只处理新增帧（基于 `frame_id`）
- 自动写表头

典型流程：
1. 初始化 `GameStateManager`
2. 初始化并 `start()` `AsyncPersistence`
3. 主循环持续 `update_frame`
4. 结束时 `stop()` + `join()`

## 6. 对外稳定导出（建议只用此入口）

```python
from core import (
    Team,
    PlayerState,
    BallState,
    FrameState,
    FoulEvent,
    OffsideQuery,
    GameStateManager,
    AsyncPersistence,
)
```

## 7. 新模块接入建议

- 优先通过 `GameStateManager` 交互，不要在模块间直接共享可变字典
- 模块内部尽量“只读消费、集中写入”
- 若需新增字段，优先向 `PlayerState` / `FrameState` 做向后兼容扩展
- 若需要高频输出，优先通过回调或异步方式，避免阻塞主循环
