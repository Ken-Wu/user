# 用户路径分析工具 PRD

> 版本：v1.0　|　状态：草稿　|　文档类型：产品需求文档

---

## 一、背景与目标

### 1.1 背景

公司已有自建埋点平台，可采集页面与元素级别的行为数据，并能通过 SQL 取数（如页面曝光 PV/UV、元素点击等）。但目前数据是"散点"状态：

- 单页面的 PV/UV 能查到，但**页面之间如何流转、用户在哪一步流失**看不清；
- 页面内**哪些功能被点、哪些被忽视**缺乏直观呈现；
- 无法从**单个用户/分群**视角回放真实的使用路径。

### 1.2 目标

构建一个用户路径分析工具，把埋点数据转化为可交互的路径洞察，支撑产品/运营做体验诊断与转化优化。三大核心能力：

| 模块 | 能力 | 解决的问题 |
|------|------|-----------|
| ① 流量流转桑吉图 | 可视化页面间流转，逐层下钻 | 用户从哪来、到哪去、在哪流失 |
| ② 页面功能下钻 | 单页面功能点击率 + 点击热力图 | 页面内哪些功能有效、布局是否合理 |
| ③ 用户路径回放 | 单用户/分群的真实访问序列 | 还原真实使用场景、定位异常路径 |

### 1.3 目标用户

- 产品经理：转化漏斗诊断、功能价值评估、改版前后对比
- 运营：活动落地页路径分析、流失节点定位
- 数据分析师：路径探索、自定义路径漏斗

---

## 二、核心概念与数据基础

### 2.1 核心概念定义

| 概念 | 定义 |
|------|------|
| 页面（Page） | 由埋点平台分配的页面 ID 唯一标识的视图单元 |
| 元素（Element） | 页面内可交互组件（按钮、入口、卡片等），有独立元素 ID |
| 事件（Event） | 一次行为，分为页面曝光（page_view）、元素曝光（element_view）、元素点击（element_click）等 |
| 会话（Session） | 同一用户一次连续使用的事件序列，超时（如 30 分钟无操作）切分新会话 |
| 步（Step / Stage） | 路径中的位置序号，桑吉图按步分层（第1步、第2步…） |
| 流转边（Edge） | `源页面 → 目标页面` 的有向流量，承载 PV/UV |

### 2.2 数据需求（依赖埋点平台）

最少需要以下字段才能跑通全部功能：

```
user_id        用户唯一标识
session_id     会话 ID（若埋点无，则用 user_id + 时间窗自行切分）
event_time     事件时间戳（毫秒）
event_type     事件类型：page_view / element_view / element_click
page_id        页面 ID
page_name      页面名称
element_id     元素 ID（点击/曝光事件必填）
element_name   元素名称
pos_x, pos_y   元素坐标 / 点击坐标（热力图用，可选）
platform       端：Android / iOS / Web / 小程序 等
app_version    版本号
ext_props      扩展属性（JSON，承载业务参数）
```

> 热力图依赖 `pos_x/pos_y`（点击坐标或元素相对坐标）。若埋点暂不采集坐标，热力图可降级为"元素点击率分布图"（按元素聚合）。

---

## 三、功能详述

## 模块一：流量流转桑吉图（Sankey）

### 3.1 功能描述

以桑吉图呈现用户在页面间的流转。参考示意图：

- **纵向**：每一列代表路径的"第 N 步"（第1步、第2步、第3步…）；
- **节点**：每一步上出现的页面，节点高度 = 流量大小（PV 或 UV）；
- **连线**：相邻两步之间的流转量，线宽 = 流量；
- **起始锚点**：可指定"起始事件/起始页面"作为路径起点（如示意图左侧的「搜索」按端拆分）。

### 3.2 关键交互

| 交互 | 说明 |
|------|------|
| 指标切换 | PV / UV 切换；流转量 / 占比切换 |
| 起点设置 | 指定起始页面或起始事件；不指定则取全量入口 |
| 层数控制 | 展示前 N 步（默认 4~5 步，参考示意图） |
| 维度拆分 | 按 platform / app_version / 新老用户 / 渠道拆分（如示意图按 Android/iOS/Windows） |
| "更多"折叠 | 每一步只展示 Top N 页面，其余聚合为「更多」节点（参考示意图绿色「更多」块） |
| 节点点击下钻 | 点击任一页面节点 → 进入模块二（该页面功能下钻） |
| 连线 hover | 显示该流转的 PV/UV、转化率、流失率 |
| 路径高亮 | 点击某节点，高亮经过它的完整路径 |
| 时间/筛选 | 时间范围、端、版本、人群包等全局筛选 |

