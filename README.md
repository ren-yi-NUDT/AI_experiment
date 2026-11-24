# CS188 Project 4: Bayes Nets — 作业指引

本目录用于完成 UC Berkeley **CS188 Project 4: Bayes Nets**（贝叶斯网络；官方页面标题为 Ghostbusters）。
官方项目页:<https://inst.eecs.berkeley.edu/~cs188/fa25/projects/proj4/>

一句话概括：在"听声辨位抓幽灵"的场景里实现贝叶斯网络推理的四块积木——**搭网络结构 → 因子相乘（join）→ 因子求和消元（eliminate）→ 变量消元推理（variable elimination）**，
依次拿下 Bayes Net Structure、Join Factors、Eliminate、Variable Elimination 四关（各 25 分）。
全程是纯 Python 的概率表运算：不用 PyTorch，也不需要图形界面。

> 本仓库是官方项目的精简版（`bayesnets/VERSION` 为 v4.000）：官方共 11 题，这里只保留平台要求的 Q1–Q4（`test_cases/` 下只有 q1–q4）。
> 官方列出的先修内容中，与这四题相关的是：概率（Video 12, Note 6.1–6.2）、贝叶斯网络表示（Video 13, Note 6.3–6.4）、贝叶斯网络推断（Video 15, Note 6.6）。

---

## 1. 环境说明

| 项目 | 状态 |
|---|---|
| 环境管理 | conda（Miniconda 26.9.4，装在 `~/miniconda3`）。复用已有的 **`cs188`** 环境即可，不用新建 |
| Python | 3.12.16 |
| 第三方依赖 | 无需安装。项目只 import 标准库和本地模块；图形模块会用到的 `tkinter` 也是标准库的一部分（`cs188` 环境已带 tk 8.6.13） |
| autograder 补丁 | `bayesnets/autograder.py:21-36` 已适配 Python 3.12：标准库 `imp` 被移除，改用 `types.ModuleType` 等价替代（提交 f77d4f1） |
| 图形窗口 | 本机 `DISPLAY` 为空（无 X 服务）。Q1–Q4 的测试不画图，autograder 实测可正常运行 |
| 代码位置 | `bayesnets/` 子目录。**所有命令在该目录下、激活 `cs188` 环境后执行** |

```bash
conda activate cs188
cd /home/ren/Code/AI_experiments/experiment4/bayesnets
python autograder.py -q q1
```

> 运行开头会打印若干行 `SyntaxWarning: invalid escape sequence`，来自官方代码里的正则字符串字面量和一段 ASCII 字符画。
> Python 3.12 起无效转义序列会报这种警告，无害，忽略即可。
> 每次运行固定 7 行（`autograder.py` 6 行、`bayesHMMTestClasses.py:689` 1 行）；刚克隆、还没有 `__pycache__` 时的第一次运行会多 5 行
> （`grading.py:108` 1 行、`testParser.py` 4 行）。

---

## 2. 目录结构（只需关心标 ★ 的两个文件）

```
experiment4/
├── README.md                    ← 本文档
├── docs/requirements.txt        ← 平台题目与分值；docs/ 被 .gitignore 忽略，未入库
└── bayesnets/                   ← 官方项目目录（VERSION: v4.000），所有命令在这里执行
    ├── inference.py             ← ★ 要写：constructBayesNet（Q1）、inferenceByVariableElimination（Q4）
    ├── factorOperations.py      ← ★ 要写：joinFactors（Q2）、eliminate（Q3）
    ├── bayesNet.py              ← BayesNet / Factor 类（不改，必看）
    ├── autograder.py            ← 评测入口（只跑，不改）
    ├── grading.py               ← 计分与输出格式
    ├── testParser.py            ← .test / .solution 文件解析
    ├── testClasses.py           ← 题目与用例的基类（PassAllTestsQuestion 在这里）
    ├── bayesHMMTestClasses.py   ← 各题的测试类
    ├── projectParams.py         ← 学生文件清单、测试类文件名
    ├── submission_autograder.py ← 官方提交检查脚本（代码经过混淆，文件头注明禁止修改）；本地评测用不到
    ├── test_cases/              ← q1…q4 共 21 个用例：.test 是输入，.solution 是标准答案
    ├── hunters.py               ← hunters 游戏引擎（Q1 用到它的 GameState）
    ├── busters.py               ← Ghostbusters 游戏引擎
    ├── pacman.py                ← 经典 Pacman 引擎
    ├── bustersAgents.py / bustersGhostAgents.py / ghostAgents.py / pacmanAgents.py / keyboardAgents.py
    │                              ← 游戏里的各种 agent
    ├── game.py / layout.py / util.py / distanceCalculator.py
    │                              ← 通用基础：状态与动作、地图解析、工具函数、迷宫距离
    ├── layouts/                 ← 4 张游戏地图（bigHunt / oneHunt / openHunt / smallHunt）
    └── graphicsDisplay.py / graphicsUtils.py / textDisplay.py
                                   ← 图形 / 文本显示
```

