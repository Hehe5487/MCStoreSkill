# 麦当劳门店查询工具参数参考（mcd-mcp）

工具前缀统一为 `mcp__mcd-mcp__`。下方省略前缀，仅列工具名与参数。

## 1. query-nearby-stores（到店门店）

按位置或收藏搜索可点餐门店。

| 参数 | 必填 | 说明 |
|------|------|------|
| beType | 是 | `1`=到店自取；`5`=得来速车道取餐(Drive Through)。beType=5 返回的门店带 beCode，后续调用须传 |
| searchType | 是 | `2`=按位置搜索（需 city+keyword）；`1`=搜索收藏餐厅（默认）。无收藏改用 2 |
| city | searchType=2 时必填 | 城市名，如「上海」 |
| keyword | searchType=2 时必填 | 位置关键词/地址，如「人民广场」 |

返回值要点：`storeCode`（主键）、`beCode`（仅 beType=5）、`reservation`/`reservationTimeOptions`/`today`。

## 2. delivery-query-stores（外送门店）

按配送地址查询可送达门店。**前置**：需先用 `delivery-query-addresses` 取得 `addressId`。到店场景不要用本工具。

| 参数 | 必填 | 说明 |
|------|------|------|
| addressId | 是 | 地址 ID，来源 `delivery-query-addresses` |
| beType | 是 | `2`=麦乐送；`6`=团餐 |

返回值要点：`storeCode`、门店名称。

## 3. query-party-city（派对城市）

活动派对可办城市列表。**前置**：需先确定 `spuId`（来源 `mall-points-products` / `mall-product-detail`）。

| 参数 | 必填 | 说明 |
|------|------|------|
| spuId | 是 | 商品 spuId（用户从可兑换商品列表中选择） |

返回值要点：`data[].code`、`data[].latitude`、`data[].longitude`（供下一步）。

## 4. query-party-store（派对门店）

查询指定城市下支持主题派对的门店列表。

| 参数 | 必填 | 说明 |
|------|------|------|
| code | 是 | 城市 code，来源 `query-party-city` |
| latitude | 否 | 纬度，来源 `query-party-city` 的 `data[].latitude` |
| longitude | 否 | 经度，来源 `query-party-city` 的 `data[].longitude` |
| spuId | 否* | 商品 spuId（建议带上以缩小范围） |

返回值要点：`storeCode`（供下一步）。

## 5. query-party-store-date（派对可预约日期）

查询指定门店可预约的活动日期。**前置**：需 `spuId` 与 `storeCode`（来自 `query-party-store`）。

| 参数 | 必填 | 说明 |
|------|------|------|
| storeCode | 是 | 门店 code，来源 `query-party-store` |
| spuId | 是 | 商品 spuId |

返回值要点：可预约日期列表。

## 调用链路速查

- 到店：`query-nearby-stores`（beType/city/keyword）
- 外送：`delivery-query-addresses` → `delivery-query-stores`（addressId/beType）
- 派对：`query-party-city`（spuId） → `query-party-store`（code/lat/lng/spuId） → `query-party-store-date`（storeCode/spuId）
