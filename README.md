# EPR-COE

面向**约束优化问题**的差分进化（DE）求解框架，MATLAB Live Script 实现。
测试基准为 **CEC2017 约束优化测试集**（28 个约束函数，支持 10 / 30 / 50 / 100 维）。

EPR-COE 在标准 DE 骨架上叠加了三类机制：

- **自适应权衡权重**：目标值与违约度（constraint violation）按随代数变化的权重加权；
- **分阶段约束处理**：`IDFR`（改进的 ε 约束）规则 + `FR`（可行性规则）按问题约束难度切换；
- **搜索痕迹驱动的种群重启**：利用各维度分箱网格上的探索/可行/不可行分布，针对"长期无可行解""早熟收敛""搜索停滞"三种场景做定向重启。

> 仓库未附带论文原文，缩写的正式展开请以作者论文为准。

---

## 1. 目录结构

```
EPR-COE/
├── README.md                    # 本文件
├── main.mlx                     # 主程序（入口）
├── fitness.mlx                  # CEC2017 28 个约束测试函数（目标值 + 违约度）
├── range.mlx                    # 各函数的变量上下界
├── tribution.mlx                # 搜索痕迹统计：每维分箱的探索/可行/不可行计数
├── generateBest.mlx             # 变异交叉策略 A（最优个体引导，current-to-best 型）
├── generateRand.mlx             # 变异交叉策略 B（rand/1 型）
├── select1.mlx                  # 选择策略 1：加权目标 + 违约度
├── select2.mlx                  # 选择策略 2：IDFR 规则
├── select3.mlx                  # 选择策略 3：可行性规则（FR）
├── InfeasibleRestart.mlx        # 重启 1：长期无可行解
├── PrematureRestart.mlx         # 重启 2：早熟收敛
├── StagnateRestart.mlx          # 重启 3：搜索停滞
├── ifRemature.mlx               # 早熟/停滞判定（维度聚集度）
├── evolutionRate.mlx            # 进化率（目标值相对改进量）
├── updateBest.mlx               # 全局最优解维护
├── getdata.mlx                  # 结果导出：每次运行最终最优值
├── getList.mlx                  # 结果导出：历代最优解 / 违约度曲线
├── getRestart.mlx               # 结果导出：重启记录
├── num2ExcelCol.mlx             # 工具：列号 → Excel 列字母
├── roh.mlx                      # 绘图：重启概率曲线（论文图 3）
├── fes.mlx                      # 绘图：单函数收敛曲线
├── shoulian.mlx                 # 绘图：多算法归一化收敛曲线对比
├── picture.mlx                  # 绘图：算法优劣统计柱状图
├── Function1.mat … Function11.mat   # f1–f11 的位移向量 o（及 f2/f5 的旋转矩阵）
└── ShiftAndRotation.mat         # f12–f28 的位移向量 o 与旋转矩阵 M
```

---

## 2. 环境要求

| 项目 | 要求 |
| --- | --- |
| MATLAB | **R2019a 或更新版本**（导出/绘图脚本用到 `writematrix` / `readmatrix`；只跑 `main` 时 R2017a+ 即可） |
| 文件格式 | Live Script（`.mlx`），需要 MATLAB R2016a+ |
| 工具箱 | 核心算法只依赖基础 MATLAB；`shoulian.mlx` / `picture.mlx` 使用 `categorical`、`bar`，均为基础功能 |
| 可选 | Parallel Computing Toolbox（把外层循环改为 `parfor` 加速） |

无需编译，解压即可运行。

---

## 3. 快速开始

1. 解压整个 `EPR-COE` 文件夹（**`.mlx` 与 `.mat` 必须在同一目录**，`fitness` 用 `load Function1` 这类相对路径读取数据）。
2. 用 MATLAB 打开 `main.mlx`。
3. **修改结果路径**（第 1 段代码，必需）：

   ```matlab
   filePath = "C:\Users\wxp20\Desktop\研究\实验\收敛\2017\结果";  % 改成你自己的目录
   ```

   该目录需事先存在；若只跑算法、不导出 Excel，可暂时忽略（导出语句默认被注释）。