> **只改 `inference.py` 和 `factorOperations.py`。** 其余文件不用改，也不要删：
> `inference.py` 和测试类会 import `game.py`、`hunters.py`、`busters.py`、`layout.py`、`bustersAgents.py` 等。

---

## 3. 评分构成与完成位置一览

**本课程平台要求**（`docs/requirements.txt`）：

| 平台题号 | 内容 | 平台分值 | 修改位置 | 本地分值 | 用例数 |
|---|---|---|---|---|---|
| Q1 | Bayes Net Structure | 25 | `inference.py` → `constructBayesNet` | 2 | 3 |
| Q2 | Join Factors | 25 | `factorOperations.py` → `joinFactors` | 3 | 6 |
| Q3 | Eliminate | 25 | `factorOperations.py` → `eliminate` | 2 | 6 |
| Q4 | Variable Elimination | 25 | `inference.py` → `inferenceByVariableElimination` | 2 | 6 |

本地满分 2 + 3 + 2 + 2 = **9**。官方页面对这四题标注的分值是 2 / 4 / 3 / 2，与本地 `CONFIG`（2 / 3 / 2 / 2）略有出入，以本地 autograder 的输出为准。

官方完整项目还有 Q5–Q11（观测概率、精确 / 近似推断跟踪、粒子滤波、Greedy Busters Agent 等），**平台未要求，不做**。
`inference.py:411-434` 里 `InferenceModule` 的四个 `raise NotImplementedError` 属于那部分，不用填。

> 四道题的 `CONFIG` 都是 `PassAllTestsQuestion`：**该题所有用例全过才得分，没有部分分。**

> ⚠️ **autograder 按名字找入口，三处不能动：**
> 1. 四个函数的名字和签名。autograder 按名字取函数（`bayesHMMTestClasses.py:47`、`:232`、`:271`）。
> 2. Q4 里标着 `# this is for autograding -- don't modify` 的两行（`inference.py:176-178`）。autograder 靠它们记录 join / eliminate 的调用序列。
> 3. 答案只能放在 `inference.py` 和 `factorOperations.py` 里。autograder 只加载这两个学生文件（`projectParams.py` 的 `STUDENT_CODE_DEFAULT`）。

---

## 4. 预备知识：四题共用的 Factor 数据模型

`Factor` 就是一张概率表。记号 `P(D | W)` 中，竖线左边是**非条件变量**（unconditioned），右边是**条件变量**（conditioned）。
表里的每一行对应一个 assignmentDict，例如 `{'D': 'wet', 'W': 'sun'}`。因子各行之和不一定是 1；
只有"恰有一个非条件变量，且每个条件取值下各行之和为 1"的因子才是 CPT。

