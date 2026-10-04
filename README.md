# CS188 Pacman Project 1: Search — 作业指引

本目录用于完成 UC Berkeley **CS188 (fa25) Project 1: Search in Pacman**。
官方项目页:<https://inst.eecs.berkeley.edu/~cs188/fa25/projects/proj1/>(本机已下载 **v1.004**)

---

## 1. 环境说明

| 项目 | 状态 |
|---|---|
| Python | 3.12.3,**命令为 `python3`**(系统没有 `python` 别名) |
| tkinter | ✅ 已安装(8.6),图形界面依赖已满足 |
| 图形窗口 | 服务器 `DISPLAY` 为空(无 X 服务)。直接跑 `python3 pacman.py -l mediumMaze ...` 会报 `no display name`。**三种解决办法**:① 在自己桌面机跑;② `ssh -X` 转发;③ 一律加 `-t` 用文本模式(推荐,做作业完全够用) |
| 代码位置 | `search/` 子目录,**所有命令都在该目录下执行** |

```bash
cd /home/ren/Code/AI_experiments/experiment1/search
```

## 2. 目录结构(只需关心加粗的两个文件)

```
search/
├── search.py          ← ★ 你要写的文件 1:Q1–Q4 四个通用搜索算法
├── searchAgents.py    ← ★ 你要写的文件 2:Q5–Q8 四个问题/启发式
├── autograder.py      ← 自动评测器(只跑,不改)
├── pacman.py          ← 游戏引擎(不改)
├── util.py            ← 提供现成的 Stack / Queue / PriorityQueue(直接用,不用自己写数据结构)
├── searchTestClasses.py / testClasses.py / testParser.py / grading.py  ← 评测内部逻辑
├── test_cases/        ← 各题测试用例(q1–q8)
├── layouts/           ← 地图文件(.lay)
└── 其余 pacman/game/graphics*.py 等 ← 引擎与显示(不改)
```

> ⚠️ fa25 版本与网上旧教程的差异:没有 `-q StayEast/StayWest` 评分参数;内置 agent 只有 `LeftTurnAgent`、`GreedyAgent`(以及各题的 `SearchAgent` 系列)。

## 3. 评分构成与完成位置一览

| 题号 | 内容 | 分值 | 修改位置 |
|---|---|---|---|
| Q1 | 深度优先搜索 DFS | 10 | `search.py` → `depthFirstSearch()` |
| Q2 | 广度优先搜索 BFS | 10 | `search.py` → `breadthFirstSearch()` |
| Q3 | 变代价搜索 UCS | 10 | `search.py` → `uniformCostSearch()` |
| Q4 | A* 搜索 | 20 | `search.py` → `aStarSearch()` |
| Q5 | 角落问题(搜索问题建模) | 10 | `searchAgents.py` → `CornersProblem`(约 L298/L305/L328 三处 `*** YOUR CODE HERE ***`) |
| Q6 | 角落问题启发式 | 15 | `searchAgents.py` → `cornersHeuristic()`(约 L363) |
| Q7 | 吃光豆子启发式 | 15 | `searchAgents.py` → `foodHeuristic()`(约 L457) |
| Q8 | 次优搜索(找最近豆子) | 10 | `searchAgents.py` → `findPathToClosestDot()`(约 L488)与 `AnyFoodSearchProblem.isGoalState()`(约 L524) |

**评测每个题:**

```bash
python3 autograder.py -q q1 --no-graphics   # 换 q2…q8
python3 autograder.py --no-graphics         # 全部(基线 0/25)
```

## 4. 逐题指引

搜索框架统一约定:写出的函数返回**从起点到目标的动作列表**(如 `['South','South','West',...]`),必须做**图搜索**(记录已扩展状态,防止绕圈死循环)。`util.py` 已备好 `util.Stack`、`util.Queue`、`util.PriorityQueue`(支持 `push(item, priority)`、`pop()`、`update(item, priority)`)。

**推荐实现路线:先写一个通用的 A*(Q4),让 DFS/BFS/UCS 全部由它退化而来**;或者独立写四份,思路更直观。以下按顺序说明。

### Q1 — DFS(10 分)

- 用 `util.Stack` 做 fringe;**先检查弹出节点的目标判定与"是否已扩展"再压入或弹出均可,但扩展时必须标记**,否则 infinite 测试会死循环。
- `problem.getSuccessors(state)` 返回 `(successor, action, stepCost)` 三元组列表;只需一路记录 `action` 回溯出路径(可用"状态 → (路径)"或"状态 → 父节点"字典)。
- 测试:

```bash
python3 pacman.py -l tinyMaze   -p SearchAgent -t   # 应能到达终点
python3 pacman.py -l mediumMaze -p SearchAgent -t   # 注意:DFS 解不是最短路,只要求合法到达
python3 autograder.py -q q1 --no-graphics
```

### Q2 — BFS(10 分)

- 把 Q1 的 `Stack` 换成 `util.Queue` 即可得到 BFS(BFS 天然最短步数)。`graph_infinite`、`graph_backtrack` 等图测试会同时校验两种遍历顺序下的正确性。
- 测试:

```bash
python3 pacman.py -l mediumMaze -p SearchAgent -a fn=bfs -t
python3 autograder.py -q q2 --no-graphics
```

### Q3 — UCS(10 分)

- 用 `util.PriorityQueue`,优先级为**累积代价 g(n)**;弹出时(或压入时)以 `(状态)` 为键去重扩展。
- 注意重复检测时机:**以"扩展(弹出)时"为准**才能保证最优性;`util.PriorityQueue.update(item, priority)` 可用于更松弛。
- 测试(图测试会检查扩展顺序):

