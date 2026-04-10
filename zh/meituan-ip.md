# 美团 IP 定位 API

来源：[forum.naixi.net/thread-3705-1-1.html](https://forum.naixi.net/thread-3705-1-1.html)

美团移动端内部接口，无需鉴权。通过美团外卖业务积累的 IP-GPS 关联数据，返回 IP 的大致地理位置。


---

## 原理

美团在用户使用 APP 点外卖时，会将 IP 地址与 GPS 定位、收货地址关联并记录。这些数据被用于风控和智能定位，但相关接口未设置鉴权，可被外部直接调用。

---

## 1. IP 地理定位

根据 IP 地址返回经纬度及省市区信息。

### 请求

```
GET https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip={IP地址}
```

### 参数

| 参数 | 说明 |
|------|------|
| `ip` | 要查询的 IP 地址 |
| `rgeo` | 是否返回逆地理编码信息，设为 `true` |

### 用法示例

```bash
curl "https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip=1.2.3.4"
```

### 返回示例

```json
{
  "data": {
    "lng": 116.6,
    "lat": 35.4,
    "rgeo": {
      "country": "中国",
      "province": "山东省",
      "city": "济宁市",
      "district": "任城区",
      "fromwhere": "mars-mt"
    }
  }
}
```

经纬度仅保留 1 位小数，对应约 11 km × 11 km 的网格。`fromwhere: mars-mt` 表示使用的是粗粒度坐标源。

未被美团收录的 IP（如海外机房段）返回的 `rgeo` 可能为空对象。

---

## 2. 经纬度反查地址

根据经纬度返回详细的地址信息。

### 请求

```
GET https://apimobile.meituan.com/group/v1/city/latlng/{纬度},{经度}?tag=0
```

### 参数

| 参数 | 说明 |
|------|------|
| `纬度,经度` | 路径参数，从第一个接口获取 |
| `tag` | 固定为 `0` |

### 用法示例

```bash
# 先获取经纬度
curl "https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip=1.2.3.4"

# 再用经纬度反查详细地址（假设返回纬度 39.9, 经度 116.3）
curl "https://apimobile.meituan.com/group/v1/city/latlng/39.9,116.3?tag=0"
```

---

## 完整检测脚本

```bash
#!/bin/bash

IP="${1:?用法: $0 <IP地址>}"

echo "=== 第一步：IP 定位 ==="
RESULT=$(curl -s "https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip=${IP}")
echo "$RESULT" | jq .

LAT=$(echo "$RESULT" | jq -r '.data.lat // empty')
LNG=$(echo "$RESULT" | jq -r '.data.lng // empty')

if [ -n "$LAT" ] && [ -n "$LNG" ]; then
  echo ""
  echo "=== 第二步：地址反查 ==="
  curl -s "https://apimobile.meituan.com/group/v1/city/latlng/${LAT},${LNG}?tag=0" | jq .
else
  echo "未能获取经纬度信息"
fi
```

---

## 注意事项

- **精度上限**：经纬度被服务端截断至 1 位小数（约 11 km），即使传 `precise=true`、`accurate=1` 等参数也无效；`v1`、`v3` 版本的路径返回 404；POST 请求返回空
- **数据范围**：国内 IP 基本都能定位到地级市/区县级别；海外 IP 能定位到国家/城市；完全陌生的 IP 段返回空 `rgeo`
- **不支持 IPv6**
- **代理关联风险依然存在**：如果你通过某个代理 IP 使用过美团 APP，该 IP 在美团内部仍被打上你的真实地址标签，只是公开接口看不到具体坐标
- **隐私建议**：使用代理/VPS 时避免访问美团、大众点评、饿了么等外卖/打车/本地生活平台，防止 IP 被持续打标
