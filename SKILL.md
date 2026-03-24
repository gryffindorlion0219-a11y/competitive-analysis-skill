---
name: competitive-analysis
version: 1.0.0
description: 将任意竞争态势数据（Excel/表格/粘贴数据）转化为可交互的 HTML 看板，含平台市占、增速对比、分品类明细、核心 Insights。适用于电商/消费品行业竞争分析、市场份额追踪、平台对比报告。输出可直接访问的 HTML 文件，无需任何框架或服务器。
---

# Competitive Analysis Skill

将竞争态势数据变成一份可交互的 HTML 看板报告。

**输入：** Excel / CSV / 粘贴的表格数据
**输出：** 单文件 HTML，本地可访问，可分享

---

## 第一步：理解数据结构

拿到数据后，先识别以下维度：

### 必须有
- **行业/品类**（消费电子、家电、家具…）
- **平台**（抖音、京东、天猫、拼多多、小红书…）
- **市占率**（%）或 **GMV 绝对值**

### 尽量有
- **同比变化**（YoY % 或 pp）
- **多个时间段**（当期 / 上期 / 全年对比）
- **预测值**（26E / 25E 等）

### 解析 Excel

```python
import openpyxl, json

wb = openpyxl.load_workbook('/path/to/file.xlsx', data_only=True)
for sheet_name in wb.sheetnames:
    ws = wb[sheet_name]
    print(f"=== {sheet_name} ({ws.max_row}行 x {ws.max_column}列) ===")
    for row in ws.iter_rows(min_row=1, max_row=3, values_only=True):
        print(row)  # 先看前3行，理解列结构
```

---

## 第二步：数据清洗

```python
def clean_pct(val):
    """处理各种格式的百分比：0.034 / '3.4%' / 3.4 → 统一返回 3.4"""
    if val is None: return None
    if isinstance(val, str):
        val = val.replace('%','').replace('％','').strip()
        try: val = float(val)
        except: return None
    if isinstance(val, float) and abs(val) < 1 and val != 0:
        val = val * 100  # 小数转百分比
    return round(float(val), 2)

def clean_growth(val):
    """处理同比/增速，正负号都保留"""
    v = clean_pct(val)
    return v  # 可以为负

def detect_unit(vals):
    """猜测 GMV 单位：亿/万亿/百亿"""
    clean = [v for v in vals if v and isinstance(v, (int,float))]
    if not clean: return '亿'
    avg = sum(clean)/len(clean)
    if avg > 10000: return '亿'
    if avg > 100: return '亿'
    return '亿'
```

---

## 第三步：分析框架

拿到数据后，按这个框架思考，驱动 Insights 写作：

### 市占分析

```
对每个平台，看三个维度：
1. 当前市占（绝对水平）—— 这是基本盘
2. 市占变化（pp）—— 正在扩张还是收缩
3. 变化背后的逻辑 —— 见下方「平台解读字典」
```

### 平台解读字典

| 信号 | 解读方向 |
|---|---|
| 抖音市占持续 +pp | 内容电商蚕食传统货架；检查是哪个品类驱动 |
| 京东市占 -pp | 传统货架失守；分品类看：3C 和家电是本阵地，失守要警惕 |
| 京东市占 +pp（家具/家居） | 标品化建材受益，逻辑不同于防守，是品类结构变化 |
| 拼多多市占 +pp | 低价竞争加剧；对高客单品类影响有限，对白牌影响大 |
| 小红书 YoY 高但市占低 | 早期高增阶段，基数小；重点看趋势方向而非绝对值 |
| 天猫市占 -pp | 检查是否流向抖音（内容化）还是拼多多（低价化） |

### 异常值识别

```python
def flag_anomalies(data, col='market_share_change'):
    """标记超出均值±1.5倍标准差的异常值"""
    import statistics
    vals = [d[col] for d in data if d.get(col) is not None]
    if len(vals) < 3: return data
    mean = statistics.mean(vals)
    stdev = statistics.stdev(vals) if len(vals) > 1 else 0
    for d in data:
        v = d.get(col)
        if v is not None and stdev > 0:
            d['is_anomaly'] = abs(v - mean) > 1.5 * stdev
            d['anomaly_direction'] = '显著上升' if v > mean else '显著下降'
    return data
```

### Insights 写作模板

每条 Insight 遵循：**现象 → 数字 → 逻辑 → 对我们的启示**

```
【{行业}】{平台}市占{变化方向}{数字}pp

现象：{平台}在{行业}市占{YoY/环比}变动{数字}pp，
      {当期市占}%→{上期市占}%

逻辑：{从平台解读字典里选}

启示：{对自己团队/平台的行动建议}
```

