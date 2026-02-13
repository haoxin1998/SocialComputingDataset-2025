# SocialComputingDataset-2025
# V2 US-Teen 候选集（strict / balanced / recall）构建说明

本 README 解释以下文件如何从头构建、以及它们之间的关系：

- 原始数据：`/data/SocialComputing-Dataset/tiktok_merged_data_deduplicated.csv`
- V2 输出（三档 US-Teen 候选）：
  - `/data/SocialComputing-Dataset/Analysis/outputs_v2/us_teen_v2_strict.csv`
  - `/data/SocialComputing-Dataset/Analysis/outputs_v2/us_teen_v2_balanced.csv`
  - `/data/SocialComputing-Dataset/Analysis/outputs_v2/us_teen_v2_recall.csv`

> 重要限制：原始数据不包含“年龄/国家/地区”等人口统计字段，因此 **US-Teen 只能做启发式/弱监督候选集**。这些 CSV 是“待核验候选”，不是 100% 真值。

---

## 0. 原始数据是什么？

`tiktok_merged_data_deduplicated.csv` 是按“视频”粒度的表，核心字段包括：

- `description`：视频描述文本
- `hashtags`：标签（逗号分隔字符串）
- `author`：作者账号名
- `likes/comments/shares/plays`：互动与播放
- `create_time`、`video_url` 等

---

## 1. teen_candidates=257 是如何来的？

V1 脚本：`/data/SocialComputing-Dataset/Analysis/teen_extract_analyze.py`

它对每条视频计算 `teen_score`（只用 `description + hashtags` 的文本/标签线索），并产生布尔标记：

- `is_teen_candidate = (teen_score >= teen_threshold)`  

本次运行使用默认参数（你当前的 `outputs/` 产物）：

- `teen_min=13, teen_max=17`
- `teen_threshold=2`

因此：

- `outputs/labeled_videos.csv`：全量 7225 行 + 规则打分列（`teen_score/teen_reasons/...`）
- `outputs/teen_candidates.csv`：从 `labeled_videos.csv` 过滤 `is_teen_candidate=True` 得到 **257 行**

### teen_score 的构成（简述）

`teen_score` 来自几类信号（命中会累加）：

1) **年龄表达（强信号）**：如 “I’m 15 / as a 16 year old / 16 years old”等（只计 13–17）  
2) **高中/校园高精度关键词（强/中信号）**：如 `highschool / prom / homecoming / freshman / sophomore / junior year / senior year / class of 20xx`  
3) **校园低精度关键词（弱信号）**：如 `student / teacher / homework / gpa`（可能带来误报）
4) **标签子串**：如 `#prom2025/#classof2025/#highschool` 等

输出中的 `teen_reasons` 记录了命中的规则，便于人工核验。

---

## 2. V2 如何从 teen_candidates 里识别 “US”？

V2 脚本：`/data/SocialComputing-Dataset/Analysis/us_teen_pipeline_v2.py`

它读取 V1 的 `outputs/labeled_videos.csv`，在其基础上新增 US 相关字段（写入 `outputs_v2/labeled_v2.csv`），主要有三类 US 证据：

### 2.1 规则（可解释）：us_geo_score_v2 / us_inst_score_v2

- `us_geo_score_v2`（地理/国家强证据）：
  - `usa / united states / american / 🇺🇸`
  - 州名/城市名（如 `texas / florida / chicago / nashville`）
  - hashtag 中相对“稳”的州缩写（如 `tx/ny/ca/...`）
- `us_inst_score_v2`（机构/体系弱证据）：
  - 体育联盟/体系：`nfl/nba/mlb/wnba/nhl/ncaa`
  - 少量品牌/机构：`walmart/target/costco/...`

并记录 `us_reasons_v2`（命中原因串）用于人工核验。

### 2.2 弱监督文本模型：us_ml_prob_v2

因为很多视频不会直接写 “USA/纽约/加州”，V2 用规则生成 pseudo label 来训练一个 US 文本分类器：

- 正样本：`us_geo_score_v2 >= 4`（强地理/国家证据）
- 负样本：`us_geo_score_v2==0 且 us_inst_score_v2==0`

