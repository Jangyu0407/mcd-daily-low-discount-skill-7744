# WorkBuddy 开发对话全记录

## 项目：麦麦当日薅券助手（mcd-daily-coupon-hunter）

| 项    | 内容                                   |
| ---- | ------------------------------------ |
| 技能名称 | 麦麦当日薅券助手                             |
| Slug | mcd-daily-coupon-hunter              |
| 作者   | Jangyu0407                           |
| 最终版本 | v1.1.0                               |
| 日期   | 2026-10-21                           |
| 用途   | WorkBuddy 专项奖励参赛材料                   |
| 环境   | WorkBuddy（workbuddy.host），已连通麦当劳 MCP |

---

## 概述

本会话从一次「查询麦当劳菜单」的简单需求出发，逐步演进出参赛 Skill「**麦麦当日薅券助手**」的完整开发过程。核心能力：帮用户找出麦当劳**当天折扣力度最大的套餐**和**全场价格最低的套餐**，只做查询比价推荐、不领券不下单。

开发全程动态调用麦当劳 MCP 接口，**零硬编码**——所有门店、套餐、菜品、价格、券面值均来自接口实时返回。

### 最终交付物

| 文件               | 路径                                                                |
| ---------------- | ----------------------------------------------------------------- |
| SKILL.md（v1.1.0） | `C:\Users\LK\.workbuddy\skills\mcd-daily-coupon-hunter\SKILL.md`  |
| README.md        | `C:\Users\LK\.workbuddy\skills\mcd-daily-coupon-hunter\README.md` |
| 本对话记录            | `E:\2026-10-21-09-14-10\workbuddy-dev-chat.md`                    |

---

# 阶段一 · 需求探索：麦当劳 MCP 菜单查询

## 1.1 用户消息

> 调用麦当劳mcp，查询菜单列表

## 1.2 开发过程（工具调用链）

### 步骤 1：加载「菜单查询」工具 Schema

调用 `ToolSearch`，精确加载工具：

```
tool_names: ["mcp__mcdonalds__query-meals"]
```

**关键发现**：`query-meals` 为「餐品列表」接口，入参必填 `storeCode`（门店编码）、`orderType`、`beType`。其中 `storeCode` 不能为空，因此需先定位门店。

### 步骤 2：加载「门店定位」工具 Schema

调用 `ToolSearch`，精确加载工具：

```
tool_names: ["mcp__mcdonalds__query-nearby-stores", "mcp__mcdonalds__delivery-query-stores"]
```

**关键发现**：

- `query-nearby-stores`：到店场景查门店，入参 `searchType`（1=收藏、2=按位置）、`beType`（1=到店自取、5=得来速）、`city`、`keyword`。
- `delivery-query-stores`：外送场景查可配送门店，入参 `addressId`（来自配送地址）+ `beType`（2=麦乐送、6=团餐）。

### 步骤 3：尝试查收藏门店（返回为空）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__query-nearby-stores
{ "searchType": 1, "beType": 1 }
```

**返回**：

```json
{
  "success": false,
  "code": 600050,
  "message": "收藏餐厅列表为空，请您提供城市+关键词进行搜索"
}
```

> 结论：用户无收藏门店，需走「按位置搜索」路径。

### 步骤 4：向用户询问点餐场景与位置

调用 `AskUserQuestion`，询问两个问题：

1. 点餐场景（到店自取 / 麦乐送 / 得来速）
2. 城市 + 位置关键词

**用户回答**：

- 点餐场景 → **到店自取**
- 位置 → **温州**

### 步骤 5：按位置搜索门店（温州）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__query-nearby-stores
{ "beType": 1, "searchType": 2, "city": "温州", "keyword": "温州" }
```

**返回**（5 家门店，按距离排序）：

| storeCode | 门店名称           | 地址                  | 距离    | 营业时间        |
| --------- | -------------- | ------------------- | ----- | ----------- |
| 3330439   | 麦当劳温州火车南站到达层餐厅 | 温州南站1层              | 86m   | 07:00–23:00 |
| 3330198   | 麦当劳温州大象城餐厅     | 潘桥街道宁波路2889号大象城商贸中心 | 683m  | 07:00–21:00 |
| 3330138   | 麦当劳温州大西洋银泰城餐厅  | 娄桥街道瓯海大道1299号       | 1789m | 09:00–22:00 |
| 3330385   | 麦当劳温州盛宏锦园餐厅    | 娄桥街道平天路187号         | 2971m | 00:00–23:59 |
| 3330556   | 麦当劳温州亚运汽车公园餐厅  | 娄桥街道瓯海大道997号        | 3004m | 07:00–22:00 |

> 选择最近门店：**麦当劳温州火车南站到达层餐厅**（storeCode=3330439，距离 86m）。

