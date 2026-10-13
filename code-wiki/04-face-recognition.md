# 04 · 人脸识别神经网络

> 目录：[`neural-networks-for-face-recognition/`](file:///workspace/neural-networks-for-face-recognition) ｜ 方法：三层前馈神经网络 + 反向传播 ｜ 语言：C（K&R 风格）

## 1. 背景与任务

源自 CMU 15-681 Machine Learning 课程作业（数据集为 20 人、每人多张 32×30 灰度 `.pgm` 人脸图，文件名携带 `userid, head, expression, eyes, scale` 元信息）。项目用一个人工神经网络识别多种人脸特征。

同一个网络骨架被实例化为 **5 个独立的识别器**（`src/` 下 5 个目录，每个目录各含一套完整可编译的源码）：

| 目录 | 任务 | 隐藏层 | 输出层 | 目标编码依据 | 权值文件 |
| --- | --- | --- | --- | --- | --- |
| [`face-recognizer`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer) | 识别是 20 人中的哪一个 | 20 | 20 | `userid`（20 类 one-of-N） | `face.net` |
| [`specific-person-recognizer`](file:///workspace/neural-networks-for-face-recognition/src/specific-person-recognizer) | 识别是否为特定人（glickman） | 4 | 1 | `userid == "glickman"` | `shades.net` |
| [`sunglass-recognizer`](file:///workspace/neural-networks-for-face-recognition/src/sunglass-recognizer) | 识别是否戴墨镜 | 4 | 1 | `eyes == "sunglasses"` | `shades.net` |
| [`pose-recognizer`](file:///workspace/neural-networks-for-face-recognition/src/pose-recognizer) | 识别人脸朝向 | 6 | 4 | `head ∈ {left,right,straight,up}` | `pose.net` |
| [`expression-regonizer`](file:///workspace/neural-networks-for-face-recognition/src/expression-regonizer) | 识别人脸表情 | 6 | 4 | `expression ∈ {happy,neutral,sad,angry}` | `expression.net` / `newexpression.net` |

> 目录名 `expression-regonizer` 保留了仓库中的原始拼写（recognizer 的笔误），未作修改。
> 网络规模（隐藏层/输出层）由各目录 `facetrain.c` 中 `bpnn_create(imgsize, H, O)` 的实参决定；输入层单元数 = 图像像素数（32×30 = 960）。

## 2. 公共模块结构

每个识别器目录包含相同的文件骨架：

```text
Makefile          构建脚本（多个可执行目标）
backprop.h/.c     神经网络核心（引擎层）
pgmimage.h/.c     PGM 图像与图像列表 I/O
imagenet.c        图像→输入层、图像名→目标向量（★任务相关）
facetrain.c       训练主程序（命令行解析、训练循环、评估）
hidtopgm.c        隐藏层权值可视化 → pgm
outtopgm.c        输出层权值可视化 → pgm
*.list            训练/测试集文件清单
*.net             已训练网络权值（二进制）
```

## 3. 网络引擎：`backprop.c`

定义数据结构 [`BPNN`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/backprop.h#L28-L48)：三层全连接前馈网络（输入/隐藏/输出），**每层 0 号单元为阈值单元**，其余单元下标从 1 开始；每个下一层单元对上一层所有单元（含阈值）加权求和。

| 函数 | 位置 | 说明 |
| --- | --- | --- |
| `bpnn_initialize(seed)` | backprop.c | 用种子初始化随机数发生器（`srandom`） |
| `drnd() / dpn1()` | backprop.c | 生成 `[0,1)` / `[-1,1)` 随机数，用于权值初始化 |
| `squash(x)` | backprop.c | 激活函数：Sigmoid `1/(1+e^{-x})` |
| `alloc_1d_dbl` / `alloc_2d_dbl` | backprop.c | 动态分配 1D/2D double 数组 |
| `bpnn_internal_create` / `bpnn_create` | backprop.c | 创建网络（各层的大小为 `n+1`，含阈值单元）；`bpnn_create` 随机初始化输入/隐藏权值，`INITZERO` 宏下输入权值置零 |
| `bpnn_free` | backprop.c | 释放网络全部内存 |
| `bpnn_layerforward(l1,l2,conn,n1,n2)` | backprop.c | **前向**：计算某层到下一层的加权和并 squash |
| `bpnn_output_error(delta,target,output,nj,err)` | backprop.c | 计算输出层误差 `δ = o(1-o)(t-o)`，返回误差和 |
| `bpnn_hidden_error(...)` | backprop.c | 计算隐藏层误差 `δ = h(1-h)·Σ(δ_o·w)` |
| `bpnn_adjust_weights(delta,ly,w,oldw,lr,momentum)` | backprop.c | **权值更新**：`new_dw = lr·δ·ly + momentum·oldw`，并保存增量 |
| `bpnn_feedforward(net)` | backprop.c | 纯前向传播（用于评估/预测） |
| `bpnn_train(net,lr,mom,erro,errh)` | backprop.c | 一次「前向 + 输出误差 + 隐藏误差 + 权值更新」完整反向传播 |
| `bpnn_save(net,filename)` / `bpnn_read(filename)` | backprop.c | 网络权值的二进制读写（先存 `n1,n2,n3`，再存两组权值矩阵） |

## 4. 图像模块：`pgmimage.c`

数据结构 [`IMAGE` / `IMAGELIST`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/pgmimage.h#L14-L23) 与函数：

| 函数 | 说明 |
| --- | --- |
| `img_open(filename)` | 读入 PGM（P5/P2 格式，8-bit）；解析尺寸与像素 |
| `img_creat(name,nr,nc)` | 新建空白图像 |
| `img_setpixel` / `img_getpixel` | 像素读写（行主序，一维数组存储） |
| `img_write(img,filename)` | 以 P2 文本格式写回 `.pgm` |
| `img_free` | 释放图像 |
| `imgl_alloc` / `imgl_add` / `imgl_free` | 图像列表管理 |
| `imgl_load_images_from_textfile(il,filename)` | 从 `.list` 文本文件逐行读取路径并加载图像 |

## 5. 任务相关模块：`imagenet.c`

| 函数 | 说明 |
| --- | --- |
| `load_input_with_image(img,net)` | 把图像像素归一化（`pixel/255.0`）依次填入输入层单元；像素数与 `input_n` 不符则报错退出 |
| `load_target(img,net)` | **★各识别器差异所在**：解析文件名（`sscanf(NAME, "%[^_]_%[^_]_%[^_]_%[^_]_%d.%[^_]")` → `userid, head, expression, eyes, scale, photo`），按任务把目标向量设为 `TARGET_HIGH(0.9)` / `TARGET_LOW(0.1)` |

各变体 `load_target` 的判定逻辑：

- **face-recognizer**：`userid` 命中 20 人之一 → 对应输出单元置高（one-of-20）。
- **specific-person-recognizer**：`userid == "glickman"` → `target[1]=HIGH`，否则 LOW。
- **sunglass-recognizer**：`eyes == "sunglasses"` → `target[1]=HIGH`，否则 LOW。
- **pose-recognizer**：`head` ∈ {`left`,`right`,`straight`,`up`} → `target[1..4]` 对应置高。
- **expression-regonizer**：`expression` ∈ {`happy`,`neutral`,`sad`,`angry`} → `target[1..4]` 对应置高。

> 设计要点：目标值用 0.9/0.1 而非 0/1，因为 Sigmoid 单元在有限权值下无法精确输出 0/1，强行拟合会迫使权值无限增大（见 [`imagenet.c` L17-L20 注释](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/imagenet.c#L17-L20)）。

## 6. 训练主程序：`facetrain.c`

### 6.1 关键函数

| 函数 | 说明 |
| --- | --- |
| `evaluate_performance(net,err)` | 计算输出层均方误差和；并按 0.5 阈值判定本样例是否分类正确（1/0） |
| `performance_on_imagelist(net,il,list_errors)` | 遍历图像列表：前向 → 载入目标 → 统计准确率与平均误差；`list_errors=1` 时打印分类失败图片名 |
| `result_on_imagelist(net,il,list_errors)` | 面向「预测结果」的评估：打印每张图片名称、猜测的人物/类别与对错标记（带 emoji） |
| `backprop_face(train,test1,test2,epochs,savedelta,netname,list_errors)` | **训练核心**：读入或新建网络 → 迭代 `epochs` 次「逐图前向+反向（lr=0.3, momentum=0.3）」→ 每 `savedelta` 代保存 → 迭代后输出预测 |
| `printusage(prog)` | 打印命令行用法 |
| `main(argc,argv)` | 解析命令行参数、创建三个图像列表、加载 `.list`、初始化网络包、启动训练 |

### 6.2 命令行参数

| 选项 | 含义 | 默认 |
| --- | --- | --- |
| `-n <network file>` | 网络权值文件（必需，保存/加载） | — |
| `-e <epochs>` | 训练迭代次数 | 100 |
| `-s <seed>` | 随机数种子 | 102194 |
| `-S <save-delta>` | 每多少代保存一次网络 | 100 |
| `-t <train list>` | 训练集清单 | — |
| `-1 <test1 list>` / `-2 <test2 list>` | 测试集清单 | — |
| `-T` | 仅测试（`epochs=0`），并输出分类失败清单 | 关闭 |

### 6.3 网络规模与超参数

- 网络：`bpnn_create(imgsize, H, O)`，其中 `imgsize = 32×30 = 960`；`H/O` 随任务不同（见第 1 节表）。
- 学习率 0.3、冲量 0.3（[`facetrain.c#L582`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/facetrain.c#L582)）。
- 权值随机初始化范围 `[-1,1]`。

## 7. 权值可视化工具

- [`hidtopgm.c`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/hidtopgm.c)：把某个隐藏单元的输入层权值按 min-max 归一化到 0..255 并输出为 `.pgm`，直观看出该单元「关注」的图像模式。
  用法：`hidtopgm net-file image-file x y hidden-unit-num`。
- [`outtopgm.c`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/outtopgm.c)：把某个输出单元到隐藏层的权值可视化为（放大一倍的）`.pgm`。
  用法：`outtopgm net-file image-file x y output-unit-num`。

## 8. 构建（Makefile）

[`Makefile`](file:///workspace/neural-networks-for-face-recognition/src/face-recognizer/Makefile) 定义的目标：

| 目标 | 由哪些 .o 链接 | 说明 |
| --- | --- | --- |
| `facetrain` | facetrain.o + imagenet.o + **backprop_initr.o** + pgmimage.o | 随机初始化网络的训练器 |
| `facetrain_init0` | facetrain.o + imagenet.o + **backprop_init0.o** + pgmimage.o | 输入权值置零（`-DINITZERO`）的变体 |
| `hidtopgm` | hidtopgm.o + pgmimage.o + backprop_initr.o | 隐藏层权值可视化 |
| `outtopgm` | outtopgm.o + pgmimage.o + backprop_initr.o | 输出层权值可视化 |

`backprop.c` 被编译两次：默认 `backprop_initr.o`；加 `-DINITZERO` 得 `backprop_init0.o`。链接时统一加 `-lm`。

## 9. 依赖关系

```text
facetrain.c ─> backprop.h/.c (BPNN 引擎) ─> libm
            ─> imagenet.c     (load_target / load_input_with_image)
            ─> pgmimage.h/.c  (图像 I/O)
hidtopgm.c / outtopgm.c ─> backprop.h/.c + pgmimage.h/.c
```

外部依赖：仅 libc、libm；数据为外部 `.pgm` 图像（不在仓库内，通过 `.list` 路径引用）。

## 10. 运行方式

```bash
cd neural-networks-for-face-recognition/src/face-recognizer
make
# 按需修改 *.list 中的图片绝对路径，使其指向本地人脸图像
./facetrain -n face.net -e 100 -s 102194 -S 100 \
            -t all_train.list -1 all_test1.list -2 all_test2.list

# 仅测试并列出分类失败图片：
./facetrain -n face.net -T -1 all_test1.list -2 all_test2.list
```

⚠️ **数据准备**：仓库未包含人脸图像，且 `.list` 文件中的路径为作者机器的绝对路径（`/Users/yanshengjia/Desktop/...`），运行前必须改成本地路径。数据集见 README 给出的 CMU faces 链接。详细实验过程与结果见 [`doc/ysjai01.pdf`](file:///workspace/neural-networks-for-face-recognition/doc) 与 [`doc/hw97.pdf`](file:///workspace/neural-networks-for-face-recognition/doc)。
