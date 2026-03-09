# Soccer AI 使用文档（中文）

本文档面向本仓库 `examples/soccer` 示例，目标是让你快速完成：
- 环境安装与资源准备
- 各模式运行
- `core` 数据接口模板接入与导出
- 代码结构与每个代码文件职责理解

## 1. 项目结构

```text
sports-main/
├─ setup.py
├─ README.md
├─ core/
│  ├─ __init__.py
│  └─ state.py
├─ sports/
│  ├─ __init__.py
│  ├─ annotators/
│  │  ├─ __init__.py
│  │  └─ soccer.py
│  ├─ common/
│  │  ├─ __init__.py
│  │  ├─ ball.py
│  │  ├─ team.py
│  │  └─ view.py
│  └─ configs/
│     ├─ __init__.py
│     └─ soccer.py
└─ examples/
   └─ soccer/
      ├─ README.md
      ├─ main.py
      ├─ setup.sh
      ├─ requirements.txt
      └─ notebooks/
         ├─ train_player_detector.ipynb
         ├─ train_ball_detector.ipynb
         └─ train_pitch_keypoint_detector.ipynb
```

## 2. 每个代码文件说明

### 2.1 根目录

- `setup.py`
  - Python 包构建入口。
  - 使用 `setuptools.find_packages()` 自动收集 `sports` 与 `core` 包。
  - 声明基础依赖（`supervision`、`opencv-python`、`transformers`、`umap-learn` 等）。

- `README.md`
  - 项目总览文档（上游介绍、挑战方向、数据集链接）。

### 2.2 core 数据接口层

- `core/__init__.py`
  - 对外统一导出数据接口类型，支持：
    - `from core import GameStateManager, FrameState, PlayerState, ...`

- `core/state.py`
  - 比赛状态数据模型与线程安全状态管理器。
  - 核心内容：
    - 数据类型：`Team`、`PlayerState`、`BallState`、`FrameState`
    - 业务事件：`FoulEvent`、`OffsideQuery`
    - 状态容器：`GameStateManager`（读写、索引、回调、统计）
    - 异步落盘：`AsyncPersistence`（按帧输出 CSV）

### 2.3 sports 公共算法层

- `sports/configs/soccer.py`
  - 定义足球场几何配置：尺寸、关键点顶点、连线、标注标签和颜色。

- `sports/annotators/soccer.py`
  - 负责在“俯视球场图”上绘制场地、点、轨迹。
  - 主要用于 `RADAR` 模式可视化。

- `sports/common/view.py`
  - 透视变换工具 `ViewTransformer`。
  - 支持点坐标变换与整图变换（单应矩阵）。

- `sports/common/ball.py`
  - `BallTracker`：基于历史中心点做简单球目标稳定选择。
  - `BallAnnotator`：在视频帧上绘制球轨迹样式。

- `sports/common/team.py`
  - `TeamClassifier`：SigLIP 特征提取 + UMAP 降维 + KMeans 聚类做球队分类。
  - `create_batches`：批处理工具函数。

- `sports/__init__.py`、`sports/common/__init__.py`、`sports/configs/__init__.py`、`sports/annotators/__init__.py`
  - 包初始化文件（当前无复杂逻辑）。

### 2.4 examples/soccer 示例层

- `examples/soccer/main.py`
  - 示例主程序，包含 7 种运行模式：
    - `PITCH_DETECTION`
    - `PLAYER_DETECTION`
    - `BALL_DETECTION`
    - `PLAYER_TRACKING`
    - `TEAM_CLASSIFICATION`
    - `PLAYER_TEAM_CLASSIFICATION`
    - `RADAR`
  - 已接入 `core` 状态模板（可选）：
    - `--state_output_path`
    - `--state_flush_interval`
  - 按帧构建 `FrameState` 并通过 `AsyncPersistence` 异步写 CSV。

- `examples/soccer/setup.sh`
  - 下载模型权重与示例视频到 `examples/soccer/data/`。
  - 依赖 `gdown`。

- `examples/soccer/requirements.txt`
  - 示例额外依赖：`ultralytics`、`gdown`。

- `examples/soccer/notebooks/*.ipynb`
  - 模型训练示例（球员检测、足球检测、球场关键点检测）。

