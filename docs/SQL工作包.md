# B 的 SQL 工作包（含表设计与查询清单）

> 制定：2026-10-13｜对象：成员 B（Python + MySQL）
> 用途：把项目里"能用 SQL 做的分析"集中到一个人手上。**SQL 适合所有"按维度分组统计"的工作，本项目的统计中枢就在这里。**

---

## 一、分工边界（先说清，避免和 A、R 撞车）

| 工作 | 谁做 | 为什么 |
|---|---|---|
| 读原始 CSV、清洗、写 `flights_clean.csv` | **A（Java）** | 清洗是一次性的工程活，产出唯一可信数据源 |
| **入库、分组统计、高峰识别、基线、交叉分析** | **★ B（SQL）** | **分组聚合是 SQL 的主场** |
| 时间格式转数值、名义属性、生成 ARFF | A（Java） | Weka 的输入格式必须由 Java 拼 |
| 调用 Weka 训练 Stacking、生成预测概率 | A（Java） | 模型框架 |
| 调整规则生成 | A（Java）+ B 提供目标航班名单 | 规则逻辑在代码里更清楚 |
| 画图、统计检验 | D（R） | 可视化 |
| 核对数字、写报告 | C | — |

**关键分界线**：**B 的每条查询都必须能独立跑、独立复现，并把结果导出成 CSV。** A 和 D 只消费这些 CSV，不重写 SQL。

---

## 二、数据库与表设计

```sql
CREATE DATABASE IF NOT EXISTS flight_analysis
  DEFAULT CHARACTER SET utf8mb4;

USE flight_analysis;
```

### 表 1：清洗后主表（所有分析的唯一数据源）

```sql
CREATE TABLE flights (
    id              INT AUTO_INCREMENT PRIMARY KEY,
    flight_date     DATE          NOT NULL,
    flight_no       VARCHAR(16)   NOT NULL,
    airline         VARCHAR(32),
    origin          CHAR(3),
    dest            CHAR(3),
    sched_dep_time  DATETIME      NOT NULL,
    dep_delay       INT           NOT NULL,
    is_delayed      TINYINT(1)    NOT NULL,   -- dep_delay > 15 记 1
    INDEX idx_hour (sched_dep_time),
    INDEX idx_date (flight_date),
    INDEX idx_route (origin, dest)
) ENGINE=InnoDB;
```

> **`is_delayed` 的阈值 15 分钟是法定口径**（《航班正常管理规定》第三条第三款），**硬编码进表里**，全组统一。

### 表 2：原始表（可选，用于对账）

```sql
CREATE TABLE flights_raw LIKE flights;      -- 结构相同，入库前未经清洗的数据
```

**用途**：`flights_raw` 与 `flights` 行数对比 → **报告里的"清洗丢了多少条"有据可查**。

### 表 3：SQL 产出的分析结果表

```sql
CREATE TABLE hourly_stats (
    hour          TINYINT PRIMARY KEY,
    flight_count  INT,
    avg_delay     DECIMAL(6,1),
    delay_rate    DECIMAL(5,3),
    is_peak       TINYINT(1) DEFAULT 0
) ENGINE=InnoDB;

CREATE TABLE route_stats (
    route         VARCHAR(8) PRIMARY KEY,
    flight_count  INT,
    avg_delay     DECIMAL(6,1),
    delay_rate    DECIMAL(5,3)
) ENGINE=InnoDB;

CREATE TABLE peak_hours (
    hour          TINYINT PRIMARY KEY,
    flight_count  INT,
    avg_delay     DECIMAL(6,1),
    delay_rate    DECIMAL(5,3)
) ENGINE=InnoDB;

-- 极值航班清单：人工复核用，不用等数组下标轮转
CREATE TABLE extreme_flights (
    flight_no     VARCHAR(16),
    flight_date   DATE,
    sched_dep_time DATETIME,
    dep_delay     INT,
    reason        VARCHAR(16)      -- 越界值 / 异常大值
) ENGINE=InnoDB;
```

**为什么用表而不只是导出 CSV**：`hourly_stats.is_peak` 标记可以被 A 直接查（`is_peak_hour` 特征的来源），也是"高峰时段识别"这一步的**持久化证据**。

---

## 三、九组查询（B 的交付清单）