### 3.3 流失展示

每一步节点应能展示：
- 进入量、流出量、**流失量（未进入下一步的用户）**；
- 流失可单列为「流失/跳出」终止节点，便于定位断点。

### 3.4 取数逻辑（SQL 思路）

核心是把事件流转换成相邻页面对（Edge）。

**Step 1：会话内排序，生成页面访问序列**

```sql
-- 取页面浏览事件，按会话+时间排序，标记步序
WITH page_seq AS (
    SELECT
        user_id,
        session_id,
        page_id,
        page_name,
        platform,
        event_time,
        ROW_NUMBER() OVER (PARTITION BY session_id ORDER BY event_time) AS step_no
    FROM dwd_track_event
    WHERE event_type = 'page_view'
      AND pt BETWEEN '${start_date}' AND '${end_date}'
      -- 可选：AND page_id IN (起点路径约束)
)
```

**Step 2：自连接生成相邻流转边**

```sql
, edges AS (
    SELECT
        a.step_no                         AS from_step,
        a.page_id                         AS from_page,
        a.page_name                       AS from_name,
        b.page_id                         AS to_page,
        b.page_name                       AS to_name,
        a.platform,
        COUNT(1)                          AS pv,
        COUNT(DISTINCT a.user_id)         AS uv
    FROM page_seq a
    JOIN page_seq b
      ON a.session_id = b.session_id
     AND b.step_no = a.step_no + 1        -- 相邻两步
    WHERE a.step_no <= ${max_step}
    GROUP BY a.step_no, a.page_id, a.page_name, b.page_id, b.page_name, a.platform
)
SELECT * FROM edges
```

**Step 3：Top N + 「更多」聚合（应用层处理）**

每个 `from_step` 内按流量排序，保留 Top N 页面，其余 page_id 归并为虚拟节点 `__more__`。

> 性能要点：
> - 大表自连接成本高，建议预聚合为 `dws_page_edge_di`（按天 + 维度预算 edge），查询层直接取宽表；
> - UV 跨步去重需用 `COUNT(DISTINCT)`，预聚合时注意 UV 不可加和，需用 bitmap/HLL 近似或按需实时算；
> - 起点约束建议在 `page_seq` 阶段下推过滤。

### 3.5 验收标准

- 桑吉图能正确渲染 ≥5 步、每步 Top N 页面 + 「更多」；
- 支持 PV/UV 切换、端维度拆分；
- 节点可点击下钻，连线 hover 显示流转量与转化率；
- 单次查询（30 天数据）响应 < 5s（依赖预聚合）。

---

## 模块二：页面功能下钻（点击率 + 热力图）

### 3.6 功能描述

从桑吉图点击某页面节点进入，展示该页面内部的功能使用情况：

1. **页面概览**：页面 PV/UV、平均停留时长、跳出率、主要来源页/去向页（Top 5）；
2. **功能点击率排行**：页面内各元素的曝光、点击、点击率（CTR），按 CTR 或点击量排序；
3. **点击热力图**：基于点击坐标的热力分布，叠加在页面截图/线框上；
4. **元素曝光-点击漏斗**：元素曝光 UV → 点击 UV 的转化。

### 3.7 关键交互

| 交互 | 说明 |
|------|------|
| 指标维度 | 点击率（点击UV/曝光UV）、点击量、人均点击次数 |
| 元素列表 | 表格展示 element_name、曝光、点击、CTR，支持排序/搜索 |
| 热力图模式 | 点击热力 / 曝光热力 / CTR 热力 切换 |
| 截图叠加 | 上传或抓取页面截图作为底图，坐标按比例映射（需处理分辨率适配） |
| 元素联动 | 点击元素列表项 → 热力图高亮对应区域 |
| 同环比 | 与上一周期对比 CTR 变化 |
| 二次下钻 | 点击元素 → 查看点击该元素后用户的去向（回到模块一的子路径） |

### 3.8 点击率取数（SQL 思路）