**示例：**
> 【消费电子】抖音市占+4.7pp，是三行业增幅最大
> 内容电商持续蚕食传统货架，手机/PC 品类是主战场。
> 对小红书的启示：消电品类内容决策属性强，是防御抖音蚕食的关键窗口。

---

## 第四步：生成 HTML 看板

### 基础结构

```html
<!DOCTYPE html>
<html lang="zh-CN">
<head>
<meta charset="UTF-8">
<title>{行业} 竞争态势看板</title>
<style>
:root {
  --bg: #0E0B14;        /* 深色底 */
  --card: #1A1428;      /* 卡片背景 */
  --border: rgba(155,106,172,0.15);
  --text: #F0EAF5;
  --text-mid: #C8B8D0;
  --text-soft: #7A6880;
  --accent: #E8B4C0;    /* 主强调色 */
  --positive: #7bc47f;  /* 正增长 */
  --negative: #e88;     /* 负增长 */
  --anomaly: #f0c070;   /* 异常值 */
}
/* ... 其余样式 */
</style>
</head>
<body>
<!-- Tab 导航 -->
<nav>总览 | 平台市占 | 增速对比 | 分品类 | Insights</nav>
<!-- 各 Tab 内容 -->
</body>
</html>
```

### 必备组件

**1. KPI 卡片**
```html
<div class="kpi-card">
  <div class="kpi-label">消费电子 小红书 YoY</div>
  <div class="kpi-value positive">+91%</div>
  <div class="kpi-sub">市占 0.18% → 0.34%</div>
</div>
```

**2. 横向条形图（纯 CSS，不需要 Chart.js）**
```html
<div class="bar-row">
  <div class="bar-label">抖音</div>
  <div class="bar-track">
    <div class="bar-fill" style="width:23.5%;background:#E8B4C0"></div>
  </div>
  <div class="bar-value">23.5%</div>
  <div class="bar-change positive">+4.7pp</div>
</div>
```

**3. 异常值高亮**
```html
<span class="anomaly-badge">⚡ 异常</span>
```

**4. Tab 切换**
```js
function showTab(id, btn) {
  document.querySelectorAll('.tab-panel').forEach(p => p.classList.remove('active'));
  document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
  document.getElementById(id).classList.add('active');
  btn.classList.add('active');
}
```

### 颜色规则

```js
function getChangeColor(val) {
  if (val === null || val === undefined) return 'var(--text-soft)';
  if (val > 2)  return 'var(--positive)';   // 显著正增长
  if (val > 0)  return '#a8d8a8';            // 轻微正增长
  if (val < -2) return 'var(--negative)';   // 显著负增长
  return '#d4a0a8';                          // 轻微负增长
}
```

---

## 第五步：本地访问

```bash
# 启动 HTTP server
cd /path/to/output/dir
python3 -m http.server 7788

# 访问地址
echo "http://localhost:7788/output.html"
```

---

## 完整工作流

```
1. 接收数据文件（Excel/CSV/粘贴）
2. 解析数据，识别维度和列结构
3. 清洗数值（统一单位/格式）
4. 套用分析框架，识别异常值
5. 生成 HTML 文件（含数据内嵌，单文件，无依赖）
6. 启动本地 server，输出访问 URL
7. 写 Insights（现象→数字→逻辑→启示）
```

---

## 踩坑记录

| 坑 | 解法 |
|---|---|
| Excel 百分比存为小数（0.034 = 3.4%） | `clean_pct()` 自动检测并转换 |
| 跨域访问 Excel/Redoc 数据 | 用 openpyxl 本地解析，或 browser 工具截图读取 |
| HTML 文件太大无法直接发送 | 启动本地 HTTP server，发内网 IP 链接 |
| localhost 对用户不可达 | 用 `0.0.0.0` 绑定，发内网 IP（`hostname -I`） |
| 多 sheet Excel 结构复杂 | 先打印前3行理解列结构，再按 sheet 分别解析 |

---

## 输出示例结构

```
看板 Tab 1：总览
  - 各行业 KPI 卡片（当期市占、YoY、平台排名变化）
  - 异常值 highlight

看板 Tab 2：平台市占
  - 横向条形图，当期 vs 上期对比
  - hover tooltip 显示详细数字

看板 Tab 3：增速对比
  - 各平台 YoY 气泡图或条形图
  - 正负增速颜色区分

看板 Tab 4：分品类明细
  - 可折叠的品类详情表格
  - 异常值自动标注 ⚡

看板 Tab 5：核心 Insights
  - 按行业分组
  - 现象→数字→逻辑→启示 四段式
  - 可复制为 PPT 文案
```
