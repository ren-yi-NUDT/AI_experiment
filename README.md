# CS188 Project 5: Machine Learning — 作业指引

本目录用于完成 UC Berkeley **CS188 Project 5: Machine Learning**（机器学习）。
官方项目页:<https://inst.eecs.berkeley.edu/~cs188/fa25/projects/proj5/>

一句话概括：给模型装上"会学习"的脑子——**感知机 → 全连接神经网络 → 卷积神经网络**，
依次拿下二分类、`sin(x)` 非线性回归、MNIST 手写数字识别（≥97%）和简化版 MNIST 卷积分类（≥80%）四关。
这是 CS188 系列里唯一不碰 `pacman.py` 的项目——这次的主角是 **PyTorch**。

---

## 1. 环境说明

| 项目 | 状态 |
|---|---|
| 环境管理 | conda（Miniconda 26.9.4，装在 `~/miniconda3`），专用环境名 **`cs188`** |
| Python | 3.12.16（conda-forge 频道装包，无 Anaconda ToS 限制） |
| PyTorch | 2.15.0+**cpu**（官方推荐 CPU 版） |
| numpy / matplotlib | 2.5.4 / 3.11.2 |
| 图形窗口 | 服务器无 X 服务。**autograder 一律加 `--no-graphics`**；`--check-dependencies` 本应弹出的画图窗口在服务器上会静默通过（exit 0），**不是出错** |
| 代码位置 | `machinelearning/` 子目录，**所有命令在该目录下、且激活 `cs188` 环境后执行** |

```bash
conda activate cs188
cd /home/ren/Code/AI_experiments/experiment3/machinelearning
python autograder.py --check-dependencies   # 依赖自检
```

> 官方文档提过"numpy 报错就降到 <2.0"。当前 torch 2.15 + numpy 2.5 组合已实测通过依赖检查和 q1 全流程，真报错时再降级即可。

**数据集说明**：`data/mnist.npz`（18 MB）因体积未入 git（`.gitignore` 已忽略），`lang_id.npz`（137 KB）已入库。
克隆到新机器后若报找不到 MNIST 数据，重新从官方 zip 解出即可：

```bash
curl -L -o /tmp/ml.zip https://inst.eecs.berkeley.edu/~cs188/fa25/assets/projects/machinelearning.zip
unzip -j -o /tmp/ml.zip 'machinelearning/data/mnist.npz' -d machinelearning/data/
```

## 2. 目录结构（只需关心加粗的三个文件）

```
machinelearning/
├── models.py     ← ★ 要写：PerceptronModel / RegressionModel / DigitClassificationModel
│                    / LanguageIDModel / Convolve / DigitConvolutionalModel / Attention
├── train.py      ← ★ 要写：train_perceptron / train_regression / train_digitclassifier / …
├── losses.py     ← ★ 要写：regression_loss / digitclassifier_loss / …
├── autograder.py ← 自动评测器（只跑，不改）
├── backend.py    ← 各任务后端逻辑（数据加载、算验证集准确率等，不改）
├── data/         ← mnist.npz（手写数字）、lang_id.npz（语言识别）
├── gpt_model.py + chargpt.py  ← 官方 Q7 字符GPT（0分趣味题，平台不要求）
└── VERSION       ← v2.0
```

> **只改 `models.py` / `train.py` / `losses.py` 三个文件。**官方要求：不要改其它文件，
> **不要改任何函数/类名与签名**，否则 autograder 找不到入口直接判错。

## 3. 评分构成与完成位置一览

**本课程平台（头歌）要求**：

| 平台题号 | 内容 | 平台分值 | 官方题号 | 修改位置 |
|---|---|---|---|---|
| Q1 | Neural Network Tips（感知机） | 25 | q1（5分） | `models.py` → `PerceptronModel`；`train.py` → `train_perceptron` |
| Q2 | Non-linear Regression | 25 | q2（5分） | `models.py` → `RegressionModel`；`train.py` → `train_regression`；`losses.py` → `regression_loss` |
| Q3 | Digit Classification | 40 | q3（5分） | `models.py` → `DigitClassificationModel`；`train.py` → `train_digitclassifier`；`losses.py` → `digitclassifier_loss` |
| Q5 | CNN | 10 | q5（3分） | `models.py` → `Convolve` + `DigitConvolutionalModel`；`train.py` → `Train_DigitConvolution`；`losses.py` → `digitconvolution_Loss` |

