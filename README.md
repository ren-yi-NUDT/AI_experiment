# CS188 Pacman Project 2: Multi-Agent Pacman — 作业指引

本目录用于完成 UC Berkeley **CS188 Project 2: Multi-Agent Pacman**（多智能体对抗搜索）。
官方项目页:<https://inst.eecs.berkeley.edu/~cs188/archive/fa25/projects/proj2/>

一句话概括：给 Pacman 装上"会思考的脑子"——**Reflex Agent（反射式）→ Minimax（极小极大）→ Alpha-Beta（剪枝加速）→ 评估函数**，
让吃豆人在有幽灵追杀的棋盘上做出理性决策。

---

## 1. 环境说明

| 项目 | 状态 |
|---|---|
| Python | 3.12.3,**命令为 `python3`**(系统没有 `python` 别名) |
| tkinter | ✅ 已安装(8.6),图形界面依赖已满足 |
| 图形窗口 | 服务器 `DISPLAY` 为空(无 X 服务)。直接跑 `python3 pacman.py` 会报 `no display name`。**三种解决办法**:① 在自己桌面机跑;② `ssh -X` 转发;③ 一律加 `-t` 用文本模式(推荐,做作业完全够用) |
| 代码位置 | `multiagent/` 子目录,**所有命令都在该目录下执行** |

```bash
cd /home/ren/Code/AI_experiments/experiment2/multiagent
```

## 2. 目录结构(只需关心加粗的**一个**文件)

```
multiagent/
├── multiAgents.py       ← ★ 唯一要写的文件:全部 5 个 agent / 评估函数都在这里
├── pacman.py            ← 游戏引擎;定义 GameState(你会大量用到)
├── game.py              ← 世界规则与基础类型:AgentState / Direction / Grid
├── util.py              ← 工具:manhattanDistance、PriorityQueue、lookup 等
├── ghostAgents.py       ← 幽灵策略:RandomGhost / DirectionalGhost
├── layout.py + layouts/ ← 地图读取与 .lay 地图文件
├── autograder.py        ← 自动评测器(只跑,不改)
├── multiagentTestClasses.py / testClasses.py / testParser.py / grading.py  ← 评测内部逻辑
├── test_cases/          ← 各题测试用例(q1–q5)
└── 其余 graphics*/textDisplay/keyboardAgents 等 ← 引擎与显示(不改)
```

> **只改 `multiAgents.py` 这一个文件。**项目页明确要求：不要改动发行包里的其它文件，也不要从 Project 1 拷代码过来（跨项目混用文件会导致评测失败）。

## 3. 评分构成与完成位置一览

**本课程平台（头歌）要求**：

| 题号 | 内容 | 平台分值 | 要求 | 修改位置 |
|---|---|---|---|---|
| Q1 | Reflex Agent | 15 | 必做 | `multiAgents.py` → `ReflexAgent.evaluationFunction()` |
| Q2 | Minimax | 40 | 必做 | `multiAgents.py` → `MinimaxAgent.getAction()` |
| Q3 | Alpha-Beta Pruning | 40 | 必做 | `multiAgents.py` → `AlphaBetaAgent.getAction()` |
| Q4 | Evaluation Function | 5 | 选做 | `multiAgents.py` → `betterEvaluationFunction()` |

**官方 fa25 原始项目**（本地 autograder 按它计分，共 25 分，用于自测）：

| 官方题号 | 内容 | 官方分值 | 本地命令 | 对应平台题号 |
|---|---|---|---|---|
| Q1 | Reflex Agent | 4 | `-q q1` | Q1 |
| Q2 | Minimax | 5 | `-q q2` | Q2 |
| Q3 | Alpha-Beta Pruning | 5 | `-q q3` | Q3 |
| Q4 | Expectimax | 5 | `-q q4` | 平台未要求,**但见下** |
| Q5 | Evaluation Function | 6 | `-q q5` | Q4(选做) |

> ⚠️ **两处易混，务必记住：**
> 1. **官方 Q5 = 平台 Q4**（评估函数）；官方 Q4 是 Expectimax。
> 2. **官方的评估函数测试是拿 `ExpectimaxAgent` 当载具跑的**：`test_cases/q5/grade-agent.test` 里写的是
>    `agentName: "ExpectimaxAgent"` + `agentArgs: "evalFn=better"`。所以即使 Expectimax 不在平台清单上，
>    **也必须把它写了**，否则本地 `python3 autograder.py -q q5` 会因 `util.raiseNotDefined()` 直接报错、选做的 5 分无从验证。
>    因此 `multiAgents.py` 里 5 个位置全部要填。

## 4. 逐题指引