## 3. 环境准备

### 3.1 Python 版本

- 推荐 `Python 3.11`。
- 建议使用虚拟环境（`venv` / `conda`）。

### 3.2 安装步骤

在仓库根目录执行：

```bash
pip install -e .
cd examples/soccer
pip install -r requirements.txt
./setup.sh
```

说明：
- `pip install -e .` 会安装当前仓库源码（包含 `sports` 和 `core`）。
- `./setup.sh` 会拉取模型与视频到 `examples/soccer/data/`。

## 4. 快速开始

### 4.1 最小可运行命令

```bash
cd examples/soccer
python main.py \
  --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-player-tracking.mp4 \
  --device cpu \
  --mode PLAYER_TRACKING
```

参数说明：
- `--source_video_path`：输入视频路径（必填）
- `--target_video_path`：输出视频路径（必填）
- `--device`：推理设备，常用 `cpu`、`cuda`、`mps`
- `--mode`：运行模式（见下节）
- `--state_output_path`：可选，启用 `core` 状态 CSV 导出
- `--state_flush_interval`：可选，状态异步落盘间隔（秒）

## 5. 各模式用法

### 5.1 PITCH_DETECTION（球场关键点）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-pitch-detection.mp4 \
  --device cpu --mode PITCH_DETECTION
```

### 5.2 PLAYER_DETECTION（球员/守门员/裁判检测）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-player-detection.mp4 \
  --device cpu --mode PLAYER_DETECTION
```

### 5.3 BALL_DETECTION（足球检测）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-ball-detection.mp4 \
  --device cpu --mode BALL_DETECTION
```

### 5.4 PLAYER_TRACKING（球员跟踪）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-player-tracking.mp4 \
  --device cpu --mode PLAYER_TRACKING
```

### 5.5 TEAM_CLASSIFICATION（球队分类）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-team-classification.mp4 \
  --device cpu --mode TEAM_CLASSIFICATION
```

### 5.6 PLAYER_TEAM_CLASSIFICATION（角色区分 + 球队分类）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-player-team-classification.mp4 \
  --device cpu --mode PLAYER_TEAM_CLASSIFICATION
```

### 5.7 RADAR（雷达图叠加）

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-radar.mp4 \
  --device cpu --mode RADAR
```

## 6. core 数据接口模板用法

### 6.1 启用状态导出

```bash
python main.py --source_video_path data/2e57b9_0.mp4 \
  --target_video_path data/2e57b9_0-player-tracking.mp4 \
  --device cpu --mode PLAYER_TRACKING \
  --state_output_path data/2e57b9_0-player-tracking-state.csv \
  --state_flush_interval 0.5
```

行为说明：
- 未传 `--state_output_path`：与旧行为一致，仅输出视频。
- 传入 `--state_output_path`：额外启用 `GameStateManager + AsyncPersistence` 导出状态。
- 当前 CSV 默认仅写球员行（不单独写球）。

### 6.2 CSV 字段说明（当前版本）

- `frame_id`：帧序号（从 0 递增）
- `timestamp`：写入时间戳（`time.time()`）
- `video_ts`：视频时间戳（当前为空）
- `source`：模式名（如 `PLAYER_TRACKING`）
- `player_id`：跟踪 ID（若无 tracker 则使用索引）
- `team`：`HOME` / `AWAY` / `REFEREE` / `UNKNOWN`
- `pixel_x`, `pixel_y`：像素坐标（底部锚点）
- `field_x`, `field_y`：场地坐标（当前默认 0，可后续接投影模块）
- `speed`：速度（当前默认 0）
- `confidence`：检测置信度
- `jersey_number`：球衣号（当前默认空）

## 7. 常见问题

- 权重下载失败
  - 检查是否已安装 `gdown`，以及网络是否可访问 Google Drive。

- 运行很慢
  - 优先使用 `--device cuda`（NVIDIA）或 `--device mps`（Apple Silicon）。

- 找不到 `core` 包
  - 确保在仓库根目录执行过 `pip install -e .`。

## 8. 许可说明

- `ultralytics`（YOLOv8）遵循 AGPL-3.0。
- 本仓库 `sports` 代码遵循 MIT（见根目录 `LICENSE`）。
