# 新增模块接入指南（中文）

本文档用于指导你把新的分析模块快速接入到当前 `core + tracking` 架构。

## 1. 接入原则

- 保持 `core` 作为统一状态中台
- 新模块与旧模块通过 `GameStateManager` 交换数据
- 读写分离：计算模块产出状态，展示/导出模块消费状态
- 尽量不改已有主流程接口，优先“旁路接入”

## 2. 推荐目录

新增模块建议按功能新增子目录（例如 `/yyy/xxx.py`）

## 3. 最小接入步骤

1. 定义输入输出契约  
输入通常是检测结果、帧图像、历史状态；输出建议归一到 `PlayerState` / `BallState` / 事件结构。

2. 在主循环中构造标准状态  
将新模块输出映射到 `FrameState`，并调用 `game_state.update_frame(frame_state)`。

3. 如有事件输出  
调用 `add_foul_event` 或 `add_offside_query`，必要时新增事件类型。

4. 如需异步落盘  
复用 `AsyncPersistence`，或新增独立后台线程。

## 4. 代码模板（示意）

```python
from core import GameStateManager, FrameState
import time

state = GameStateManager()

def process_one_frame(frame_id: int, players_dict):
    frame_state = FrameState(
        frame_id=frame_id,
        timestamp=time.time(),
        video_ts=None,
        players=players_dict,
        ball=None,
        source="NEW_MODULE",
    )
    state.update_frame(frame_state)
```

## 5. 回调式扩展（推荐）

你可以在不改核心逻辑的情况下添加订阅者：

```python
def on_new_frame(frame_state):
    # 做二次分析、告警、统计等
    pass

state.on_frame(on_new_frame)
```

## 6. 数据结构扩展策略

- 小范围扩展优先使用可选字段（`Optional`）
- 不破坏已有字段语义与默认值
- 变更后同步更新：
  - `core/state.py` 注释
  - `core/__init__.py` 导出
  - 文档（`core/README.md`）

## 7. 接入完成检查清单

- 新模块可独立运行并产出稳定结果
- 主流程无明显性能回退
- `FrameState` / 事件结构字段完整
- CSV 输出可用且字段语义正确
- 异常不会导致主循环崩溃

## 8. 常见问题

- `ModuleNotFoundError: core`  
  先执行 `pip install -e .`，并尽量从仓库根目录运行。

- 新模块拖慢主流程  
  将耗时逻辑放到异步线程或回调消费者中。

- 状态字段不一致  
  统一走 `core` 的 dataclass，不在模块内定义同名“影子结构”。
