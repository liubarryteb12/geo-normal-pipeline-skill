# L1 GEO Bulk 常规挖掘流水线

> 本仓库是 **L1 生产分析线**的源码镜像。完整技术说明、输入契约、步骤细节与实测基线见
> [`src/line1_bulk/README.md`](src/line1_bulk/README.md)。

## 三线体系

| 线 | 仓库 | 数据 | 核心问题 |
|---|---|---|---|
| **L1** | **本仓库：`geo-normal-pipeline-skill`** | bulk 表达矩阵（series matrix + GPL 注释） | 差异表达、共表达、富集、PPI、机器学习与外部验证 |
| L2 | [`scrna-pipeline-skill`](https://github.com/liubarryteb12/scrna-pipeline-skill) | 单细胞计数 | 细胞类型、轨迹、通讯、调控、虚拟敲除 |
| L3 | [`spatial-pipeline-skill`](https://github.com/liubarryteb12/spatial-pipeline-skill) | Visium 空间转录组 | 空间域、空间统计、通讯、定位、虚拟敲除 |

三条线默认互不交叉。线内允许按工具分工混用 R 与 Python，线间不互相取参考、不合并数据集、不跨线验证。约束全文见 [`src/line1_bulk/docs/isolation.md`](src/line1_bulk/docs/isolation.md)。

## 这套框架是什么

- **模块注册表**：根目录 [`workflows.json`](workflows.json) 是本线 stage、mod、依赖产物、成本估算与约束的唯一事实源。
- **步骤母版**：`src/line1_bulk/scripts/` 下每个文件就是一个可执行步骤，文件名即 step 名。
- **跨语言桥**：`src/bridge_guard.sh` 在 Python 与 R 之间传 `.h5ad` / `.rds` 前校验基因名、矩阵方向、细胞名、embedding 四项断言。
- **环境声明**：`src/line1_bulk/envs/manifest.txt` 记录主语言、版本、环境名与容器镜像，分析中途不更换。
- **检查点**：`src/line1_bulk/checkpoint/` 放跨语言检查点脚本，命名 `checkpoint_*.sh`。
- **共享规范**：`src/line1_bulk/GLOSSARY.md`、`src/line1_bulk/docs/` 保存术语、隔离、目录、编码与出图规范。

## 分析阶段

| 阶段 | 模块 | 脚本 |
|---|---|---|
| 1 数据接入 | 矩阵装配与注释 | `prep_matrix` |
| 2 质控与探索 | 样本 QC、PCA 分组验证 | `qc_matrix`、`pca_matrix` |
| 3 差异与共表达 | limma 差异表达、WGCNA 模块 | `deg_limma`、`wgcna_modules` |
| 4 功能富集与网络 | GO/KEGG、富集图件、STRING PPI | `enrichment_bulk`、`enrichment_figures`、`ppi_string` |
| 5 机器学习与验证 | 交集基因、建模、Hub、诊断、外部验证、Cox | `key_genes`、`ml_model`、`ml_hub_analysis`、`ml_diagnostics`、`external_validation`、`cox_analysis` |
| 6 图件汇总 | 线 1 图件包 | `bulk_figures` |

## 输入

- GEO series matrix 与 GPL 平台注释。
- `grouping.txt` 分组表。
- 项目数据目录：`<BIO_DATA_ROOT>/projects/<project>/line1_bulk/raw_data/`。

## 产出

产物落在 `<BIO_DATA_ROOT>/projects/<project>/line1_bulk/`：

```text
raw_data/    原始数据
processed/   中间矩阵与工作目录
results/     统计表、报告、图注
figures/     单图输出
```

每个步骤同时输出 `*_report.txt`，记录口径、阈值、诊断量、计时与产物清单。自校验不过不落盘。

## 运行方式

生产环境由任务引擎执行，不依赖 GitHub Actions：

1. 运维层把 `src/line1_bulk/scripts/` 部署到算力节点的 `bin/line1_bulk/`，并注入 PATH。
2. 任务引擎把工作目录切到 `<line>/processed/`，传入 `--input`、`--output`、`--seed`。
3. 引擎注入 `BIOANALYS_SEED`；缺失即退出码 3，不设默认值。
4. 环境版本与随机种子记录在 `src/line1_bulk/envs/manifest.txt`，分析中途不更换。

退出码约定：

| 码 | 含义 |
|---|---|
| 0 | 成功，产物与台账齐 |
| 1 | 自校验失败（计算或产物不合口径） |
| 2 | 参数错误或输入契约不满足 |
| 3 | `BIOANALYS_SEED` 未注入 |
| 4 | 同步骤锁互斥 |
| 5 | 启动前内存余量不足 |

本仓库只保留源码与规范，不包含数据、产物、凭证或设备身份信息。路径中的 `<BIO_DATA_ROOT>` 由部署环境提供。

## 目录结构

```text
workflows.json              L1 模块注册表
src/
  bridge_guard.sh           跨语言桥接四项断言
  line1_bulk/
    README.md               生产说明与实测基线
    GLOSSARY.md             领域术语表
    docs/                   隔离、目录、编码规范
    envs/manifest.txt       环境与版本声明
    checkpoint/             桥接检查点
    scripts/                步骤母版
assets/sponsor-qr.png       赞赏码
```

## 迁移说明

2026-11-13 起，本仓库只保留线 1 生产母版。旧版 GEOquery / limma 全流程脚本、旧 CI 与旧图表模板已移除。生产行为以 `src/line1_bulk/` 与 `workflows.json` 为准。

## 赞赏支持

如果这个仓库帮你节省了时间，欢迎请作者喝杯咖啡。赞赏完全自愿，不影响代码的 MIT 许可与后续更新。

<p align="center">
  <img src="assets/sponsor-qr.png" alt="伊霖的赞赏码" width="260">
</p>

> 感谢支持，我会尽快更新代码，维护代码。

## 许可

MIT