### 步骤 6：查询菜单

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__query-meals
{ "storeCode": "3330439", "orderType": 1, "beType": 1 }
```

**返回结构**（关键字段）：

- `data.categories[]`：菜单分类（`name` + `meals[].code`）
- `data.meals{}`：餐品详情映射（`name` / `currentPrice` / `originalPrice` / `discountType` / `canWithOrder`）

**返回的完整菜单数据（早餐时段，按分类）**：

- **人气热卖**：厚松饼猪柳蛋堡三件套（¥42.5）、猪柳蛋麦满分套餐（¥38.5）、麦咖啡™奶铁（¥9.9/原价25）
- **早餐汉堡**：东南亚风味椰香鸡扒堡套餐（¥31.5）、椰香鸡扒堡组合（¥24.5）、椰香鸡扒堡（¥14）
- **厚松饼堡系列**：猪柳蛋堡套餐（¥42.5）、猪柳堡套餐（¥38.5）、猪柳蛋堡组合（¥33.5）、猪柳堡组合（¥29.5）、猪柳蛋堡单品（¥23）、猪柳堡单品（¥19）
- **麦满分系列**（套餐）：双层原味板烧鸡腿（¥39.5）、猪柳蛋（¥38.5）、原味板烧鸡腿（¥35.5）、猪柳（¥34.5）、大脆鸡扒（¥32）、火腿扒（¥31）、吉士蛋（¥30）
- **麦满分系列**（单品）：双层猪柳蛋（¥22）、双层原味板烧鸡腿（¥20）、猪柳蛋（¥19）、原味板烧鸡腿（¥16）、猪柳（¥15）、大脆鸡扒（¥12.5）、火腿扒（¥11.5）、吉士蛋（¥10.5）
- **麦咖啡™**：美禄可可雪冰（¥27）、浓浓燕麦黑巧（¥27）、奶铁雪冰（¥27）、浓浓黑巧（¥25）、卡布奇诺（¥25）、燕麦奶（¥25）、浓缩咖啡（¥19）、鲜萃咖啡（¥10.5）、奶铁（¥9.9/原价25）、美式（¥9.9/原价23）、燕麦奶铁（¥9.9/原价27）、热美式/热奶铁中杯（¥9.9）
- **炒双蛋堡系列**：原味板烧鸡腿套餐（¥39）、猪柳套餐（¥39）、猪柳单品（¥19.5）、板烧单品（¥19.5）
- **早安营养卷**：火腿扒套餐（¥36）、图林根香肠套餐（¥36）、火腿扒单品（¥16.5）、图林根单品（¥16.5）
- **小食甜品/其他**：旋旋水手帽（¥48.8）、蓝莓爆爆珠麦旋风（¥17）、泷愿成真麦旋风（¥17）、经典麦旋风（¥16）、玉米杯（¥14.5）、新地（¥14）、派（¥9.5）、德式图林根香肠（¥9.5）、脆薯饼（¥9）、脆香油条（¥9）、灰焰圆筒（¥5）、圆筒冰淇淋（¥5）、脆香油条豆浆组合（¥29.5）
- **饮品**：多汁柠柠（¥12.5）、黄金橙橙（¥14）、优品豆浆（¥10.5）、纯牛奶（¥12）、100%苹果汁（¥12.5）、可乐（¥10）、雪碧（¥10）、锡兰红茶（¥12.5）、冰牛奶（¥23）、怡泉+C（¥13）、阳光柠檬红茶（¥10）、纯悦（¥8.5）、鲜萃咖啡（¥10.5）

> 注：麦咖啡™ 中「¥9.9」的商品 `discountType` 为「随单购早餐卡优惠」，需随单购早餐卡才享此价，否则按原价。

### 步骤 7：输出菜单

助手以表格形式向用户输出完整菜单，并提示可按预算搭配或下单。

---

# 阶段二 · 开发参赛 Skill「麦麦当日薅券助手」

## 2.1 用户需求

> 基于已经连通的麦当劳MCP，开发参赛Skill，技能名字：麦麦当日薅券助手。  
> 核心功能：帮用户找出【当天折扣力度最大的套餐】以及【全场价格最低的套餐】。  
> 要求：
>
> 1. 输出完整SKILL.md，包含：技能名称、触发词、功能描述、MCP调用流程、输出格式说明。
> 2. 必须动态调用MCP接口：查询当日可用优惠券、查询门店菜单，**禁止硬编码任何套餐、菜品、价格**。
> 3. 逻辑流程：①获取优惠券 ②获取菜单 ③计算折扣排序TOP1 ④筛选最低价套餐 ⑤输出
> 4. 需要向用户询问必要信息：门店编码、取餐方式
> 5. 不做下单功能，只做查询比价推荐
> 6. 本次全部开发对话完整保留，导出为 workbuddy-dev-chat.md

## 2.2 摸底 MCP 接口（6 个工具 Schema + 3 次实测）

### 步骤 1：加载优惠券与算价相关工具 Schema

调用 `ToolSearch`，一次性加载 6 个工具：

```
tool_names: [
  "mcp__mcdonalds__available-coupons",
  "mcp__mcdonalds__query-my-coupons",
  "mcp__mcdonalds__query-store-coupons",
  "mcp__mcdonalds__query-promotions",
  "mcp__mcdonalds__query-meal-detail",
  "mcp__mcdonalds__calculate-price"
]
```

**关键结论**（优惠券有三个数据源）：

| 工具                    | 用途                   | 入参                                  | 返回特点                                               |
| --------------------- | -------------------- | ----------------------------------- | -------------------------------------------------- |
| `query-store-coupons` | **指定门店+订单类型下可用券**（主） | storeCode/orderType/beType[,beCode] | title/couponId/couponCode/tradeDateTime/products[] |
| `available-coupons`   | 可领取的麦麦省券             | 无                                   | title（含面值）/状态/图片，无 couponId                        |
| `query-my-coupons`    | 用户卡包已持有券             | page/pageSize                       | 卡包券资产                                              |

**`calculate-price`（精确算价）** 是核心：传入 `items[{productCode, quantity, couponId?, couponCode?}]`，返回含优惠的精确价格，且返回单位为「分」。

### 步骤 2：实测门店券接口（返回为空）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__query-store-coupons
{ "storeCode": "3330439", "orderType": 1, "beType": 1 }
```

**返回**：

```json
{ "success": true, "code": 200, "data": [] }
```

> 结论：这家店当前（早餐时段）无可用门店券，`data` 为空数组。**券接口可能为空，Skill 需处理此情况。**

### 步骤 3：实测可领取券接口（返回 5 张）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__available-coupons
{}
```

**返回**（5 张可领取券，券名含面值）：

1. 19.9元王牌炸鸡任选2（可领取）
2. 11.9元麦辣鸡腿堡（可领取）
3. 免费脆薯饼（可领取）
4. 人气麦旋风买一送一（可领取）
5. 9.9元中杯冰美式（可领取）

> 结论：`available-coupons` 返回的券仅有 `title` + 状态 + 图片，**无 couponId、无适用商品结构**。券面值需从标题解析（如「11.9元…」「免费…」「买一送一」）。

### 步骤 4：实测卡包券接口（返回为空）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__query-my-coupons
{ "page": "1", "pageSize": "20" }
```

**返回**：「暂无可用优惠券」。

### 步骤 5：实测精确算价接口（确认「分」单位）

调用 `DeferExecuteTool`：

