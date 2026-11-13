# GEO 常规表达谱流水线（bulk 芯片）

<p align="center">
  <a href="https://github.com/liubarryteb12/geo-normal-pipeline-skill/actions/workflows/geo_analysis.yml"><img alt="GEO pipeline" src="https://github.com/liubarryteb12/geo-normal-pipeline-skill/actions/workflows/geo_analysis.yml/badge.svg?branch=main"></a>
  <a href="https://github.com/liubarryteb12/geo-normal-pipeline-skill/actions/workflows/docs_check.yml"><img alt="Docs check" src="https://github.com/liubarryteb12/geo-normal-pipeline-skill/actions/workflows/docs_check.yml/badge.svg?branch=main"></a>
  <img alt="R" src="https://img.shields.io/badge/R-4.3%2B-276DC3">
  <img alt="Node" src="https://img.shields.io/badge/Node-18%2B-339933">
  <img alt="License" src="https://img.shields.io/badge/License-MIT-green">
</p>

> **L1 分析线：bulk 转录组。** 面向 GEO 人源基因表达芯片的端到端流水线，覆盖数据门禁、清洗、QC、差异表达、通路富集、PPI、WGCNA、LASSO-Cox 与 TF 调控，产物以 GitHub Actions artifact 形式交付。

## 三线体系

