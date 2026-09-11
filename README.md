# 个性化五音养生音乐生成与评估系统 · 学术实验网站（v4.1 双端科研版）

本目录是硕士课题《基于人体生物信号与中医辨证的养生音乐 AIGC 系统设计与实现》的
**被试实验端 + 研究者技术端一体化系统**，纯静态单文件（`index.html`），零外网依赖、无 CDN/外链字体，
可双击离线运行，也可整体托管到 GitHub Pages / 对象存储后用链接发给被试与导师。

v4.0 在 v3.5（纯被试端）基础上补齐两块科研短板：
1. **研究者技术端**：完整对应导师计划书「四大模块」，做到技术链透明、材料可溯源、统计可复算；
2. **客观生理指标**：被试内对照设计，用 Apple Watch Series 11 采集听前后 **HR + HRV(RMSSD)**，
   与主观 SAM / STAI-S 配对分析并可视化，避免评价全靠主观打分。

---

## 一、与导师计划书「四大模块」的对应（研究者端 6 个标签页）

| 计划书模块 | 网页标签页 | 实现内容 |
|---|---|---|
| 总览 | 总览/架构 | 端到端系统架构 SVG（数据获取→Theory Mapper→AIGC 生成→闭环评估）、技术栈 KPI、量表开关 |
| 模块 1 数据获取 | 模块 1 数据获取 | PhysioNet（MIT-BIH/MIMIC）成人静息口径表；中医体质(9)+脏腑(5)+时辰(11)=**25 维 One-hot 嵌入向量实时演示** |
| 模块 2 Theory Mapper | 模块 2 映射 | 生理(HR/HRV/EDA/BP/SpO₂)+辨证 → z 标准化 → A_phys/V_phys → 五音/BPM 控制向量，**逐步显示公式可当场验算**；附 V/A 直演与规则表 |
| 模块 3 AIGC 生成 | 模块 3 AIGC 材料库 | 路线 A（符号 Music Transformer/VAE→MIDI→SoundFont）vs 路线 B（波形 Audio Diffusion / **本研究采用的 MusicGen**）对比；**15 首材料档案表**：文件/五音/五行脏腑/BPM/音色/随机种子/时长/响度 LUFS/峰值/RMS/最长弱音/是否削波/完整生成 prompt，全部来自 `materials_manifest.csv` 与 `audio_qc_report.csv` |
| 模块 4 闭环评估 | 模块 4 评估看板 | 配对样本 t、Wilcoxon（小样本以此为准）、Cohen dz、Pearson/Spearman 相关；HR/HRV/STAI/V 前后对比 SVG 图、ΔHRV–ΔSTAI 相关散点；主客观相关；材料客观指标试算 |
| 数据 | 数据管理 | 在线飞书收集表对接、多文件 JSON/CSV 导入去重聚合、导出 SPSS 宽表 CSV/JSON、清空 |

---

## 二、实验设计（被试内两阶段对照，写死在 CONFIG 便于论文方法学描述）

- **两阶段**：每位被试完成两段，①个性化五音段 `matched`（MusicGen 真音频）②中性对照段 `control`；
- **顺序平衡（AB/BA）**：`orderForSubject(被试编号)` 按编号哈希奇偶分配 matched>control 或 control>matched，抵消顺序效应；
- **对照四型**（`CONFIG.controlType`，v4.1 起默认柔和棕噪声）：
  - `brown`（默认）：Web Audio 离线合成**棕噪声 Brownian（功率谱 1/f²，能量集中低频）**，深度低通 480 Hz + 缓慢“呼吸”起伏 + 4 s 首尾淡变 + 峰值压到 0.30（五音为 0.891），听感低沉最不刺耳；白噪声全频等功率、高频嘶声最重，比粉/棕噪更易致烦躁，故不采用；
  - `pink`：柔和粉红噪声（1/f，Voss-McCartney 近似 + 低通），保留为可选项；
  - `silent`：静坐静音对照（仅计时、不发声）；
  - `unmatched`：随机播放一首「非推荐调式」的真音频，控制音乐存在本身；
- **每段聆听 120 s（约 2 分钟）**，墙钟累计、防快进、听满才解锁，改 `LISTEN_SECONDS` 一处即可调整；
- **客观指标只采 HR + HRV(RMSSD)**（导师拍板，消费级 Apple Watch 无 EDA，血氧/血压不进被试端）：
  **单共同基线设计**——实验最开始测 1 次听前基线（静坐 3–5 min、同一 App、HRV 测满 1 min 取 RMSSD），两段听后各测 1 次，全程客观共 3 次；分析时 matchedΔ=听后五音−共同基线、controlΔ=听后对照−共同基线，配合 AB/BA 抵消顺序与残留；