```json
// mcp__mcdonalds__calculate-price
{
  "storeCode": "3330439",
  "orderType": 1,
  "beType": 1,
  "items": [{ "productCode": "9900003497", "quantity": 1 }]
}
```

**返回**：

```json
{
  "productOriginalPrice": 3850,
  "productPrice": 3850,
  "originalPrice": 3850,
  "discount": 0,
  "price": 3850,
  "productList": [
    { "productCode": "9900003497", "productName": "猪柳蛋麦满分套餐", "quantity": 1, "originalSubtotal": 3850, "subtotal": 3850 }
  ],
  "takeWayList": [
    { "code": "eat-in", "title": "堂食" },
    { "code": "take-in-store", "title": "外带" }
  ]
}
```

> **关键结论**：
>
> - `calculate-price` 返回字段：`price`（现价）、`originalPrice`（原价）、`discount`（优惠金额），**单位全部为「分」**，展示需 ÷100 转元。
> - 验证：猪柳蛋麦满分套餐 9900003497，3850 分 = ¥38.5，与菜单 `currentPrice` 一致 ✓
> - 配合 `couponId`/`couponCode` 即可精确算出「券后价」。
> - 另有 `enjoyed`/`enjoyable` 字段（amountType：1特价/2折扣/3立减，realDiscount 单位元），可辅助解读优惠类型。

## 2.3 参考现有 Skill 规范格式

先后探索了用户级 skills 目录与 builtin 插件目录，读取了两个现有 Skill 作格式参考：

- `brand-material-alignment/SKILL.md`（简洁版：name + description + agent_created）
- `voice-to-minutes/SKILL.md`（参赛完整版：含 slug/display_name/description_zh/description_en/version/license/author/category/pricing/platforms/allowed-tools/trigger）

**确定采用「参赛完整版」frontmatter 规范**。

## 2.4 生成 SKILL.md v1.0.0

调用 `Write`，生成 `C:\Users\LK\.workbuddy\skills\mcd-daily-coupon-hunter\SKILL.md`。

**v1.0.0 设计要点**：

- frontmatter：name/slug/display_name/description（含触发词）/version/license/author(占位)/category/allowed-tools/trigger
- 正文 8 章：触发场景、功能边界、前置信息收集、MCP 调用流程（6 步）、参数速查、价格单位与陷阱、输出格式、示例输出
- 核心算法：三源聚合券 + 精确算价/标题解析双路径 + 优惠金额/折扣率双口径排序 + 最低价套餐筛选

**v1.0.0 完整内容见【附录 A】**。

---

# 阶段三 · 流程优化：自动定位门店（免输门店编码）

## 3.1 用户优化需求

> 补充优化：刚才的Skill流程里，不要让用户手动输入门店编码。改成：
>
> 1. 先问用户取餐方式（到店自取/外送）和所在城市+位置关键词。
> 2. 到店自取就调用 query-nearby-stores 接口，传入 searchType=2、beType=1、城市、位置关键词，自动返回附近门店列表。
> 3. 外送就调用 delivery-query-stores 接口，根据用户地址返回可配送门店。
> 4. 把门店列表展示给用户选，选中后自动拿到 storeCode 和 beCode，再继续查优惠券和菜单。
> 5. 用户全程不需要知道什么是门店编码，只需要说自己在哪、怎么取餐就行。

## 3.2 确认外送地址接口

调用 `ToolSearch`：

```
tool_names: ["mcp__mcdonalds__delivery-query-addresses"]
```

**关键发现**：`delivery-query-addresses` 无入参，返回用户配送地址列表。外送完整链路为：

```
delivery-query-addresses()  → 用户选地址（或 delivery-create-address 新建）
   → delivery-query-stores(addressId, beType)  → 可配送门店（storeCode + beCode）
```

## 3.3 生成 SKILL.md v1.1.0（重写）

调用 `Write`，重写 `SKILL.md`。

**v1.1.0 相对 v1.0.0 的变更点**：

| 项           | v1.0.0                          | v1.1.0                                                  |
| ----------- | ------------------------------- | ------------------------------------------------------- |
| 前置信息收集      | 问门店编码 + 取餐方式                    | 只问「取餐方式 + 城市/关键词」（**不涉及门店编码**）                          |
| Step 1 定位门店 | 「仅当用户没给 storeCode 时」            | **必走**，按到店/外送分三条分支自动定位                                  |
| 门店选择        | 无明确交互                           | 展示门店列表（名称/地址/距离）让用户选，选中自动拿 storeCode+beCode             |
| 外送链路        | 简单带过                            | 补全 `delivery-query-addresses` → `delivery-query-stores` |
| frontmatter | version 1.0.0，allowed-tools 9 项 | version 1.1.0，allowed-tools 增至 11 项（加地址类工具）             |
| 陷阱第 6 条     | 无                               | 新增「不要向用户索要门店编码」                                         |

**v1.1.0 完整内容见【附录 B】（最终交付版）**。

---

# 阶段四 · 收尾：author + README

## 4.1 用户消息

> author写Jangyu0407 要补一个

## 4.2 执行

1. 调用 `Edit`，将 SKILL.md 的 `author: （请填写作者名）` 改为 `author: Jangyu0407`。
2. 调用 `Write`，生成 `README.md`（功能一览 + 工作流程图 + MCP 数据来源 + 触发词 + 目录结构 + 依赖 + 作者）。

**README.md 完整内容见【附录 C】**。

---

# 阶段五 · 问答：作者名字段位置

## 5.1 用户消息

> skill里作者名在哪里啊

## 5.2 回答

作者名位于 SKILL.md 文件开头的 YAML 头部（frontmatter）第 11 行：`author: Jangyu0407`。README.md 末尾「## 作者」一栏也写了 Jangyu0407，两处一致。

---

# 附录 A · SKILL.md v1.0.0（初版，完整内容）