官方 fa25 原始项目还有 q4 语言识别（5分）、q6 Attention（2分）、q7 字符GPT（0分趣味题），**平台未要求，不做**。

> ⚠️ **三处命名/脚手架陷阱（autograder 按名字反射查找，错了=0分）：**
> 1. 卷积的训练函数是大写 **`Train_DigitConvolution`**（不是 `train_digitconvolution`）；
> 2. 卷积的损失函数是大写 L 的 **`digitconvolution_Loss`**（不是 `…_loss`）；
> 3. `train_perceptron` 里**已写好** `with no_grad():` 和 `DataLoader(dataset, batch_size=1, shuffle=True)` 脚手架，别删别改，在其下填代码。

## 4. 预备知识：所有题共用的"训练五步"

官方用线性回归示例给出的固定套路。除 Q1 感知机（手工更新、不用梯度）外，Q2/Q3/Q5 的训练循环都是这五步：

```python
optimizer = optim.Adam(model.parameters(), lr=lr)   # 循环外建一次
for epoch in range(epochs):
    for batch in DataLoader(dataset, batch_size=B):
        x, label = batch['x'], batch['label']       # 每个 batch 固定长这样
        optimizer.zero_grad()                        # ① 清梯度
        predicted_y = model(x)                       # ② 前向
        loss = loss_fn(predicted_y, label)           # ③ 算损失（regression_loss / digitclassifier_loss）
        loss.backward()                              # ④ 反向传播
        optimizer.step()                             # ⑤ 更新参数
```

**import 红线**：`train.py` / `losses.py` 文件头注释原话 "Your code will not pass if the gradescope
autograder detects any changed imports"——**只能用已 import 的东西，不要新增 import**：

| 文件 | 已有可用 import |
|---|---|
| `train.py` | `optim`、`tensor`、`no_grad`、`DataLoader`、`movedim` |
| `losses.py` | `mse_loss`、`cross_entropy` |
| `models.py` | `Parameter`、`Linear`、`Module`、`tensor`、`tensordot`、`ones`、`matmul`、`zeros`、`relu`、`softmax`、`movedim` |

## 5. 逐题指引

### Q1 — 感知机（官方 q1，5分；平台 25 分）

线性二分类器，数据保证线性可分，标签只有 `1` / `−1`。四处填写：

- `PerceptronModel.__init__(dimensions)`：`self.w = Parameter(ones((1, dimensions)))`。
  **必须用 `Parameter` 包装**；且 autograder.py:303 明确要求权重 shape 为 **`(1, dimensions)`**（二维！
  写成一维 `(dimensions,)` 形状检查直接挂）。初始化用 `ones` 而非随机值——权重更新一致性检查要求精确相等。
- `forward(x)`：权重与输入的点积。x 带 batch 维（shape `(1, dimensions)`），
  `tensordot(self.get_weights(), x, dims=([1], [1]))` 沿第 1 维收缩，返回单个分数（shape `(1,1)`）。
  ⚠️ `dims=1` 的简写在这里**不行**（会把 w 的第 0 维对上 x 的第 0 维，维度不匹配报错）。
- `get_prediction(x)`：分数 **≥ 0 判 `1`，否则 `−1`**（starter 已把 `score = self(x)` 写好）。
- `train_perceptron(model, dataset)`：经典感知机训练——遍历数据，遇到错分样本就
  `model.w += label * x`（方向×幅度，幅度取 1；**不能写 `model.get_weights() += …`，
  对函数调用做增强赋值是 Python 语法错误**）；**跑完一整轮零错分即收敛、停止训练**。

**验收**：

```bash
python autograder.py -q q1 --no-graphics
```

> 正确实现 30 秒内跑完；卡很久 = 有 bug（autograder 只给你有限次更新机会，不收敛 = 超时判错）。