`GameState`（定义在 `pacman.py`）是贯穿全项目的核心对象，常用接口先列在这里：

| 调用 | 含义 |
|---|---|
| `state.getLegalActions(agentIndex)` | 该 agent 的合法动作列表；**agentIndex=0 是 Pacman，≥1 是幽灵** |
| `state.generateSuccessor(agentIndex, action)` | 执行动作后的新 `GameState` |
| `state.getNumAgents()` | 总 agent 数(Pacman + 幽灵数) |
| `state.isWin()` / `state.isLose()` | 终局判定 |
| `state.getScore()` | 当前分数(float) |
| `state.getPacmanPosition()` | Pacman 坐标 `(x, y)` |
| `state.getGhostStates()` / `getGhostPositions()` | 幽灵状态列表 / 坐标列表 |
| `state.getFood()` → `.asList()` | 剩余豆子网格 / 坐标列表 |
| `state.getNumFood()` / `state.getCapsules()` | 剩余豆子数 / 剩余能量豆坐标 |
| `state.getWalls()` | 墙网格 |

`AgentState`（`game.py`）代表某个 agent 的状态：`getPosition()`、`getDirection()`、`scaredTimer`（被惊吓剩余回合数）。

### Q1 — ReflexAgent 评估函数(必做)

改 `ReflexAgent.evaluationFunction(self, currentGameState, action)`。
调用它的 `getAction()` 已经写好了：对每个合法动作打分，取最高分（并列随机选），所以**你只要写"打分"这一段**。

- **重点：这一问评估的是"状态-动作对"(`state-action pair`)，后续 Q2–Q4 评估的是"状态"(`state`)。**别搞反。
- 已经从 `successorGameState` 里取好了：`newPos`、`newFood`、`newGhostStates`、`newScaredTimes`。
- 官方提示：**特征的倒数比特征本身好用**（比如"到食物距离的倒数"，越近分越高，且不会有量纲/截断问题）。
- 必须**同时考虑食物和幽灵**：只追豆子会被吃，只躲幽灵会饿死。
  - 食物项：到最近豆子距离的倒数（`newFood.asList()` 拿到所有豆子坐标，配 `util.manhattanDistance`）。
  - 幽灵项：对每只幽灵，非惊吓态且贴脸 ⇒ 重罚（例如 `- 1/dist` 或距离小于阈值时给大负分）；`scaredTimer > 0` 时反过来给奖励（可以去吃它）。
  - 兜底项：`successorGameState.getScore()`（吃豆加分、等待扣分），以及剩余豆子数的少量惩罚。

**验收**：`python3 autograder.py -q q1 --no-graphics`
官方在 `openClassic` 上跑 **10 局**（`RandomGhost`，固定随机种子 0），满分 4 分的分档：

| 档位 | 条件 | 分值 |
|---|---|---|
| 不超时 | 10 局全部不超时（`nonTimeoutMinimum: 10`，**任何一局超时/崩溃 → 本题直接 0 分**） | 必须 |
| 胜场 | 至少赢 5 局 +1；赢满 10 局再 +1 | 0–2 |
| 平均分 | >500 +1；>1000 再 +1 | 0–2 |

手动跑：

```bash
python3 pacman.py -p ReflexAgent -l testClassic -t          # 应能稳定通关
python3 pacman.py --frameTime 0 -p ReflexAgent -k 1 -t      # mediumClassic 1 幽灵
python3 pacman.py --frameTime 0 -p ReflexAgent -k 2 -t      # 2 幽灵，容易翻车
```

### Q2 — Minimax(必做，40 分)

改 `MinimaxAgent.getAction(self, gameState)`，实现极小极大搜索。

- 用 `self.depth` 与 `self.evaluationFunction`——**不要硬编码调用 `scoreEvaluationFunction`**，因为后面 Q4 会换评估函数（通过 `-a evalFn=better` 传入）。
- **1 ply(一层) = Pacman 走一步 + 所有幽灵各走一步**，所以 `depth=2` 指 Pacman 和每只幽灵都动两次。
- **多幽灵的树比课上讲的更宽**：每个 max 层下面有**与幽灵数相同个数**的 min 层（幽灵按 agentIndex 递增依次行动）。
  典型写法是递归 `(agentIndex, depth)`：`agentIndex == self.index` 时是 max 层；
  `agentIndex` 走到最后一只幽灵后，下一个 agent 就绕回 Pacman 并把 `depth - 1`。