```markdown
---
name: mcd-daily-coupon-hunter
slug: mcd-daily-coupon-hunter
display_name: 麦麦当日薅券助手
display_name_en: McDonald's Daily Coupon Hunter
description: "帮用户找出麦当劳「当天折扣力度最大的套餐」和「全场价格最低的套餐」，只做查询比价推荐、不领券不下单。基于已连通的麦当劳 MCP 动态调用：查门店可用券（query-store-coupons / available-coupons / query-my-coupons）、查门店菜单（query-meals）、精确算价（calculate-price），实时计算每张券的优惠金额与折扣率并排序。所有套餐名、菜名、价格、券面值均来自接口实时返回，绝不硬编码。当用户说「薅券」「薅羊毛」「今天麦当劳什么最划算」「今日麦当劳优惠」「麦当劳最便宜套餐」「麦当劳折扣最大」「麦当劳比价」「麦当劳优惠券」等时触发。"
description_zh: "帮用户找出麦当劳「当天折扣力度最大的套餐」和「全场价格最低的套餐」，只做查询比价推荐、不领券不下单。基于已连通的麦当劳 MCP 动态调用：查门店可用券（query-store-coupons / available-coupons / query-my-coupons）、查门店菜单（query-meals）、精确算价（calculate-price），实时计算每张券的优惠金额与折扣率并排序。所有套餐名、菜名、价格、券面值均来自接口实时返回，绝不硬编码。当用户说「薅券」「薅羊毛」「今天麦当劳什么最划算」「今日麦当劳优惠」「麦当劳最便宜套餐」「麦当劳折扣最大」「麦当劳比价」「麦当劳优惠券」等时触发。"
description_en: "Find the McDonald's set meal with the biggest discount today and the cheapest set meal on the whole menu — query-and-compare only, no coupon claiming and no ordering. Drives the connected McDonald's MCP dynamically: fetch store-available coupons (query-store-coupons / available-coupons / query-my-coupons), fetch the store menu (query-meals), and compute exact prices (calculate-price), then rank every coupon by savings amount and discount rate. All meal names, item names, prices and coupon face values come from live API responses — nothing is hardcoded. Triggers on phrases like '薅券', '今天麦当劳什么最划算', '麦当劳最便宜套餐', '麦当劳折扣最大', '麦当劳比价'."
version: 1.0.0
license: "MIT"
author: （请填写作者名）
category: life-service
agent_created: true
pricing: 免费
platforms: [WorkBuddy]
allowed-tools:
  - AskUserQuestion
  - mcp__mcdonalds__query-nearby-stores
  - mcp__mcdonalds__delivery-query-stores
  - mcp__mcdonalds__query-store-coupons
  - mcp__mcdonalds__available-coupons
  - mcp__mcdonalds__query-my-coupons
  - mcp__mcdonalds__query-meals
  - mcp__mcdonalds__query-meal-detail
  - mcp__mcdonalds__calculate-price
trigger: ["薅券","薅羊毛","麦当劳优惠","今天麦当劳什么最划算","今日麦当劳优惠","麦当劳最便宜","麦当劳最便宜套餐","麦当劳折扣","麦当劳折扣最大","麦当劳比价","麦当劳券","麦当劳优惠券","薅麦当劳","麦当劳划算","最划算套餐","折扣最大"]
---

# 麦麦当日薅券助手（mcd-daily-coupon-hunter）

> 一句话：连上麦当劳 MCP，实时算出「今天在这家店，用哪张券最划算」和「哪份套餐最便宜」。**只比价、不领券、不下单。**

## 一、触发场景

- 用户问「薅券 / 薅羊毛 / 今天麦当劳什么最划算 / 麦当劳最便宜套餐 / 折扣最大 / 有没有优惠」
- 用户想在某家麦当劳门店下单前，先看当天能省多少
- 用户明确给了门店或地点，说「帮我看看这店今天怎么吃最划算」

**触发门槛（防误触发）**：明确带「优惠 / 券 / 划算 / 便宜 / 折扣 / 比价 / 薅」类意图才执行；纯闲聊、普通点餐、与比价无关的消息**不触发**，正常对话即可。

## 二、功能边界（重要）

| 做 | 不做 |
|---|---|
| 查当日可用券、查菜单、算折扣、比价推荐 | 领券、下单、支付、创建订单 |
| 实时调用 MCP，数据全部来自接口返回 | 硬编码任何套餐名 / 菜名 / 价格 / 券面值 |
| 输出原价、券后价、优惠金额、使用条件 | 替用户决定买不买、下不下单 |

## 三、前置信息收集（必问，缺一不可）

用 `AskUserQuestion` **一次性问清**，避免多轮来回：

1. **门店编码 storeCode**（纯数字）
   - 用户不知道 → 走 Step 1 先定位门店
2. **取餐方式 beType**
   - 到店自取（1）/ 麦乐送（2）/ 得来速（5）/ 团餐（6）
   - 外送类（2 / 6）还需要配送地址 → 走 Step 1 拿地址与 beCode

## 四、MCP 调用流程

### Step 1 · 定位门店（仅当用户没给 storeCode 时）

- **到店 / 得来速**：`mcp__mcdonalds__query-nearby-stores`
  - 入参：`searchType=2`、`city`（城市）、`keyword`（位置关键词）、`beType`
  - 拿到 `storeCode`；得来速（beType=5）额外拿到 `beCode`
- **外送**：先查地址 `mcp__mcdonalds__delivery-query-addresses`，再 `mcp__mcdonalds__delivery-query-stores(addressId, beType)`
  - 拿到 `storeCode` + `beCode`

### Step 2 · 抓当日可用券（三源聚合，禁止硬编码）

1. **主**：`mcp__mcdonalds__query-store-coupons(storeCode, orderType, beType[, beCode])`
   - 返回**该店 + 该取餐方式下当前可用券**，字段：`title`（券名）、`couponId`、`couponCode`、`tradeDateTime`（有效期）、`products[]`（适用商品 `productName` / `productCode`）
2. **辅**：`mcp__mcdonalds__available-coupons()`
   - 返回**可领取的麦麦省券**，字段：`title`（券名，通常含面值）、状态、图片；无 `couponId`
3. **辅**：`mcp__mcdonalds__query-my-coupons(page, pageSize)`
   - 返回**用户卡包已持有的券**

> 三源可能都为空（非营业时段 / 该店无券 / 账号无券）。**为空时如实告知，绝不编造券或优惠。**

### Step 3 · 抓完整菜单

- `mcp__mcdonalds__query-meals(storeCode, orderType, beType[, beCode])`
  - 返回 `categories[]`（分类）+ `meals{}`（每餐品的 `name` / `currentPrice` / `originalPrice` / `discountType` / `canWithOrder`）
- 需要套餐组成时：`mcp__mcdonalds__query-meal-detail(code, ...)`

### Step 4 · 逐券算折扣（核心算法）

对 Step 2 拿到的**每一张券**：

1. **定位适用商品**：券的 `products[].productCode` → 在 `meals{}` 里反查原价 `originalPrice`（无原价时用 `currentPrice`）
2. **求券后价**（两种办法，按可靠度优先）：
   - **精确法（首选，券有 couponId/couponCode 时）**：
     `mcp__mcdonalds__calculate-price(storeCode, orderType, beType, items=[{productCode, quantity:1, couponId, couponCode}])`
     → 返回 `price`（券后价，分）、`originalPrice`（原价，分）、`discount`（优惠金额，分），**÷100 转元**
   - **解析法（兜底，券无 id 仅有标题时）**：从 `title` 解析券后价
     - 「X元 + 商品名」→ 券后 X 元
     - 「免费 + 商品名」→ 券后 0 元
     - 「买一送一」→ 单件半价（原价 × 0.5）
3. **计算指标**：
   - 优惠金额 = 原价 − 券后价
   - 折扣率 = 券后价 ÷ 原价（如 0.63 → 记作 6.3 折）
   - 优惠比例 = 1 − 折扣率
4. **排序取 TOP1**：
   - 主口径「**折扣力度最大**」= **优惠金额降序**（省的钱最多）
   - 备口径「**折扣最深**」= **折扣率升序**（打的折最狠）
   - 两个口径各给 TOP1，并说明差异

### Step 5 · 筛全场最低价套餐（不需券）

1. 在 `meals{}` 里筛「**套餐**」：`tags` 含「套餐 / 三件套 / 二件套」或 `name` 含「套餐 / 组合」
2. **排除需券才享价**的：`discountType` 非空、或 `canWithOrder=true`（随单购价），只保留「直接售卖价」
3. 按 `currentPrice` **升序**，取最低（可给 TOP3 供参考）

> 单品与套餐**分开**。用户要的是「套餐」，不要把单品汉堡 / 小食混进套餐榜单。

### Step 6 · 输出

按下方「输出格式」模板整理成两张表，短句 + 表格，直白汇报。

## 五、参数速查（orderType / beType / beCode 映射）

| 场景 | beType | orderType | beCode |
|---|---|---|---|
| 到店自取 | 1 | 1 | 不传（传了会报错） |
| 得来速 DT | 5 | 1 | 必传（query-nearby-stores 获取） |
| 麦乐送 | 2 | 2 | 必传（delivery-query-stores 获取） |
| 企业团餐 | 6 | 2 | 必传（delivery-query-stores 获取） |

## 六、价格单位与陷阱（踩过的坑）

1. **`calculate-price` 返回单位是「分」**，展示前必须 ÷100 转元。
2. **到店自取不传 beCode**，误传会报错。
3. `query-meals` 的 `currentPrice` 可能是**随单购优惠价**（`canWithOrder=true`，需额外买早餐卡/麦金卡才享）——算「最低价套餐」时要用不依赖随单购的价。
4. 券三源可能都为空，**如实报告，不编造**。
5. **禁止硬编码**：所有套餐名、菜名、价格、券面值必须来自 MCP 返回，不得凭记忆填写。

## 七、输出格式（模板）

### 🎯 今日薅券推荐 · 折扣力度 TOP1

| 项 | 内容 |
|---|---|
| 套餐 | {套餐名}（来自 query-meals） |
| 原价 | ¥{原价} |
| 券后价 | ¥{券后价} |
| 优惠金额 | ¥{原价 − 券后价}（省了 ¥{优惠金额}） |
| 折扣率 | {券后价 ÷ 原价 → 折} |
| 使用券 | {券名} |
| 使用条件 | 有效期 {tradeDateTime}；适用 {products 商品名}；取餐方式 {到店/外送}；{是否需随单购} |

（可并列给出「折扣最深 TOP1」作第二行对照）

### 💰 全场最低价套餐（无需券，直接买）

| 排名 | 套餐 | 价格 |
|---|---|---|
| 1 | {套餐名} | ¥{currentPrice} |
| 2 | … | … |

### 补充说明（可选）
- 券来源说明（门店券 / 可领券 / 卡包券）
- 若某券为「买一送一」「免费」等非直减形式，单独说明换算口径

## 八、示例输出（占位，不含任何真实商品名 / 价格）
```