4. 设置要跑的函数（默认只跑 f13）：

   ```matlab
   for problemIndex = 13        % 单函数；要跑全部改成 1:problemNum
   ```

5. 按需修改 `dim`、`popsize`、`totalRun`，然后 **Run**。

一次运行会在命令行每 1000 次评估打印一行进度：

```
problemIndex：13, run：1, 2000, 0.12%, 57, 1.234567890e+02, 0.000000000e+00
```

字段依次为：函数编号、运行序号、当前 FES、可行率、当前最优个体下标、最优目标值、最优违约度。注意可行率是 0–1 的比例，代码按 `%.2f%%` 打印但未乘 100。**违约度 `conV = 0` 表示已找到可行解。**

### 最小调用示例（单独测试某个函数）

```matlab
global initial_flag
initial_flag = 0;                       % 必须清零，否则不会重新 load 数据
[objF, conV] = fitness(rand(5, 10), 1); % 5 个个体、10 维、f1
```

---

## 4. 参数说明

`main.mlx` 顶部的可调参数：

| 参数 | 默认值 | 含义 |
| --- | --- | --- |
| `filePath` | 作者本机路径 | 结果 Excel 输出目录，**使用前必须修改** |
| `problemIndex` | `13` | 测试函数编号 1–28 |
| `problemNum` | `28` | 测试集函数总数（仅用于预分配记录数组） |
| `dim` | `100` | 问题维数，可选 10 / 30 / 50 / 100 |
| `popsize` | `100` | 种群规模 |
| `totalFES` | `dim * 20000` | 总评估次数（100 维时为 2 000 000） |
| `maxGen` | `totalFES / popsize` | 最大迭代代数（100 维时为 20 000） |
| `totalRun` | `1` | 每个函数的独立运行次数 |
| `classifyGen` | `dim * 2` | 约束难度分类的判定代数 |
| `Gb` | `maxGen * 0.8` | 允许重启的阶段上限（前 80% 代数） |

内部关键量：

| 变量 | 含义 |
| --- | --- |
| `VAR0` / `VAR` / `cp` | 违约度阈值（ε 水平）及其指数衰减参数 |
| `pr` / `pmax` / `weights` | 目标—违约权衡权重的上界与每个个体的权重 |
| `classifyRate` | 前 `classifyGen` 代内违约度的相对下降比例，用于判定"强约束问题" |
| `IDFRVAR0` | IDFR 规则的初始松弛量，取 `max(conV - conVtrial)` |
| `splitNum` | 每维分箱数，`floor(sqrt(range * popsize))` |
| `hisTri` / `hisFea` / `hisInfea` | 累积的探索 / 可行 / 不可行分布（`dim × splitNum`） |
| `recordRestart` | 重启记录，上限 30 次 |

---

## 5. 算法流程

```
初始化种群（均匀随机）
for gen = 1 : maxGen
    ① 调整权衡权重
       VAR 指数衰减；可行率 > 85% 时置 0；pr 由 sigmoid 控制
       在 [0, pr] 上等距生成 popsize 个权重，再随机分配给个体
    ② 变异 / 交叉
       f1, f2, f4 → generateRand（rand/1）
       其余      → generateBest（最优个体引导）
       F ∈ {1.0, 0.8, 0.6}，CR ∈ {1.0, 0.2, 0.1}，逐个体随机
       越界：镜像反射或重置到边界
    ③ 计算试验个体 fitness → (objF, conV)
    ④ 记录搜索痕迹（仅前 Gb 代）
    ⑤ 选择
       gen < classifyGen 或已出现可行解 → select1（加权适应度）
       否则 → select2（IDFR）
              且 classifyRate <= rand（强约束）时再执行 select3（FR）
    ⑥ 重启（仅前 Gb 代）
       长期无可行解 → InfeasibleRestart
       可行率长期 = 1 且进化率停滞
           ifRemature 判定 → PrematureRestart（早熟）
           否则           → StagnateRestart（停滞）
       被重启个体 = 适应度最差的 75%，按分箱累积概率重采样
    ⑦ updateBest 更新全局最优并记录曲线
end
```

