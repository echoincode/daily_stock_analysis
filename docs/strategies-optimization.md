# 交易策略（strategies/）优化分析

> 目标：梳理 `strategies/` 目录下 15 个策略 YAML 的可优化点。
> 分析基于策略文件本身，以及代码对字段的实际消费方式：
> - `src/agent/skills/base.py`：`default_active` / `default_router` 默认 `False`，`required_tools` 默认 `[]`。
> - `src/agent/skills/defaults.py`：`get_default_active_skill_ids` 只返回首个 `default_active` 策略；`get_regime_skill_ids` 仅命中 `market_regimes` 时优先选该策略。
> - `src/agent/skills/skill_agent.py:40`：`SkillAgent` 用 `required_tools` 决定 LLM 可调用的工具集合。

---

## P0 — 正确性缺陷（会导致策略实际失效，建议优先修）

### 1. `required_tools` 与 `instructions` 引用的工具不一致

`SkillAgent` 仅把 `required_tools` 中的工具暴露给 LLM。若 instructions 让模型使用某工具但未声明，模型**无法调用该工具**，策略逻辑会缺关键判断。

| 文件 | instructions 引用但未声明的工具 | 当前 `required_tools` |
|---|---|---|
| `bottom_volume.yaml` | `get_realtime_quote`（line 28）、`search_stock_news`（line 37） | `get_daily_history`, `analyze_trend` |
| `shrink_pullback.yaml` | `search_stock_news`（line 38） | `get_daily_history`, `analyze_trend`, `get_realtime_quote` |
| `volume_breakout.yaml` | `search_stock_news`（line 40） | `get_daily_history`, `analyze_trend`, `get_realtime_quote` |

**修复**：在对应 `required_tools` 中补上缺失工具。

### 2. `shrink_pullback` 的 `market_regimes` 与前提矛盾

- `shrink_pullback.yaml:16` 的 `market_regimes: [trending_down, sideways]`
- 但其入场前提是"必须处于上升趋势 MA5 > MA10 > MA20"（`shrink_pullback.yaml:23-24`）

`get_regime_skill_ids` 只在命中该 regime 时优先启用策略，导致它在**下跌/横盘**时才被选中，而此时前提不成立。

**修复**：改为 `market_regimes: [trending_up]`，或移除让它走 router fallback。

---

## P1 — 激活 / 路由配置

### 3. `default_active` 仅 `bull_trend` 为 true

`base.py` 默认 `False`，`get_default_active_skill_ids` 只返回首个 `default_active` 策略（`defaults.py:186`）。因此 `DEFAULT_ACTIVE_SKILL_IDS` 实际只有 `bull_trend`，其余 14 个策略只能靠 router fallback 或显式 `/ask` 触发。

**建议**：明确策略分层——默认激活组 / 路由组 / 按需调用组。若需并行多策略分析，显式给目标策略设 `default_active` 或纳入 `default_router`。

### 4. `one_yang_three_yin` 缺少 `market_regimes`

该策略无任何 `market_regimes`，意味着永远只能被显式调用或走 router fallback，不会被任何 regime 自动选中。

**修复**：至少补 `market_regimes: [trending_up]`。

### 5. `emotion_cycle` 的 `market_regimes:[sector_hot]` 过窄

情绪周期是通用框架策略，不应只在板块热时启用。

**建议**：改为 `[volatile, sideways]` 或留空（走 router）。

---

## P2 — 参数一致性与市场校准

### 6. "放量"阈值不统一

| 策略 | 放量定义 |
|---|---|
| `bottom_volume` | 当日量 > 5 日均量 3 倍 |
| `volume_breakout` / `box_oscillation` | 突破/放量 > 2 倍 |
| `ma_golden_cross` | 量比 > 1.2 |

**建议**：统一口径（如"突破/启动 = 2 倍，异动激增 = 3 倍"），并在 README 约定。

### 7. 换手率阈值偏通用市场，不适合 A 股

`emotion_cycle` 把 `<0.5%/日` 当底部、`>10%` 当顶部。A 股多数个股日换手 1–5%，`<0.5%` 几乎永远不满足，下限失真。

**建议**：按 A 股 / 港股 / 美股分别校准，或放宽下限。

### 8. 乖离率阈值散乱

阈值分散为 5% / 2% / 7% / 8% / 10%，与 README 核心理念 1（乖离率 < 5% 才考虑入场）不完全对齐。

**建议**：统一为基准 5%，仅在确有必要放宽时（如龙头）注明理由（如龙头放宽至 7%）。

---

## P3 — 内容 / 命名细节

### 9. `one_yang_three_yin` 别名误称

`aliases: [一阳穿三阴, 一阳夹三阴]` 中"一阳穿三阴"是误称（"一阳穿三线"是另一形态）。

**修复**：删除"一阳穿三阴"别名。