### Q2 — 非线性回归（官方 q2，5分；平台 25 分）

用神经网络拟合 `[-2π, 2π]` 上的 `sin(x)`，**全数据集平均 MSE ≤ 0.02** 即满分。

- `RegressionModel.__init__`：定义 Linear 层，如 `self.layer1 = Linear(1, h)`。
  ⚠️ forward 的输入形状是 `(batch_size x 1)`，所以**第一层输入维度是 1**，不是 batch_size。
- `forward(x)`：返回 `(batch_size x 1)` 的预测值。一层隐藏层通常够用（官方建议 h ∈ [100, 500]）；
  **输出层不加激活**——sin 在负半轴有值，输出必须能取负。
- `regression_loss(y_pred, y)`：`mse_loss` 一行即可。
- `train_regression`：按 §4 训练五步写，自己建 `DataLoader`。
  官方要求**数据集大小能被 batch size 整除**；建议 batch = 200，epoch 循环到训练损失 ≤ 0.02 就 break（留点余量更好）。

**验收**：`python autograder.py -q q2 --no-graphics`（训练需几分钟，属正常）

### Q3 — 手写数字分类（官方 q3，5分；平台 40 分，分值大头）

MNIST：每张图 28×28 展平成 784 维向量（值在 0–1），输出 10 类得分。**测试集 ≥ 97% 才给分**。

- `DigitClassificationModel.__init__`：starter 已给 `input_size = 28*28`、`output_size = 10`。
- `forward(x)`：返回 `(batch_size x 10)` 得分（logits）。**最后一层不要加 ReLU**——损失用 cross_entropy，要原始分数。
- `digitclassifier_loss`：`return cross_entropy(y_pred, y)` 一行。⚠️ **最大的坑**：标签 y 是
  **one-hot** `(batch x 10)`，**直接当 target 传给 `cross_entropy` 即可**（PyTorch 支持概率目标）。
  ⚠️ **不要**按直觉先 `y.argmax(dim=1)`——argmax 不可导，会把标签从计算图切断，
  autograder 的"loss 必须依赖 y"梯度检查（autograder.py:484）直接挂。
- `train_digitclassifier`：训练五步（starter 已写好 `model.train()`），用
  `dataset.get_validation_accuracy()` 决定何时停。**阈值设 0.98**——验证 0.975 时实测测试集只有
  97.22%（验证/测试是同一测试集劈成的两半，天然有 ~0.3% 波动），踩线易挂；
  阈值 0.98 实测 3 次训练测试集全部 ≥ 97.54%。

**验收**：`python autograder.py -q q3 --no-graphics`（训练几分钟，参考实现约 5 个 epoch）

### Q5 — 卷积神经网络（官方 q5，3分；平台 10 分）

两件事：手写二维卷积 + 在**简化版 MNIST** 上训练 CNN（**≥ 80%**）。

- `Convolve(input, weight)`：实现标准 2D 卷积，**不 padding**——输出尺寸 = 输入边长 − 核边长 + 1（每个维度都如此）。
  输入保证比核大，且**一次只传一张**（batch 恒为 1，不用处理 batch 维）。
  双重 for 循环逐窗口做逐元素乘再求和即可；autograder 会**先单测这个函数的正确性**，再测模型。
- `DigitConvolutionalModel`：官方已把"卷积层 + 展平"写好（见 starter 里 forward 的注释），
  你只需把展平后的向量接上少量小线性层——**大段复用 Q3 的代码**。
- `train.py` → **`Train_DigitConvolution`**（大写 T）、`losses.py` → **`digitconvolution_Loss`**（大写 L）：
  损失与训练循环同 Q3。
- 你的手写卷积比 PyTorch 内置慢是预期内的（人家有 C++/CUDA 优化），官方特意给了简化数据集，不影响得分。

**验收**：`python autograder.py -q q5 --no-graphics`

## 6. 运行与评测命令汇总

```bash
conda activate cs188
cd /home/ren/Code/AI_experiments/experiment3/machinelearning

python autograder.py --check-dependencies   # 依赖自检
python autograder.py -q q1 --no-graphics    # 单题；q1…q6
python autograder.py --no-graphics          # 全部题目（官方满分 5+5+5+5+3+2=25）
```

