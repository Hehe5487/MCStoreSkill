---
name: mcd-store-query
description: 基于麦当劳 MCP（mcd-mcp）查询麦当劳门店位置与可点餐门店。当用户询问"我附近有哪些麦当劳""查一下XX市/XX位置的麦当劳门店""这个地址能送到哪些店""主题派对在哪些城市的门店举办"等场景时使用。This skill should be used when the user wants to find McDonald's stores by location, city, keyword, or delivery address, or to look up party-capable stores.
agent_created: true
---

# McDonald's 门店查询（mcd-mcp）

## Overview

本 Skill 封装麦当劳官方 MCP（`mcd-mcp`）的门店查询工具，覆盖三大场景：

1. **到店**：到店自取 / 得来速（Drive Through）车道取餐 —— 按城市+位置关键词搜索附近门店。
2. **外送**：麦乐送 / 团餐 —— 按用户配送地址查询可送达的门店。
3. **派对门店**：主题派对 / 生日派对 / 麦麦体验营等活动的可举办门店、城市与可预约日期。

所有工具都属于 `mcd-mcp` 服务，调用方式见下文「工具调用约定」。详细参数见 `references/store-tools.md`。

## 工具调用约定

- 工具前缀统一为 `mcp__mcd-mcp__`（例如 `mcp__mcd-mcp__query-nearby-stores`）。
- 通过 **DeferExecuteTool** 调用：先用 ToolSearch 加载对应工具 schema（如尚未加载），再用 DeferExecuteTool 传参执行。
- 这些工具依赖用户鉴权与会话上下文，调用前需确认 `mcd-mcp` 已连接且用户已登录麦当劳账号。
- 返回结果中的 `storeCode`（门店编码）是后续流程的主键，务必完整展示给用户。

## 场景路由（先判断再调用）

| 用户意图 | 场景 | 主工具 | 关键参数 |
|---------|------|--------|---------|
| "我附近/想驾车去/到店自提" | 到店 | `query-nearby-stores` | `beType=1`（自取）或 `beType=5`（得来速） |
| "这个地址能送到哪/麦乐送/团餐" | 外送 | `delivery-query-stores` | `addressId` + `beType=2`(麦乐送)/`6`(团餐) |
| "主题派对/生日派对哪些门店能办" | 派对 | `query-party-store`（需先 `query-party-city`） | `code`/`latitude`/`longitude`/`spuId` |

判断不清时，先用一句话向用户确认场景与城市/位置/地址。

## 场景一：到店门店查询（query-nearby-stores）

适用：用户想找可点餐的到店门店。

**参数规则**
- `beType`（必填）：`1`=到店自取；`5`=得来速车道取餐（Drive Through）。
- `searchType`：`2`=按位置搜索（**门店位置查询走这个**，需 `city`+`keyword`）；`1`=搜索用户收藏餐厅（无收藏时报错，改用 searchType=2）。
- `city` / `keyword`：searchType=2 时必填，例如 `city="上海"`、`keyword="人民广场"` 或具体地址。

**调用示例**
```
query-nearby-stores: { beType: 1, searchType: 2, city: "上海", keyword: "人民广场" }
query-nearby-stores: { beType: 5, searchType: 2, city: "北京", keyword: "国贸" }
```

**结果处理**
- 展示每个门店的 `storeCode`；当 `beType=5` 时，得来速门店还返回 `beCode`，**后续工具调用必须传 beCode**。
- 若 `reservation=true`，完整展示 `reservationTimeOptions`；`today=true` 的标注"（今天）"。
- 引导用户选定门店，记录其 `storeCode`（及得来速的 `beCode`）供后续点餐流程使用。

## 场景二：外送门店查询（delivery-query-stores）

适用：用户已选配送地址，想看该地址能送到哪些麦当劳门店。

**前置条件**
- 必须先取得 `addressId`：调用 `mcp__mcd-mcp__delivery-query-addresses` 列出用户地址，让用户选择其一。
- 本工具**仅限外送场景**；到店场景请用 `query-nearby-stores`。

**参数规则**
- `addressId`（必填）：来源 `delivery-query-addresses` 的地址 ID。
- `beType`（必填）：`2`=麦乐送；`6`=团餐。

**调用示例**
```
delivery-query-stores: { addressId: "ADDR_123", beType: 2 }
```

**结果处理**
- 展示可配送门店的 `storeCode` 与门店名称，引导用户选择。

## 场景三：派对门店查询（主题/生日派对）

适用：用户想找可举办主题派对、生日派对、麦麦体验营、品鉴会、读书会、积分兑换活动的门店。

**调用链（必须按顺序，存在前置依赖）**
1. `query-party-city`：先确定 `spuId`（来自 `mall-points-products` 或 `mall-product-detail` 让用户选择商品），返回城市列表及每个城市的 `code`/`latitude`/`longitude`。
2. `query-party-store`：用上一步的城市 `code`、`latitude`、`longitude` 与 `spuId`，返回该城市支持派对的门店列表（含 `storeCode`）。
3. `query-party-store-date`：用户选定门店后，用 `storeCode`+`spuId` 查询该门店可预约日期。

**关键依赖**
- `spuId` 贯穿整个派对流程，必须先从 `mall-points-products` / `mall-product-detail` 获取。
- 城市经纬度与 `code` 来自 `query-party-city`，不要手填。

**结果处理**
- 展示城市列表 → 门店列表（含 storeCode）→ 可预约日期，逐级引导用户选择。

## 错误处理

- `query-nearby-stores` 返回无收藏记录：提示用户改用位置搜索（searchType=2）。
- 外送场景提示「地址无法配送」：引导用户更换/新增配送地址（`delivery-query-addresses`）。
- 派对流程缺 `spuId`：先回到商品选择步骤补全 `spuId` 再继续。
- 任何工具返回鉴权/会话错误：提示用户确认 mcd-mcp 已连接且已登录麦当劳账号。

## Resources

- `references/store-tools.md`：各门店查询工具的完整参数表与返回值要点，调用前可快速核对。