模型：`char 3–5gram TF-IDF + LogisticRegression`  
输出：每条视频一个概率 `us_ml_prob_v2`（越大越像 US）。

### 2.3 相似度扩张：us_sim_to_strict_seed_v2

在 teen 子集内，以 “strict US teen” 作为种子，计算其他 teen 与种子中心的 TF‑IDF 余弦相似度，用于补漏（同一内容群但没写地名）。

---

## 3. strict / balanced / recall 三个文件的含义

这三个文件都是从 `outputs_v2/labeled_v2.csv` 过滤得到的 **US-Teen 候选集**，只是在“US 判定”上严格程度不同。

共同前提：都要求 `is_teen_candidate=True`（即来自那 257 条 teen_candidates）。

### 3.1 strict（最干净，偏高精度）

文件：`outputs_v2/us_teen_v2_strict.csv`  
定义（核心）：  

- `US-Teen strict := teen_candidate AND (us_geo_score_v2 >= 4)`

直观含义：文本/标签里出现明确的美国国家/地理强证据才算 US。

### 3.2 balanced（折中，推荐做主分析）

文件：`outputs_v2/us_teen_v2_balanced.csv`  
定义（核心）：

- `is_us_v2_balanced := (geo>=2) OR (geo>=1 & inst>=1) OR (inst>=2)`
- `US-Teen balanced := teen_candidate AND (is_us_v2_balanced OR (us_ml_prob_v2>=0.35 AND us_inst_score_v2>=1))`

直观含义：允许“中等地理证据”或“机构+少量地理”的组合；必要时让 ML 在“有一点 US 体系信号(inst)”的前提下补漏。

### 3.3 recall（最大候选池，偏高召回）

文件：`outputs_v2/us_teen_v2_recall.csv`  
定义（核心）：

- `US-Teen recall := teen_candidate AND (is_us_v2_balanced OR us_ml_prob_v2>=0.2 OR us_sim_to_strict_seed_v2>=0.25)`

直观含义：只要规则/ML/相似度任一认为“像 US”，就放进候选池。**规模大，但误报风险也最高**。

### 3.4 三者关系（包含关系）

在当前实现下：  

`strict ⊆ balanced ⊆ recall`

---

## 4. 从头构建（可复现命令）

> 使用 `llama_factory` 环境（推荐用 conda run，避免交互式 shell 差异）

### Step A：从原始数据生成 teen_candidates（V1）

```bash
source /data/conda/bin/activate
cd /data/SocialComputing-Dataset/Analysis

conda run -n llama_factory python teen_extract_analyze.py \
  --input ../tiktok_merged_data_deduplicated.csv \
  --outdir outputs
```

关键产物：

- `outputs/labeled_videos.csv`（全量打标）
- `outputs/teen_candidates.csv`（257 行：`is_teen_candidate=True`）

### Step B：在 teen_candidates 基础上构建 V2 US-Teen 三档候选

```bash
source /data/conda/bin/activate
cd /data/SocialComputing-Dataset/Analysis

conda run -n llama_factory python us_teen_pipeline_v2.py \
  --labeled-v1 outputs/labeled_videos.csv \
  --v1-us-teen outputs/us_teen_candidates.csv \
  --outdir outputs_v2
```

关键产物：

- `outputs_v2/labeled_v2.csv`（V1 labeled + V2 US 字段）
- `outputs_v2/us_teen_v2_strict.csv`
- `outputs_v2/us_teen_v2_balanced.csv`
- `outputs_v2/us_teen_v2_recall.csv`
- `outputs_v2/compare_report.md`（与 V1 的规模/统计对比）

---

## 5. 如何评估“有效性”（建议）

因为没有真实 US 标签，建议对“V2 新增部分”做抽样人工核验：

- `outputs_v2/validity_sample_v2.csv`：已为你抽样（只看新增更省时间）

你标注 `human_is_us / human_is_teen` 后，即可计算 precision 并决定最终用 strict / balanced / recall 哪一档做研究主结论（并可在论文里做稳健性检验）。