```bash
python3 pacman.py -l mediumDottedMaze -p StayEastSearchAgent -t   # legacy 命令可能不可用,以下为准:
python3 autograder.py -q q3 --no-graphics
```

### Q4 — A*(20 分,大头)

- 实现 `aStarSearch(problem, heuristic=nullHeuristic)`:优先级 = `g(n) + h(n)`。
- **Q3 的 UCS 就是 `aStarSearch` + 零启发式**;实现完 Q4 后可回头把 UCS 改为 `return aStarSearch(problem)`。
- 测试(曼哈顿启发式已由框架提供):

```bash
python3 pacman.py -l bigMaze -z .5 -p SearchAgent -a fn=astar,heuristic=manhattanHeuristic -t
python3 autograder.py -q q4 --no-graphics   # 含 goalAtDequeue 顺序测试,验证弹出时才判目标
```

### Q5 — CornersProblem 建模(10 分)

目标:从起点出发,访问地图四个角即算成功。难点是把"哪些角已访问过"编进**状态**里。

- 三处填空:`__init__` 里自定义状态(推荐 `(位置, 已访问角落元组/位掩码)`);`getSuccessors` 里走到新位置时更新"已访问";`isGoalState` 里判断四角是否全访问。
- **不要把 walls/食物等无关量放进状态**(状态空间爆炸)。
- 测试:

```bash
python3 pacman.py -l mediumCorners -p AStarCornersAgent -z 10 -t
python3 autograder.py -q q5 --no-graphics   # tiny_corner 校验最短路长度
```

### Q6 — cornersHeuristic(15 分)

- 要求**可采纳且一致**(admissible & consistent),且 mediumCorners 上扩展节点数 **≤ 2000 / 1600 / 1200 三档给分**(源码 `test_cases/q6/medium_corners.solution`)。
- 经典满分思路:`h = max(未访问角 c 的 manhattan(当前位置, c))`,可进一步用"贪心链式最近角"增强,但对满分配合 UCS 足够时优先简单方案——**注意必须基于未访问的角**,且对全部未访问角取 max(取 max 保持可采纳)。
- 测试:

```bash
python3 pacman.py -l mediumCorners -p AStarCornersAgent -z 10 -t
python3 autograder.py -q q6 --no-graphics   # 看 expanded nodes 是否进入 1200 档
```

### Q7 — foodHeuristic(15 分)

目标:吃光所有豆子。状态为 `(吃豆人位置, foodGrid)`,启发式签名 `foodHeuristic(state, problem)`。

- 满分门槛:在 `trickySearch` 上 **expanded nodes ≤ 7000**(四档阈值 `15000/12000/9000/7000`,基线 1 分 + 每过一档 1 分,详见 `searchTestClasses.py` 的 `HeuristicGrade`)。
- 其余 17 个小测试校验**可采纳/一致性**(返回的路径必须最优,启发式不能高估真实代价)。
- 经典满分思路:`h = max(豆子 d 的迷宫真实距离 manhattan 不够 → 用 mazeDistance(位置, d))`,`mazeDistance` 已在 `searchAgents.py` 底部提供(BFS 实现,可直接调用);所有豆子中取最大。注意 `foodHeuristic` 会被调用**很多次**,缓存 `problem.walls` 上的距离结果可大幅提速。
- 测试:

```bash
python3 pacman.py -l trickySearch -p AStarFoodSearchAgent -t
python3 autograder.py -q q7 --no-graphics
```

### Q8 — ClosestDotSearchAgent(10 分)

要求:逐个走向**最近的**豆子(允许次优总路径,本问拿满 10 分即可)。

- 两处填空:`AnyFoodSearchProblem.isGoalState`(判断该状态位置是否恰是目标豆 `self.target`/任一豆子)和 `findPathToClosestDot`(直接调 `search.bfs(problem)` 返回路径)。
- 用 Q2 的 BFS,不需要改 search.py。
- 测试:

```bash
python3 pacman.py -l trickySearch -p ClosestDotSearchAgent -t
python3 pacman.py -l bigSearch    -p ClosestDotSearchAgent -t
python3 autograder.py -q q8 --no-graphics
```

## 5. 提交打包(按课程要求)

1. 只交 **2 个文件**:`search.py`、`searchAgents.py`(改动了多少都只交这两个,不能多不能少,文件名不能改)。
2. 建「**学号_姓名**」文件夹(**下划线**连接,如 `20220602_张三`),把两个文件放进去,压缩 zip 上传头歌。
3. 实验报告另存为「**学号_姓名_实验1**」单独提交。

```bash
# 打包示例(替换成你的学号姓名)
cd /home/ren/Code/AI_experiments/experiment1
mkdir 20220000_你的名字 && cp search/search.py search/searchAgents.py 20220000_你的名字/
zip -r 20220000_你的名字.zip 20220000_你的名字
```

> 交前自检:`cd search && python3 autograder.py --no-graphics`,确认各题得分;并确认两个文件里**没有打印调试语句残留**(某些测试对输出敏感)。

## 6. 常见坑

- **`python` 不存在** → 一律 `python3`。
- **图形报 `no display name`** → 加 `-t`,或在有显示器的机器跑(`ssh -X` 也行)。
- **死循环/卡住** → 八成是没做图搜索的"已扩展"标记,或扩展时机不对(Q3/Q4 必须弹出时判目标)。
- **Q4 的 `goalAtDequeue` 测试挂** → 说明你在压入 fringe 时就返回了答案;应改为**从优先队列弹出时**检查 `isGoalState`。
- **Q7 跑得极慢** → 启发式里对每个豆子现算 BFS 会重复计算;按 `walls` 缓存。
- **改了不该改的文件** → 评测只读 `search.py`/`searchAgents.py`,但其余文件请保持原样以便对照。

祝顺利拿满 100 分!