三种重启共用一套"搜索痕迹"概率模型：

- **未探索区域**：`1 / hisTri` 归一化，探索越少的箱被选中概率越大；
- **不可行区域**：`1 / hisInfea` 归一化（重启 1）；
- **可行区域**：`hisFea / sum(hisFea)`（重启 2、3）；
- **最优解区域**：当前最优个体所在箱（重启 3）。

重启 2、3 只重采样"聚集维度"（`ifRemature` 判定为过密的维度）或"非聚集维度"，从而在保持已收敛维度的同时探索其余维度。

---

## 6. 数据文件（`.mat`）

| 文件 | 变量 | 尺寸 | 用途 |
| --- | --- | --- | --- |
| `Function1.mat`, `Function3`–`Function11.mat` | `o` | 1×100 | 位移向量（shift） |
| `Function2.mat` | `o`, `M_10`, `M_30`, `M_50`, `M_100` | 1×100；10/30/50/100 阶 | 位移 + 旋转矩阵 |
| `Function5.mat` | `o`, `M1_*`, `M2_*` | 同上 | 位移 + 两组旋转矩阵 |
| `ShiftAndRotation.mat` | `o`, `M_10`, `M_30`, `M_50`, `M_100` | 同上 | f12–f28 的位移；f21–f28 另用旋转矩阵 |

- `o` 统一存为 100 维，`fitness` 内部按 `o(1:D)` 截取，因此同一套数据支持 10 / 30 / 50 / 100 维。
- `fitness` 用 `global initial_flag` + `persistent` 变量缓存，只 `load` 一次；**切换函数或维数时必须把 `initial_flag` 置 0**（`main` 每次 run 都会重置）。
- 约束处理：不等式 `g <= 0`、等式 `|h| <= 1e-4`；违约度 `conV = sum(max(0, [g, |h|-1e-4]))`，可行解满足 `conV = 0`。

---

## 7. 输出与结果导出

运行时在内存中记录：

| 变量 | 尺寸 | 内容 |
| --- | --- | --- |
| `recordListobjF` | `maxGen × totalRun × problemNum` | 历代最优目标值 |
| `recordListconV` | `maxGen × totalRun × problemNum` | 历代最优违约度 |
| `recordRestart` | `30 × 2 × totalRun × problemNum` | 第 1 列 = 重启代数，第 2 列 = 类型（1 无可行解 / 2 早熟 / 3 停滞） |

导出函数（`main.mlx` 末尾默认被注释，取消注释即可）：

```matlab
getdata(problemIndex, dim, totalRun, maxGen, recordListobjF, recordListconV, filePath);
getList(problemIndex, dim, totalRun, maxGen, recordListobjF, recordListconV, filePath);
getRestart(problemNum, totalRun, recordRestart, filePath);
```

| 函数 | 输出文件 | 工作表 / 区域 |
| --- | --- | --- |
| `getdata` | `实验结果.xlsx` | `EPR-COE-1_<dim>D`，第 `3f-1` ~ `3f` 列（每个函数占 2 列） |
| `getList` | `EPR-COE_<dim>D历代最优解.xlsx` | `EPR-COE_<f>_<dim>_2`，每次运行占 3 列宽（目标值 + 违约度） |
| `getRestart` | `实验结果.xlsx` | `重启记录表`，每个函数占 30 行 |

导出的列位置由 `num2ExcelCol` 换算，依赖固定的模板布局；换用别的表格时需同步修改。

---

## 8. 复现论文图表

以下脚本都读取本机绝对路径，**运行前需修改其中的 `filePath`**：