- 递归终止：`state.isWin()` / `isLose()` 或 `depth == 0` ⇒ 返回 `self.evaluationFunction(state)`。
- **autograder 会严格比对 `GameState.generateSuccessor` 的调用次数**（以及展开出的状态集合与根动作是否与参考解一致），
  调用多一次或少一次都会判错——所以**不要为了"提前判断胜负"而多调用**，也不要漏掉合法的子节点。

**验收**：

```bash
python3 autograder.py -q q2 --no-graphics
python3 pacman.py -p MinimaxAgent -l minimaxClassic -a depth=4 -t
```

- 官方给出 `minimaxClassic` 布局初始状态的 minimax 值：**depth 1/2/3/4 分别为 9 / 8 / 7 / -492**，可用来校验你的实现。
- 若在 `trappedClassic` 上看到 Pacman 主动冲向最近的幽灵，那是**正确**行为：它认为必死，于是尽早结束以少扣"存活惩罚"。
- `minimaxClassic` 上 depth 4 的移动通常很慢（几秒/步），属正常。

### Q3 — Alpha-Beta Pruning(必做，40 分)

新建逻辑写进 `AlphaBetaAgent.getAction(self, gameState)`：在 Minimax 基础上加 α-β 剪枝。

- **剪枝后得到的 minimax 值与 Q2 完全相同**（选出的动作可能不同，因为并列时的打破顺序不同）。
- 同样要**扩展到多幽灵**：多只幽灵时，α 要沿 min 链一路往上传（max 层只需把 `alpha` 更新后传给下一只幽灵）。
- **两条硬性要求，违反就会和 autograder 的参考状态集对不上**：
  1. **不得重排子节点**——必须严格按 `getLegalActions()` 返回的顺序展开。
  2. **不得在相等时剪枝**——只允许严格不等号：
     max 层 `if v > beta: return v`；min 层 `if v < alpha: return v`（写成 `>=` / `<=` 会剪掉参考解里应被展开的节点，典型对照用例是 `test_cases/q3/6-tied-root`）。
- 目标：**depth 3 的 alpha-beta 大约和 depth 2 的 minimax 一样快**。

**验收**：

```bash
python3 autograder.py -q q3 --no-graphics
python3 pacman.py -p AlphaBetaAgent -a depth=3 -l smallClassic -t
```

### Q4 — Evaluation Function(选做，5 分)

改 `betterEvaluationFunction(currentGameState)`，注意**它评估的是 state，不是 action**（跟 Q1 的评估函数不一样）。
`multiAgents.py` 末尾已有别名 `better = betterEvaluationFunction`，所以命令行可以写 `-a evalFn=better`。

- 用 depth 2 搜索 + 你的评估函数，应当在 `smallClassic`（1 只随机幽灵）上**超过一半的局数获胜**，且平均分在赢的时候接近 1000。
- 可用的特征（示例，不限于此）：分数、到最近豆子的距离、剩余豆子数、胶囊位置与数量、与每只幽灵的距离、`scaredTimer` 剩余回合。
  建议写成"线性加权"：分数 + 各项加权项；用**曼哈顿距离**（`util.manhattanDistance`）而不是欧氏距离。
- 和 Q1 一样，**幽灵惊吓中（`scaredTimer` 大）应给奖励/让它去追；未被惊吓且贴脸时给重罚**。
- 运行太慢会丢分（见下表），所以别写太笨重的特征（例如每次调用都跑一遍全图 BFS）。

**验收**：`python3 autograder.py -q q5 --no-graphics`（**注意是官方 q5**）
官方在 `smallClassic` 上跑 10 局，满分 6 分的分档：

| 档位 | 条件 | 分值 |
|---|---|---|
| 不超时 | 10 局全部不超时（`nonTimeoutThresholds: 10`） | 0–1 |
| 胜场 | 至少赢 1 局 +1；≥5 局再 +1；赢满 10 局再 +1 | 0–3 |
| 平均分 | ≥500 +1；≥1000 再 +1 | 0–2 |

同样地，**任一档低于其下限（`minimum`），本题直接 0 分**。

### 附：Expectimax（官方 Q4，平台不要求，但 Q4 评估函数依赖它）

改 `ExpectimaxAgent.getAction(self, gameState)`：树结构和 Minimax 完全一样，唯一区别是——
**幽灵层不再取"最小值"（假设对手最优），而是对所有合法动作取"平均值"（假设对手均匀随机乱走）**：

$$\text{value} = \frac{1}{|actions|}\sum_{a} \text{value}(successor)$$

这样 Pacman 面对随机乱走的幽灵时会更大胆（敢于火中取栗多啃几颗豆），更符合 `RandomGhost` 的真实行为。

**验收**：

