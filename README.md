# SocialComputingDataset-2025
**新版(V2)识别逻辑（研究模式解释）**  
V2 不改动“青少年”抽取（仍用 V1 的 `teen_candidates=257`），只升级“是否美国(US)”识别，并输出三档 `US-Teen` 候选集（strict/balanced/recall）。代码在 `SocialComputing-Dataset/Analysis/us_teen_pipeline_v2.py:1`，本次运行报告在 `SocialComputing-Dataset/Analysis/outputs_v2/compare_report.md:1`。

1) **规则打分（可解释）**：对每条视频从 `description + hashtags + author` 里提取美国信号，得到  
- `us_geo_score_v2`：国家/州/城市/州缩写等地理信号（例如 `usa/🇺🇸/united states`、州名、城市名、`tx/ny/ca` 等 hashtag）  
- `us_inst_score_v2`：机构/体系/品牌弱信号（例如 `nfl/nba/mlb/ncaa`、`walmart/target/costco` 等）  
并记录 `us_reasons_v2` 便于核验。

2) **规则 US 判定（两档）**  
- `is_us_v2_strict = (us_geo_score_v2 >= 4)`：只认“强地理/国家级”证据  
- `is_us_v2_balanced = (geo>=2) OR (geo>=1 & inst>=1) OR (inst>=2)`：允许“中等地理”或“机构+少量地理”的组合

3) **弱监督 US 文本模型（提升召回）**  
- 用规则生成 pseudo label：`geo>=4` 当作 US 正样本；`geo==0 且 inst==0` 当作非 US 负样本  
- 训练 `char 3–5gram TF-IDF + LogisticRegression`，得到每条视频的 `us_ml_prob_v2`

4) **相似度扩张（补漏）**  
- 在 teen 子集里，用“strict US teen”作种子，计算到种子中心的 TF-IDF 余弦相似度 `us_sim_to_strict_seed_v2`，阈值（本次）`>=0.25` 视为相似。

---

**strict=17 / balanced=25 / recall=138 的具体含义**  
这三个数字都是“在 V1 的 257 条 teen_candidates 中，被判为 US 的条数”，只是使用的 US 判定强弱不同：

- `strict = 17`：`teen AND is_us_v2_strict`（强地理/国家证据才算 US）  
  - 占 teen_candidates：`17/257 = 6.61%`；占全量：`17/7225 = 0.235%`  
  - 适合做“高精度小样本”主分析或对照。

- `balanced = 25`：`teen AND (is_us_v2_balanced OR (us_ml_prob_v2>=0.35 AND us_inst_score_v2>=1))`  
  - 占 teen_candidates：`25/257 = 9.73%`；占全量：`25/7225 = 0.346%`  
  - 本次结果里，`balanced` 的 25 条**全部来自规则增强**（ML 子句没有额外新增），更可解释、噪声相对可控。

- `recall = 138`：`teen AND (is_us_v2_balanced OR us_ml_prob_v2>=0.2 OR similarity>=0.25)`  
  - 占 teen_candidates：`138/257 = 53.70%`；占全量：`138/7225 = 1.910%`  
  - 这是“撒大网”的候选池：本次 `recall` 里 **110 条几乎是仅靠 ML 阈值 0.2 进入**，地理证据平均很弱（`us_geo_score_v2` 均值约 0.78），所以更适合“人工核验/半自动清洗”，不建议直接当作高置信 US 数据集使用。

（这些规模与对比表都在 `SocialComputing-Dataset/Analysis/outputs_v2/compare_report.md:1`；阈值敏感性在 `SocialComputing-Dataset/Analysis/outputs_v2/ml_threshold_sweep.csv:1`。）

**检验建议（有效性）**  
- 用 `SocialComputing-Dataset/Analysis/outputs_v2/validity_sample_v2.csv:1` 先标注新增样本（`human_is_us/human_is_teen`），就能估计 strict/balanced/recall 各自 precision，再决定你论文/报告采用哪一档作为“US-Teen”。
