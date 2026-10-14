# 数据驱动的机场高峰时段时刻微调优化

哈尔滨工业大学 2026 级 · 大一年度项目（学院立项表第 11 号）

| 项 | 内容 |
|---|---|
| 项目类别 | 教师科研项目 |
| 学科交叉 | 是 |
| 指导教师 | 吴冲 教授（经济与管理学院） |
| 数据源 | Kaggle 公开航班时刻与延误数据 |
| 核心技术 | 两层栈式集成学习（Stacking）+ 统计方法识别高峰时段 |
| 立项答辩 | 2026-11-16 ~ 11-22 |

## 项目目标

聚焦单一机场的早高峰时段，完成「数据清洗 → 高峰时段识别 → 延误概率预测 → 时刻微调建议 → 优化效果验证」的完整流程，输出一套**可运行、可复现**的时刻微调方案生成系统。

**系统定位**：决策支持系统。输出时刻微调建议及量化依据，**不直接改动航班时刻，不直接面向旅客发布信息**。

## 技术流水线

```
Kaggle 原始 CSV
      │
      ▼  ① Python（pandas）：清洗 + 特征工程
data/features/flight_features.csv
      │
      ├──▶ ② MySQL：数据质量体检 + 高峰时段识别 + 统计基线
      │        └─▶ data/output/*.csv
      │
      ▼  ③ Java：CSV → ARFF + Weka Stacking 训练
data/model/flight_predictions.csv（延误概率）+ metrics.json
      │
      ▼  ④ Java：调整规则生成 + 调整前后对比
data/model/adjustments.csv + comparison.csv
      │
      ▼  ⑤ R：制图制表 + 统计检验
figures/*.png + tables/*.csv
```

### 模型配置

**第一层（4 个异构基模型）**

| 模型 | Weka 类 |
|---|---|
| J48 决策树 | `weka.classifiers.trees.J48` |
| RandomForest 随机森林 | `weka.classifiers.trees.RandomForest` |
| SMO 支持向量机 | `weka.classifiers.functions.SMO` |
| NaiveBayes 朴素贝叶斯 | `weka.classifiers.bayes.NaiveBayes` |

**第二层（元学习器）**：Logistic 逻辑回归 · 集成框架：`weka.classifiers.meta.Stacking`

> **关键技术点**：SMO 必须开启 `-M`（`buildCalibrationModels`），否则只输出硬 0/1，Stacking 无法获得有效概率。

## 团队成员与分工

| 成员 | 工具 | 负责环节 |
|---|---|---|
| A | **Java + Weka** | CSV→ARFF 转换、Stacking 训练与评估、调整规则生成、前后对比 |
| B | **Python + MySQL** | 数据清洗、特征工程、入库、SQL 统计与高峰识别 |
| C | Excel / Word | 数据核对、字段说明、报告撰写、答辩材料 |
| D | **R** | 制图制表、统计检验 |

## 文档索引（先读这两份）

| 文档 | 用途 |
|---|---|
| **[docs/交接契约.md](docs/交接契约.md)** | **各环节之间的文件格式、列定义与验收标准——动手前必读** |
| [docs/技术方案.md](docs/技术方案.md) | 流水线设计、各环节输入输出、风险与对策 |
| [docs/需求与设计.md](docs/需求与设计.md) | 原始需求拆解、功能范围、简化说明 |
| [docs/团队分工.md](docs/团队分工.md) | 四人职责划分 |
| [docs/每人任务清单.md](docs/每人任务清单.md) | 逐周任务与交付物 |
| [docs/SQL工作包.md](docs/SQL工作包.md) | B 的九组查询、表设计与脚本清单 |
| [docs/数据流与接口定义.md](docs/数据流与接口定义.md) | Weka 版数据流与字段对齐要求 |

## 目录结构

```
data/
  raw/       原始 Kaggle 数据（不进 Git，见 raw/README.md）
  sample/    200 行样例数据（进 Git，供全组并行开工）
  features/  Python 产出的特征表
  output/    SQL 统计结果
  model/     ARFF、预测结果、模型指标
src/
  java/      Java + Weka 代码
  python/    清洗与特征工程
sql/         建表、查询、导出脚本
r/           R 脚本
figures/     图表输出（300dpi PNG）
tables/      报告用表格
report/      报告与 PPT
docs/        项目文档
```

## 全组必须遵守的四条格式铁律

| # | 铁律 | 违反的后果 |
|---|---|---|
| 1 | **CSV 编码 UTF-8 无 BOM** | R 读出来首列名变 `X.U.FEFF.hour`；MySQL 导入首列报错 |
| 2 | **列名全小写 + 下划线**，无空格无中文 | SQL 需加反引号、R 需改名 |
| 3 | **缺值用空字段**，不写 `NULL`/`NA`/`-` | `AVG()` 算错、R 当作字符串 |
| 4 | **小数位统一**（数值 1 位、概率 4 位、比率 3 位） | 同一指标在报告里出现两种精度 |

## 关键约定

- **统计量只在 SQL 里算一次**——各算一遍会导致报告里出现多个不一致的数字
- **高峰时段（`is_peak_hour`）的唯一来源是 `peak_hours.csv`**，其他环节不得自行定义
- **每段只加工一次**，以 CSV 作为环节间的标准化接口
- **原始数据不进 Git**，聚合结果进 Git

## 运行环境

- JDK 21 + IntelliJ IDEA（Java 侧）
- Weka（详见 `docs/技术方案.md`）
- Python 3 + pandas（B 侧）
- MySQL 8（B 侧）
- R + ggplot2（D 侧）

> 具体版本与安装步骤由各负责人补充到 `docs/环境说明.md`。
