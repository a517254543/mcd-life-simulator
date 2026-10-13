# MCP 集成说明

## 用到的麦当劳 MCP Server

- 接入地址：`https://mcp.mcd.cn`
- 传输协议：Streamable HTTP
- 鉴权：请求头 `Authorization: Bearer <MCP Token>`
- 限流：每 Token 每分钟 600 次，超限返回 429

本项目实际调用 5 个 Tool（核心流程）+ 2 个 Tool（按需），全部来自麦当劳中国官方 MCP Server。

## 实际使用的 Tool

| Tool | 用途 | 报告段落 | 是否关键 |
|---|---|---|---|
| `query-nearby-stores` | 按城市与位置定位门店，取 `storeCode` | 报告头 | 关键前置 |
| `query-meals` | 拉取该门店当前真实在售菜单、分类与价格 | 专属套餐 | **核心** |
| `available-coupons` | 当前可领取的优惠券列表 | 本月运势·小吉 | **核心** |
| `campaign-calendar` | 当月营销活动日历 | 本月运势·上上签 | **核心** |
| `list-nutrition-foods` | 能量、蛋白质、脂肪、碳水、钠、钙 | 隐藏属性·脂肪 | 关键 |
| `query-meal-detail` | 餐品配料与特调能力（验证名实不符项） | 本月运势·注意 | 按需 |
| `calculate-price` | 特调后算价 | — | 按需（本环境有已知问题） |

核心洞察：这个项目证明 MCP 不仅能做「决策工具」，也能做「内容生成器的数据底座」。
娱乐化的叙事外壳负责传播，MCP 的实时数据负责让推荐真实可点。

## 调用流程

```
① query-nearby-stores(searchType=2, beType=1, city, keyword)
      └─ 取 storeCode，向用户展示门店名与编号

② query-meals(storeCode, orderType=1, beType=1)
      └─ 取菜单：code / name / currentPrice / tags / discountType
      └─ 按价位带（由月收入决定）粗筛候选餐品

③ available-coupons()
      └─ 取状态为「可领取」的券名称

④ campaign-calendar()
      └─ 取「今日」及未来日期的活动（往期不取）

⑤ list-nutrition-foods()
      └─ 取候选餐品的 energyKcal 等，用于脂肪属性折算

⑥（按需）query-meal-detail(storeCode, orderType=1, beType=1, code)
      └─ 仅当需要验证「菜名与配料是否名实不符」时调用
      └─ 如龙焰芝士棒鸡腿堡：名称含芝士，配料为辣椒黄油风味酱 + 切块生菜

⑦ AI 按 references/report-template.md 生成六段报告 + 自检清单
```

## 报告数据映射

| 报告段落 | 数据来源 | 硬约束 |
|---|---|---|
| 专属套餐 | `query-meals` | 餐品必须在返回里；价格用 `currentPrice` 原值 |
| 本月运势·上上签 | `campaign-calendar` | 只取进行中/未来活动；改写营销语气、保留事实 |
| 本月运势·小吉 | `available-coupons` | 只取「可领取」状态；不引用会过期的签名图片 URL |
| 本月运势·注意 | `query-meal-detail` | 名实不符项必须验证过才写 |
| 隐藏属性·脂肪 | `list-nutrition-foods` | 必须附 kcal 折算依据；标注娱乐推算 |
| 隐藏属性·效率/快乐/钱包 | ②③④ 的真实优惠与活动 | 每个数值关联一个真实机制 |

## 业务价值

「人生模拟器」类内容在社交平台有天然传播属性（MBTI、星座、AI 未来照片都验证过这一点），
但纯生成的版本有一个致命问题：**推荐是假的**。用户按报告点到店里，没有这个餐品，
乐趣立刻归零。

本项目把 MCP 的实时数据接进娱乐框架：

- **传播层**：六段结构自带截图点（人生轨迹、运势打卡、反差梗）
- **信任层**：每份套餐、每个价格、每张券都能追到接口返回，附数据基准日
- **复用层**：`report-template.md` 的文案规则可直接迁移到其他品牌的
  「实时数据 × 娱乐生成」场景

## 数据边界（如实声明）

- `list-nutrition-foods` 返回字段仅
  `productName / nutritionDescription / energyKj / energyKcal / protein / fat / carbohydrate / sodium / calcium`。
  不含过敏原、配料表。本项目的营养引用仅用于「脂肪」属性的趣味折算，
  不构成营养建议。
- `available-coupons` 的券图片为带签名的临时 URL，会过期，项目不引用。
- `campaign-calendar` 混合往期/今日/未来三种状态，项目只取今日及未来。
- `calculate-price` 的 `items` 数组参数在当前工具调用层报
  `/items: must be array`（最简合法入参同样失败），判断为客户端序列化问题。
  核心流程不经过该接口；用到时失败即如实告知，不伪造结果。
- 所有价格为实测快照，报告每次生成都必须重新调用接口取当前值。

项目不做医疗或营养建议，不与其他品牌做任何对比，
不下单（除非用户明确要求）。