### 查询 1｜入库对账与数据质量体检（最优先，A、C 都等这个）

```sql
-- 1.1 原始 vs 清洗后行数对账
SELECT (SELECT COUNT(*) FROM flights_raw) AS raw_rows,
       (SELECT COUNT(*) FROM flights)     AS clean_rows,
       (SELECT COUNT(*) FROM flights_raw) - (SELECT COUNT(*) FROM flights) AS dropped;

-- 1.2 各字段缺失统计（决定清洗规则是否合理）
SELECT SUM(flight_no IS NULL OR flight_no = '')          AS no_flight_no,
       SUM(origin IS NULL OR origin = '')                AS no_origin,
       SUM(sched_dep_time IS NULL)                       AS no_time,
       SUM(dep_delay IS NULL)                            AS no_delay
FROM flights_raw;

-- 1.3 延误值异常范围检查
SELECT MIN(dep_delay) AS min_delay,
       MAX(dep_delay) AS max_delay,
       AVG(dep_delay) AS avg_delay,
       SUM(dep_delay > 1440 OR dep_delay < -60) AS suspicious_count
FROM flights;

-- 1.4 时间范围与数据完整性（按日期看记录数，查缺口）
SELECT flight_date,
       COUNT(*) AS flights,
       ROUND(AVG(dep_delay), 1) AS avg_delay
FROM flights
GROUP BY flight_date
ORDER BY flight_date;

-- 1.5 重复航班检查（同一航班同一天同一起飞时间不应重复）
SELECT flight_no, flight_date, sched_dep_time, COUNT(*) AS cnt
FROM flights
GROUP BY flight_no, flight_date, sched_dep_time
HAVING cnt > 1;
```

| 项 | 内容 |
|---|---|
| **输出** | `data/output/data_quality.csv`（把上面 5 段的结果汇总） |
| **服务报告** | 研究内容第 1 条"数据清洗"——**"丢了多少条、按什么规则丢"** |
| **消费者** | **C 用于抽样核对**（他会拿这份去找出该丢没丢、不该丢却丢了的行） |
| 优先级 | **最高**——A 的清洗规则需要它验证，C 的核对需要它做基准 |

---

### 查询 2｜高峰时段识别（本项目的关键统计步骤）

```sql
-- 2.1 按小时统计
INSERT INTO hourly_stats (hour, flight_count, avg_delay, delay_rate)
SELECT HOUR(sched_dep_time) AS hour,
       COUNT(*)              AS flight_count,
       ROUND(AVG(dep_delay), 1)                                        AS avg_delay,
       ROUND(SUM(CASE WHEN dep_delay > 15 THEN 1 ELSE 0 END) / COUNT(*), 3) AS delay_rate
FROM flights
GROUP BY HOUR(sched_dep_time)
ORDER BY hour;

-- 2.2 高峰时段标记（延误率 > 全样本均值的 1.5 倍）
UPDATE hourly_stats
SET is_peak = 1
WHERE delay_rate > 1.5 * (SELECT AVG(delay_rate) FROM hourly_stats);

-- 2.3 或者用航班量口径：航班量前 25% 的小时
-- UPDATE hourly_stats SET is_peak = 1
-- WHERE flight_count >= (SELECT flight_count FROM hourly_stats ORDER BY flight_count DESC LIMIT 1) * 0.75;

-- 2.4 导出高峰时段
INSERT INTO peak_hours SELECT * FROM hourly_stats WHERE is_peak = 1;
```

**这段必须写进报告**：

| 候选定义 | 口径 | 优点 |
|---|---|---|
| 航班量前 25% 的小时 | `flight_count` 排序 | 直观 |
| **延误率 > 均值 1.5 倍** | `delay_rate` | **与研究目标（降低延误）直接相关** ← 推荐 |
| 两者取交集 | — | 最保守，但可能只剩 1～2 个小时 |

**推荐用法 + 敏感性分析**：主用延误率口径，同时给"航班量口径"下的结果做对照。**敏感性分析是加分项**，说明结论不依赖口径选择。

| 项 | 内容 |
|---|---|
| **输出** | `data/output/hourly_stats.csv`、`data/output/peak_hours.csv` |
| **消费者** | **A（`is_peak_hour` 特征直接查这张表）**、D（画高峰图） |
| 服务报告 | 研究内容第 2 条 |

