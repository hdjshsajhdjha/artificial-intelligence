# 05 · 构建与运行指南

> 本文汇总四个子项目的依赖、编译方式、运行命令与数据准备。各项目相互独立，可单独构建。

## 1. 依赖环境总览

| 项目 | 语言/运行时 | 构建工具 | 第三方库 | 操作系统建议 |
| --- | --- | --- | --- | --- |
| 24-puzzle 求解器 | C++（STL） | g++ / make | 无 | 跨平台 |
| 24-puzzle GUI | C++ / Qt5 | qmake + make | Qt5（core, gui, widgets） | Windows/macOS/Linux |
| 遗传算法（MATLAB） | MATLAB | MATLAB 脚本 | MATLAB 内置 | 任意含 MATLAB 的环境 |
| 遗传算法（TensorFlow） | Python 3.6 | python3 | tensorflow 1.4.1、numpy 1.13.3、matplotlib 2.1.1 | 跨平台 |
| 百万皇后 | C++（STL） | g++ | 无 | 跨平台（需较大内存） |
| 人脸识别 | C（K&R） | gcc + make | libm | 类 Unix（POSIX 头文件） |

> `.gitignore` 已忽略 `*.o`、`*.exe`、`*.pyc`、`.idea`、`.vscode`、`.DS_Store` 等构建/编辑器产物。

## 2. 24-Puzzle

### 2.1 求解器

```bash
cd 24-puzzle/src
g++ -O2 -o 24puzzle 24puzzle.cpp
./24puzzle
```

- 目标/启发式由 [`24puzzle.cpp`](file:///workspace/24-puzzle/src/24puzzle.cpp#L27-L28) 的 `SPIRAL`（0/1）与 `NORMAL`（0/1）宏在编译期选择（两者互斥）。
- 初始状态在 `g_start[]` 的 `#if 0/#elif` 分支中选择。
- 输出：终端打印解路径，并写入 `data.txt`。

### 2.2 GUI

```bash
cd 24-puzzle/puzzle
qmake puzzle.pro
make
./puzzle          # 点击 "start" 开始动画
```

**数据准备**：把求解器生成的走步序列整理为 GUI 读取格式（第一行=目标状态，第二行=初始状态，其后每行 `r2 c2 r1 c1`），放到 GUI 构造函数中读取的路径（源码写死为 `.../puzzle/data_normal.txt`，需按实际情况修改 [`mainwindow.cpp#L15`](file:///workspace/24-puzzle/puzzle/mainwindow.cpp#L15)）。

## 3. 遗传算法

### 3.1 MATLAB 版

```matlab
cd genetic-algorithm-for-functional-maximum-problem/src/matlab-ga
test          % 使用 test.m 中的默认参数运行
```

默认参数：种群 100、染色体 17 位、迭代 200、交叉率 0.6、变异率 0.01、精英保留开启。运行后会绘图并输出最优个体、最优适应度、对应自变量值与达到最优的迭代次数。

### 3.2 TensorFlow 版

```bash
pip install tensorflow==1.4.1 numpy==1.13.3 matplotlib==2.1.1
cd genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga
python3 train_ga.py
```

运行后每 5 代打印一次最优适应度（最低 MSE），结束弹出对比图。

## 4. 百万皇后

```bash
cd million-queens/src
g++ -O2 -o nqueens nqueens.cpp
./nqueens
# 输入皇后数量，例如 3000000
```

- 运行时输入 N；程序重复 `TIMES`（默认 10）次并输出平均耗时与步数。
- 小规模 N 时可将 [`nqueens.cpp`](file:///workspace/million-queens/src/nqueens.cpp#L12-L13) 的 `LOCATION` 或 `CODE` 置 1 以导出解文件。
- 大 N（百万级）需要相应内存：`queen[]` 为 N 个 `long long`，`b/c` 各为 2N 个 `long long`。

## 5. 人脸识别

```bash
cd neural-networks-for-face-recognition/src/<识别器目录>   # 5 选 1
make
# 训练：
./facetrain -n <netfile> -e 100 -s 102194 -S 100 \
            -t <train.list> -1 <test1.list> -2 <test2.list>
# 仅测试（列出分类失败图片）：
./facetrain -n <netfile> -T -1 <test1.list> -2 <test2.list>
```

- 可选目标：`facetrain`、`facetrain_init0`（`-DINITZERO`）、`hidtopgm`、`outtopgm`。
- **数据准备（必做）**：编辑各 `.list` 文件中的图片绝对路径，指向本地 `faces_4/*.pgm` 图像；数据集来源见项目 README 的 CMU 链接。
- 权值可视化：

  ```bash
  ./hidtopgm <netfile> <out.pgm> <x> <y> <hidden-unit>
  ./outtopgm <netfile> <out.pgm> <x> <y> <output-unit>
  ```

## 6. 常见问题与注意事项

| 问题 | 说明 |
| --- | --- |
| 硬编码绝对路径 | 24-Puzzle GUI 与所有人脸识别 `.list` 文件均含作者机器路径（`/Users/yanshengjia/...`），运行前必须替换为本地路径 |
| 数据缺失 | 人脸图像数据集、部分 `.net` 权值文件需自行准备/生成 |
| 编译期行为开关 | 24-Puzzle（`SPIRAL/NORMAL`、`DEBUG_LISTS`）、百万皇后（`LOCATION/CODE`）通过宏在编译期切换，修改后需重新编译 |
| MATLAB 文件名/函数名不一致 | `plot_ga.m` 内函数名为 `plotGA`，单独调用绘图脚本时需留意 |
| TensorFlow 版本 | TF 版基于 TensorFlow 1.x（`tf.Session`、`ops.reset_default_graph`），在 TF 2.x 下需兼容层或改写 |
| C 代码风格 | 人脸识别项目为 K&R 风格（旧式函数定义），`gcc` 编译时可能产生隐式声明告警，属正常现象 |
| 大 N 输出 | 百万皇后开启 `LOCATION` 会为每行输出 N 个字符，百万级会产生巨量输出，仅限小 N 使用 |

## 7. 一键速查

```bash
# 24-puzzle 求解器
g++ -O2 -o 24puzzle 24-puzzle/src/24puzzle.cpp && ./24puzzle

# GPU-less 环境下的 TensorFlow GA
python3 genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga/train_ga.py

# 百万皇后
g++ -O2 -o nqueens million-queens/src/nqueens.cpp && ./nqueens

# 人脸识别（示例）
cd neural-networks-for-face-recognition/src/face-recognizer && make && ./facetrain -n face.net -e 100 -t all_train.list -1 all_test1.list -2 all_test2.list
```
