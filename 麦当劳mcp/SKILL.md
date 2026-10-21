\---  
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
  \- AskUserQuestion  
  \- mcp\_\_mcdonalds\_\_query-nearby-stores  
  \- mcp\_\_mcdonalds\_\_delivery-query-addresses  
  \- mcp\_\_mcdonalds\_\_delivery-create-address  
  \- mcp\_\_mcdonalds\_\_delivery-query-stores  
  \- mcp\_\_mcdonalds\_\_query-store-coupons  
  \- mcp\_\_mcdonalds\_\_available-coupons  
  \- mcp\_\_mcdonalds\_\_query-my-coupons  
  \- mcp\_\_mcdonalds\_\_query-meals  
  \- mcp\_\_mcdonalds\_\_query-meal-detail  
  \- mcp\_\_mcdonalds\_\_calculate-price  
trigger: ["薅券","薅羊毛","麦当劳优惠","今天麦当劳什么最划算","今日麦当劳优惠","麦当劳最便宜","麦当劳最便宜套餐","麦当劳折扣","麦当劳折扣最大","麦当劳比价","麦当劳券","麦当劳优惠券","薅麦当劳","麦当劳划算","最划算套餐","折扣最大"]  
\---

# 麦麦当日薅券助手（mcd-daily-coupon-hunter）

> 一句话：连上麦当劳 MCP，实时算出「今天在这家店，用哪张券最划算」和「哪份套餐最便宜」。**只比价、不领券、不下单。**  
> 用户体验：**只需说「怎么取餐 + 我在哪」，剩下全自动**——自动定位门店、自动查券查菜单、自动算折扣。

## 一、触发场景

- 用户问「薅券 / 薅羊毛 / 今天麦当劳什么最划算 / 麦当劳最便宜套餐 / 折扣最大 / 有没有优惠」
- 用户想在某家麦当劳门店下单前，先看当天能省多少
- 用户说「帮我看下附近麦当劳今天怎么吃最划算」

**触发门槛（防误触发）**：明确带「优惠 / 券 / 划算 / 便宜 / 折扣 / 比价 / 薅」类意图才执行；纯闲聊、普通点餐、与比价无关的消息**不触发**，正常对话即可。

## 二、功能边界（重要）

| 做                          | 不做                       |
| -------------------------- | ------------------------ |
| 自动定位门店、查当日可用券、查菜单、算折扣、比价推荐 | 领券、下单、支付、创建订单            |
| 实时调用 MCP，数据全部来自接口返回        | 硬编码任何套餐名 / 菜名 / 价格 / 券面值 |
| 输出原价、券后价、优惠金额、使用条件         | 让用户手动输入门店编码 / 替用户下单      |

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

| 场景     | beType | orderType | beCode                       |
| ------ | ------ | --------- | ---------------------------- |
| 到店自取   | 1      | 1         | 不传（传了会报错）                    |
| 得来速 DT | 5      | 1         | 必传（query-nearby-stores 获取）   |
| 麦乐送    | 2      | 2         | 必传（delivery-query-stores 获取） |
| 企业团餐   | 6      | 2         | 必传（delivery-query-stores 获取） |

## 六、价格单位与陷阱（踩过的坑）

1. **`calculate-price` 返回单位是「分」**，展示前必须 ÷100 转元。
2. **到店自取不传 beCode**，误传会报错。
3. `query-meals` 的 `currentPrice` 可能是**随单购优惠价**（`canWithOrder=true`，需额外买早餐卡/麦金卡才享）——算「最低价套餐」时要用不依赖随单购的价。
4. 券三源可能都为空，**如实报告，不编造**。
5. **禁止硬编码**：所有套餐名、菜名、价格、券面值必须来自 MCP 返回，不得凭记忆填写。
6. **不要向用户索要门店编码**：门店由「城市 + 关键词 / 配送地址」自动定位，再由用户从列表中选，全程无感。

## 七、输出格式（模板）

### 🎯 今日薅券推荐 · 折扣力度 TOP1

| 项    | 内容                                                          |
| ---- | ----------------------------------------------------------- |
| 套餐   | {套餐名}（来自 query-meals）                                       |
| 原价   | ¥{原价}                                                       |
| 券后价  | ¥{券后价}                                                      |
| 优惠金额 | ¥{原价 − 券后价}（省了 ¥{优惠金额}）                                     |
| 折扣率  | {券后价 ÷ 原价 → 折}                                              |
| 使用券  | {券名}                                                        |
| 使用条件 | 有效期 {tradeDateTime}；适用 {products 商品名}；取餐方式 {到店/外送}；{是否需随单购} |

（可并列给出「折扣最深 TOP1」作第二行对照）

### 💰 全场最低价套餐（无需券，直接买）

| 排名 | 套餐    | 价格              |
| -- | ----- | --------------- |
| 1  | {套餐名} | ¥{currentPrice} |
| 2  | …     | …               |

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
