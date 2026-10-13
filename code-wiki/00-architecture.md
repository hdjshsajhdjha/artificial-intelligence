# 00 · 架构总览

> 本文描述 `SEU CS AI Projects` 仓库的整体架构、模块职责、依赖关系与目录结构。各项目的算法与函数细节见对应专题文档。

## 1. 仓库性质与总体架构

仓库是一个 **多项目管理型（multi-project）工作区**，而非微服务或分层单体应用。其架构特征为：

```text
SEU CS AI Projects (repo root)
├── LICENSE                 Apache-2.0
├── README.md               仓库索引
├── .gitignore
│
├── 24-puzzle/              [项目 A] 搜索算法 + 图形界面
├── genetic-algorithm-.../  [项目 B] 进化算法（双实现）
├── million-queens/         [项目 C] 局部搜索算法
└── neural-networks-.../    [项目 D] 神经网络
```

四个项目 **物理隔离、语义独立**：每个目录自带源码、数据与构建脚本，可单独拷贝出去编译运行。它们之间不存在头文件包含、包引用或运行时联动。因此本仓库的「架构」本质上是一组 **并列的算法实验单元**，而不是一个具有内部调用链的系统。

### 1.1 统一的设计范式

尽管语言各异，四个项目共享同一套「课程作业」范式：

```text
[问题定义]  →  [算法核心 / 数据建模]  →  [可选：结果可视化]
```

- **算法核心**：每个项目都有一个主导算法（A* / GA / QS4 / Backprop）。
- **可视化层**：24-Puzzle 使用 Qt 动画回放；人脸识别使用 `.pgm` 权重可视化工具；GA 使用 MATLAB/TF 绘图。
- **结果落盘**：24-Puzzle 输出走步序列到文本文件；GA 输出最优个体；人脸识别保存/读取 `.net` 网络权重。

## 2. 模块职责划分