- **主观量表**：SAM 效价/唤醒/支配（自绘图形规避官方版权）+ STAI-S 20 题（自动反向计分），
  **同样只在最开始测 1 次共同基线、两段听后各 1 套（全程主观共 3 次，减少重复填写疲劳）**；
  声音评价 4 项 9 点（喜好/匹配/舒适/熟悉）+ 开放备注；
- **统计口径**：p<0.05；差值正态用配对 t，否则 Wilcoxon；效应量 Cohen dz；主客观做相关；网页结果为现场速览，正式分析用 SPSS 复核，论文需写明 Apple Watch 为消费级设备的局限。

---

## 三、audio/ 音频材料（本地 MusicGen-small 批量生成，已随仓库发布）

网页按 `audio/{五音拼音}_{BPM}.mp3` 探测，命中即用 MusicGen 真音频（`audio_source=musicgen`），
缺失才回退 Web Audio 五声合成兜底（`audio_source=websynth`，正式分析不纳入）；对照段 `audio_source=control`。
共 **15 首**：宫 gong / 商 shang / 角 jue / 徵 zhi / 羽 yu × 55 / 72 / 88 BPM，
每首 120 s、-14 LUFS 等响度、峰值 0.891、零削波、无歌词无人声、丝竹/古筝/竹笛/古琴类音色。
母带与溯源清单在 `E:\zytai\ai_to_music\music_library\wuyin_five_tones\`
（15 wav + 15 mp3 + `materials_manifest.csv` + `audio_qc_report.csv`），生成脚本为 `generate_wuyin_materials.py`（固定随机种子可复现）。

---

## 四、在线数据自动收集（飞书多维表格表单，国内可达）

- 已创建本课题专用多维表格「五音养生AIGC实验数据库」与收集表，**互联网匿名可填、免登录、27 个字段与系统一一对应**，
  默认地址固化在 `CONFIG.feishuFormDefault`，线上版开箱即用；数据管理页可粘贴自己的表覆盖。
- 被试点完成页「提交到研究数据库」时，系统用 `?prefill_题目=值` 把本阶段全部字段自动预填，
  被试核对后点最后一次提交即可（飞书不允许跨域无感 POST，故保留这一次人工确认）；
  主试在多维表格实时看到所有记录，可直接导出 SPSS。表单题目名必须与 `FEISHU_Q` 的中文名完全一致。
- 双保险：每条记录同时存浏览器 localStorage，可下载 JSON；主试可用「多文件导入聚合」合并多台机器数据后统一导出。
- 国外表单后端（Formspree 等）因国内网络不可达已排除，故采用飞书。

---

## 五、部署（面向国内被试，零外网依赖）

- **线上（发给导师/被试）**：把本目录传到 GitHub 仓库根目录（`index.html`）与 `audio/`，GitHub Pages 托管；
  更新后等 1–2 分钟生效，破缓存给链接加 `?v=数字`。本机无 git，走 GitHub 网页端「Add file→Upload files」上传即可。
- **离线**：把本目录整体拷给被试，双击 `index.html` 即可（无需联网、无 CDN/外链字体）。
- 正式假设检验只纳入 `audio_source=musicgen` 且 `listen_completed=1` 的记录。

---

## 六、CSV 字段（SPSS 宽格式，以 `CONFIG.csvCols` 为唯一准，当前 53 列）

字段分五组：①登记与实验设计（subject_id/age/gender/music_training/phase_order/trial_no/condition/control_type）；
②客观生理（pre_hr_bpm/post_hr_bpm/pre_hrv_ms/post_hrv_ms/hr_delta/hrv_delta/device/hrv_metric）；
③主观量表（SAM 三维听前后、stai_pre/post_total 与 delta、liking/fit/comfort/familiarity/comment）；
④映射与材料（five_tone/element/organ/bpm/audio_source/audio_file/listen_seconds/listen_completed/play_count）；
⑤溯源（trial_id/时间戳/app_version）。`five_tone` 存拼音、选择类题面存中文；逐列名称与顺序以 `index.html` 中 `CONFIG.csvCols` 为准。

---

## 七、量表版权与学术合规

- SAM 人形图为 Bradley & Lang (1994) 版权图形，本网页用自绘等价图形替代，论文中注明；
- STAI-S（Spielberger）为标准量表，正式研究需取得使用授权，题目与反向计分题号已内置；
- 客观生理用消费级智能手表（Apple Watch Series 11，HRV 取 RMSSD），论文需写明其与医疗级设备的精度差异、无 EDA 的局限。
