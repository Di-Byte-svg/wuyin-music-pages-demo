# 个性化五音养生音乐生成与评估系统 · 学术实验网站（v3.1）

本目录是硕士课题《基于人体生物信号与中医辨证的养生音乐 AIGC 系统设计与实现》的
**被试实验入口 + 研究者计算工具**，纯静态单文件（`index.html`），零外网依赖、可双击离线运行，
对应课题执行计划书的**模块 2（Theory Mapper）前端可复现版**与**模块 4（闭环评估）数据采集端**。

---

## 一、与导师计划书「四大模块」的对应关系（汇报口径）

| 计划书模块 | 主要实现位置 | 本网页承担的部分 |
|---|---|---|
| 模块 1 数据源（DEAP / PhysioNet：HR、HRV、血压、血氧、皮电；中医体质/脏腑/时辰标签） | Python `src/data_loader.py` | 研究者面板 Theory Mapper 提供**录入界面与归一化基准**（PhysioNet 成人静息口径，见 `CONFIG.physioBase`）；被试端走 SAM 主观 V/A 路线 |
| 模块 2 数字化中医五音映射 Theory Mapper | Python `src/theory_mapper.py` | **网页内置同方程的 JS 参考实现**，逐步显示 z 分数→激活/效价指数→五音/BPM 控制向量，保证论文公式可复现、可被审稿人当场验算 |
| 模块 3 AIGC 生成（Music Transformer/VAE→MIDI→SoundFont；或 MusicGen/AudioLDM） | Python `src/generator.py` | 双引擎：`audio/` 下**预生成真音频优先**；缺失时用 Web Audio 五声调式离线合成兜底（`audio_source` 字段区分，正式分析只取 musicgen） |
| 模块 4 闭环评估（客观纯净度/节奏稳定性；主观 SAM、STAI-S、POMS；听前后生理；配对检验） | Python `src/evaluator.py` + SPSS | 网页负责**全部主观量表与生理补录的采集、CSV 宽格式导出**，并对合成材料试算客观指标；真音频客观指标由 `evaluator.py`（librosa）按同口径计算 |

**两条映射路线在五音空间汇合，即小论文的「多模态融合」点：**
- 被试端（情绪路线）：主观状态 → V/A（效价/唤醒）→ 五音；
- 研究者端（生理路线）：HR/HRV/EDA/血压/血氧 + 中医辨证 → A_phys/V_phys → 五音。

---

## 二、建议的仓库结构（对齐导师 wellness-music-aigc 规范）

```
wellness-music-aigc/
├─ src/
│  ├─ data_loader.py      # 模块1：DEAP s01.dat / PhysioNet(MIT-BIH,MIMIC) 读取、HRV/血压周期/血氧特征
│  ├─ theory_mapper.py    # 模块2：与本网页 CONFIG 同构的生理→五音映射方程（单一事实来源）
│  ├─ generator.py        # 模块3：MusicGen / Music Transformer 生成，输出 120s（约2分钟）五音材料到 audio/
│  └─ evaluator.py        # 模块4：librosa 计算五声纯净度、节奏CV；读 CSV 做配对 t / Wilcoxon
├─ experiment_web/        # ← 本目录整体放这里
│  ├─ index.html
│  ├─ audio/              # 预生成材料（见第四节）
│  └─ README.md
├─ results/
│  ├─ generated_samples/  # 3–5 首代表音频
│  └─ hrv_analysis.png
├─ README.md
└─ .gitignore
```

---

## 三、网页功能清单（v3.0）

**被试端（七步，被试内前后测）**
1. 知情同意（勾选才可开始）；2. 匿名登记（编号/年龄/性别/音乐训练）；
3. 听前基线：SAM（V/A/D 9 点，自绘图形规避官方 SAM 版权）+ STAI-S 状态焦虑 20 题（自动反向计分）；
4. 状态与目标选择，展示完整推理链；
5. **强制聆听约 2 分钟（120 秒）**（墙钟累计、防快进、听满才解锁，可重复播放并计数，时长由 `CONFIG.LISTEN_SECONDS` 一处控制）；
6. 听后同套 SAM + STAI-S；7. 音乐评价（喜好/匹配/舒适/熟悉 各 9 点）+ 开放备注；
8. 完成页：主试可补录听前后 HRV/血压/皮电，按同一 trial 对齐；下载单条 JSON。

**研究者面板**
- 情绪路线全部映射规则表（可复现）；
- **Theory Mapper 计算器**：输入 6 项生理指标 + 体质/脏腑/时辰，输出逐步计算过程与五音控制向量；
  - 连续层：z 标准化 → `A_phys=5+2(0.5z_HR+0.3z_EDA+0.2z_HRV)`，血压/血氧/辨证修正 V_phys；
  - 规则层（优先，导师指定）：阳虚/心率缓/低血压→**徵音火、高 BPM 补心阳**；肝郁/高压/压力→**角音木、中速丝竹疏肝**；脏腑-五音（肝角/心徵/脾宫/肺商/肾羽）；