```
🎯 今日薅券推荐 · 折扣力度 TOP1
套餐：{A 套餐}
原价 ¥{P1} → 券后 ¥{P2}，省 ¥{P1−P2}，约 {折扣} 折
用券：{券名}（{有效期}）
条件：到店自取，适用于 {商品名}；随单购价需另购 {卡名}

💰 全场最低价套餐（无需券）
1. {B 套餐} ¥{P3}
2. {C 套餐} ¥{P4}
3. {D 套餐} ¥{P5}
```
```

---


# 附录 B · SKILL.md v1.1.0（最终交付版，完整内容）

> 以下为最终版全文（`author` 已更新为 Jangyu0407）。

```markdown
---
name: mcd-daily-coupon-hunter
slug: mcd-daily-coupon-hunter
display_name: 麦麦当日薅券助手
display_name_en: McDonald's Daily Coupon Hunter
description: "帮用户找出麦当劳「当天折扣力度最大的套餐」和「全场价格最低的套餐」，只做查询比价推荐、不领券不下单。基于已连通的麦当劳 MCP 动态调用：自动定位门店（query-nearby-stores / delivery-query-stores）、查门店可用券（query-store-coupons / available-coupons / query-my-coupons）、查门店菜单（query-meals）、精确算价（calculate-price），实时计算每张券的优惠金额与折扣率并排序。用户全程只需告诉「取餐方式 + 自己在哪」，无需知道门店编码。所有套餐名、菜名、价格、券面值均来自接口实时返回，绝不硬编码。当用户说「薅券」「薅羊毛」「今天麦当劳什么最划算」「今日麦当劳优惠」「麦当劳最便宜套餐」「麦当劳折扣最大」「麦当劳比价」「麦当劳优惠券」等时触发。"
description_zh: "帮用户找出麦当劳「当天折扣力度最大的套餐」和「全场价格最低的套餐」，只做查询比价推荐、不领券不下单。基于已连通的麦当劳 MCP 动态调用：自动定位门店（query-nearby-stores / delivery-query-stores）、查门店可用券（query-store-coupons / available-coupons / query-my-coupons）、查门店菜单（query-meals）、精确算价（calculate-price），实时计算每张券的优惠金额与折扣率并排序。用户全程只需告诉「取餐方式 + 自己在哪」，无需知道门店编码。所有套餐名、菜名、价格、券面值均来自接口实时返回，绝不硬编码。当用户说「薅券」「薅羊毛」「今天麦当劳什么最划算」「今日麦当劳优惠」「麦当劳最便宜套餐」「麦当劳折扣最大」「麦当劳比价」「麦当劳优惠券」等时触发。"
description_en: "Find the McDonald's set meal with the biggest discount today and the cheapest set meal on the whole menu — query-and-compare only, no coupon claiming and no ordering. Drives the connected McDonald's MCP dynamically: auto-locate the store (query-nearby-stores / delivery-query-stores), fetch store-available coupons (query-store-coupons / available-coupons / query-my-coupons), fetch the store menu (query-meals), and compute exact prices (calculate-price), then rank every coupon by savings amount and discount rate. The user only tells the pickup method and location — no store code needed. All meal names, item names, prices and coupon face values come from live API responses — nothing is hardcoded. Triggers on phrases like '薅券', '今天麦当劳什么最划算', '麦当劳最便宜套餐', '麦当劳折扣最大', '麦当劳比价'."
version: 1.1.0
license: "MIT"
author: Jangyu0407
category: life-service
agent_created: true
pricing: 免费
platforms: [WorkBuddy]
allowed-tools:
  - AskUserQuestion
  - mcp__mcdonalds__query-nearby-stores
  - mcp__mcdonalds__delivery-query-addresses
  - mcp__mcdonalds__delivery-create-address
  - mcp__mcdonalds__delivery-query-stores
  - mcp__mcdonalds__query-store-coupons
  - mcp__mcdonalds__available-coupons
  - mcp__mcdonalds__query-my-coupons
  - mcp__mcdonalds__query-meals
  - mcp__mcdonalds__query-meal-detail
  - mcp__mcdonalds__calculate-price