| 线 | 仓库 | 数据 | 核心问题 |
|---|---|---|---|
| **L1** | **本仓库：`geo-normal-pipeline-skill`** | bulk 芯片 | 差异表达、通路富集、共表达、预后签名 |
| L2 | [`scrna-pipeline-skill`](https://github.com/liubarryteb12/scrna-pipeline-skill) | 单细胞 | 细胞类型、轨迹、通讯、调控 |
| L3 | [`spatial-pipeline-skill`](https://github.com/liubarryteb12/spatial-pipeline-skill) | Visium 空间转录组 | 空间域、空间高变基因、邻域、空间通讯 |

三条线共用同一套工程约定：配置驱动、状态落盘、验收判据、静态门禁。本仓库负责把 bulk 芯片的每一步做对。

## 这个仓库是什么

- **框架，不是焊死的流水线。** 步骤、参数、产物与验收项都随分析需求变化。加一步、换一种方法、关掉某个可选步骤，都是预期用法。
- **疾病无关。** `scripts/` 中没有一处依赖疾病或平台。换 `assets/config.<GSE>.yml` 就换分析。已验证的两个数据集恰好是乳腺癌，但代码不携带疾病假设。
- **配置驱动。** 一个数据集一个配置文件、一个产物目录。`--config` 必须显式指定，避免多数据集下静默跑错。
- **云端仓库是唯一真源。** 本 skill 不需要安装，也不依赖某台机器上的目录，按需从 GitHub 拉取即可。

## 分析流程

| 步骤 | 脚本 | 主要内容 | required |
|---|---|---|---|
| 00 输入门禁 | `00_validate_inputs.R` | 物种、数据类型、样本量、分组四项校验 | 是 |
| 01 下载清洗 | `01_download_clean.R` | GEO 取数、样本表整理、标准化 | 是 |
| 02 QC / PCA / 相关性 | `02_qc_pca_correlation.R` | 箱线图、密度图、PCA、样本相关热图 | 是 |
| 03 差异表达 | `03_deg.R` | limma 全基因结果、火山图、p 值直方图 | 是 |
| 04 热图与富集 | `04_heatmap_enrichment.R` | top DEG 热图、preranked GSEA、ORA GO/KEGG | 热图是，富集否 |
| 05 PPI | `05_ppi.R` | STRING 网络、hub 基因、边表、图注 | 否 |
| 06 WGCNA | `06_wgcna.R` | 软阈值、共表达模块、模块-性状 | 否，仅 `cohort` |
| 07 LASSO-Cox | `07_lasso.R` | 系数、风险评分、训练/CV/外部 C-index | 否，需随访终点 |
| 08 TF 调控 | `08_tf_regulation.R` | DoRothEA 调控子富集与活性组间比较 | 否 |
| 09 导出靶点 | `09_export_targets.R` | 候选靶基因 CSV，交给下游 | 否 |
| 10 生存诊断 | `10_survival_diagnostics.R` | timeROC 与校准，只读 07 的风险评分 | 否 |

可选步骤失败不会让 CI 变红，但每一步都必须在状态文件中写下 `status` 与原因。`not_done` 表示明确记录“做不了以及为什么”，与“做了没问题”严格区分。

## 快速开始

### 环境

- R 4.3+ 与 Bioconductor：`GEOquery`、`limma`、`clusterProfiler`、`org.Hs.eg.db`、`enrichplot`、`STRINGdb`、`pheatmap`、`WGCNA`、`glmnet`、`survival`、`dorothea` 等。
- Node 18+，仅用于数据集预检与静态门禁。
- 需要访问 NCBI GEO、STRINGdb 与 KEGG。

### 数据预检

```bash
node scripts/find_dataset.mjs check GSE42568
node tools/check_sample_structure.mjs GSE42568
node tools/check_clinical_endpoints.mjs GSE42568
```

第一条检查数据集是否可用；第二条检查分组是否与批次混杂；第三条给出随访终点、事件数与 EPV 换算的签名基因数上限。

### 运行

```bash
Rscript scripts/main_analysis.R --config assets/config.GSE42568.yml
ls results/GSE42568/
cat results/GSE42568/state.json
```

产物按 `dataset_id` 分目录，跑第二个数据集不会覆盖第一个。

### GitHub Actions

```bash
gh workflow run geo_analysis.yml -f dataset=GSE42568
gh run watch
gh run download --name geo-results-GSE42568
```

手动触发时指定数据集；push 到 `main` 时两个数据集各跑一个 job。artifact 名带数据集，下载后不会混。

### 作为 Agent Skill 使用

```bash
./use.sh                          # 拉取或更新到 ~/.cache/dsh-skills/，打印路径
./use.sh --register               # 需要本机 agent 直接发现时加
./use.sh --ref v1.0               # 钉住某一版
SKILL=$(./use.sh --print-path)    # 只取路径，便于脚本使用
./use.sh --clean                  # 清理缓存副本并撤销注册
```

拉取后会校验 `SKILL.md` 存在。远端改名或换结构时会明确报错，不会安静地给一个空目录。

## 两套设计模式

由配置中的 `design_mode` 选择，不是一个门禁换个阈值。

| | `small_sample` | `cohort` |
|---|---|---|
| 规模 | 总样本 < 10，每组 ≥ 3 | 总样本 ≥ 15，每组 ≥ 10 |
| 用途 | 探索性、假设生成 | 队列级验证 |
| 典型数据 | [GSE64790](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE64790)：6 例，3 TNBC vs 3 配对正常，GPL19612 | [GSE42568](https://www.ncbi.nlm.nih.gov/geo/query/acc.cgi?acc=GSE42568)：121 例，104 癌 vs 17 正常，GPL570 |
| 富集措辞 | “在最显著的 N 个基因里富集到……” | “差异基因富集到……” |

GSE64790 在 n=6 时没有任何基因能通过全基因组 FDR，最小 `adj.P` 为 0.394。富集与 PPI 走明确标注的 `ranked_fallback` 降级路径，结果只能作假设生成。GSE42568 用来把这个假设放到有功效的队列上验证。

## 配置

```yaml
design_mode: small_sample
dataset_id: GSE64790

group_field: characteristics_ch1
group_values:
  tumor:  ["tnbc"]
  normal: ["matched normal"]
contrast: ["tumor", "normal"]

paired: true
pairs:
  - ["GSM1580581", "GSM1580584"]
  - ["GSM1580582", "GSM1580585"]
  - ["GSM1580583", "GSM1580586"]

thresholds:
  adj_p: 0.05
  log2fc: 1.0
  string_score: 400

analysis:
  ranked_fallback_genes: 500
```

配对关系显式声明，不解析样本标题后缀。`dataset_id` 派生 `results/<GSE>/` 与 `data/<GSE>/`，脚本与配置中不写死目录。

## 实测基线

| 数据集 | 设计 | 关键结果 |
|---|---|---|
| GSE64790 | 6 例，3 对配对 | 无基因通过 FDR，最小 `adj.P` = 0.394；GSEA 在完整排序表上仍能给出通路信号，但只能作假设生成 |
| GSE42568 | 121 例，104 癌 vs 17 正常 | WGCNA 15 个模块（14 个非 grey），power=8；LASSO 16 基因签名 C-index 训练 0.879 / CV 0.793 / 外部 0.672；TF 87 个富集（BH<0.05）、190 个活性组间差异 |

这些数字是当前配置与参考数据集的基线，不是所有数据集的预期值。

## 产物

所有产物落在 `results/<GSE>/` 下。

| 文件 | 内容 |
|---|---|
| `state.json` | 每步执行状态、耗时与验收结果 |
| `run_manifest.json` | 本轮运行的证据：代码版本、参数、环境 |
| `deg_table.csv` | 全基因 limma 结果：gene、logFC、P.Value、adj.P.Val |
| `01-02-*` | 标准化前后箱线图、密度图、PCA、样本相关热图 |
| `01-03-*` | 火山图、p 值直方图 |
| `01-04-*` | top50 热图、GSEA GO/KEGG、ORA GO/KEGG、富集表 |
| `01-05-*` | STRING PPI 网络、hub 基因、边表、布局表、图注 |
| `wgcna_*.csv` + `01-06-*` | WGCNA 模块、模块-性状、软阈值与诊断图 |
| `lasso_*.csv` + `01-07-*` | LASSO 系数、风险评分、稳定性、KM 曲线 |
| `tf_*.csv` + `01-08-*` | TF 调控子富集、活性、组间差异 |
| `*_status.json` | 富集、PPI、WGCNA、LASSO、TF 的方法与限制 |
| `label_decisions.csv` | 每张图的标签显示决策与容量计算 |

每张图同时输出 PDF（矢量）与 PNG（可直接预览）。规范文件名用于验收和下游脚本，另有一份带 commit 短 SHA 的副本，用于区分不同版本的同名图。

## 质量门禁

```bash
node tools/check_r_syntax.mjs
node tools/check_fig_names.mjs
node tools/check_palette.mjs
node tools/check_legend_convention.mjs
node tools/check_figures.mjs results/GSE42568
node tools/check_fig_sizes.mjs results/GSE42568
node tools/check_artifact_paths.mjs
node tools/check_doc_refs.mjs
```

CI 还会跑 `check_legend_convention.mjs --selftest` 与 `check_doc_refs.mjs --selftest`。门禁自带反向标定用例，避免“检查了但什么都没查”的假阴性。

## 适用范围与限制

- **这是探索性分析，不是机制结论。** 肿瘤 vs 全组织正常的差异主要来自组织成分（上皮、脂肪、基质、免疫细胞），不等同于肿瘤特异驱动事件。
- **小样本的 FDR 边界是设计本身决定的。** n=6、3 对配对时残差自由度只有 2，全基因组校正后可能没有任何基因显著。这不是分析错误，也不应通过调大阈值来掩盖。
- **可选步骤的结论依赖前置条件。** WGCNA 需要 `cohort` 规模；LASSO-Cox 需要随访终点与足够事件数；TF 活性在每组 3 个样本时没有统计功效，流水线会如实报 0。
- **换数据集先查三件事。** 分组是否与批次混杂、平台注释是否包含基因 symbol、是否有随访终点。前两项任一不合格，数据集不可用。

完整的统计功效边界、分组依据与被否决数据集，见 [`EXPERIMENTAL_DESIGN.md`](EXPERIMENTAL_DESIGN.md)。

## 目录结构

```text
assets/       config.<GSE>.yml、TF 调控子最小表
scripts/      00-10 步骤脚本、main_analysis.R、find_dataset.mjs
scripts/lib/  common.R（配置、日志、色板、绘图工具）
tools/        静态门禁与数据集预检
references/   module0.md、troubleshooting.md
docs/         figure_groups.md
AGENTS.md     工程规则与判据
SKILL.md      Agent Skill 元数据
```

## 文档导航

| 文档 | 内容 |
|---|---|
| [`AGENTS.md`](AGENTS.md) | 工程规则、验收判据、踩坑记录 |
| [`EXPERIMENTAL_DESIGN.md`](EXPERIMENTAL_DESIGN.md) | 统计功效、分组依据、被否决数据集 |
| [`references/module0.md`](references/module0.md) | 模块零运行时规范 |
| [`references/troubleshooting.md`](references/troubleshooting.md) | 故障排查 |
| [`docs/figure_groups.md`](docs/figure_groups.md) | 图组清单与命名 |

## 赞赏支持

如果这个仓库帮你节省了时间，欢迎请作者喝杯咖啡。赞赏完全自愿，不影响代码的 MIT 许可与后续更新。

<p align="center">
  <img src="assets/sponsor-qr.png" alt="伊霖的赞赏码" width="260">
</p>

> 感谢支持，我会尽快更新代码，维护代码。

## 许可

MIT