```sql
-- 单页面元素曝光/点击聚合
SELECT
    element_id,
    element_name,
    COUNT(DISTINCT CASE WHEN event_type='element_view'  THEN user_id END) AS expo_uv,
    COUNT(DISTINCT CASE WHEN event_type='element_click' THEN user_id END) AS click_uv,
    SUM(CASE WHEN event_type='element_click' THEN 1 ELSE 0 END)           AS click_pv
FROM dwd_track_event
WHERE page_id = '${page_id}'
  AND pt BETWEEN '${start_date}' AND '${end_date}'
  AND event_type IN ('element_view','element_click')
GROUP BY element_id, element_name
```

CTR = `click_uv / expo_uv`（曝光点击率）。

### 3.9 热力图取数与渲染

```sql
-- 点击坐标聚合（按网格降采样，避免返回海量散点）
SELECT
    FLOOR(pos_x / ${grid}) AS gx,
    FLOOR(pos_y / ${grid}) AS gy,
    COUNT(1)               AS click_cnt
FROM dwd_track_event
WHERE page_id = '${page_id}'
  AND event_type = 'element_click'
  AND pos_x IS NOT NULL
  AND pt BETWEEN '${start_date}' AND '${end_date}'
GROUP BY FLOOR(pos_x / ${grid}), FLOOR(pos_y / ${grid})
```

渲染要点：
- 前端用 heatmap.js / ECharts heatmap，将网格点映射到底图坐标系；
- **分辨率适配**：坐标按页面宽度归一化（pos_x / screen_width），渲染时再乘以底图宽度，避免多机型坐标错位；
- 无坐标时降级为元素维度的"气泡热力"（用元素中心点近似）。

### 3.10 验收标准

- 能展示页面内全部元素的曝光/点击/CTR，且与埋点平台对数一致；
- 热力图能正确叠加底图并随筛选刷新；
- 元素列表与热力图联动高亮；
- 支持回到桑吉图查看点击元素后的去向。

---

## 模块三：用户路径回放与探索

### 3.11 功能描述

从用户视角还原真实路径，分两种视角：

1. **单用户路径回放**：输入 user_id，按时间线展示其完整事件序列（页面→点击→页面…），含时间间隔；
2. **分群路径探索**：
   - **路径检索**：指定起点页面 + 终点页面，看中间经过的高频路径（Top N）；
   - **路径漏斗**：自定义 N 步漏斗（页面/事件），看逐步转化率；
   - **异常路径**：识别"回退、反复横跳、长时间停留"等异常模式。

### 3.12 关键交互

| 交互 | 说明 |
|------|------|
| 单用户检索 | user_id / 设备号检索，时间线 + 卡片流展示 |
| 事件详情 | 点击事件查看 ext_props 业务参数 |
| 起终点路径 | 选起点页 A、终点页 B，输出 A→…→B 的路径分布 |
| 路径方向 | 正向（从起点出发）/ 逆向（到终点为止）探索 |
| 漏斗自定义 | 拖拽编排步骤，支持"严格相邻"或"宽松（中间允许其他页）" |
| 人群圈选 | 按 platform/version/业务标签筛选人群 |
| 导出 | 路径明细、漏斗结果导出 |

### 3.13 取数逻辑（SQL 思路）

**单用户时间线**

```sql
SELECT event_time, event_type, page_id, page_name, element_id, element_name, ext_props
FROM dwd_track_event
WHERE user_id = '${user_id}'
  AND pt BETWEEN '${start_date}' AND '${end_date}'
ORDER BY event_time
```

**起点→终点路径聚合**

```sql
-- 拼接每个会话的页面路径串，筛选起点开头/终点结尾
WITH seq AS (
    SELECT session_id, user_id,
           COLLECT_LIST(page_name) OVER (
               PARTITION BY session_id ORDER BY event_time
               ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
           ) AS path_arr
    FROM dwd_track_event
    WHERE event_type='page_view' AND pt BETWEEN '${start_date}' AND '${end_date}'
)
SELECT path_str, COUNT(DISTINCT user_id) AS uv
FROM (
    SELECT session_id, user_id, CONCAT_WS(' → ', path_arr) AS path_str
    FROM seq
) t
WHERE path_str LIKE '${start_page}%'        -- 起点约束
  AND path_str LIKE '%${end_page}'          -- 终点约束
GROUP BY path_str
ORDER BY uv DESC
LIMIT ${topN}
```

