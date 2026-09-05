# RoboMemo @ AlxOrigin Summit —— 中期报告（9/5–9/6）

**Raw Data in, Robot-Ready Out.**

> 官网：**[robomemo.ai](https://robomemo.ai/)** ｜ 主项目见 [`RoboMemo/robomemo-core`](https://github.com/RoboMemo/robomemo-core)：具身智能数据基础设施——把人类演示视频变成机器人可训练数据集（π0 / GR00T / UniVLA 兼容）。
> 本仓库是 **AlxOrigin Summit（9/5–9/6）的中期进展报告**， Summit 期间主攻方向：**三阶段数据管线 + 定量评审体系**——让"这条数据值不值得入库、值不值得训、训出来打不打得过"变成有公式、有阈值、有 gate 的自动判定。

---

## 系统全景

```
┌────────────────────────────────────────────────────────────────────┐
│                        数据入口（多源）                             │
│   佩戴设备(4×相机+IMU) │ 第三人称视频 │ Ego2L X5 产线 │ ReViV 动捕   │
│   Status: ✅ 全部接入                                                │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│              三阶段管线（pipeline3stages 框架）                      │
│                                                                    │
│  ① 筛选&路由 ──▶ ② 标注&转换 ──▶ ③ 打包&检查                       │
│  search/route      pose/VLM标注      export LeRobot V2.1           │
│  upload vs ingest  SMPLify/ReViV     gen modality.json             │
│  retarget G1/A5                                    │               │
│                                                    ▼               │
│  ┌──────────────── 定量评审赛制 ─────────────────┐                 │
│  │ L0 格式入场券（validate_dataset，一票否决）    │                 │
│  │ 初赛 L1 数据质量：RMSE/DTW/物理可行性 → scorecard│   Status: 🏗️  │
│  │ 复赛 L2 部署性能：帧率/延迟/稳定/流畅/泛化      │（框架已定，     │
│  │ 总决赛 L3 任务效果：VLM judge + 鲁棒性 + 创新   │  L1 实现中）    │
│  └────────────────────────────────────────────────┘                │
└──────────────────────────────┬─────────────────────────────────────┘
                               ▼
┌────────────────────────────────────────────────────────────────────┐
│   机器人可训练数据集（LeRobot V2.1 + scorecard.json 评分证据）       │
│   Status: ✅ 导出链路通 / 🔧 评分卡落地中                            │
└────────────────────────────────────────────────────────────────────┘
```

---

## ✅ 已完成的部分

### 1. 三阶段框架与定量评审体系——设计定稿
- 管线收敛为 **筛选&路由 → 标注&转换 → 打包&检查**，与现有 11 节点 DAG 逐一映射完毕；
- 评审赛制定稿：**L0 格式入场券（一票否决）→ 初赛 L1 数据质量（100 分制：任务空间 RMSE + DTW 相似度 + 物理可行性加权，≥75 入库 / 60–75 人工复核 / <60 打回）→ 复赛 L2 部署性能 → 总决赛 L3 任务效果**；
- 三轮共用统一 `scorecard.json` schema（value / score / threshold / verdict / evidence），评分过程产出证据而非只给一个数；
- 设计文档已评审通过，拆出 P0–P3 落地清单。

### 2. 主线合并——两条产线两线合一（PR #80 / #86）
- live 树全量合入 `main`：**Ego2L X5 产线 + Teleport 实时/回放 + Console 对等功能**，结束长期双线并行；
- Ego2L 本地录制链路合入，佩戴设备直采可直接进管线。

### 3. Agent 平台稳定化
- 修复**多会话并发卡死**（并发 session 抢占导致死锁）；
- 新增 **Azure OpenAI provider**（gpt-5.4），推理与自动标注的模型可切换；
- 新增数据集 **zip 流式下载 API**（`/api/agent/jobs/{id}/download`）。

### 4. Console（前端）体验
- 聊天记录持久化（localStorage + 运行时 shape 校验），切页/刷新不丢会话；
- 修复**中文任务文本上传事故**（详见技术难点 #4），非 ASCII 单文件上传改走 batch 通道。

### 5. 采集链路鲁棒性
- 板子中途掉线不再挂死录制锁（finalize/scp 全部带超时与未捕获异常兜底）；
- ReViV 第一人称 egocentric pose 后端集成完毕。

### 6. CI 全绿
- Python Agent job 补装 `opencv-python-headless`，全分支 CI 红清零。

---

## 🔧 剩余待完成的部分及计划（9/6）

| 优先级 | 事项 | 说明 | 状态 |
|---|---|---|---|
| **P0** | `check_quality_l1` 工具实现 | 读 `g1_motion.pkl` + `gmr_report.json` + `ref_keypoints.npz`，FK + DTW 纯 numpy 实现，产出 scorecard.json；输入已齐备，可当天落地 | 🏗️ 进行中 |
| **P0** | `run_gmr_on_fit.py` 落盘 `ref_keypoints.npz` | retarget 时顺手保存对齐后参考关键点（双腕/双肘/双肩，身高归一化） | 计划 9/6 上午 |
| **P1** | DAG / Console 接线 | pipeline 视图加 L1 节点（retarget 后预检 + export 后终检各一次），verdict 进 orchestrator gate | 计划 9/6 |
| **P1** | scorecard 入 job 记录 | 评分历史可追溯 | 计划 9/6 |
| **P1** | 阈值 golden 集标定 | 用 346 真实帧 golden 集跑分布，定 R₀ / τ / R_hard；阈值全部进 config | 计划 9/6 |
| **P1** | 端到端演示准备 | 视频 → 管线 → LeRobot 数据集 + scorecard 全链路 live demo | 计划 9/6 |
| P2 | L2 部署性能 harness | 仿真/真机 run log 采集 + 离线算分（依赖部署侧 runner） | 后置 |
| P3 | L3 VLM judge | 阶段化 checklist 判分 + 遮挡/变速注入器 | 后置 |

**需要拍板的待定项**：初赛两口径权重（当前 0.40/0.35/0.25）、复赛 FPS_target（30 vs 50 Hz）、G1 速度上限数值来源、创新性"新颖度"用帧嵌入还是动作嵌入。

### 🎯 下一阶段目标（采集 × 感知 × 重定向）

| # | 目标 | 现状 → 目标 |
|---|---|---|
| 1 | **Ego2L / Ego4 本地录制** | Ego2L（双目 LiDAR）与 Ego4（四目）目前**仅支持串流**，录制链路待打通——能落盘才能量产进管线 |
| 2 | **第一人称 / 第三人称区分** | 管线暂不区分视角；计划在①筛选&路由阶段显式标记视角，作为骨架检测后端路由的前置 |
| 3 | **全身+双手骨架检测——按视角路由后端** | 第一人称单目 → **ReViV**（已集成）；第三人称 → **WHAM**（待接入，路线已明确，见下）；第一/第三混合 → **EgoExo**（待接入） |
| 4 | **Retarget 双后端测试** | **XRobokot** 与 **GMR** 均已部署、**尚未测试**——待用 golden 集验证，并接入 L1 评分 gate |

### 补充：第三人称技术路线——WHAM → GMR → MuJoCo（8 阶段端到端）

核心思路是**复用预训练模型、对新视频零训练推理**：预训练 WHAM 人体运动估计 + 基于约束的 GMR 重定向 + MuJoCo 仿真验证，而**不是**针对每段视频重训一个端到端模仿学习模型。

| 阶段 | 模块 | 输入 → 输出 |
|---|---|---|
| 1 视频准备 | `examples/*.mp4` | 单人全身 RGB 视频（太极、演讲等）→ 帧序列 |
| 2 人体运动估计 | **WHAM** | 视频帧 → 人体 3D 姿态 / 根节点运动 / SMPL 运动序列 |
| 3 流式动作传递 | `handle_wham_gmr.py` | WHAM 动作流 → `npz_stream` 中间数据 |
| 4 坐标系统一 | `yup_to_zup` | Y-up 人体坐标 → MuJoCo/GMR 的 Z-up 约定 |
| 5 动作重定向 | GMR / IK / 关节映射 | 人体关节 + 根轨迹 → `unitree_g1` 关节轨迹（限位与结构约束下） |
| 6 轨迹后处理 | `smooth_alpha=0.35`、`height_adjust` | 原始轨迹 → 降抖动、改善脚部接触与整体高度 |
| 7 仿真回放 | MuJoCo Viewer / 录制 | G1 关节轨迹 → 实时窗口 + MP4 |
| 8 数据导出 | CSV / PKL | 最终结果 → CSV + PKL + MP4，供分析 / 训练 / 复用对比 |

```
单目视频 → WHAM（检测/跟踪/3D 动作估计）→ SMPL 动作流 → Y-up→Z-up
        → GMR（人体→G1 关节重定向）→ 平滑/身高调整/根节点处理
        → MuJoCo 仿真回放与录制 → CSV + PKL + MP4
```

工程现状：**WHAM→GMR→MuJoCo 链路已在 GPU 上跑通**（见 `demo/demo1.mp4` / `demo2` / `demo3.mp4` 实测回放），已为不同视频集建立独立运行入口（复制 `.bat` 模板 → 设 `VIDEO` 与 `OUTPUT_ROOT`），避免输出互相覆盖。剩余工作：**迁移到 RoboMemoClaw 的 A100 后端，并接入 agent 的 tool 体系**（注册为 pipeline tool 后即可被一键 orchestrator 与 LLM ReAct 两条路径调用）。

---

## ⚠️ 遇到的技术难点

**1. 单目深度模糊（无标定 transl 近似）**
单目视频没有标定，SMPLify 的平移解存在 scale/depth 歧义，只能取近似值；重投影阈值 `p95 ≤ 0.15` 帧边长本质是经验值。
→ 缓解：相似度评分改用**任务空间关键点 RMSE**（根对齐 + 身高缩放）为主口径，弱化绝对平移依赖。遗留：待 SMPLer-X + MANO 介入。

**2. 夹爪通道占位**
手部可见率仅 ~36%，MediaPipe 手部关键点不稳，`export_lerobot` 的 gripper 值目前固定 0.5 占位。
→ 处置：导出元数据中显式标注 known-gap，不静默。遗留：MANO 手部拟合补真值。

**3. 无灵巧手硬件，手部 Retarget 无法闭环验证**
G1 侧没有灵巧手，手部 retarget 算法的输出**没有任何执行端可以消费**——重定向对不对、是否真正支持灵巧操作，既不能用仿真回放确认（缺手部模型对标），也没有真机反馈，只能停在算法层。
→ 影响：手部动作质量无法量化，L1 评分 gate 对手部通道暂时失明。
→ 缓解：身体通道（29 DoF）先按现有链路验证；手部先做离线可视化（MANO 关键点 vs 重定向输出叠加）做定性校验，等灵巧手到位后回补真值。

**4. 并发 / 竞态类故障（9/4–9/5 连爆三例）**
多会话并发死锁、uvloop 对已退出进程 `kill()` 抛 `ProcessLookupError`、板子中途掉线挂死录制锁。
→ 教训已固化为规范：**所有外部资源（子进程 / SSH / 板载连接）必须带超时 + 幂等清理**，异常路径不允许裸 await。

**5. HTTP header 非 ASCII 事故（9/5 15:32）**
中文任务文本放进 `X-Task` header → 浏览器 `fetch` 直接 TypeError，上传链路中断。
→ 修复：header 只承载 ASCII id，业务载荷全部走 JSON body（单文件上传改走 batch 通道）。
→ 教训：**header 永远不传业务文本**，多语言内容一律 body 化。

**6. 定量评审缺乏 ground truth 分界**
三阶段框架最大的不确定性：R₀ / τ 等阈值没有现成真值，定松了放进坏数据、定紧了误杀好数据。
→ 缓解：golden 集（346 真实帧 + 近期 job 产物）跑分布，阈值定在"好样本明显通过、已知坏样本明显打回"的分界；阈值全部进 config 不改代码可调；60–75 灰区一律 `ask_human` 人工复核兜底。

**7. canonical 中间产物单 job 限制**
`extract_pose` / `fit_smplify` 写 hardcoded 路径，同一时刻只能支持一个活跃 job，演示时无法多任务并行。
→ 计划 9/6 演示脚本按单任务串行规避；workdir 参数化列入后续重构。

---

*Maintained by RoboMemo team · 中期节点 2026-09-05 · 终期报告随 9/6 演示更新*
