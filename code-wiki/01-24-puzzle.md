# 01 · 24-Puzzle（A* 求解 + Qt 演示）

> 目录：[`24-puzzle/`](file:///workspace/24-puzzle) ｜ 方法：启发式搜索（A*） ｜ 语言：C++ / Qt5

## 1. 解决的问题

求解 **5×5 数码（24-Puzzle）**：棋盘上 24 个编号方块 + 1 个空格，通过移动方块（等价于移动空格）使初始状态到达目标状态。项目由两层组成：

1. **求解层**（`src/`）：C++ 控制台程序，用 A* 求出走步序列。
2. **演示层**（`puzzle/`）：Qt5 图形界面，读取走步序列并动画回放。

目标状态有两种，由宏切换：

- `NORMAL`（普通）：方块 1..24 顺序排列，末位为空格。
- `SPIRAL`（螺旋）：方块按阿基米德螺旋排列。

## 2. 目录与文件

| 文件 | 作用 |
| --- | --- |
| [`src/24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp) | 主程序；定义 `PuzzleState` 并驱动 A* |
| [`src/stlastar.h`](file:///workspace/24-puzzle/src/stlastar.h) | 通用 A* 模板引擎（第三方，Justin Heyes-Jones） |
| [`src/fsa.h`](file:///workspace/24-puzzle/src/fsa.h) | 固定大小内存分配器 |
| [`puzzle/main.cpp`](file:///workspace/24-puzzle/puzzle/main.cpp) | Qt 程序入口 |
| [`puzzle/mainwindow.h`](file:///workspace/24-puzzle/puzzle/mainwindow.h) / [`.cpp`](file:///workspace/24-puzzle/puzzle/mainwindow.cpp) | 主窗口逻辑：加载数据、动画回放 |
| [`puzzle/mylabel.h`](file:///workspace/24-puzzle/puzzle/mylabel.h) | 自定义数字方块标签 `MyLabel` / 背景 `BigLabel` |
| [`puzzle/mainwindow.ui`](file:///workspace/24-puzzle/puzzle/mainwindow.ui) | Qt Designer 界面（一个 `start` 按钮） |
| [`data/data_normal.txt`](file:///workspace/24-puzzle/data/data_normal.txt) | 普通目标的初始状态与走步序列 |
| [`data/data_spiral.txt`](file:///workspace/24-puzzle/data/data_spiral.txt) | 螺旋目标的初始状态与走步序列 |

## 3. 求解层设计（C++）

### 3.1 配置宏

在 [`24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp#L21-L38) 顶部集中定义：

| 宏 | 默认 | 含义 |
| --- | --- | --- |
| `NUM_TIMES_TO_RUN_SEARCH` | 1 | 搜索重复次数 |
| `DISPLAY_SOLUTION_FORWARDS` | 1 | 是否正向打印解路径 |
| `DISPLAY_SOLUTION_BACKWARDS` | 0 | 是否反向打印解路径 |
| `DEBUG_LISTS` | 0 | 是否打印每步的 Open/Closed 列表 |
| `SPIRAL` / `NORMAL` | `SPIRAL=0, NORMAL=1` | 选择目标状态与启发式（二者互斥，用 `#if/#elif`） |
| `BOARD_WIDTH` / `BOARD_HEIGHT` | 5 / 5 | 棋盘尺寸 |

`g_start[]` 的初始状态由一组 `#if 0/#elif 1` 分支示例提供（螺旋 a/b/c、普通 a/b/c/d），编译期选择。

### 3.2 关键类：`PuzzleState`

定义于 [`24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp#L78-L150)，是 A* 引擎的「问题状态」实现，需实现 `AStarState` 约定的 5 个回调：

| 成员 | 说明 |
| --- | --- |
| `TILE` 枚举 | `TL_SPACE(0)`、`TL_1..TL_24`，为方块取值类型 |
| `static TILE g_goal[25]` | 目标状态（由 `SPIRAL/NORMAL` 决定） |
| `static TILE g_start[25]` | 初始状态 |
| `TILE tiles[25]` | 当前节点的棋盘数据 |
| `GoalDistanceEstimate(nodeGoal)` | **启发式函数 h**：`NORMAL` 用曼哈顿距离；`SPIRAL` 用 Nilsson 序列分数 |
| `IsGoal(nodeGoal)` | 是否到达目标（委托 `IsSameState`） |
| `GetSuccessors(search, parent)` | 枚举空格四个方向的合法移动，调用 `AddSuccessor` |
| `GetCost(successor)` | 单步代价，恒为 `1.0f` |
| `IsSameState(rhs)` | 逐格比较是否同状态 |
| `PrintNodeInfo()` / `WriteNodeInfo()` / `WriteDream()` | 终端打印 / 写入文件 / 追加目标状态 |
| `GetSpacePosition` / `LegalMove` / `GetMap`（private） | 空格定位、合法性判定与地图编码 |

#### 启发式函数细节

- **NORMAL**（[`24puzzle.cpp#L517-L603`](file:///workspace/24-puzzle/src/24puzzle.cpp#L517-L603)）：对每个非空格方块累加其目标坐标与当前坐标的曼哈顿距离之和，得到 h。
- **SPIRAL**（[`24puzzle.cpp#L315-L513`](file:///workspace/24-puzzle/src/24puzzle.cpp#L315-L513)）：即 **Nilsson 序列分数** `t = h + 3*s`。
  - `h` = 各方块曼哈顿距离之和；
  - `s` = 序列分数：中心方块不对得 1 分，每个方块若其顺时针后继方块与目标不符得 2 分；
  - 借助三张查表 `correct_follower_to[]`、`clockwise_tile_of[]`、`tile_x[]/tile_y[]` 计算。

> 该启发式对螺旋目标比纯曼哈顿距离更有信息量（更强的可采纳/一致性倾向），是选择 `SPIRAL` 模式的原因。

### 3.3 通用 A* 引擎：`AStarSearch<UserState>`

定义于 [`stlastar.h`](file:///workspace/24-puzzle/src/stlastar.h#L59-L785)：

| 组成部分 | 说明 |
| --- | --- |
| `Node` | 搜索节点：`parent/child` 指针、`g/h/f` 值、内嵌 `UserState m_UserState` |
| `HeapCompare_f` | 堆比较器（按 `f` 逆序，实现小顶堆） |
| `SetStartAndGoalStates(Start, Goal)` | 初始化起止节点、计算 `f`、压入 Open 堆 |
| `SearchStep()` | **单步搜索**：弹出最小 f 节点；若为目标则回溯设置 `child` 指针并成功返回；否则展开后继、按 g 值去重/更新 Open 与 Closed 列表 |
| `AddSuccessor(State)` | 由用户状态回调调用，分配节点并加入后继列表 |
| `GetSolutionStart/Next/End/Prev()` | 沿 `child`/`parent` 正向或反向遍历解 |
| `GetSolutionCost()` | 返回最终 g 值（失败返回 `FLT_MAX`） |
| `GetOpenListStart/Next`、`GetClosedListStart/Next` | 调试用列表遍历 |
| `FreeSolutionNodes` / `FreeUnusedNodes` / `FreeAllNodes` | 节点内存回收 |
| `AllocateNode` / `FreeNode` | 节点分配，可选启用 `FixedSizeAllocator`（`USE_FSA_MEMORY`，默认关闭） |

搜索状态枚举：`SEARCH_STATE_NOT_INITIALISED / SEARCHING / SUCCEEDED / FAILED / OUT_OF_MEMORY / INVALID`。

抽象接口 [`AStarState<T>`](file:///workspace/24-puzzle/src/stlastar.h#L787-L796) 定义了上述 5 个纯虚回调，`PuzzleState` 即其实现。

### 3.4 固定大小内存分配器 `FixedSizeAllocator<USER_TYPE>`

定义于 [`fsa.h`](file:///workspace/24-puzzle/src/fsa.h#L51-L250)：一次申请大块内存，维护「空闲链表 + 已用链表」双向链表，实现 **O(1) 分配与释放**，避免频繁 `new/delete`。提供 `alloc()`、`free()`、`GetFirst/GetNext` 迭代、`Debug()`。

### 3.5 主流程 `main()`

见 [`24puzzle.cpp#L744-L957`](file:///workspace/24-puzzle/src/24puzzle.cpp#L744-L957)：

1. 可选：从命令行参数解析初始状态字符（数字 0-9）。
2. 创建 `AStarSearch<PuzzleState>`，`SetStartAndGoalStates(nodeStart, nodeEnd)`。
3. 循环 `SearchStep()` 直到状态不再是 `SEARCHING`。
4. 若成功：正/反向打印各节点棋盘；若失败或内存耗尽：打印提示。
5. 收集空格移动轨迹 `Move`，并把走步序列（`r2 c2 r1 c1`）写入全局文件流 `out`（`data.txt`）。

## 4. 演示层设计（Qt5）

### 4.1 数据结构

- [`Movement`](file:///workspace/24-puzzle/puzzle/mainwindow.h#L16-L24)：一次移动 `(r1,c1) → (r2,c2)`。
- `MainWindow::moveQueue`：静态 `queue<Movement>`，存放待回放的走步序列。

### 4.2 `MyLabel` / `BigLabel`

见 [`mylabel.h`](file:///workspace/24-puzzle/puzzle/mylabel.h)：

- `MyLabel(x, r, c, par)`：显示数字方块的 `QLabel`，设置位置 `QRect(c*len, r*len, 100, 100)`、字体、边框；`changeColor(1)` 变黄底表示「该方块已就位」。
- `BigLabel`：棋盘背景层，`lower()` 置于底层。

### 4.3 `MainWindow` 行为

见 [`mainwindow.cpp`](file:///workspace/24-puzzle/puzzle/mainwindow.cpp)：

| 方法 | 说明 |
| --- | --- |
| 构造函数 | 读取数据文件：前 25 个数为 `destArr`（目标），随后 25 个为初始布局；剩余行为走步序列 `moveQueue`；连接 `start` 按钮 |
| `start()` | 创建 `QTimer`，周期 `tt+tt2` 触发 `timerUpdate` |
| `timerUpdate()` | 取出队首移动，调用 `exchange` 并出队 |
| `exchange(r1,c1,r2,c2)` | 用 `QPropertyAnimation` 把方块从 `(r1,c1)` 平移到 `(r2,c2)`，更新 `labelArr` 指针，延时后变色 |
| `changeColor()` | 判断已移动方块是否与目标一致，切换高亮 |
| `getXY(r,c)` | 由行列计算像素坐标 |

### 4.4 求解层 ↔ 演示层的数据契约

求解器产物为文本行：**第一行 25 个数 = 目标状态，第二行 25 个数 = 初始状态，其后每行 4 个数 = 空格移动 `r2 c2 r1 c1`**（见 [`data_normal.txt`](file:///workspace/24-puzzle/data/data_normal.txt) / [`data_spiral.txt`](file:///workspace/24-puzzle/data/data_spiral.txt)）。

⚠️ **注意**：求解器默认写入 `data.txt`，而 GUI 读取写死路径（`/Users/yanshengjia/Desktop/puzzle/data_normal.txt`）——即需要把 `data.txt` 按目标类型重命名/拷贝到 GUI 读取路径。两者的状态顺序也需与 GUI 解析顺序（先目标、后初始）保持一致。

## 5. 依赖关系

```text
24puzzle.cpp ─include─> stlastar.h ─include─> fsa.h        (C++ 标准库)
main.cpp ─> mainwindow.cpp ─> mylabel.h / mainwindow.ui    (Qt5: core gui widgets)
```

## 6. 运行方式

**求解器（命令行）**

```bash
cd 24-puzzle/src
g++ -O2 -o 24puzzle 24puzzle.cpp        # 依赖 STL，无需额外库
./24puzzle                             # 使用 g_start 中当前选定的初始状态
# 产出 data.txt（棋盘 + 走步序列）
```

**GUI（Qt5）**

```bash
cd 24-puzzle/puzzle
qmake puzzle.pro && make
./puzzle                                # 需先将 data.txt 放到程序读取的路径
```

编译期切换目标/启发式：编辑 [`24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp#L27-L28) 中 `SPIRAL`、`NORMAL` 宏并重新编译。