参考耗时（staff 参考解）：q1 < 30 秒；q2/q3 各几分钟；q5 分钟级；**全套约 12 分钟**。
你的实现显著慢于这个量级就该检查效率（最常见：print 刷屏、batch=1 跑 Q3、Convolve 写得太笨）。

## 7. 调参速查（官方建议）

| 超参 | 建议范围 | 说明 |
|---|---|---|
| 隐藏层大小 | 100–500 | 太宽 → 训练慢、易过拟合 |
| batch size | 1–128 | Q2/Q3 要求**数据集能被 batch 整除**；batch 越小，lr 越要小 |
| 学习率 | 0.0001–0.01 | **最重要的超参**：NaN/Inf = 太大；loss 纹丝不动 = 太小 |
| 隐藏层数 | 1–3 | **从浅开始**：先用一层找好 lr 和层宽，再考虑加深 |

经验法则：
- 系统记录每次尝试的「架构 + 超参 + 结果」。参数随机初始化、数据乱序，**偶尔挂一次重跑就好**；**连挂两次**才值得换架构。
- 损失函数选择：Q2 回归用 `mse_loss`；Q3/Q5 分类用 `cross_entropy`。

## 8. 提交打包（按平台要求）

1. 平台要交的代码：`models.py`、`train.py`、`losses.py` 三个文件（不能改名）。
2. 建「**学号_姓名**」文件夹（**下划线**连接，如 `20220602_张三`），把三个文件放进去，压缩 zip 上传头歌。
3. 实验报告另存为「**学号_姓名_实验3**」单独提交。

```bash
# 打包示例（替换成你的学号姓名）
cd /home/ren/Code/AI_experiments/experiment3
mkdir 20220000_你的名字 && cp machinelearning/{models,train,losses}.py 20220000_你的名字/
zip -r 20220000_你的名字.zip 20220000_你的名字
```

> 交前自检：`python autograder.py --no-graphics` 确认 q1/q2/q3/q5 四题得分；
> 并确认三个文件里**没有 print 调试残留**——Q3 训练是几万个 batch 的循环，一个 print 就能把评测拖到超时。

## 9. 常见坑

- **`ModuleNotFoundError: No module named 'torch'`** → 忘了 `conda activate cs188`。
  `which python` 应指向 `~/miniconda3/envs/cs188/bin/python`。
- **`no display name` / 弹窗失败** → 服务器无 X 服务，autograder 一律加 `--no-graphics`。
- **gradescope 判过不了但本地看着对** → 检查是否**新增了 import**（官方检测 changed imports 直接判负，见 §4 红线）。
- **cross_entropy 报维度错 / 得分怪异** → 标签是 one-hot，**直接整张传给 `cross_entropy`**（概率目标形式）；
  **别做 `argmax` 转换**——argmax 不可导会切断对 y 的梯度，过不了 autograder 的依赖检查（Q3/Q5 同理）。
- **Tensor 布尔陷阱** → `if pred == label:` 单元素 Tensor 会抛 "Boolean value of Tensor is ambiguous"，
  先 `.item()` 转成 Python 数再比较（Q1 感知机最容易踩）。
- **Q2 拟合不出 sin 的负半轴** → 输出层（最后一层 Linear 之后）加了 ReLU。去掉，只在隐藏层间用。
- **感知机不收敛 / 权重不更新** → 权重没包 `Parameter`；或 `no_grad()` 外面做了 `w.data` 之外的原地操作；
  或收敛判定写成了"单轮无错就停"以外的条件（必须整轮零错分）。
- **`mnist.npz` 找不到** → 数据集未入 git，按 §1 的命令重新下载。
- **Convolve 输出尺寸不对** → 无 padding 时输出 = 输入 − 核 + 1，逐维检查；单测先过再训练模型。
- **改了不该改的文件 / 函数名** → 评测只认 `models.py` / `train.py` / `losses.py` 里的原函数名，
  大小写陷阱见 §3（`Train_DigitConvolution`、`digitconvolution_Loss`）。
