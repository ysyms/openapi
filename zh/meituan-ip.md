# 美团 IP 精准定位 API

来源：[forum.naixi.net/thread-3705-1-1.html](https://forum.naixi.net/thread-3705-1-1.html)

美团移动端内部接口，无需鉴权。通过美团外卖业务积累的 IP-GPS 关联数据，实现远超常规 IP 库的定位精度（可达 50 米级别）。

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
# 查询指定 IP 的定位
curl "https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip=1.2.3.4"
```

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

# 提取经纬度
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

- 定位精度取决于该 IP 是否有美团用户使用记录，无记录的 IP 精度较低
- 不支持 IPv6
- 如果你通过代理使用过美团 APP，代理 IP 会与你的真实地址绑定
- **隐私建议**：使用代理时避免访问美团等外卖/打车平台，防止 IP 与真实地址关联
