# Apple App Store 公开 API

来源：[developer.apple.com](https://developer.apple.com/library/archive/documentation/AudioVideo/Conceptual/iTuneSearchAPI/index.html) / [rss.marketingtools.apple.com](https://rss.marketingtools.apple.com/)

Apple 官方提供的三组公开接口，**无需鉴权，无需 API Key**，可直接调用。

> **并发建议：100 并发 + aiohttp 连接池复用**，实测 ~400 req/s，成功率 99%。超过 200 并发后成功率开始下降。

---

## 1. 应用搜索 API

按关键词搜索 App Store 应用。

### 请求

```
GET https://itunes.apple.com/search
```

### 参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `term` | 是 | 搜索关键词 |
| `country` | 否 | 国家/地区代码（默认 US），如 `CN` `JP` `HK` |
| `entity` | 否 | 内容类型，App 用 `software`（默认全类型） |
| `limit` | 否 | 返回数量，最大 `200`（默认 50） |
| `lang` | 否 | 返回语言，如 `zh_cn` |

### 示例

```bash
# 搜索 "procreate"，美区，最多 10 个结果
curl "https://itunes.apple.com/search?term=procreate&country=US&entity=software&limit=10"
```

### 响应

```json
{
  "resultCount": 3,
  "results": [
    {
      "trackId": 916366645,
      "trackName": "Procreate Pocket",
      "bundleId": "com.savage.procreate",
      "price": 5.99,
      "currency": "USD",
      "formattedPrice": "$5.99",
      "primaryGenreName": "Entertainment",
      "version": "4.3.9",
      "currentVersionReleaseDate": "2024-11-01T00:00:00Z",
      "trackViewUrl": "https://apps.apple.com/us/app/procreate-pocket/id916366645"
    }
  ]
}
```

---

## 2. Top 榜单 API

获取各区 App Store 排行榜，Apple 官方 RSS Feed。

### 请求

```
GET https://rss.marketingtools.apple.com/api/v2/{country}/apps/{feed}/100/apps.json
```

### 参数

| 参数 | 说明 |
|------|------|
| `country` | 国家/地区代码，如 `us` `cn` `jp` `hk` `tw` `gb` |
| `feed` | 榜单类型，见下表 |

**榜单类型（feed）**

| 值 | 说明 |
|----|------|
| `top-free` | 免费榜 |
| `top-paid` | 付费榜 |
| `new-apps-we-love` | 新上榜推荐 |
| `new-games-we-love` | 新游戏推荐 |

> 每次最多返回 100 条。

### 示例

```bash
# 美区 Top 100 免费 App
curl "https://rss.marketingtools.apple.com/api/v2/us/apps/top-free/100/apps.json"

# 中国区 Top 25 付费 App
curl "https://rss.marketingtools.apple.com/api/v2/cn/apps/top-paid/25/apps.json"
```

### 响应

```json
{
  "feed": {
    "title": "Top Free Apps",
    "results": [
      {
        "id": "932747118",
        "name": "Shadowrocket",
        "genres": [
          { "genreId": "6002", "name": "Utilities" }
        ]
      }
    ]
  }
}
```

> 榜单接口只返回 `id` 和 `name`，不含价格。需配合 Lookup API 查询价格。

---

## 3. 应用详情 / 价格 Lookup API

按 App ID 查询指定地区的详细信息，**含本地货币价格**。

### 请求

```
GET https://itunes.apple.com/lookup
```

### 参数

| 参数 | 必填 | 说明 |
|------|------|------|
| `id` | 是 | App 的 trackId（数字） |
| `country` | 否 | 国家/地区代码（默认 US） |

### 示例

```bash
# 查询 Procreate Pocket 在日区的价格
curl "https://itunes.apple.com/lookup?id=916366645&country=JP"

# 多区价格对比（需多次请求）
curl "https://itunes.apple.com/lookup?id=916366645&country=US"
curl "https://itunes.apple.com/lookup?id=916366645&country=CN"
curl "https://itunes.apple.com/lookup?id=916366645&country=HK"
```

### 响应（关键字段）

```json
{
  "resultCount": 1,
  "results": [
    {
      "trackId": 916366645,
      "trackName": "Procreate Pocket",
      "bundleId": "com.savage.procreate",
      "price": 1000.0,
      "currency": "JPY",
      "formattedPrice": "¥1,000",
      "version": "4.3.9",
      "sellerName": "Savage Interactive Pty Ltd",
      "primaryGenreName": "Entertainment",
      "averageUserRating": 4.8,
      "userRatingCount": 12345,
      "trackViewUrl": "https://apps.apple.com/jp/app/id916366645",
      "currentVersionReleaseDate": "2024-11-01T00:00:00Z"
    }
  ]
}
```

---

## 多区价格对比示例

```python
import asyncio
import aiohttp

COUNTRIES = ['US', 'CN', 'JP', 'HK', 'TW', 'GB', 'KR', 'SG', 'AU', 'DE']

async def get_price(session, app_id: int, country: str) -> dict:
    url = f'https://itunes.apple.com/lookup?id={app_id}&country={country}'
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as resp:
            data = await resp.json(content_type=None)
            if data.get('resultCount', 0) > 0:
                r = data['results'][0]
                return {
                    'country': country,
                    'price': r.get('price'),
                    'currency': r.get('currency'),
                    'formatted': r.get('formattedPrice'),
                }
    except:
        pass
    return {'country': country, 'price': None}

async def compare_prices(app_id: int):
    # 最佳并发：100，实测 ~400 req/s，成功率 99%
    # 超过 200 并发后成功率开始下降
    connector = aiohttp.TCPConnector(ssl=True, limit=100)
    async with aiohttp.ClientSession(connector=connector) as session:
        tasks = [get_price(session, app_id, c) for c in COUNTRIES]
        results = await asyncio.gather(*tasks)

    for r in sorted(results, key=lambda x: x['price'] or 9999):
        if r['price'] is not None:
            print(f"{r['country']:4s}  {r['formatted']}")

asyncio.run(compare_prices(916366645))
```

输出示例：
```
TW    $190.00
HK    HK$48.00
CN    ¥38.00
JP    ¥1,000
US    $5.99
```

---

## 限流说明

| 客户端方案 | 并发 | 成功率 | 速率 |
|-----------|------|--------|------|
| urllib（每次新建连接） | 10 | ~60% | — |
| aiohttp（连接池复用） | 100 | 99% | ~400 req/s |
| aiohttp | 200 | 99% | ~500 req/s |
| aiohttp | 300 | 88% | — |
| aiohttp | 1000 | 5% | — |

- Apple **不返回 429**，限流方式是直接断 TLS 连接
- 失败可直接重试，无封禁惩罚
- 关键：**复用连接池**（keep-alive），避免频繁 TLS 握手触发限制
