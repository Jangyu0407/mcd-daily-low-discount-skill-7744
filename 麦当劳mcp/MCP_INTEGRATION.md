# MCP Integration Document
## Project Overview
This project is a McDonald's MCP Agent skill, which helps users query nearby McDonald's stores, obtain store‑specific available coupons, and check meal information to assist users in selecting food.

## Used MCP Tools
1. `query‑nearby‑stores`：Query nearby McDonald's stores, get storeCode
2. `delivery‑query‑stores`：Query delivery‑support stores
3. `query‑store‑coupons`：Get available coupons for the specified store
4. `query‑meals`：Query meal list and meal details

## Calling Process
1. User asks for nearby stores → call `query‑nearby‑stores` to get store list and storeCode
2. Select target store, call `query‑store‑coupons` with storeCode to fetch store coupons
3. Call `query‑meals` to get meal information of the store
4. Return store, coupon and meal information to user

## Environment Configuration
MCP server address: `https://mcp.mcdonalds.com`
API Key: `YOUR_MCP_TOKEN`

## Test Record
- Successfully obtained nearby store list
- Successfully pulled store‑specific coupon data
- Successfully queried meal information
- No real user data is used, all test data is simulated