---

### 查询 3｜航线维度分析

```sql
INSERT INTO route_stats (route, flight_count, avg_delay, delay_rate)
SELECT CONCAT(origin, '-', dest) AS route,
       COUNT(*)                  AS flight_count,
       ROUND(AVG(dep_delay), 1)  AS avg_delay,
       ROUND(SUM(CASE WHEN dep_delay > 15 THEN 1 ELSE 0 END) / COUNT(*), 3) AS delay_rate
FROM flights
GROUP BY origin, dest
HAVING flight_count >= 30          -- 样本太少的航线结论不可靠
ORDER BY avg_delay DESC;
```

**`HAVING flight_count >= 30` 这一句要写进报告**——**样本量小于 30 的航线不纳入排名**，否则一条只飞过 2 次、恰巧都延误的航线会排到第一。**答辩时这是"你懂统计"的体现。**

| 输出 | `data/output/route_stats.csv`、`top10_delayed_routes.csv` |
|---|---|
| 消费者 | D（Top N 条形图）、C（核对） |

---

### 查询 4｜统计基线（**机器学习结果的对照物，必须有**）

```sql
-- 4.1 全样本基线
SELECT COUNT(*) AS n, ROUND(AVG(dep_delay), 1) AS avg_delay,
       ROUND(SUM(is_delayed) / COUNT(*), 3) AS delay_rate
FROM flights;

-- 4.2 分时段基线（高峰 / 非高峰）
SELECT h.is_peak,
       COUNT(*) AS n,
       ROUND(AVG(f.dep_delay), 1) AS avg_delay,
       ROUND(SUM(f.is_delayed) / COUNT(*), 3) AS delay_rate
FROM flights f JOIN hourly_stats h ON HOUR(f.sched_dep_time) = h.hour
GROUP BY h.is_peak;

-- 4.3 分基线的预测规则：若预测"高峰时段且延误率高于基线"则判 positive
--     （这等价于一个规则基线分类器，用于和 Stacking 模型比 F1）
```

**为什么必须有**：

> 报告里要能写："与**纯统计基线**（预测某航班延误 = 该小时历史平均延误是否超阈值）相比，Stacking 模型的 F1 提升了 X"。
> **没有基线对照的机器学习结果，答辩站不住。**

| 输出 | `data/output/baseline_metrics.csv` |
|---|---|
| 消费者 | A（对比模型指标）、D（指标对比图）、C（报告结论） |

---

### 查询 5｜延误的时间分布与趋势（支撑 D 的图）

```sql
-- 5.1 星期 × 小时的热力表（给 D 画热力图）
SELECT DAYOFWEEK(flight_date) AS dow,
       HOUR(sched_dep_time)   AS hour,
       COUNT(*)               AS flights,
       ROUND(AVG(dep_delay), 1) AS avg_delay
FROM flights
GROUP BY dow, hour
ORDER BY dow, hour;

-- 5.2 按月的延误趋势（趋势分析的证据）
SELECT DATE_FORMAT(flight_date, '%Y-%m') AS ym,
       COUNT(*) AS flights,
       ROUND(AVG(dep_delay), 1) AS avg_delay,
       ROUND(SUM(is_delayed) / COUNT(*), 3) AS delay_rate
FROM flights
GROUP BY ym
ORDER BY ym;
```

| 输出 | `data/output/dow_hour_heatmap.csv`、`data/output/monthly_trend.csv` |
|---|---|
| 消费者 | **D**（热力图、趋势折线图） |
| 服务报告 | 研究内容第 5 条"延误趋势"——**这里明确是"趋势分析"，不是"预测"** |

---

### 查询 6｜航空公司 / 机场维度（附加分析，答辞材料）

```sql
-- 6.1 各航司延误表现
SELECT airline,
       COUNT(*) AS flights,
       ROUND(AVG(dep_delay), 1) AS avg_delay,
       ROUND(SUM(is_delayed) / COUNT(*), 3) AS delay_rate,
       RANK() OVER (ORDER BY AVG(dep_delay) DESC) AS rnk
FROM flights
GROUP BY airline
HAVING flights >= 50;

-- 6.2 起降机场组合的延误差异
SELECT origin, dest, COUNT(*) AS flights, ROUND(AVG(dep_delay), 1) AS avg_delay
FROM flights
GROUP BY origin, dest
HAVING flights >= 30
ORDER BY avg_delay DESC
LIMIT 10;

-- 6.3 早晨首发航班 vs 后续航班（"延误累积效应"的直接证据）
SELECT CASE WHEN HOUR(sched_dep_time) < 9 THEN '早班(9点前)' ELSE '其他' END AS seg,
       COUNT(*) AS flights,
       ROUND(AVG(dep_delay), 1) AS avg_delay
FROM flights
GROUP BY seg;
```