| 接口 | 作用 | 位置 |
|---|---|---|
| `Factor(unconditioned, conditioned, variableDomainsDict)` | 新建一张概率全为 0 的表 | `bayesNet.py:236` |
| `f.getAllPossibleAssignmentDicts()` | 枚举表中每一行 | `bayesNet.py:422` |
| `f.getProbability(a)` / `f.setProbability(a, p)` | 读 / 写一行 | `bayesNet.py:344` / `:366` |
| `f.unconditionedVariables()` / `f.conditionedVariables()` | 两个变量集合（`set`） | `bayesNet.py:296` / `:300` |
| `f.variableDomainsDict()` | 变量到取值列表的映射 | `bayesNet.py:284` |
| `normalize(f)` | 整表除以总和，并把取值域只剩 1 个值的非条件变量移到条件变量 | `bayesNet.py:857` |
| `constructEmptyBayesNet(variables, edges, domains)` | 搭一个空网络（Q1） | `bayesNet.py:573` |
| `bayesNet.getAllCPTsWithEvidence(evidenceDict)` | 返回按证据裁剪后的全部 CPT（Q4） | `bayesNet.py:212` |

三条性质决定了代码怎么写：

1. `a` 可以含有因子里没有的变量（多余的会被忽略），但不能缺少因子里有的变量。所以一整行 `a` 可以直接传给任意一个小因子。
2. 查询方法返回的都是副本，在返回值上 `remove` / `del` 不会影响原因子。
3. 域字典里允许出现因子用不到的变量。

想直观看这些接口怎么用，运行 `python bayesNet.py`：它会打印一个 Raining / Ballgame / Traffic 示例网络，
演示建表、读写概率、按证据裁剪 CPT 的全过程（`printStarterBayesNet`，`bayesNet.py:720`）。

---

## 5. 逐题指引

建议按 Q1 → Q2 → Q3 → Q4 的顺序做。Q4 调用的是你自己写的 `joinFactors` 和 `eliminate`，前面错了后面必然不过。

### Q1 — Bayes Net Structure（`inference.py` → `constructBayesNet`；本地 2 分）

只搭骨架：变量、边、取值域。概率表（CPT）由官方代码另外填，这里不用管。

结构（观测由位置产生，所以边从位置指向观测）：

```
Pacman → Observation0 ← Ghost0
Pacman → Observation1 ← Ghost1
```

- 变量名必须用函数里给好的常量 `PAC` / `GHOST0` / `GHOST1` / `OBS0` / `OBS1`（`inference.py:50-54`）。
- 边是 `(父, 子)` 元组，和变量、取值域一起交给 `bn.constructEmptyBayesNet(...)`（`inference.py:67`）。

| 变量 | 取值域 |
|---|---|
| `Pacman`、`Ghost0`、`Ghost1` | 棋盘上**所有**格子 `(x, y)`，`0 ≤ x < X_RANGE`、`0 ≤ y < Y_RANGE`，**包括墙** |
| `Observation0`、`Observation1` | 整数 `list(range(X_RANGE + Y_RANGE + MAX_NOISE - 1))` |

依据：官方页面写明 "Pacman and the two ghosts can be anywhere in the grid (we ignore walls for this)"；
观测是非负的、与真实曼哈顿距离相差不超过 `MAX_NOISE`（7）的距离，最远距离为 `(宽-1) + (高-1)`，所以上限是 `宽 + 高 - 2 + 7`。

三个用例的规模（已对照标准答案核对）：

| 用例 | 棋盘 | 位置域个数 | 观测取值 |
|---|---|---|---|
| `1-small-board` | 15 × 11 | 165 | 0–31 |
| `2-long-bottom` | 11 × 15 | 165 | 0–31 |
| `3-wide-inverted` | 19 × 15 | 285 | 0–39 |

**评分方式**（`GraphEqualityTest`，`bayesHMMTestClasses.py:51-100`）：只比较变量集、边集、每个变量的取值集合。
比较前会排序，所以与顺序无关；不看 CPT。不一致时会列出缺少 / 多余的变量、边和取值。

**易错**：
- 位置只取可通行的格子（例如用 `getWalls().asList(False)`），取值集合会对不上。
- 观测值写成字符串 `'0'`、`'1'`，标准答案是 `int`。
- 边的方向反了。

**验收**：`python autograder.py -q q1`

### Q2 — Join Factors（`factorOperations.py` → `joinFactors`；本地 3 分）

因子相乘：多张表逐行相乘成一张。以用例 `1-product-rule` 为例，`P(D | W) × P(W) = P(D, W)`：

```
            A = P(D|W)   B = P(W)   A × B
wet, sun       0.1         0.8      0.08
dry, sun       0.9         0.8      0.72
wet, rain      0.7         0.2      0.14
dry, rain      0.3         0.2      0.06
```