trigger: ["薅券","薅羊毛","麦当劳优惠","今天麦当劳什么最划算","今日麦当劳优惠","麦当劳最便宜","麦当劳最便宜套餐","麦当劳折扣","麦当劳折扣最大","麦当劳比价","麦当劳券","麦当劳优惠券","薅麦当劳","麦当劳划算","最划算套餐","折扣最大"]
---

# 麦麦当日薅券助手（mcd-daily-coupon-hunter）

> 一句话：连上麦当劳 MCP，实时算出「今天在这家店，用哪张券最划算」和「哪份套餐最便宜」。**只比价、不领券、不下单。**
> 用户体验：**只需说「怎么取餐 + 我在哪」，剩下全自动**——自动定位门店、自动查券查菜单、自动算折扣。

## 一、触发场景

- 用户问「薅券 / 薅羊毛 / 今天麦当劳什么最划算 / 麦当劳最便宜套餐 / 折扣最大 / 有没有优惠」
- 用户想在某家麦当劳门店下单前，先看当天能省多少
- 用户说「帮我看下附近麦当劳今天怎么吃最划算」

**触发门槛（防误触发）**：明确带「优惠 / 券 / 划算 / 便宜 / 折扣 / 比价 / 薅」类意图才执行；纯闲聊、普通点餐、与比价无关的消息**不触发**，正常对话即可。

## 二、功能边界（重要）

| 做 | 不做 |
|---|---|
| 自动定位门店、查当日可用券、查菜单、算折扣、比价推荐 | 领券、下单、支付、创建订单 |
| 实时调用 MCP，数据全部来自接口返回 | 硬编码任何套餐名 / 菜名 / 价格 / 券面值 |
| 输出原价、券后价、优惠金额、使用条件 | 让用户手动输入门店编码 / 替用户下单 |

## 三、前置信息收集（必问，但**不涉及门店编码**）

用 `AskUserQuestion` **一次性问清**，只需两样：

1. **取餐方式 beType**：到店自取（1）/ 麦乐送（2）/ 得来速（5）/ 团餐（6）
2. **位置**（按取餐方式区分）：
   - **到店自取 / 得来速**：城市 + 位置关键词（如「温州 火车南站」）
   - **麦乐送 / 团餐**：配送地址（从地址列表选一个，或新建）

> 用户全程不需要知道、也无需输入「门店编码 storeCode」「业务编码 beCode」——这些由下面的流程自动拿到。

## 四、MCP 调用流程

### Step 1 · 自动定位门店（必走，用户无感知）

按取餐方式分两条分支，拿到门店后**展示列表让用户选**：

**分支 A：到店自取（beType=1）**

- 调 `mcp__mcdonalds__query-nearby-stores`，入参：`searchType=2`、`beType=1`、`city`（城市）、`keyword`（位置关键词）
- 返回附近门店列表（`storeName` / `address` / `distance` / `storeCode` / `reservation` 等）

**分支 B：得来速（beType=5）**

- 调 `mcp__mcdonalds__query-nearby-stores`，入参：`searchType=2`、`beType=5`、`city`、`keyword`
- 返回门店列表，**额外含 `beCode`**（后续调用必须携带）

**分支 C：麦乐送（beType=2）/ 团餐（beType=6）**