> Hive/Presto 语法差异：`COLLECT_LIST`（Hive）/ `array_agg`（Presto）；路径串可能很长，建议限制最大步数。

### 3.14 验收标准

- 单用户时间线完整、按时间正确排序、可查事件参数；
- 起终点路径检索结果与桑吉图整体口径一致；
- 自定义漏斗逐步转化率计算正确，支持严格/宽松模式。

---

## 四、模块联动设计

三个模块构成"宏观 → 中观 → 微观"的下钻闭环：

```
桑吉图（看整体流转）
   │ 点击页面节点
   ▼
页面下钻（看页面内功能 + 热力图）
   │ 点击元素 → 看点击后去向
   ▼
用户路径（看真实个体/分群路径，验证假设）
```

全局筛选器（时间、端、版本、人群）在三个模块间保持状态同步。

---

## 五、技术架构建议

### 5.1 分层架构

```
┌─────────────────────────────────────────┐
│  前端：ECharts(Sankey) + heatmap.js + 表格 │
├─────────────────────────────────────────┤
│  应用层：查询服务 / 路径计算 / Top N聚合      │
├─────────────────────────────────────────┤
│  数据层：                                  │
│   - 明细层 dwd_track_event（埋点清洗）       │
│   - 汇总层 dws_page_edge_di（流转边预聚合）   │
│           dws_page_element_di（元素点击预聚合）│
│   - 缓存：高频查询结果缓存（Redis）           │
└─────────────────────────────────────────┘
```

### 5.2 关键技术点

| 问题 | 方案 |
|------|------|
| 大表自连接慢 | 预聚合流转边到 `dws_page_edge_di`（按天+维度） |
| UV 跨步去重不可加 | 实时 `COUNT(DISTINCT)` 或 HLL/bitmap 近似 |
| 会话切分 | 优先用埋点 session_id；否则按 30min 超时窗口切分 |
| 热力图海量散点 | 网格降采样（GROUP BY FLOOR(x/grid)）后返回 |
| 多机型坐标错位 | 坐标按屏宽归一化存储，渲染时还原 |
| 路径串过长 | 限制最大步数 / Top N 截断 |
| 查询响应慢 | 预聚合 + 结果缓存 + 异步查询任务化 |

### 5.3 性能目标

- 桑吉图（30 天，Top10×5 步）：< 5s
- 页面下钻：< 3s
- 单用户路径：< 2s
- 分群路径检索：异步任务，大查询 < 30s

---

## 六、里程碑规划

| 阶段 | 范围 | 产出 |
|------|------|------|
| MVP（P0） | 模块一桑吉图（PV/UV、Top N+更多、端拆分、节点下钻） | 整体流转可视化 |
| P1 | 模块二页面下钻（点击率排行 + 元素漏斗），热力图先降级版 | 页面功能诊断 |
| P1 | 模块三单用户路径回放 | 个体路径还原 |
| P2 | 真·热力图（坐标）、分群路径检索、自定义漏斗 | 完整探索能力 |
| P2 | 同环比、人群包、改版前后对比 | 高级分析 |

---

## 七、风险与依赖

| 项 | 说明 |
|----|------|
| 埋点完整性 | 页面/元素埋点覆盖率决定分析质量，需先做埋点盘点 |
| 坐标采集 | 热力图依赖点击坐标，若未采集需推动埋点补充 |
| session_id | 若埋点无会话 ID，需自行切分，口径需对齐 |
| UV 去重性能 | 跨步精确去重成本高，需评估精确 vs 近似 |
| 取数权限 | 工具取数依赖现有 SQL 引擎权限与稳定性 |

---

## 八、待确认问题（QA）

1. 埋点是否已采集 `session_id` 与点击坐标 `pos_x/pos_y`？
2. 页面 ID 与元素 ID 的稳定性如何（改版后是否变化）？
3. 桑吉图默认起点：全量入口 vs 指定起始事件，哪个为主？
4. UV 去重要求精确还是可接受 HLL 近似（影响性能）？
5. 是否需要实时/准实时，还是 T+1 离线即可？
6. 底图来源：人工上传截图 vs 自动抓取？