**规则**：
1. 结果的非条件变量 = 各因子非条件变量的并集。
2. 结果的条件变量 = 各因子条件变量的并集，再减去第 1 步的结果。
3. 结果每一行的概率 = 各因子对应行的概率之积。

**做法**：按前两条算出变量集合，取任一输入因子的 `variableDomainsDict()`（题目保证各因子的域字典相同），新建一个 `Factor`。
遍历它的 `getAllPossibleAssignmentDicts()`，把每一行 `a` 直接传给每个输入因子的 `getProbability`，相乘后 `setProbability`。
两个因子没有公共变量时（用例 `3-disjoint-right`），这种写法自然得到外积。

**评分方式**（`FactorEqualityTest`，`bayesHMMTestClasses.py:159-189`）：返回的因子要与标准因子 `==`，
即非条件 / 条件变量集合相同（`bayesNet.py:314-316`），且每行概率的误差不超过 `10e-13`。

**易错**：
- 测试传进来的 `factors` 是 `dict_values`，不是 list（`bayesHMMTestClasses.py:234`）。不能写 `factors[0]`，要先转成 list 或直接用 `for` 遍历。
- 忘了"减去"：`W` 在 `P(D | W)` 里是条件变量、在 `P(W)` 里是非条件变量，在结果 `P(D, W)` 里它是非条件变量。
- 输入因子各行之和不一定是 1。不要归一化，也不要做任何"和为 1"的假设。

**验收**：`python autograder.py -q q2`；单跑一个用例：`python autograder.py -t test_cases/q2/1-product-rule`

### Q3 — Eliminate（`factorOperations.py` → `eliminate`；本地 2 分）

求和消元：把某个非条件变量消掉。接着上面的例子，对 `W` 求和：`Σ_W P(D, W) = P(D)`，
wet 为 `0.08 + 0.14 = 0.22`，dry 为 `0.72 + 0.06 = 0.78`。

**规则**：
1. 结果的非条件变量 = 原来的非条件变量去掉 `eliminationVariable`；条件变量原样保留。
2. 结果每一行 = 原因子中"其余变量取值相同"的所有行之和。

**做法**：新建 `Factor`，域字典直接沿用原因子的（官方要求返回因子保持与输入相同的 `variableDomainsDict`）。
遍历原因子的每一行 `a`，按顺序做三件事：
1. 读出 `factor.getProbability(a)`。
2. `del a[eliminationVariable]`。
3. 把读出的概率累加到新因子的这一行上。

**易错**：
- 三步的顺序不能换。先 `del` 再读会抛 `ValueError`，因为 `getProbability` 要求 `a` 含有该因子的全部变量。
- **不要归一化**。用例 `6-simple-eliminate-int` 的标准结果是 `P(D=wet) = 2.0`、`P(D=dry) = 0.0`，各行之和不是 1。归一化是 Q4 最后一步的事。
- 条件变量要保留：`P(B, T, W | D)` 消去 `B` 得到 `P(T, W | D)`（用例 `3-eliminate-conditioned`）。

**验收**：`python autograder.py -q q3`；单跑一个用例：`python autograder.py -t test_cases/q3/1-simple-eliminate`

### Q4 — Variable Elimination（`inference.py` → `inferenceByVariableElimination`；本地 2 分）

用"join 一个变量、立刻消去它"的方式算 `P(queryVariables | evidenceDict)`。同文件里的 `inferenceByEnumeration`（`inference.py:71-120`）
是参考写法，但它先把所有因子合成一张完整的联合分布表再消元，表太大。变量消元则按 `eliminationOrder` 交替进行 join 和 eliminate。

**骨架**（标了 `# this is for autograding -- don't modify` 的两行不要改）：

```python
currentFactorsList = bayesNet.getAllCPTsWithEvidence(evidenceDict)   # 已按证据裁剪的 CPT
for joinVariable in eliminationOrder:
    currentFactorsList, joinedFactor = joinFactorsByVariable(currentFactorsList, joinVariable)
    if len(joinedFactor.unconditionedVariables()) == 1:
        ...   # 丢弃 joinedFactor，不调用 eliminate
    else:
        ...   # eliminate 后放回 currentFactorsList
return normalize(joinFactors(currentFactorsList))
```

