# 心理旋转实验 · Web 版重构

复现 **Cooper & Shepard (1973)** 经典字符旋转任务的可直接运行网页：被试看到一个随机旋转 0°–300° 的大写 **R**（正写）或其水平镜像（反写 **Я**），任务是忽略角度、尽快判断正写还是反写。

原项目 [`Yideal/mental-rotation-experiment`](https://github.com/Yideal/mental-rotation-experiment) 是北京师范大学认知心理学课程的实验报告与单文件实验程序。本仓库保留原有实验逻辑与设计（2 字符类型 × 6 角度、练习 + 正式、F/J 按键、CSV 导出），对实验程序做了完整重写：修正了计时与呈现上的几处关键缺陷，并重做了界面与数据分析。

**在线地址：** https://clonedman9.github.io/mental-rotation-experiment/

---

## 快速开始

直接打开 `index.html` 即可（零依赖、无需构建、可离线）。或本地起一个静态服务：

```bash
python3 -m http.server 8000
# 打开 http://localhost:8000/
```

URL 参数可预填实验配置，便于课堂演示或小样本收集：

| 参数 | 含义 | 示例 |
| --- | --- | --- |
| `id` | 被试编号 | `?id=S007` |
| `per` | 每条件每组的试次数（12 条件） | `?per=5` → 60 试次/组 |
| `blocks` | 正式组数 | `?blocks=2` |
| `practice` | 练习每条件试次数 | `?practice=2` |
| `fix` | 注视点时长（ms） | `?fix=500` |
| `timeout` | 反应时限（ms） | `?timeout=4000` |
| `break` | 组间休息（秒） | `?break=120` |
| `stim` | 刺激大小（px） | `?stim=110` |

示例：`?id=S001&per=5&blocks=1&break=30` 会把正式实验压缩到 60 试次，适合快速演示。

---

## 与原版程序的差异

原程序能跑通整个流程，但存在几处会直接影响数据解释的问题。重构时逐项修正：

| # | 问题 | 原版 | 重构版 |
| --- | --- | --- | --- |
| 1 | **镜像与旋转的变换顺序** | `scale(-1,1) rotate(θ)`：先旋转后镜像，镜像轴随字符一起转动，镜像试次等效于反向旋转 | `rotate(θ) scale(-1,1)`：先水平翻转再整体旋转，镜像轴固定在字符自身坐标系内 |
| 2 | **反应时计时起点** | `trialStartTime` 在注视点呈现前赋值，RT 把 500 ms 注视点时间算进去（报告里 0° 平均 545 ms ≈ 500 + 45，正是这个偏移） | 以刺激实际绘制的时刻（双 `requestAnimationFrame` 之后）为零点，注视点与空屏不计入 RT |
| 3 | **按键门控** | 注视点期间与试次间隔内按键都会被记录，试次间隔内重复按键还会二次记录 | 仅在刺激呈现窗口内接受按键；按下即锁定，忽略长按重复（`e.repeat`） |
| 4 | **正式实验结构** | 代码为单个 block × 240 试次，中途休息 120 次处触发；与报告的“2 block × 120”不一致 | 严格按 `blocks × 12 × per` 组织，组间休息，休息可跳过 |
| 5 | **练习正确率标准** | 报告写“≥ 80% 才能进入正式实验”，代码未实现 | 练习后计算正确率，未达标提示重练一轮，练习数据不计入正式数据 |
| 6 | **刺激渲染** | `d3` 渲染 `<text font-family="Arial">`，字体缺失时字形被替换，且 48 px 在 1080p 上偏小 | 使用矢量字形轮廓（DejaVu Sans Bold 的 R，Bitstream Vera 许可），跨设备完全一致，几何中心即旋转原点 |
| 7 | **依赖** | 从 CDN 加载 D3.js v7，断网/被墙即白屏 | 零外部依赖，单文件，可离线、可托管任意静态空间 |
| 8 | **数据完整性** | 仅内存保存，刷新即丢失 | 每个试次写入 `localStorage`，刷新后可恢复；另有离开页面提醒 |
| 9 | **结果反馈** | 只给平均 RT 与正确率 | 内置条件描述统计、0°–180° 线性回归与旋转速度（°/s）、对称性检验、镜像成本、RT × 角度折线图 |
| 10 | **可用性** | 固定白色页面、无响应式、无触屏支持 | 浅色/深色配色、响应式布局、触屏虚拟按键、全屏模式、键盘焦点与 `aria-live` |

---

## 实验设计

- **设计**：2（字符类型：正写 / 反写）× 6（旋转角度：0°、60°、120°、180°、240°、300°）被试内设计。
- **刺激**：大写 R，正写或水平镜像，绕字形几何中心顺时针旋转。
- **反应**：`F` = 正写，`J` = 反写；单次作答，按下即记录。
- **流程**：指导语 → 练习（默认 24 试次，有反馈，正确率 ≥ 80%）→ 正式实验（默认 2 组 × 120 试次，无反馈）→ 组间休息 120 s → 结果页。
- **时序**：注视点 500 ms → 刺激（上限 4000 ms）→ 练习阶段反馈 450/950 ms，正式阶段空屏 600 ms。

默认参数下每人 **240 个正式试次**，与课程报告的 2 × 120 一致。

---

## 数据

结果页可导出 CSV / JSON。CSV 为 UTF-8（含 BOM，Excel 直接打开不乱码），字段如下：

| 字段 | 说明 |
| --- | --- |
| `subject_id` | 被试编号 |
| `session_id` | 本次会话 ID |
| `phase` | `practice` / `main` |
| `practice_round` | 练习轮次（仅练习） |
| `block` | 正式组序号（仅正式） |
| `trial_in_block` | 组内序号 |
| `global_index` | 全会话累计序号 |
| `angle` | 旋转角度（0/60/120/180/240/300） |
| `char_type` | `normal` / `mirror` |
| `expected_key` | 正确按键 |
| `response_key` | 实际按键（超时为空） |
| `rt_ms` | 反应时（ms，从刺激呈现起算） |
| `correct` | 1 正确 / 0 错误 |
| `is_timeout` | 1 超时 |
| `is_outlier` | 1 表示 RT < 200 或 > 3000 ms（按报告口径标记，不删除） |
| `stim_px` | 本次刺激高度（px），便于复现视觉条件 |
| `timestamp` | ISO 时间戳 |

清洗建议沿用报告口径：剔除 `< 200 ms` 与 `> 3000 ms`，再按被试 `M ± 3SD` 剔除极端值，仅保留正确试次。`is_outlier` 只是标记，不替使用者做删除决定。

**隐私**：所有数据只存在访问者自己的浏览器 `localStorage` 中，页面不发送任何网络请求（除静态资源本身）。

---

## 数据分析模块

页面：`/analysis.html`（实验页右上角与结果页都有入口）

**输入**：拖拽或选择多份文件，全部在浏览器本地计算，不上传。

- 本程序导出的 `CSV` / `JSON`
- 原版程序的 CSV（`subject_id, block, angle, char_type, rt, correct`）——自动识别 `block 0 = 练习`
- 一个文件多人、多人多文件均可，按 `subject_id` 自动合并

**清洗口径**（页面上可改，改完立即重算）：仅正式阶段 → 去掉超时与错误 → RT ∈ [200, 3000] ms → 剔除每名被试 ±3SD。每一步剔除了多少试次都会列出来。

**统计内容**

| 模块 | 方法 |
| --- | --- |
| 群体反应时曲线 | 被试条件均值的 M ± SE，带误差线 |
| 重复测量 ANOVA | 2（类型）× 6（角度），三个效应各用「效应 × 被试」误差项，报告 F、df、p、偏 η² |
| 心理旋转速度 | 逐被试 0°–180° 线性回归，速度 = 1000 / 斜率，报告 M、SD、中位数、范围与分布直方图 |
| 镜像成本 | 反写 − 正写，被试内配对 t 检验 + Cohen's dz |
| 对称性 | 60° vs 300°、120° vs 240°，配对 t 检验 |
| 线性 vs 二次 | 0°–180° 条件均值 OLS，比较 R²、AIC、二次项系数 t |
| ex-Gaussian | 逐被试、逐条件 MLE（Nelder–Mead 多起点）；另给合并拟合参考值；对 μ、τ 做配对 t 检验 |

**导出**：条件汇总 CSV、被试汇总 CSV、Markdown 报告（可直接粘进课程报告）。

**统计实现与局限**（写在页面注释里，也在这里说明）

- 全部零依赖手写：不完全 beta 函数求 t/F 的 p 值，OLS 用正规方程 + Gauss–Jordan 求逆。
- ANOVA 未做 Greenhouse–Geisser 校正，也未检验球形性；角度水平多时建议用 R `afex::aov_ez` 复核。
- ex-Gaussian 拟合前会先对「角度 × 类型」单元格中心化，否则 0°–300° 的系统差异会被误读为长尾（σ、τ 虚高）。每条件约 120 试次时 τ 会低估 15–25%，**条件间差异比绝对值可靠**。
- 多个配对检验未做多重比较校正。
- 页面提供「示例数据（24 人 · 模拟）」，可先看输出形态；示例按 Cooper & Shepard 的典型结果生成，不是真实被试。

## 部署（GitHub Pages）

```bash
git clone https://github.com/Clonedman9/mental-rotation-experiment.git
cd mental-rotation-experiment
# 修改 index.html 后：
git add -A && git commit -m "update" && git push
```

仓库已开启 Pages：`Settings → Pages → Deploy from a branch → main / (root)`。仓库根目录含 `.nojekyll`，避免 Jekyll 处理静态资源。

---

## 许可与致谢

- 代码：MIT License。
- 刺激字形轮廓来自 DejaVu Sans Bold（Bitstream Vera / Public Domain 衍生许可）。
- 实验范式：Cooper, L. A., & Shepard, R. N. (1973). Chronometric studies of the rotation of mental images. In *Visual Information Processing* (pp. 75–176). Academic Press.
- 原始实验逻辑与报告：[@Yideal](https://github.com/Yideal) 等的课程作业。