**6.3 的价值**：如果"早班延误明显低"，就为"**把部分航班提前到早高峰前**"的调整规则提供了依据——**直接支撑核心交付**。

| 输出 | `data/output/airline_stats.csv`、`route_pairs.csv`、`segment_compare.csv` |

---

### 查询 7｜调整目标航班的筛选（与 A 的规则生成配合）

A 生成 `adjustments.csv` 后，**入库做前后对比**：

```sql
CREATE TABLE adjustments (
    flight_no      VARCHAR(16),
    flight_date    DATE,
    original_time  DATETIME,
    adjusted_time  DATETIME,
    delta_minutes  INT,
    reason         VARCHAR(64),
    PRIMARY KEY (flight_no, flight_date)
) ENGINE=InnoDB;

CREATE TABLE predictions (
    flight_no        VARCHAR(16),
    flight_date      DATE,
    delay_probability DECIMAL(5,4),
    PRIMARY KEY (flight_no, flight_date)
) ENGINE=InnoDB;

-- 7.1 统计每个高峰小时被调整的航班数（验证"少量航班"约束）
SELECT HOUR(original_time) AS hour, COUNT(*) AS adjusted_count
FROM adjustments
GROUP BY hour
ORDER BY hour;

-- 7.2 调整量的分布（验证是否都在 ±10—15 分钟）
SELECT delta_minutes, COUNT(*) AS cnt
FROM adjustments
GROUP BY delta_minutes
ORDER BY delta_minutes;

-- 7.3 校准校验：预测概率分桶 vs 实际延误率
--     （如果模型说"70% 概率延误"，实际应接近 70% —— 这是模型可信度的直接证据）
SELECT FLOOR(delay_probability * 10) / 10 AS prob_bucket,
       COUNT(*) AS n,
       ROUND(AVG(f.is_delayed), 3) AS actual_delay_rate
FROM predictions p JOIN flights f
  ON p.flight_no = f.flight_no AND p.flight_date = f.flight_date
GROUP BY prob_bucket
ORDER BY prob_bucket;
```

**7.3 是加分项**：它检验模型**概率是否可信**（也叫可靠性图 / calibration curve）。**能主动做这一步，评委会觉得你们真的懂模型评估**，而不只是"跑了个准确率"。

---

### 查询 8｜调整前后对比（核心交付的验证）

```sql
-- 8.1 构造"调整后"的延误：被调整航班按 ±delta 平移延误
CREATE TABLE flights_adjusted AS
SELECT f.*,
       CASE WHEN a.delta_minutes IS NULL THEN f.dep_delay
            ELSE f.dep_delay + a.delta_minutes END AS dep_delay_adj,
       CASE WHEN a.delta_minutes IS NULL THEN f.is_delayed
            WHEN (f.dep_delay + a.delta_minutes) > 15 THEN 1
            ELSE 0 END AS is_delayed_adj
FROM flights f
LEFT JOIN adjustments a
  ON f.flight_no = a.flight_no AND f.flight_date = a.flight_date;

-- 8.2 前后对比
SELECT 'original' AS scenario, ROUND(AVG(dep_delay), 1) AS avg_delay,
       ROUND(SUM(is_delayed) / COUNT(*), 3) AS delay_rate FROM flights
UNION ALL
SELECT 'adjusted', ROUND(AVG(dep_delay_adj), 1),
       ROUND(SUM(is_delayed_adj) / COUNT(*), 3) FROM flights_adjusted;

-- 8.3 只在高风险航班子集上对比（更能体现规则效果）
SELECT 'original' AS scenario, COUNT(*) AS n, ROUND(AVG(dep_delay), 1) AS avg_delay
FROM flights WHERE is_delayed = 1
UNION ALL
SELECT 'adjusted', COUNT(*), ROUND(AVG(dep_delay_adj), 1)
FROM flights_adjusted WHERE is_delayed = 1;
```