丢弃的理由：对单个变量求和的结果恒为 1（例如 `Σ_T P(T | W=sun) = 1`），乘不乘都不影响答案；
而对只剩一个非条件变量的因子调用 `eliminate` 会直接抛 `ValueError`（`factorOperations.py:148-153` 的类型检查）。

**手算用例 `1-disconnected-eliminate`**（网络 `W → D`、`W → T`；查询 `D`；证据 `W = sun`；消元顺序 `[T]`）：
1. 初始因子：`P(D | W=sun)`、`P(W=sun)`、`P(T | W=sun)`。
2. join `T`：只有 `P(T | W=sun)` 含 `T`，join 后只剩 1 个非条件变量，丢弃。
3. 剩下 `P(D | W=sun)` 和 `P(W=sun)`，`joinFactors` 后 `normalize`，得到 `P(D | W=sun)`：dry 0.9、wet 0.1。

对照用例 `2-independent-eliminate`（查询 `W`，无证据，顺序 `[T, D]`）：join `T` 时 `P(D | T, W)` 与 `P(T)` 合成 `P(D, T | W)`，
有两个非条件变量，这次要 eliminate `T`，得到 `P(D | W)`；join `D` 时只剩一个非条件变量，丢弃；最后留下 `P(W)`。

六个用例的标准调用序列：

| 用例 | 标准调用序列 |
|---|---|
| `1-disconnected-eliminate` | join `T` |
| `2-independent-eliminate` | join `T` → eliminate `T` → join `D` |
| `3-independent-eliminate-extended` | join `S` → join `D` |
| `4-common-effect-eliminate` | join `T` → eliminate `T` → join `D` → eliminate `D` |
| `5-grade-var-elim` | join `C` → join `D` → join `E` |
| `6-large-bayesNet-elim` | join `X3` → eliminate `X3` → … → join `X10` → eliminate `X10` → join `Z` → eliminate `Z` |

**评分方式**（`bayesHMMTestClasses.py:166-176`）：结果因子 `==` 标准因子，**并且**调用序列与标准序列完全相同，两个条件都满足才通过。

**易错**：
- 顺序里的每个变量都要调用一次 `joinFactorsByVariable`，即使随后要丢弃；也不能多调用。
  `joinFactorsByVariable` 和 `eliminate` 每被调用一次就往 `callTrackingList` 里记一条（`factorOperations.py:37-38`、`:137-138`），
  所以最后合并剩余因子要用不带记录的 `joinFactors`。
- "只剩 1 个非条件变量"要数 join 之后的那个因子。证据变量在 `normalize` 之前仍算非条件变量（只是取值域被裁成一个值），
  所以 `P(T | W=sun)` 是 1 个，不是 2 个。
- 最后的 `normalize` 不能省。它把整张表除以总和，并把取值域只剩 1 个值的变量从非条件移到条件变量（`bayesNet.py:857-882`），
  结果的变量划分才能和标准答案一致。
- `joinFactorsByVariable` 返回的是二元组（未参与 join 的因子列表，join 结果），未参与的那部分要留着。

**验收**：`python autograder.py -q q4`；单跑一个用例：`python autograder.py -t test_cases/q4/1-disconnected-eliminate`

---

## 6. 运行与评测命令汇总

```bash
conda activate cs188
cd /home/ren/Code/AI_experiments/experiment4/bayesnets

python autograder.py -q q1                                # 单题；q1…q4
python autograder.py -t test_cases/q3/1-simple-eliminate   # 只跑一个用例（不带 .test 后缀）
python autograder.py                                      # 全部四题
```

- `-t` 的路径不带 `.test` 后缀，autograder 会自动拼接（`autograder.py:228`）。
- `--no-graphics` 可加可不加。`-q` / `-t` 模式下 autograder 默认会构造图形显示对象（`autograder.py:340-351`），
  但 Q1–Q4 的测试类不画图，本机无 X 服务，实测不加也能正常跑。