### 10. 一阳夹三阴"实体 > 股价的 2%"表述歧义

应明确为"实体涨幅 > 2%"或"实体 > 前收盘 2%"。

### 11. 评分幅度无上限 / 归一化

`sentiment_score` 调整从 +5 到 +20、负向 −5 到 −20，多策略叠加会无界累加。

**建议**：在 README 增加"单策略评分上限 + 聚合时 clamp"约定。

---

## P4 — 结构 / 设计（可选）

### 12. 策略簇高度重叠

- **趋势簇**：`bull_trend` / `ma_golden_cross` / `shrink_pullback` / `volume_breakout` 同属 `trend`，逻辑重叠（多头排列 + 回踩 + 突破 + 金叉）。
- **框架簇**：`hot_theme` / `dragon_head` / `emotion_cycle` / `event_driven` / `expectation_repricing` / `growth_quality` 都要求 `search_stock_news + get_sector_rankings`，评分逻辑相似。

**建议**：
1. 明确各策略互斥 / 互补边界，避免重复分析。
2. 对主观性强的 `chan_theory` / `wave_theory` 增加 `confidence` 权重字段供聚合器降权。

### 13. 多日确认机制缺失

如 `volume_breakout` 要求"次日开盘在突破位之上"区分真假突破，但策略无持续跟踪 / 状态字段。

**建议**：补充后续验证的触发 / 状态机制（可选）。

---

## 实施计划（最小侵入改动）

**核心原则**：只改 YAML 字段，**不动任何 Python 代码**。路由（`src/agent/skills/defaults.py`）与工具门禁（`src/agent/skills/skill_agent.py:40`）本来就直接读取这些 YAML 字段，因此改 YAML 即可生效，侵入面最小。

共需改 **5 个文件**（全在 `strategies/`）。

### 1. `strategies/bottom_volume.yaml` — 补 `required_tools`
```yaml
required_tools:
  - get_daily_history
  - analyze_trend
  - get_realtime_quote      # 新增（instructions line 28 用到）
  - search_stock_news       # 新增（instructions line 37 用到）
```

### 2. `strategies/shrink_pullback.yaml` — 补工具 + 修正 regime
```yaml
required_tools:
  - get_daily_history
  - analyze_trend
  - get_realtime_quote
  - search_stock_news       # 新增（instructions line 38 用到）

market_regimes: [trending_up]   # 原 [trending_down, sideways] → 改为上升（前提要求 MA5>MA10>MA20）
```

### 3. `strategies/volume_breakout.yaml` — 补 `required_tools`
```yaml
required_tools:
  - get_daily_history
  - analyze_trend
  - get_realtime_quote
  - search_stock_news       # 新增（instructions line 40 用到）
```

### 4. `strategies/one_yang_three_yin.yaml` — 补 regime + 修正别名
```yaml
aliases: [一阳夹三阴]            # 删掉误称的「一阳穿三阴」
market_regimes: [trending_up]    # 新增（否则永不被 regime 自动选中）
```

### 5. `strategies/emotion_cycle.yaml` — 放宽 regime
```yaml
market_regimes: [volatile, sideways]   # 原 [sector_hot] → 情绪周期是通用框架
```
> 或直接**删除该行**，走 router fallback。删行比改值更"最小"。

### 明确不要动的项（避免扩大侵入面）

- **`src/agent/skills/*.py`**：字段已被消费，无需改。
- **P1-#3 `default_active`**：这是行为开关（决定默认并行跑几个策略），改动影响全局分析量，**先不动**，待确认要多策略并行再单独评估。
- **P2 阈值校准 / P3 评分规范 / P4 重叠拆分**：涉及分析质量与大面积改写，不属于"最小侵入修失效"，建议单独一轮处理。
- **README / CHANGELOG**：本次纯 YAML 字段修正，可不更；若需留痕可加一条 CHANGELOG（非必须）。

### 验证（最小）
```bash
python -m py_compile src/agent/skills/*.py   # 未改 py，仅确认无回归
# 加载校验：启动后用 /ask 触发对应策略，确认模型能调用 search_stock_news 等工具
```

## 改动优先级建议

| 优先级 | 项 | 改动量 | 收益 |
|---|---|---|---|
| P0 | 1. 补 `required_tools`（3 文件） | 小 | 高（修复失效策略） |
| P0 | 2. 修正 `shrink_pullback` regime | 极小 | 高 |
| P1 | 3/4/5. 激活与 regime 配置 | 小 | 中（影响覆盖范围） |
| P2 | 6/7/8. 阈值统一与校准 | 中 | 中（分析质量） |
| P3 | 9/10/11. 命名与评分规范 | 小 | 低-中 |
| P4 | 12/13. 结构设计与状态机制 | 大 | 可选 |

> 注：如前一份文档 `docs/tushare-custom-url-change.md` 因功能已存在且变量名有误而失效，建议同步清理，避免误导。
