# 03 · 百万皇后问题（QS4 算法）

> 目录：[`million-queens/`](file:///workspace/million-queens) ｜ 方法：QS4 最小冲突局部搜索 ｜ 语言：C++

## 1. 解决的问题

**N 皇后问题**：在 N×N 棋盘上放置 N 个皇后，使任意两个皇后不在同一行、列、对角线上。本项目面向 **超大规模 N（默认 300 万）**，采用 Sosic & Gu 提出的 **QS4 算法**（[参考文献](file:///workspace/million-queens/README.md)）。

核心思想：把「行」固定为皇后的天然行号，用数组 `queen[i]` 表示第 i 行皇后所在列，从而只需处理 **列冲突与两条斜线冲突**；通过不断交换两行皇后来降低冲突总数。

## 2. 文件与配置

单文件实现：[`src/nqueens.cpp`](file:///workspace/million-queens/src/nqueens.cpp)。

| 宏 | 默认 | 含义 |
| --- | --- | --- |
| `TIMES` | 10 | 重复运行次数（用于统计平均耗时） |
| `LOCATION` | 0 | 是否把解的棋盘（Q/.）写入 `million_queen_location.txt` |
| `CODE` | 0 | 是否把解编码（每行皇后列号）写入 `million_queen_code.txt` |
| `ll` | `long long int` | 长整型别名 |
| `C[] = {30,50,80,100}` | — | 不同 N 规模下建议保留的冲突数上限 |

全局：`n`（皇后数量，默认 3,000,000，运行时由 `cin` 覆盖）、`m`（无冲突皇后数）、`step`（步数）、计时变量、随机种子 `RandSeed`。

## 3. 关键函数

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `conflict(n)` | [L32-L40](file:///workspace/million-queens/src/nqueens.cpp#L32-L40) | 根据 N 规模返回「允许残留冲突数」阈值的经验值（N≤8 取 8；N<100 取 N；否则取 `C[]` 中对应档位） |
| `main()` | [L43-L86](file:///workspace/million-queens/src/nqueens.cpp#L43-L86) | 读入 N，计算 `m = n - conflict(n)`，循环 `TIMES` 次调用 `QS4()` 并统计平均耗时 |
| `BRandom(max)` | [L90-L103](file:///workspace/million-queens/src/nqueens.cpp#L90-L103) | 线性同余式伪随机数发生器，返回 `[0, max)` 内整数，用于初始化时随机选列 |
| `swap(a,b)` | [L105-L114](file:///workspace/million-queens/src/nqueens.cpp#L105-L114) | 交换两个长整型 |
| `init_4(queen, n, m, b, c)` | [L117-L167](file:///workspace/million-queens/src/nqueens.cpp#L117-L167) | **初始化**：先把前 m 行皇后贪心地放置到「无对角冲突」的列（借助 `bb/cc` 布尔标记），剩余行随机放置；同时维护对角线冲突计数数组 `b`、`c` |
| `sum(queen, n, b, c)` | [L169-L183](file:///workspace/million-queens/src/nqueens.cpp#L169-L183) | 计算当前棋盘冲突总数（对 `b/c` 中计数 >1 的斜线，累加组合数 `k(k-1)/2`） |
| `delta(i, j, b, c, q, n)` | [L186-L203](file:///workspace/million-queens/src/nqueens.cpp#L186-L203) | **增量冲突变化**：O(1) 计算交换第 i、j 行皇后所引起的冲突数变化；若二者已同斜线则额外加 2 |
| `QS4()` | [L206-L293](file:///workspace/million-queens/src/nqueens.cpp#L206-L293) | **主算法**：见下方流程 |

## 4. QS4 主算法流程

见 [`QS4()` L206-L293](file:///workspace/million-queens/src/nqueens.cpp#L206-L293)：

```text
QS4():
  分配 queen[n]、b[2n]、c[2n]
  init_4()                          初始化（前 m 行无对角冲突）
  t = sum(...)                      当前冲突数
  step = 0
  while (t > 0):
      从第 m 行起扫描每个「有冲突」的皇后 i
        对每个其它皇后 j，用 delta(i,j) 试探交换代价
        只要找到使冲突下降的交换（temp < 0）就执行：
            step++，更新 b/c 计数，swap(queen[i], queen[j])，t += temp
        （在 m 行之前的部分已无对角冲突，参与交换会破坏初始化优势，故从 m 开始）
      若整轮都找不到下降交换（陷入局部最优）：
            重新 init_4()（重启动 / restart），重算 t
  #if LOCATION / CODE：输出解
  释放内存
```

### 数据结构要点

- `queen[i]`：第 i 行皇后所在列。
- `b[i - queen[i] + n - 1]`：第 i 行皇后所在 **反对角线**（`i - col`）上的皇后计数。
- `c[i + queen[i]]`：第 i 行皇后所在 **主对角线**（`i + col`）上的皇后计数。
- 冲突总数 = 各斜线上「同线皇后对数」之和；交换两皇后时用 `delta` 只更新受影响的 4 条斜线计数，实现 **O(1) 增量更新**，这是该算法能扩展到百万级的关键。

## 5. 复杂度与工程意义

- 通过「行固定 + 增量子交换 + 冲突计数数组」，把每次移动的代价降到常数级。
- `init_4` 的贪心预置（前 m 行无对角线冲突）+ 局部最优时的 **重启动** 策略，保证了在超大 N 下仍能快速收敛。
- 默认 `n = 3000000`，即 README 引用的「百万皇后」规模。

## 6. 依赖关系

```text
nqueens.cpp ← 仅依赖 C++ 标准库（<stdio.h> <stdlib.h> <time.h> <iostream>），无第三方依赖
```

## 7. 运行方式

```bash
cd million-queens/src
g++ -O2 -o nqueens nqueens.cpp
./nqueens
# 程序提示 "Enter the number of queens:"，输入 N（如 3000000）
```

运行结果会打印每轮耗时、步数，并在结尾输出 `TIMES` 次的平均耗时。

**可选输出**：把 [`nqueens.cpp`](file:///workspace/million-queens/src/nqueens.cpp#L12-L25) 顶部 `LOCATION` 或 `CODE` 宏置为 1 后重新编译，即可生成棋盘解（`million_queen_location.txt`）或解编码（`million_queen_code.txt`）。

> ⚠️ 注意：`LOCATION` 模式会为每行打印 N 个字符（Q/.），在 N=300 万时会产生数 TB 级别的输出，仅适合小规模 N 使用。