- 必须在 `bayesnets/` 下运行。autograder 用相对路径找 `factorOperations.py`、`inference.py` 和 `test_cases/`，
  在仓库根目录运行会报 `FileNotFoundError: [Errno 2] No such file or directory: 'factorOperations.py'`。

**当前状态**（四个函数尚未实现）的真实输出：

```
Question q1
===========
*** Method not implemented: constructBayesNet at line 64 of inference.py
*** FAIL: Terminated with a string exception.

### Question q1: 0/2 ###
…（q2、q3、q4 同理）…

Provisional grades
==================
Question q1: 0/2
Question q2: 0/3
Question q3: 0/2
Question q4: 0/2
------------------
Total: 0/9
```

四题全部完成后，计分栏应为下面这样（预期值，尚未实测）：

```
Question q1: 2/2
Question q2: 3/3
Question q3: 2/2
Question q4: 2/2
------------------
Total: 9/9
```

测试没过时，autograder 会把差异打印出来：

- 代码抛异常：`*** FAIL: Exception raised: ...`，后面跟完整 traceback。
- Q1：列出缺少 / 多余的变量、边和取值（`bayesHMMTestClasses.py:59-99`）。
- Q2 / Q3 / Q4：`Factors are not equal.`，后面是"你的因子"和"标准因子"两张表；两边概率总和不同时还会打印总和（`:177-189`）。
- Q4 另有调用序列的比对：`Order of joining by variables and elimination by variables is incorrect`，
  并分别打印你的序列和标准序列（`:170-174`）。

---

## 7. 提交打包

**待确认**：`docs/requirements.txt` 只写了题目和分值，没有写本实验的提交要求。下面是能从代码确定的部分：

- 本实验要交的代码只涉及 `bayesnets/inference.py` 和 `bayesnets/factorOperations.py`。
  autograder 也只把这两个文件当作学生代码加载（`projectParams.py` 的 `STUDENT_CODE_DEFAULT`）。
- 交前自检：`python autograder.py` 四题全部满分（Total: 9/9），并去掉调试用的 `print`，避免干扰 autograder 的输出。

实验 3 的打包约定是：代码放进「学号_姓名」文件夹压缩成 zip 上传，实验报告另存为「学号_姓名_实验3」单独提交。
本实验是否沿用（报告名是否相应改为「实验4」），请以平台本关的说明为准。

---

## 8. 常见坑

- **`Method not implemented: ... at line N of ...`** → 对应函数还是 starter 的占位（`raiseNotDefined()`），尚未实现。
- **`FileNotFoundError: [Errno 2] No such file or directory: 'factorOperations.py'`** → 没在 `bayesnets/` 目录下运行 autograder。
- **`SyntaxWarning: invalid escape sequence`** → 见 §1，忽略。
- **Q1 报 `Student solution has incorrect values dictionary.`** → 取值域不对，看下面列出的 missing / extra assignments。
  最常见的是位置漏了墙、观测范围算错。
- **Q2 报 `'dict_values' object is not subscriptable`** → 对入参用了下标，先 `list(factors)`。
- **`ValueError: The input assignmentDict is not contained in this factor`** → 传给 `getProbability` / `setProbability` 的 dict
  缺少该因子的变量（多了不要紧）。Q3 里"先 `del` 后读"最容易触发。
- **Q3 数值的比例对，但判为不相等** → 多做了归一化。`eliminate` 只求和。
- **Q4 报 `Factor has only one unconditioned variable, so you can't eliminate`** → 该丢弃的因子却调用了 `eliminate`。
- **Q4 报 `Order of joining by variables and elimination by variables is incorrect`** → 调用序列与标准不一致，
  对照 §5 Q4 的序列表检查。
- **Q4 报 `The factor to be normalized must have only one assignment of the conditional variables`** →
  最终因子里还有取值域大于 1 的条件变量，通常是漏掉了某个变量自己的 CPT（先验），例如 `P(W)`，
  导致 `W` 没能变成非条件变量。
- **改了函数名 / 签名，或把答案写到别的文件** → autograder 按名字在两个学生文件里找入口，找不到直接判错（见 §3）。