| 子项目 | 核心模块 | 职责 | 语言 |
| --- | --- | --- | --- |
| 24-puzzle | [`src/24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp) | 定义 `PuzzleState` 状态空间与启发式，驱动 A* 求解 24 数码 | C++ |
| 24-puzzle | [`src/stlastar.h`](file:///workspace/24-puzzle/src/stlastar.h) | 通用 A* 模板引擎（开放/关闭列表、节点内存管理、解路径回溯） | C++ 头文件 |
| 24-puzzle | [`src/fsa.h`](file:///workspace/24-puzzle/src/fsa.h) | 固定大小内存分配器（O(1) 分配/释放） | C++ 头文件 |
| 24-puzzle | [`puzzle/mainwindow.cpp`](file:///workspace/24-puzzle/puzzle/mainwindow.cpp) | Qt GUI：读取走步序列并按动画回放 | C++ / Qt5 |
| 遗传算法 | [`src/matlab-ga/genetic_algorithm.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/genetic_algorithm.m) | MATLAB 版 GA 主循环 | MATLAB |
| 遗传算法 | [`src/tensorflow-ga/train_ga.py`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga/train_ga.py) | TensorFlow 版 GA（以张量运算实现种群演化） | Python |
| 百万皇后 | [`src/nqueens.cpp`](file:///workspace/million-queens/src/nqueens.cpp) | QS4 算法：初始化、冲突计数、增量子交换 | C++ |
| 人脸识别 | `src/*/backprop.c` | 神经网络核心：创建/前向/反向/权值更新/存取 | C |
| 人脸识别 | `src/*/imagenet.c` | 图像→输入层、图像名→目标向量编码（任务相关） | C |
| 人脸识别 | `src/*/pgmimage.c` | PGM 图像与图像列表的 I/O | C |
| 人脸识别 | `src/*/facetrain.c` | 训练主程序：命令行解析、训练循环、性能评估 | C |

## 3. 依赖关系

### 3.1 项目间依赖

**无。** 四个子项目彼此不引用，可独立构建。

### 3.2 项目内依赖（组件关系）

**24-Puzzle（C++）**

```text
24puzzle.cpp ──include──> stlastar.h ──include──> fsa.h
     │  (定义 PuzzleState，实现 A* 回调接口 AStarState)
     └──输出──> data.txt ──(手工拷贝)──> data_normal.txt / data_spiral.txt
                                                    │
puzzle/main.cpp ──> mainwindow.(h/cpp) ──> mylabel.h ──> mainwindow.ui
                                                    └──读取──> data_normal.txt
```

**遗传算法**：两个实现完全独立。

```text
test.m ──调用──> genetic_algorithm.m ──调用──> init.m / fitness.m / rank.m
                                          └──> selection.m / crossover.m / mutation.m / plot_ga.m
（target.m 为 MATLAB 全局 GA 工具箱提供目标函数，与本 GA 主流程平行）

train_ga.py   （单文件，自包含）
```

**百万皇后**：单文件 `nqueens.cpp`，函数内部自洽。

**人脸识别**（每个识别器目录结构相同）：

```text
facetrain.c ──> backprop.h/c   (神经网络)
            ──> imagenet.c     (load_target / load_input_with_image)
            ──> pgmimage.h/c   (图像 I/O)
hidtopgm.c ──> backprop.h/c + pgmimage.h/c
outtopgm.c ──> backprop.h/c + pgmimage.h/c
```

### 3.3 外部依赖

| 项目 | 外部依赖 |
| --- | --- |
| 24-puzzle 求解器 | C++ 标准库（`<vector> <set> <algorithm> <cfloat>`），`libstdc++` |
| 24-puzzle GUI | Qt5（`core gui widgets`）、qmake |
| MATLAB 版 GA | MATLAB（全局变量、`plot`、`rand`、`disp`） |
| TensorFlow 版 GA | Python 3.6、TensorFlow 1.4.1、numpy 1.13.3、matplotlib 2.1.1 |
| 百万皇后 | C++ 标准库、`<time.h>` |
| 人脸识别 | libc、libm（Makefile 链接 `-lm`）；无第三方库 |

## 4. 目录结构（详细）

```text
.
├── 24-puzzle/
│   ├── README.md                          项目说明（启发式搜索）
│   ├── data/
│   │   ├── data_normal.txt                普通目标解的初始状态 + 走步序列
│   │   └── data_spiral.txt                螺旋目标解的初始状态 + 走步序列
│   ├── puzzle/                            Qt GUI 子项目
│   │   ├── main.cpp / mainwindow.{h,cpp} / mylabel.h / mainwindow.ui
│   │   └── puzzle.pro                     qmake 工程文件
│   └── src/
│       ├── 24puzzle.cpp                   A* 求解器主程序
│       ├── stlastar.h                     A* 模板引擎（第三方）
│       └── fsa.h                          固定大小分配器（第三方）
│
├── genetic-algorithm-for-functional-maximum-problem/
│   └── src/
│       ├── matlab-ga/  (genetic_algorithm.m, init.m, fitness.m, rank.m,
│       │                selection.m, crossover.m, mutation.m, plot_ga.m,
│       │                target.m, test.m)
│       └── tensorflow-ga/  (train_ga.py, README.md)
│
├── million-queens/
│   └── src/nqueens.cpp                    QS4 求解器
│
└── neural-networks-for-face-recognition/
    ├── doc/                               hw97.pdf、ysjai01.pdf（实验报告）
    └── src/
        ├── face-recognizer/               H1：识别 20 人中的哪一个
        ├── specific-person-recognizer/    H2：识别是否为特定人
        ├── sunglass-recognizer/           H3：识别是否戴墨镜
        ├── pose-recognizer/               H4：识别人脸朝向
        └── expression-regonizer/          H5：识别表情（目录名保持原样，含拼写笔误）
            └── 每个目录：Makefile, backprop.{c,h}, facetrain.c, imagenet.c,
                pgmimage.{c,h}, hidtopgm.c, outtopgm.c, *.list, *.net
```

> 说明：人脸识别 5 个识别器目录里同名文件（`backprop.c`、`pgmimage.c` 等）内容基本一致，**主要差异集中在 `imagenet.c` 的 `load_target` 函数（目标向量编码）与 `facetrain.c` 的网络规模与输出层解析**。

## 5. 数据流概览

| 项目 | 输入 | 核心处理 | 输出 |
| --- | --- | --- | --- |
| 24-puzzle | 硬编码 / 命令行初始状态；`g_goal` 目标状态 | A* 搜索（f = g + h） | 终端打印解 + `data.txt`（棋盘与走步序列） |
| 遗传算法 | 函数 `f(x)`、GA 参数 | 选择/交叉/变异迭代 | 最优个体、最优适应度、迭代曲线 |
| 百万皇后 | 皇后数量 N（命令行输入） | QS4 最小冲突交换 | 无冲突解、耗时与步数 |
| 人脸识别 | `.pgm` 图像列表（`.list`） | 前馈 + 反向传播训练 | 准确率/误差、`.net` 权值文件、可视化 `.pgm` |

## 6. 关键设计要点小结

- **可替换的算法核心**：`stlastar.h` 通过 `AStarState` 抽象接口把「搜索引擎」与「问题状态」解耦；人脸识别的 `backprop.c` 把「网络引擎」与「任务编码（imagenet.c）」解耦。
- **第三方复用**：两处明确复用 CMU / 开源代码（A* 模板、神经网络与图像库），文件头保留版权。
- **配置即宏**：24-Puzzle 与百万皇后大量使用 `#define` 开关（如 `SPIRAL/NORMAL`、`LOCATION/CODE`）在编译期切换行为。
- **可视化作为独立消费者**：24-Puzzle 的 GUI 不参与求解，仅消费文本走步序列做动画回放，求解器与演示层解耦。
