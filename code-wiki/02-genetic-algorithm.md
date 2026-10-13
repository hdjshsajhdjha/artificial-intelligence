# 02 · 遗传算法求函数最大值

> 目录：[`genetic-algorithm-for-functional-maximum-problem/`](file:///workspace/genetic-algorithm-for-functional-maximum-problem) ｜ 方法：遗传算法 ｜ 语言：MATLAB / Python+TensorFlow

## 1. 解决的问题

求函数

```text
f(x) = x + 10·sin(5x) + 7·cos(4x)，  x ∈ [0, 9]
```

的最大值。参考真值：`f_max = 24.8553627239529`（`x ≈ 7.857`），见 [README](file:///workspace/genetic-algorithm-for-functional-maximum-problem/README.md)。

仓库提供 **两种实现**：

- **MATLAB 版**（[`src/matlab-ga/`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga)）：完整实现遗传算法各算子，直接对 `f(x)` 做最大化。
- **TensorFlow 版**（[`src/tensorflow-ga/train_ga.py`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga/train_ga.py)）：用张量运算实现遗传算法流程（源自 TensorFlow Cookbook 示例，优化目标为「让种群逼近一条正弦波形」）。

## 2. MATLAB 实现

### 2.1 文件职责

| 文件 | 函数 | 职责 |
| --- | --- | --- |
| [`genetic_algorithm.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/genetic_algorithm.m) | `genetic_algorithm(...)` | **GA 主驱动**：初始化→循环（适应度/排序/选择/交叉/变异）→绘图→解码最优个体 |
| [`init.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/init.m) | `init` | 随机初始化种群（二进制串） |
| [`fitness.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/fitness.m) | `fitness` | 计算每个个体的适应度（解码 + 代入 f(x)） |
| [`rank.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/rank.m) | `rank` | 按适应度升序排序，累计适应度，记录历代最优 |
| [`selection.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/selection.m) | `selection` | 轮盘赌选择（排中法二分查找），支持精英保留 |
| [`crossover.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/crossover.m) | `crossover` | 单点交叉 |
| [`mutation.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/mutation.m) | `mutation` | 单点变异 |
| [`plot_ga.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/plot_ga.m) | `plotGA` | 绘制平均适应度随迭代变化曲线 |
| [`target.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/target.m) | `target` | 供 MATLAB 全局 GA 工具箱使用：返回取反的目标函数（工具箱求最小值） |
| [`test.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/test.m) | `test` | 参数设置与调用入口（测试脚本） |

### 2.2 全局变量（MATLAB GA 状态）

主驱动通过 `global` 共享以下状态：

| 变量 | 含义 |
| --- | --- |
| `population` | 当前种群（`population_size × chromosome_size` 二进制矩阵） |
| `population_new` | 选择产生的新种群 |
| `fitness_value` | 当前代各个体适应度 |
| `fitness_sum` | 适应度前缀和（用于轮盘赌） |
| `fitness_average` | 历代平均适应度 |
| `best_fitness` / `best_individual` / `best_generation` | 历代最优适应度 / 最优个体 / 出现代数 |
| `G` | 当前迭代次数 |

### 2.3 算法流程

```text
genetic_algorithm(population_size, chromosome_size, generation_size, cross_rate, mutate_rate, elitism)
   │
   ├─ init()                 随机初始化种群
   └─ for G = 1 : generation_size
         fitness()            计算适应度
         rank()               排序 + 累计适应度 + 记录最优
         selection()          轮盘赌 + 精英保留
         crossover()          单点交叉
         mutation()           单点变异
   │
   ├─ plotGA()               绘制平均适应度曲线
   └─ 输出 m,n,p,q           最优个体/适应度/代数/自变量值
```

### 2.4 关键点

- **编码/解码**（[`fitness.m#L18-L26`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/fitness.m#L18-L26) 与 [`genetic_algorithm.m#L51-L57`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/genetic_algorithm.m#L51-L57)）：`population[i][j]` 二进制串与自变量二进制位的顺序 **相反**（`j` 从低位到高位，权重 `2^(j-1)`）；再按

  ```text
  x = lower_bound + q · (upper_bound - lower_bound) / (2^chromosome_size - 1)
  ```

  解码到 `[0,9]`。
- **适应度**：直接取 `f(x)` 值（最大化），因此 `rank` 按适应度升序，`population(population_size)` 为当代最优。
- **选择**（[`selection.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/selection.m)）：生成 `[0, 总适应度]` 随机数后，用「排中法」在累计适应度数组上二分定位中选个体。
- **精英保留**：`elitism=true` 时，新种群只覆盖前 `population_size-1` 个个体，保留最优个体。

### 2.5 参数（来自 [`test.m`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/matlab-ga/test.m)）

| 参数 | 值 | 含义 |
| --- | --- | --- |
| `elitism` | true | 启用精英选择 |
| `population_size` | 100 | 种群规模 |
| `chromosome_size` | 17 | 染色体长度（二分位，精度约小数点后 4 位） |
| `generation_size` | 200 | 最大迭代代数 |
| `cross_rate` | 0.6 | 交叉概率 |
| `mutate_rate` | 0.01 | 变异概率 |

> 说明：`plot_ga.m` 中函数名为 `plotGA`，而文件名与函数名不一致（MATLAB 要求主函数名与文件名一致），属于仓库中的既有问题；单独调用绘图脚本时需注意。

## 3. TensorFlow 实现

文件：[`train_ga.py`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga/train_ga.py)（单文件、无依赖拆分）。

### 3.1 参数

| 参数 | 值 |
| --- | --- |
| `pop_size` | 100（种群规模） |
| `features` | 50（每个个体的基因维度） |
| `selection` | 0.2（父代比例） |
| `mutation` | 1/200（变异概率） |
| `generations` | 200 |
| `num_parents / num_children` | 20 / 80 |

### 3.2 计算图（关键张量）

| 张量 | 作用 |
| --- | --- |
| `population` | `tf.Variable`，形状 `[pop_size, features]` |
| `truth` | 目标波形 `sin(2π·arange(features)/features)`，作为「真值」 |
| `fitness` | `-mean(square(population - truth))`，即与真值的负 MSE（越大越优） |
| `top_vals, top_ind` | 按适应度取前 `pop_size`（`tf.nn.top_k`） |
| `best_individual` | 最优个体（`tf.gather`） |
| `parents` | 从排序后的种群切出前 `num_parents` 个作为父代 |
| `crossover_mat_ph` / `mutation_val_ph` | 交叉/变异的占位符（由 Python 侧随机生成后喂入） |
| `children_after_sel` | 交叉结果 = 父1×交叉掩码 + 父2×(1−掩码) |
| `mutated_children` | 交叉结果 + 变异值 |
| `new_population` | `concat([parents, mutated_children])`，`step` 将其 `assign` 回 `population` |

### 3.3 训练循环

每代由 Python 侧生成交叉掩码矩阵（随机交叉点，前段置 0）与变异矩阵（按 `mutation` 概率保留高斯扰动），通过 `feed_dict` 喂给计算图；`step.run()` 执行一次「选择→交叉→变异→替换」的种群演化；每 5 代打印最优适应度（最低 MSE）。最后用 matplotlib 对比「真值波形」与「最优个体」（[`train_ga.py#L101-L109`](file:///workspace/genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga/train_ga.py#L101-L109)）。

> ℹ️ **重要差异**：TF 版本源自 TensorFlow Cookbook 的 GA 示例，其「优化目标」是让种群逼近一条正弦曲线（MSE 最小化），**并非直接求解 `f(x)=x+10sin(5x)+7cos(4x)` 的最大值**；它演示的是「如何用 TensorFlow 张量运算实现遗传算法这一过程式算法」。README 也明确指出 TensorFlow 并非实现 GA 的最佳选择，此例用于展示可行性。

## 4. 依赖关系

```text
matlab-ga:  test.m → genetic_algorithm.m → {init, fitness, rank, selection, crossover, mutation, plot_ga}.m
            target.m 独立（供 MATLAB 全局 GA 工具箱）

tensorflow-ga:  train_ga.py（自包含单文件）
```

外部依赖：MATLAB（`plot/rand/disp` 等）；Python 3.6 + TensorFlow 1.4.1 + numpy 1.13.3 + matplotlib 2.1.1。

## 5. 运行方式

**MATLAB 版**

```matlab
% 在 src/matlab-ga 目录下，运行 test.m 即可
test
% 或直接指定参数调用：
[best_individual,best_fitness,iterations,x] = ...
    genetic_algorithm(100, 17, 200, 0.6, 0.01, true);
```

**TensorFlow 版**

```bash
cd genetic-algorithm-for-functional-maximum-problem/src/tensorflow-ga
python3 train_ga.py        # 需先安装 tensorflow 1.4.1 / numpy / matplotlib
```

运行后会打印每 5 代的最优适应度，并弹出真值波形与最优个体的对比图。
