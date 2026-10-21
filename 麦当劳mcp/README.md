# 麦麦当日薅券助手（mcd-daily-coupon-hunter）

> 连上麦当劳 MCP，实时算出「今天在这家店，用哪张券最划算」和「哪份套餐最便宜」——**只比价、不领券、不下单**。

## 一句话体验

用户只需说「**怎么取餐 + 我在哪**」，技能自动完成：定位门店 → 查当日可用券 → 查菜单 → 逐券算折扣 → 输出两张榜单。全程无需知道门店编码。

## 核心功能

| 功能           | 说明                               |
| ------------ | -------------------------------- |
| 🎯 折扣力度 TOP1 | 逐券计算「优惠金额」和「折扣率」，排序筛出当天最划算的用券套餐  |
| 💰 全场最低价套餐   | 从菜单里筛出无需券、直接售卖价最低的套餐             |
| 📍 门店自动定位    | 到店按「城市+关键词」、外送按「配送地址」自动返回门店列表供选择 |
| 🔒 只比价不下单    | 纯查询推荐，不做领券/下单/支付                 |

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
