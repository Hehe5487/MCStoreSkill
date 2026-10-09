# 麦当劳门店查询 (mcd-store-query)

## 项目介绍

mcd-store-query 是一个基于麦当劳官方 MCP（`mcd-mcp`）「门店查询」能力开发的 WorkBuddy Skill。它帮用户快速找到附近的麦当劳门店——按城市 / 位置关键词查到店自取或得来速门店、按配送地址查麦乐送 / 团餐外送范围，以及查主题派对 / 生日派对可举办的门店。

**为什么做这个：** 麦当劳 `mcd-mcp` 提供了完整的门店查询能力（`query-nearby-stores` / `delivery-query-stores` / `query-party-*`），但不同场景对应不同工具、不同参数（`beType`、`searchType`、`addressId`、`spuId`、城市 `code` 等），很容易用错或漏传必填项（例如得来速的 `beCode`、派对链路缺失的 `spuId`）。本 Skill 把这套「场景路由 + 参数约束 + 调用链路」固化下来，让用户只说一句话就能查到正确的店。

## 目标用户

- 想吃麦当劳、想找最近门店或某位置门店的普通用户
- 想用麦乐送 / 团餐外送、需先确认配送范围的人
- 想办主题派对 / 生日派对、需要先查哪些门店支持的家长或活动组织者

## 安装方法

### 方式一：作为 WorkBuddy 技能安装（推荐）

1. **配置 mcd-mcp 连接器**（见官方 README「WorkBuddy 开发指南」）：左侧边栏【专家 · 技能 · 连接器】→【连接器】→【自定义连接器】→【配置 MCP】，填入：

   ```json
   {
     "mcpServers": {
       "mcd-mcp": {
         "type": "streamablehttp",
         "url": "https://mcp.mcd.cn",
         "headers": {
           "Authorization": "Bearer YOUR_MCP_TOKEN"
         }
       }
     }
   }
   ```

   并将 `mcd-mcp` 启用（`YOUR_MCP_TOKEN` 替换为你的麦当劳 MCP Token）。

2. **安装本技能**：将本仓库 `mcd-store-query/` 目录放到 WorkBuddy 技能扫描目录：

   ```bash
   # 方式 A：直接复制
   cp -r mcd-store-query ~/.workbuddy/skills/mcd-store-query

   # 方式 B：软链（便于跟随仓库更新）
   ln -sfn "$(pwd)/mcd-store-query" ~/.workbuddy/skills/mcd-store-query
   ```

3. 刷新 / 重启 WorkBuddy，技能面板即可看到「麦当劳门店查询」。

### 方式二：直接对话触发

在已接入 `mcd-mcp` 的 WorkBuddy 会话中，直接说「查附近的麦当劳 / 这个地址能送到哪 / 主题派对哪些门店能办」，AI 会按 Skill 流程引导。

## 使用示例

- 用户：「我附近有哪些麦当劳？」→ 调用 `query-nearby-stores`（`beType=1` 到店自取，`searchType=2` 按位置），展示门店与 `storeCode`。
- 用户：「我想驾车去麦当劳」→ 调用 `query-nearby-stores`（`beType=5` 得来速），返回带 `beCode` 的门店，后续调用自动带上 `beCode`。
- 用户：「这个地址能送到哪些店？」→ 先 `delivery-query-addresses` 取 `addressId`，再 `delivery-query-stores`（`beType=2` 麦乐送 / `6` 团餐）查可配送门店。
- 用户：「主题派对在哪些门店能办？」→ 链路 `query-party-city`（`spuId`）→ `query-party-store`（城市 `code`/经纬度）→ `query-party-store-date`（可预约日期），逐级引导选择。

## 场景一览

Skill 按用户意图路由到对应工具（截图可放 `docs/` 目录补充）：

| 场景 | 触发话术举例 | 主工具 | 关键参数 |
|------|------|--------|---------|
| 到店自取 | 「附近有哪些麦当劳」 | `query-nearby-stores` | `beType=1`、`searchType=2`、`city`+`keyword` |
| 得来速 | 「驾车去麦当劳」 | `query-nearby-stores` | `beType=5`、返回带 `beCode` |
| 麦乐送 / 团餐 | 「这个地址能送哪」 | `delivery-query-stores` | `addressId`、`beType=2`/`6` |
| 派对门店 | 「主题派对哪些店能办」 | `query-party-city` → `query-party-store` → `query-party-store-date` | `spuId`、城市 `code`/经纬度、`storeCode` |

**到店确认卡** — 展示门店列表、`storeCode`，得来速额外展示 `beCode`，引导用户选择门店。

**外送范围卡** — 展示该地址可配送门店列表，引导选择。

**派对链路卡** — 城市 → 门店 → 可预约日期，逐级收敛到具体场次。

## 合规与说明

- 本 Skill 严格遵守 `mcd-mcp` 门店查询接口的调用约定，区分到店 / 外送 / 派对场景，不混用工具。
- 得来速（`beType=5`）返回门店的 `beCode` 在后续工具调用中必须携带；派对链路必须先确定 `spuId` 与城市 `code`/经纬度，不可手填。
- 返回值以 `storeCode` 为主键完整展示给用户，供后续点餐 / 外送流程复用。
- 项目输出仅供参考，门店 / 配送范围 / 派对场次以麦当劳官方实时数据为准。