| 脚本 | 说明 | 依赖 |
| --- | --- | --- |
| `roh.mlx` | 重启概率曲线（图 3），对比 sigmoid 与线性项 | 无 |
| `fes.mlx` | 单函数收敛曲线，并与 IUDE / ECO-HCT / DeCODE 对比（默认遍历 f3 / f10 / f12，标题写死为 C24） | `EPR-COE_<dim>D历代最优解.xlsx` |
| `shoulian.mlx` | 9 个算法的归一化收敛曲线 + FEs 标记点 | 工作区变量 `data`（`20000×9`，每列一个算法） |
| `picture.mlx` | 优劣/相似函数个数统计柱状图（默认 100 维） | 无（数据写死在脚本里） |

`fes.mlx` 读取的工作表名是 `EPR-COE_<f>_<dim>`，而 `getList` 写出的是 `EPR-COE_<f>_<dim>_2`，直接配合使用前请统一工作表名。

---

## 9. 常见问题与注意事项

1. **只跑了一个函数**：`for problemIndex = 13` 是单函数调试设置。跑全测试集改为：

   ```matlab
   for problemIndex = 1 : problemNum
       for run = 1 : totalRun
   ```

   如需并行，可把外层 `for` 换成 `parfor`：`main` 每轮都会把 `initial_flag` 置 0，各 worker 会各自 `load` 数据，通常可直接运行。但 `rand('seed', sum(100 * clock))` 在同时启动的 worker 上可能取到相同种子，建议改成 `rng('shuffle')` 或按 `run` 编号设种子。

2. **"未定义变量 o / M_10"**：`.mat` 未与 `.mlx` 放在同一目录，或 `initial_flag` 未被重置导致跳过了 `load`。

3. **路径写死**：`main.mlx`、`fes.mlx`、`roh.mlx`、`picture.mlx` 中都有作者本机路径（`C:\Users\wxp20\...`），迁移到别的机器必须逐个修改。

4. **`fitness.mlx` 中 f18 / f27 的疑似缺陷**：Rosenbrock 项写成 `x(1,1:D-1)`、`x(1,2:D)`，只取了第 1 个个体，等同于给所有个体加上同一个常数，`h` 约束实际失效。若要严格复现基准，应改为 `x(:,1:D-1)`、`x(:,2:D)`（`sum(...,2)`）。当前代码保留原样以保证与作者结果一致。

5. **重启数组上限 30**：`recordRestart` 预分配 30 次；重启次数更多时 `recordRestart(restartNum+1, ...)` 会越界。需要更多重启请增大第一维。

6. **重启采样可能越界**：`InfeasibleRestart` / `PrematureRestart` / `StagnateRestart` 中的

   ```matlab
   while p - pro(c, t + 1) >= 0
       t = t + 1;
   end
   ```

   依赖 `pro` 每行严格和为 1。`hisFea` 用第 1 行的总数做全局归一化，其他维度行和可能小于 1，浮点误差下有越界风险，可加 `t = min(t, splitNum - 1)` 保护。

7. **搜索痕迹的轻微滞后**：`InfeasibleRestart` 后调用 `tribution` 时传入的是重启前的 `conVtrial`，而非重新计算的 `conV`，统计略有偏差（不影响收敛性）。

8. **随机种子**：`rand('seed', sum(100 * clock))` 是旧式写法，新版本建议改为 `rng('shuffle')`（可复现实验则用 `rng(k)`）。

9. **中断即丢失**：结果只在全部 run 结束后写 Excel；长实验建议在循环内周期性 `save` 或在 `mod(FES, 1000) == 0` 处落盘。

10. **`shoulian.mlx` 依赖工作区变量 `data`**：需先在 workspace 中准备好对比算法的收敛矩阵，否则报"未定义变量"。

---

## 10. 引用与许可

- 基准问题：CEC2017 约束优化测试集（28 个函数）。
- 若在论文中使用本代码，请引用作者对应的 EPR-COE 论文；压缩包内未包含文献信息与 LICENSE，正式发布前请补充。
- 代码中保留了大量中文注释与调试痕迹，二次开发时建议先清理硬编码路径。
