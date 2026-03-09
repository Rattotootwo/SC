# Sports Main 项目说明（中文）

本仓库是一个以足球视频分析为主的视觉项目，当前结构已统一为：
- `core/`：稳定的数据结构与状态管理中台
- `tracking/`：检测、跟踪、分类、雷达可视化等业务实现

## 1. 目录结构

```text
sports-main/
├─ requirements.txt
├─ setup.py
├─ README.md
├─ core/
│  ├─ __init__.py
│  ├─ state.py
│  └─ README.md
└─ tracking/
   ├─ __init__.py
   ├─ main.py
   ├─ README.md
   ├─ setup.sh
   ├─ requirements.txt
   ├─ annotators/
   ├─ common/
   ├─ configs/
   ├─ data/
   ├─ input_videos/
   ├─ output_videos/
   └─ notebooks/
```

## 2. 功能概览

- 球场关键点检测（`PITCH_DETECTION`）
- 球员检测（`PLAYER_DETECTION`）
- 足球检测（`BALL_DETECTION`）
- 球员跟踪（`PLAYER_TRACKING`）
- 球队分类（`TEAM_CLASSIFICATION`）
- 球员+球队分类（`PLAYER_TEAM_CLASSIFICATION`）
- 雷达视图叠加（`RADAR`）
- 可选状态输出（CSV），由 `core` 提供统一数据结构与异步持久化

## 3. 环境安装（统一依赖）

推荐 Python `>=3.8`（建议 3.11）。

```bash
python -m venv .venv
source .venv/bin/activate
pip install -U pip
pip install -r requirements.txt
pip install -e .
```

说明：
- `requirements.txt` 是主目录统一依赖清单，可覆盖完整运行所需依赖（含 `tracking` 运行依赖）。
- `pip install -e .` 用于本地可编辑安装，确保 `core` / `tracking` 在任意工作目录都可导入。

## 4. 准备模型与示例视频

```bash
cd tracking
./setup.sh
cd ..
```

资源会下载到 `tracking/data/`。

## 5. 快速开始

建议在仓库根目录执行：

```bash
python tracking/main.py \
  --source_video_path tracking/data/2e57b9_0.mp4 \
  --target_video_path tracking/data/2e57b9_0-player-tracking.mp4 \
  --device cpu \
  --mode PLAYER_TRACKING
```

启用状态输出：

```bash
python tracking/main.py \
  --source_video_path tracking/data/2e57b9_0.mp4 \
  --target_video_path tracking/data/2e57b9_0-player-tracking.mp4 \
  --device cpu \
  --mode PLAYER_TRACKING \
  --state_output_path tracking/data/2e57b9_0-player-tracking-state.csv \
  --state_flush_interval 0.5
```

## 6. 参数说明

- `--source_video_path`：输入视频路径（必填）
- `--target_video_path`：输出视频路径（必填）
- `--device`：`cpu` / `cuda` / `mps`
- `--mode`：运行模式
- `--state_output_path`：可选，启用 CSV 状态导出
- `--state_flush_interval`：可选，异步刷盘间隔（秒）

## 7. 与 core 的关系

`tracking/main.py` 在运行时可选创建：
- `GameStateManager`
- `AsyncPersistence`

并按帧写入 `FrameState`。  
`core` 的详细接口见 `core/README.md`。

## 8. 扩展开发

新增模块接入规范见主目录文档：
- `MODULE_INTEGRATION.md`

## 9. 许可

- 本仓库代码遵循 MIT（见 `LICENSE`）
- `ultralytics` 遵循 AGPL-3.0，请按其许可条款使用