- 先调 `mcp__mcdonalds__delivery-query-addresses` 拿配送地址列表 → 用户选地址（无地址则引导 `delivery-create-address` 新建）
- 再调 `mcp__mcdonalds__delivery-query-stores(addressId, beType)` 拿**可配送门店列表**（含 `storeCode` + `beCode`）

**展示门店给用户选**：按距离/营业状态列出（名称、地址、距离、营业时间），引导用户「选第几个」。
**选中后自动得到**：`storeCode`（+ 外送/得来速场景的 `beCode`），无需用户手动输入。

### Step 2 · 抓当日可用券（三源聚合，禁止硬编码）

拿到 storeCode（+ beCode）后：

1. **主**：`mcp__mcdonalds__query-store-coupons(storeCode, orderType, beType[, beCode])`
   - 返回**该店 + 该取餐方式下当前可用券**，字段：`title`（券名）、`couponId`、`couponCode`、`tradeDateTime`（有效期）、`products[]`（适用商品 `productName` / `productCode`）
2. **辅**：`mcp__mcdonalds__available-coupons()`
   - 返回**可领取的麦麦省券**，字段：`title`（券名，通常含面值）、状态、图片；无 `couponId`
3. **辅**：`mcp__mcdonalds__query-my-coupons(page, pageSize)`
   - 返回**用户卡包已持有的券**

> 三源可能都为空（非营业时段 / 该店无券 / 账号无券）。**为空时如实告知，绝不编造券或优惠。**

### Step 3 · 抓完整菜单

- `mcp__mcdonalds__query-meals(storeCode, orderType, beType[, beCode])`
  - 返回 `categories[]`（分类）+ `meals{}`（每餐品的 `name` / `currentPrice` / `originalPrice` / `discountType` / `canWithOrder`）
- 需要套餐组成时：`mcp__mcdonalds__query-meal-detail(code, ...)`

### Step 4 · 逐券算折扣（核心算法）

对 Step 2 拿到的**每一张券**：

1. **定位适用商品**：券的 `products[].productCode` → 在 `meals{}` 里反查原价 `originalPrice`（无原价时用 `currentPrice`）
2. **求券后价**（两种办法，按可靠度优先）：
   - **精确法（首选，券有 couponId/couponCode 时）**：
     `mcp__mcdonalds__calculate-price(storeCode, orderType, beType, items=[{productCode, quantity:1, couponId, couponCode}])`
     → 返回 `price`（券后价，分）、`originalPrice`（原价，分）、`discount`（优惠金额，分），**÷100 转元**
   - **解析法（兜底，券无 id 仅有标题时）**：从 `title` 解析券后价
     - 「X元 + 商品名」→ 券后 X 元
     - 「免费 + 商品名」→ 券后 0 元
     - 「买一送一」→ 单件半价（原价 × 0.5）
3. **计算指标**：
   - 优惠金额 = 原价 − 券后价
   - 折扣率 = 券后价 ÷ 原价（如 0.63 → 记作 6.3 折）
   - 优惠比例 = 1 − 折扣率
4. **排序取 TOP1**：
   - 主口径「**折扣力度最大**」= **优惠金额降序**（省的钱最多）
   - 备口径「**折扣最深**」= **折扣率升序**（打的折最狠）
   - 两个口径各给 TOP1，并说明差异

### Step 5 · 筛全场最低价套餐（不需券）

1. 在 `meals{}` 里筛「**套餐**」：`tags` 含「套餐 / 三件套 / 二件套」或 `name` 含「套餐 / 组合」
2. **排除需券才享价**的：`discountType` 非空、或 `canWithOrder=true`（随单购价），只保留「直接售卖价」
3. 按 `currentPrice` **升序**，取最低（可给 TOP3 供参考）

> 单品与套餐**分开**。用户要的是「套餐」，不要把单品汉堡 / 小食混进套餐榜单。

### Step 6 · 输出

按下方「输出格式」模板整理成两张表，短句 + 表格，直白汇报。

## 五、参数速查（orderType / beType / beCode 映射）

| 场景 | beType | orderType | beCode |
|---|---|---|---|
| 到店自取 | 1 | 1 | 不传（传了会报错） |
| 得来速 DT | 5 | 1 | 必传（query-nearby-stores 获取） |
| 麦乐送 | 2 | 2 | 必传（delivery-query-stores 获取） |
| 企业团餐 | 6 | 2 | 必传（delivery-query-stores 获取） |

## 六、价格单位与陷阱（踩过的坑）

1. **`calculate-price` 返回单位是「分」**，展示前必须 ÷100 转元。
2. **到店自取不传 beCode**，误传会报错。
3. `query-meals` 的 `currentPrice` 可能是**随单购优惠价**（`canWithOrder=true`，需额外买早餐卡/麦金卡才享）——算「最低价套餐」时要用不依赖随单购的价。
4. 券三源可能都为空，**如实报告，不编造**。
5. **禁止硬编码**：所有套餐名、菜名、价格、券面值必须来自 MCP 返回，不得凭记忆填写。
6. **不要向用户索要门店编码**：门店由「城市 + 关键词 / 配送地址」自动定位，再由用户从列表中选，全程无感。

## 七、输出格式（模板）

### 🎯 今日薅券推荐 · 折扣力度 TOP1

| 项 | 内容 |
|---|---|
| 套餐 | {套餐名}（来自 query-meals） |
| 原价 | ¥{原价} |
| 券后价 | ¥{券后价} |
| 优惠金额 | ¥{原价 − 券后价}（省了 ¥{优惠金额}） |
| 折扣率 | {券后价 ÷ 原价 → 折} |
| 使用券 | {券名} |
| 使用条件 | 有效期 {tradeDateTime}；适用 {products 商品名}；取餐方式 {到店/外送}；{是否需随单购} |

（可并列给出「折扣最深 TOP1」作第二行对照）

### 💰 全场最低价套餐（无需券，直接买）

| 排名 | 套餐 | 价格 |
|---|---|---|
| 1 | {套餐名} | ¥{currentPrice} |
| 2 | … | … |

### 补充说明（可选）