> ⚠️ **8.1 的建模假设必须在报告里写明**：
> "本项目将调整量直接作用于该航班的历史延误值，作为**简化假设**；未模拟"时刻前移导致的后续航班连锁影响"。"
> **主动承认假设，比被评委问出来强。**

| 输出 | `data/output/comparison.csv` | 消费者 | **D（对比图）、C（核对与结论）** |

---

### 查询 9｜中间结果持久化

```sql
-- 9.1 保存 SQL 执行日志（可复现性证据）
CREATE TABLE sql_run_log (
    id          INT AUTO_INCREMENT PRIMARY KEY,
    script_name VARCHAR(64),
    run_at      DATETIME DEFAULT CURRENT_TIMESTAMP,
    row_effect  INT,
    note        VARCHAR(255)
);

-- 9.2 把关键中间结果做成视图，方便复核
CREATE VIEW v_peak_hour_detail AS
SELECT h.hour, h.flight_count, h.avg_delay, h.delay_rate, h.is_peak,
       ROUND(100 * h.flight_count / (SELECT COUNT(*) FROM flights), 1) AS pct_of_total
FROM hourly_stats h ORDER BY h.hour;
```

**为什么值得做**：报告里如果写"数据来自 SQL 分析"，评委可能问"怎么保证可复现"。**一个 `sql_run_log` 表 + 一份脚本清单，就是答案。**

---

## 四、B 的脚本交付清单

| 文件 | 内容 |
|---|---|
| `sql/01_schema.sql` | 建库建表 + 索引 |
| `sql/02_quality_checks.sql` | 查询 1 的 5 段 |
| `sql/03_hourly_stats.sql` | 查询 2（含高峰标记） |
| `sql/04_route_stats.sql` | 查询 3 |
| `sql/05_baseline.sql` | 查询 4 |
| `sql/06_trend.sql` | 查询 5 |
| `sql/07_dimension_extra.sql` | 查询 6 |
| `sql/08_predict_validate.sql` | 查询 7（含校准曲线） |
| `sql/09_comparison.sql` | 查询 8 |
| `sql/10_views_log.sql` | 查询 9 |
| `py/import_to_mysql.py` | 读 `flights_clean.csv` → 批量插入 |
| `docs/环境说明.md` | MySQL 版本、字符集、导入命令 |
| `data/output/*.csv` | 每条查询导出的结果（**导出脚本也写出来**） |

**执行顺序写在 `sql/README.md` 里**，别人拿到就能重跑。

---

## 五、SQL 与 Java 的接缝（必须对齐的字段）

| SQL 侧 | Java 侧 | 对齐要求 |
|---|---|---|
| `hourly_stats.is_peak` | `flight_train.arff` 的 `is_peak_hour` | **A 必须查 B 的表或 CSV，不能自己另定一套口径** |
| `flights.is_delayed`（>15 分钟） | ARFF 的类别标签 `delayed` | 阈值必须都是 **15 分钟** |
| `peak_hours.hour` | 特征 `hour` | **同一个小时的定义**（用计划起飞时间，不是实际起飞） |
| `predictions.delay_probability` | `adjustments.csv` | 概率决定哪些航班进调整名单 |

**这四条一旦不一致，模型输出和统计结论就会互相矛盾**——**答辩时最容易被抓住的地方**。建议：**B 把 `peak_hours.csv` 交付后，A 打印前 5 行贴群里确认。**

---

## 六、性能提醒（数据量可能几十万行）

| 问题 | 对策 |
|---|---|
| 逐行 `INSERT` 太慢 | **批量 `INSERT`**（一次 1000 行）或 `LOAD DATA LOCAL INFILE` |
| `is_delayed` 反复计算 | **入库时就算好存字段**（已在表设计里） |
| `GROUP BY HOUR(sched_dep_time)` 无法用索引 | 可加一个持久列 `dep_hour TINYINT`，入库时填好并建索引 |
| 大表联查慢 | 检查 `EXPLAIN`，确认走了索引 |

**产出**：`docs/环境说明.md` 里附一条"导入 50 万行耗时 X 秒"，**这本身就是报告里"方法可行性"的证据**。