```bash
python3 autograder.py -q q4 --no-graphics
python3 pacman.py -p ExpectimaxAgent -l minimaxClassic -a depth=3 -t
# 对照实验：trappedClassic 上 AlphaBeta 必败，Expectimax 大约能赢一半
python3 pacman.py -p AlphaBetaAgent -l trappedClassic -a depth=3 -q -n 10
python3 pacman.py -p ExpectimaxAgent -l trappedClassic -a depth=3 -q -n 10
```

## 5. 运行与评测命令汇总

**自动评测**（在 `multiagent/` 目录下）：

```bash
python3 autograder.py --no-graphics          # 全部题目（满分基线 0/25）
python3 autograder.py -q q1 --no-graphics    # 单题；q1…q5
python3 autograder.py -t q2/0-small-tree --no-graphics   # 单个测试用例
```

> ⚠️ `autograder.py -t` 的 `-t` 是 **`--test`（跑单个测试）**；
> 而 `pacman.py -t` 的 `-t` 是 **`--textGraphics`（文本模式）**。**同名不同义，别用混。**

**手动跑游戏**：

```bash
python3 pacman.py -p ReflexAgent    -l testClassic    -t
python3 pacman.py -p MinimaxAgent   -l minimaxClassic -a depth=4 -t
python3 pacman.py -p AlphaBetaAgent -l smallClassic   -a depth=3 -t
python3 pacman.py -p ExpectimaxAgent -a depth=3 -l smallClassic -t
```

**常用命令行参数**：

| 参数 | 含义 |
|---|---|
| `-p AgentName` | 指定 Pacman 的策略类（本项目的四个 agent 名） |
| `-l layout` | 指定布局，如 `testClassic` / `minimaxClassic` / `openClassic` / `smallClassic` |
| `-a depth=3,evalFn=better` | 传参给 agent 构造函数 |
| `-k N` | 幽灵数量 |
| `-n N` | 连跑 N 局 |
| `-q` | 安静文本模式（快速跑多局） |
| `-t` | 文本模式（有逐帧文字输出） |
| `-f` | 固定随机种子（便于复现同一局） |
| `--frameTime 0` | 关闭逐帧延迟，跑得更快 |

## 6. 提交打包(按平台要求)

1. 只交 **1 个文件**：`multiAgents.py`（改了多少都只交这个，文件不能改名）。
2. 建「**学号_姓名**」文件夹（**下划线**连接，如 `20220602_张三`），把该文件放进去，压缩 zip 上传头歌。
3. 实验报告另存为「**学号_姓名_实验2**」单独提交。

```bash
# 打包示例（替换成你的学号姓名）
cd /home/ren/Code/AI_experiments/experiment2
mkdir 20220000_你的名字 && cp multiagent/multiAgents.py 20220000_你的名字/
zip -r 20220000_你的名字.zip 20220000_你的名字
```

> 交前自检：`cd multiagent && python3 autograder.py --no-graphics` 看各题得分；
> 并确认 `multiAgents.py` 里**没有打印调试语句残留**（`print(...)` 在批量跑局时会刷屏且拖慢评测）。

## 7. 常见坑

- **`python` 不存在** → 一律 `python3`。
- **图形报 `no display name`** → 加 `-t`（文本模式）或 `--no-graphics`。
- **`autograder.py` 的 `-t` 会报"找不到测试"** → 别把它和 `pacman.py -t` 记混：autograder 的 `-t` 后面要跟测试名，如 `-t q2/0-small-tree`。
- **Q2/Q3 明明赢了却判错** → autograder 不看你赢没赢，看的是**展开状态集合 + 根动作**是否与参考一致。多半是：
  - `generateSuccessor` 多调/漏调；
  - 把"1 ply"理解成了 Pacman 走一步（忽略了幽灵也要算一层）；
  - 叶子判定或 `depth` 递减的位置写错。
- **Q3 值与 Q2 不一致 / 状态集合对不上** → 检查是否**重排了子节点**，或用了 `>=` / `<=` 做剪枝（必须严格不等号）。
- **Q5（评估函数）报 `NotImplementedError` / 直接崩** → `ExpectimaxAgent.getAction` 还是 `util.raiseNotDefined()`，没写。它是评估函数测试的载具。
- **Q1/Q4 得 0 分但看着"能跑"** → 平台/官方规则是**任一档低于下限则本题 0 分**；最常见的是**超时或崩溃**（例如评估函数太慢、递归深度过大）。
- **Minimax 在 `minimaxClassic` 上极慢** → depth 4 的树很大，属正常；用 `-q` 跑多局或降 depth 调试。
- **改了不该改的文件** → 评测只读 `multiAgents.py`，其余文件请保持原样以便对照。
