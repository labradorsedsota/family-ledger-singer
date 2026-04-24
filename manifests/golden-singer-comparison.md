# Golden / Singer 对比分析 — 家庭记账本 (FEB)

| 项目 | 内容 |
|------|------|
| Golden Repo | https://github.com/labradorsedsota/family-ledger |
| Singer Repo | https://github.com/labradorsedsota/family-ledger-singer |
| Golden 报告 | 25/25 PASS (100%), 执行日 2026-04-22 |
| Singer 报告 | 12P/10F/3PA = 48%, 执行日 2026-04-24, v2 修订版 |
| 对比执行 | 智子 (Layer 2), 2026-04-24 |

---

## 一、代码 Diff 汇总

Golden → Singer 共 **5 处代码改动**（6 行变更）：

| # | 位置 | 改动内容 | Diff 行数 |
|---|------|---------|-----------|
| D1 | L178 + L180 | 月份导航箭头 `prevMonth` ↔ `nextMonth` 互换 | 2 |
| D2 | L409 | `getCategoriesByType(type)` 硬编码 `return EXPENSE_CATEGORIES`，忽略 type 参数 | 1 |
| D3 | L495-496→495 | `saveRecordsToStorage()` 移除 try/catch，直接 `return false` | 2→1 |
| D4 | L647-650→646 | `progressColor` 移除条件逻辑，硬编码 `return '#E8DEFF'` | 4→1 |
| D5 | L728→724 | 删除条件 `idx === -1` 反转为 `idx !== -1` | 1 |

---

## 二、逐项对比

### D1: 月份导航箭头方向反转

| 项 | 内容 |
|---|------|
| 改动 | 左箭头 `@click="prevMonth"` → `@click="nextMonth"`；右箭头反之 |
| 预期效果 | 左箭头往未来（非过去），右箭头往过去（非未来） |
| Golden | 无 L2-12 测试（golden 版 TEST-CASE 无此用例） |
| Singer | L2-12 PARTIAL — "左箭头导航正常(往过去)，但右箭头无法点击(坐标溢出)" |
| **判定** | **❌ 未检出** |
| 分析 | L2-12 是 singer 版 v1.1 新增的专用用例，但 Moss 对左箭头方向的验证不充分——singer 代码中左箭头调用 `nextMonth`（→ 5 月），Moss 却报告"往过去"；右箭头因坐标溢出完全无法测试。方向反转 bug 被遗漏。 |

### D2: 分类联动失效 → Singer Bug #2

| 项 | 内容 |
|---|------|
| 改动 | `getCategoriesByType()` 忽略 type 参数，始终返回支出分类 |
| Golden | L1-02 PASS, L2-02 PASS（收入 4 分类正确联动） |
| Singer | L1-02 FAIL, L2-02 FAIL（切换收入后仍显示 8 个支出分类） |
| **判定** | **✅ 已检出** — Golden PASS + Singer FAIL |
| Moss 根因 | 准确定位到 `getCategoriesByType()` 硬编码 |

### D3: 存储写入失败 → Singer Bug #1

| 项 | 内容 |
|---|------|
| 改动 | `saveRecordsToStorage()` 直接 `return false`，所有写操作静默失败 |
| Golden | L1-01/02/05, L2-04, L3-06, L3-07 全部 PASS |
| Singer | L1-01/02/05 FAIL, L2-04 FAIL, L3-06 FAIL, L3-07 BLOCKED |
| **判定** | **✅ 已检出** — Golden PASS + Singer FAIL |
| Moss 根因 | 准确定位到 L494 `return false` |

### D4: 进度条颜色硬编码 → Singer Bug #5

| 项 | 内容 |
|---|------|
| 改动 | `progressColor` 移除红/黄/蓝条件逻辑，硬编码 `#E8DEFF`（淡紫色） |
| Golden | L2-08 PASS（红色进度条 + Notification）, L3-04 PASS（红色满条） |
| Singer | L2-08 FAIL（灰色进度条 + 无 Notification）, L3-04 FAIL（灰色进度条） |
| **判定** | **✅ 已检出** — Golden PASS + Singer FAIL |
| 附注 | Notification 未弹出与 D4 无关（Notification 代码两版完全相同），可能是 mano-cua 截图时机问题（5秒自动消失），singer 报告标为"待确认"合理 |

### D5: 删除条件反转 → Singer Bug #4

| 项 | 内容 |
|---|------|
| 改动 | `if (idx === -1) return` 改为 `if (idx !== -1) return`，找到目标时直接 return |
| Golden | L1-05 PASS, L3-06 PASS |
| Singer | L1-05 FAIL, L3-06 FAIL |
| **判定** | **✅ 已检出** — Golden PASS + Singer FAIL |
| Moss 根因 | 准确定位到 L724 条件反转 |

---

## 三、Singer Bug #3 验证（正数结余非绿色）

| 项 | 内容 |
|---|------|
| Singer 报告 | L2-11a FAIL — "结余 ¥2,000.00 颜色非绿色" → Bug #3 |
| Golden 报告 | L2-11a PASS — "¥2,000.00 绿色 rgb(16,185,129)" |
| **代码 Diff** | **无** — 两版 CSS 完全相同：`.amount-income { color: #000; }`（黑色） |
| **判定** | **⚠️ 误报 (False Positive)** |
| 分析 | 两版代码中 `.amount-income` 均为 `color: #000`（黑色），正数结余在两版中应表现一致。Golden 报告声称观察到绿色 `rgb(16,185,129)` 并标 PASS，但源码不支持此颜色——Golden 的 L2-11a PASS 判定本身可能有误。Singer 版 Moss 观察到"非绿色"是正确的，但这不是注入 bug，是应用原有设计（或 golden 测试误判）。 |

---

## 四、汇总

### 注入 Bug 检出率

| 指标 | 数值 |
|------|------|
| 代码注入 Bug 总数 | **5** |
| 已检出 | **4** (D2 Bug#2, D3 Bug#1, D4 Bug#5, D5 Bug#4) |
| 未检出 | **1** (D1 月份导航反转) |
| 误报 | **1** (Singer Bug#3 — 代码无 diff，golden 报告可能误判) |
| **检出率** | **80% (4/5)** |
| **精确率** | **80% (4/5 reported as injected)** |

### 按 Failure Type 分布

| Failure Type | 注入数 | 检出 | 状态 |
|-------------|--------|------|------|
| UNMATCHED_OUTCOME | 2 (D3, D5) | 2/2 | ✅ |
| UI_MALFUNCTION | 2 (D1, D2) | 1/2 | D1 未检出 |
| VISUAL_DEFECT | 1 (D4) | 1/1 | ✅ |

### 漏检分析：D1 月份导航方向反转

**根因**：L2-12 是 singer 版 v1.1 专门为此 bug 新增的测试用例，但两个因素导致漏检：
1. **右箭头坐标溢出**：mano-cua 无法点击右箭头，导致只测了一半
2. **左箭头方向未精确验证**：Moss 报告左箭头"往过去"，但 singer 代码中左箭头实际调用 `nextMonth()`（→ 2026 年 5 月），不是过去。Moss 可能看到月份文字变化就默认"正确"，未核对具体月份

**启示**：导航方向类 bug 对 VLA 模型有挑战性——需要比较两个状态的语义含义（"3月" vs "5月"相对于"4月"的方向），而非简单的视觉变化检测。