- 门店信息（已选门店名称、地址、取餐方式）
- 券来源说明（门店券 / 可领券 / 卡包券）
- 若某券为「买一送一」「免费」等非直减形式，单独说明换算口径

## 八、示例输出（占位，不含任何真实商品名 / 价格）

```
📍 已选门店：{门店名}（{地址}，{到店自取/麦乐送}）

🎯 今日薅券推荐 · 折扣力度 TOP1
套餐：{A 套餐}
原价 ¥{P1} → 券后 ¥{P2}，省 ¥{P1−P2}，约 {折扣} 折
用券：{券名}（{有效期}）
条件：到店自取，适用于 {商品名}；随单购价需另购 {卡名}

💰 全场最低价套餐（无需券）
1. {B 套餐} ¥{P3}
2. {C 套餐} ¥{P4}
3. {D 套餐} ¥{P5}
```
```

---

# 附录 C · README.md（完整内容）

```markdown
# 麦麦当日薅券助手（mcd-daily-coupon-hunter）

> 连上麦当劳 MCP，实时算出「今天在这家店，用哪张券最划算」和「哪份套餐最便宜」——**只比价、不领券、不下单**。

## 一句话体验

用户只需说「**怎么取餐 + 我在哪**」，技能自动完成：定位门店 → 查当日可用券 → 查菜单 → 逐券算折扣 → 输出两张榜单。全程无需知道门店编码。

## 核心功能

| 功能 | 说明 |
|---|---|
| 🎯 折扣力度 TOP1 | 逐券计算「优惠金额」和「折扣率」，排序筛出当天最划算的用券套餐 |
| 💰 全场最低价套餐 | 从菜单里筛出无需券、直接售卖价最低的套餐 |
| 📍 门店自动定位 | 到店按「城市+关键词」、外送按「配送地址」自动返回门店列表供选择 |
| 🔒 只比价不下单 | 纯查询推荐，不做领券/下单/支付 |

## 工作流程

```
用户：取餐方式 + 位置
        ↓
① 定位门店（query-nearby-stores / delivery-query-stores）→ 列表让用户选
        ↓
② 抓当日可用券（query-store-coupons / available-coupons / query-my-coupons）
        ↓
③ 抓完整菜单（query-meals）
        ↓
④ 逐券算折扣（calculate-price 精确算价，兜底从券名解析）
        ↓
⑤ 筛最低价套餐（按 currentPrice 升序，排除需券/随单购价）
        ↓
⑥ 输出两张榜单
```

## 数据来源（全部实时、零硬编码）

所有套餐名、菜名、价格、券面值均来自已连通的**麦当劳 MCP** 接口实时返回：

- `query-nearby-stores` / `delivery-query-stores` — 门店定位
- `delivery-query-addresses` / `delivery-create-address` — 配送地址
- `query-store-coupons` / `available-coupons` / `query-my-coupons` — 优惠券（三源聚合）
- `query-meals` / `query-meal-detail` — 菜单与套餐详情
- `calculate-price` — 精确算价（含券，返回单位「分」）

## 触发词

薅券、薅羊毛、麦当劳优惠、今天麦当劳什么最划算、今日麦当劳优惠、麦当劳最便宜套餐、麦当劳折扣最大、麦当劳比价、麦当劳优惠券……

## 目录结构

```
mcd-daily-coupon-hunter/
├── SKILL.md      # 技能主文件（frontmatter + 完整工作流）
└── README.md     # 本说明
```

## 依赖

- WorkBuddy 环境
- 已连通的麦当劳 MCP（`mcp__mcdonalds__*` 工具）

## 作者

Jangyu0407
```

---

# 附录 D · 涉及的麦当劳 MCP 工具清单

| 工具 | 作用 | 关键入参 | 关键返回字段 |
|---|---|---|---|
| `query-nearby-stores` | 到店/得来速门店定位 | searchType/city/keyword/beType | storeName/address/distance/storeCode/beCode |
| `delivery-query-addresses` | 查询配送地址列表 | 无 | 地址列表（含 addressId） |
| `delivery-create-address` | 新建配送地址 | 地址信息 | 新地址 addressId |
| `delivery-query-stores` | 外送可配送门店 | addressId/beType | storeCode/beCode |
| `query-store-coupons` | 指定门店可用券 | storeCode/orderType/beType[,beCode] | title/couponId/couponCode/tradeDateTime/products[] |
| `available-coupons` | 可领取麦麦省券 | 无 | title/状态/图片 |
| `query-my-coupons` | 用户卡包券 | page/pageSize | 卡包券资产 |
| `query-meals` | 门店菜单 | storeCode/orderType/beType[,beCode] | categories[]/meals{}（name/currentPrice/originalPrice） |
| `query-meal-detail` | 餐品详情/特调 | storeCode/code/orderType/beType | 套餐组成/特调选项 |
| `calculate-price` | 精确算价（含券） | storeCode/orderType/beType/items[] | price/originalPrice/discount（单位：分） |

---

# 附录 E · 关键技术结论（开发沉淀）

1. **券有三个数据源**，结构不同：门店券（有 couponId + 适用商品，可精确算）、可领券（仅标题含面值，需解析）、卡包券（用户资产）。
2. **精确算价靠 `calculate-price`**：传入 `productCode + couponId` 拿「券后价」，返回单位「分」，需 ÷100 转元。
3. **券无 id 时兜底解析标题**：「X元+商品名」「免费+商品名」「买一送一」三类面值口径。
4. **折扣双口径**：优惠金额（省多少钱）降序 = 力度最大；折扣率升序 = 折扣最深。
5. **最低价套餐要排除「随单购价」**：`canWithOrder=true` 或 `discountType` 非空的价不是「直接售卖价」。
6. **参数映射固定**：到店 beType=1→orderType=1 不传 beCode；得来速 5→orderType=1 必传 beCode；麦乐送 2/团餐 6→orderType=2 必传 beCode。
7. **门店自动定位**：到店用「城市+关键词」，外送用「配送地址」，用户无需知道门店编码。

---

*全文完。本记录由 WorkBuddy 会话自动整理生成，覆盖从需求探索到最终交付的全部过程。*
