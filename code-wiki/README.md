# SEU CS AI Projects —— Code Wiki

本 Wiki 是对仓库 `SEU CS AI Projects` 的结构化代码文档。该仓库是作者（严晟嘉 / Shengjia Yan，i@yanshengjia.com）在东南大学（SEU）人工智能课程中完成的 **4 个相互独立的人工智能算法作业** 的集合，同时包含一个用于可视化演示的 Qt 图形界面。

## 仓库定位

- 这不是一个共享公共库的单一工程，而是 **4 个可独立编译、独立运行** 的课程项目工作区。
- 各子项目之间 **没有代码级依赖**（不互相 include / import），唯一共同点是作者、代码风格与「用程序求解经典 AI 问题」的主题。
- 每个子项目使用不同的语言与工具链，因此本 Wiki 按「一个项目一篇文档」组织，并额外提供整体架构与构建运行两类横切文档。

## Wiki 目录

| 文档 | 内容 |
| --- | --- |
| [00-架构总览](file:///workspace/code-wiki/00-architecture.md) | 仓库整体架构、模块职责划分、依赖关系与目录结构 |
| [01-24数码问题](file:///workspace/code-wiki/01-24-puzzle.md) | A* 启发式搜索（C++） + Qt5 图形界面演示 |
| [02-遗传算法求函数最大值](file:///workspace/code-wiki/02-genetic-algorithm.md) | MATLAB 实现与 TensorFlow 实现两种版本 |
| [03-百万皇后问题](file:///workspace/code-wiki/03-million-queens.md) | QS4 局部搜索算法（C++） |
| [04-人脸识别神经网络](file:///workspace/code-wiki/04-face-recognition.md) | 反向传播前馈神经网络（C），含 5 个识别任务变体 |
| [05-构建与运行指南](file:///workspace/code-wiki/05-build-and-run.md) | 各项目依赖、编译方式、运行命令与数据准备 |

## 四个子项目速览

| 子项目目录 | 问题 | 方法 | 语言 / 工具链 |
| --- | --- | --- | --- |
| [24-puzzle](file:///workspace/24-puzzle) | 5×5 数码（24-Puzzle）的求解与演示 | A* 启发式搜索（Nilsson 序列分数 / 曼哈顿距离） | C++（STL）+ Qt5 |
| [genetic-algorithm-for-functional-maximum-problem](file:///workspace/genetic-algorithm-for-functional-maximum-problem) | 求 `f(x)=x+10sin(5x)+7cos(4x)` 在 `[0,9]` 上的最大值 | 遗传算法（选择 / 交叉 / 变异） | MATLAB；Python + TensorFlow 1.4 |
| [million-queens](file:///workspace/million-queens) | N 皇后问题（N 可达百万级） | QS4 最小冲突局部搜索 | C++ |
| [neural-networks-for-face-recognition](file:///workspace/neural-networks-for-face-recognition) | 人脸特征识别（5 类任务） | 三层前馈神经网络 + 反向传播 | C（K&R 风格） |

## 参考基准（Ground Truth）

- **遗传算法**：理论最大值 `24.8553627239529`，对应 `x ≈ 7.857`（见 [遗传算法 README](file:///workspace/genetic-algorithm-for-functional-maximum-problem/README.md)）。
- **24-Puzzle**：目标状态分为「普通（NORMAL）」与「螺旋（SPIRAL）」两种，见 [24-puzzle 数据文件](file:///workspace/24-puzzle/data)。
- **人脸识别**：来源于 CMU 15-681 Machine Learning 课程作业的数据集（20 人、每人多张 32×30 灰度图）。

## 许可

仓库整体采用 [Apache License 2.0](file:///workspace/LICENSE)。其中 24-Puzzle 的 A* 搜索模板库（`stlastar.h` / `fsa.h`）及人脸识别项目（`backprop.c` / `pgmimage.c` 等）来自第三方（Justin Heyes-Jones、Jeff Shufelt, CMU），在文件头部保留了原始版权声明。