- V/A 直演（CNN-LSTM 模型输出 V/A 后的接入验证点）；
- 客观指标试算：五声音阶纯净度、音符起始间隔变异系数 CV（节奏稳定性）；
- 量表开关（STAI-S 可关；POMS 留授权接口，默认不内置题目）；
- 数据汇总：导出 SPSS 直读 CSV（UTF-8 BOM、一行一 trial、52 字段）/ 全部 JSON / 清空。

**实验参数（写死在 CONFIG，便于论文方法学描述）**：每首聆听 120 s（约 2 分钟，改 `LISTEN_SECONDS` 一处即可调整）；基础版只做 matched 条件（预留 `condition` 字段供对照）；默认 1 轮、可加轮；显著性 p<0.05，前后差值用配对样本 t 检验（正态）或 Wilcoxon（非正态）。

---

## 四、audio/ 音频材料制备规范（阶段 7，本地 MusicGen 完成）

网页按 `audio/{五音拼音}_{BPM}.mp3` 探测，未命中再自动探测同名 `.wav`，两者命中任一即用真音频、否则网页合成兜底（v3.2 起同时支持 mp3/wav）。共需 **15 个文件**：

| 五音 | 拼音 | 55 BPM | 72 BPM | 88 BPM |
|---|---|---|---|---|
| 宫(土/脾) | gong | gong_55.wav | gong_72.wav | gong_88.wav |
| 商(金/肺) | shang | shang_55.wav | … | … |
| 角(木/肝) | jue | jue_55.wav | jue_72.wav | jue_88.wav |
| 徵(火/心) | zhi | zhi_55.wav | zhi_72.wav | zhi_88.wav |
| 羽(水/肾) | yu | yu_55.wav | yu_72.wav | yu_88.wav |

要求：时长 **120 s（约 2 分钟，与 `LISTEN_SECONDS` 一致）**、等响度 **-16 LUFS**、**无歌词无人声**、中国五声调式、丝竹/古筝/竹笛类音色、首尾淡入淡出。格式为 MusicGen 原生 **32 kHz WAV**（脚本默认，浏览器与 SPSS 均可读）；若本机 libsndfile 支持，脚本会额外导出同名 MP3，二选一放入 `audio/` 即可。

**一键制备脚本 `generate_wuyin_materials.py`**（离线加载本地 `musicgen-small`，自动完成生成→精确裁到 120 s→淡入淡出→-16 LUFS 等响度→按上表命名→写 `materials_manifest.csv` 溯源清单，支持断点续跑、固定随机种子可复现）：

```bash
# cuda113 环境（RTX A5000），在脚本所在目录执行
python generate_wuyin_materials.py --selftest   # 先跑4秒自检，确认环境
python generate_wuyin_materials.py              # 正式生成15首×120s（已存在自动跳过，可中断续跑）
python generate_wuyin_materials.py --only jue   # 只生成某一音（3首），便于先试听调提示词
```

生成后把 15 个音频（及清单）拷入 `website/audio/` 随仓库一起发布；再用 `evaluator.py`（librosa）复核纯净度与节奏稳定性，填入论文客观结果表。

---

## 五、部署（面向国内被试，零外网依赖）

- **最简**：把本目录整体拷给被试，双击 `index.html` 即可（无需联网、无任何 CDN/字体外链）；
- **线上**：推荐腾讯云 COS 默认域名静态托管（默认域名免备案）；GitHub Pages 仅作备选；
- **数据回收**：被试下载单条 JSON 交主试，或主试在同一台机器汇总导出 CSV；如需线上汇总可改用腾讯文档/金山表单收 JSON，不在网页内嵌第三方统计脚本。

---

## 六、量表版权与学术合规

- SAM 人形图为 Bradley & Lang (1994) 版权图形，本网页用**自绘等价图形**替代，论文中注明；
- STAI-S（Spielberger）为标准量表，正式研究需取得使用授权，题目与反向计分题号（1,2,5,8,10,11,15,16,19,20）已内置；
- POMS 简式同理，取得授权后将题目填入 `CONFIG.poms` 并打开开关，未授权前不内置具体题面。

---

## 七、CSV 字段（52 列，SPSS 宽格式）

`trial_id, subject_id, age, gender, music_training, trial_no, condition, mapper_route, ts_start, ts_end,`
`pre_valence, pre_arousal, pre_dominance, stai_pre_total, state_input, goal_input, activity_input,`
`va_initial_v, va_initial_a, va_adjusted_v, va_adjusted_a, five_tone, element, organ, bpm,`
`audio_source, audio_file, listen_seconds, listen_completed, play_count,`
`post_valence, post_arousal, post_dominance, stai_post_total, delta_valence, delta_arousal, delta_dominance, stai_delta,`
`pre_hrv, post_hrv, pre_sbp, post_sbp, pre_dbp, post_dbp, pre_eda, post_eda,`
`liking, fit, comfort, familiarity, comment, app_version`

- `five_tone` 存拼音（gong/shang/jue/zhi/yu），`element` 存英文（earth/metal/wood/fire/water），选择类题面存中文；
- 正式假设检验只纳入 `audio_source=musicgen` 的记录；`listen_completed=1` 为有效 trial。
