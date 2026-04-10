# 美团 HTTPDNS API

美团自建的 HTTPDNS 服务，用于客户端绕过运营商 DNS 劫持、获取 CDN 节点。本身是美团 App 的网络基础设施，对外未设鉴权，可直接调用。

响应除了返回域名的 IPv4/IPv6 解析结果外，还**附带客户端源 IP**（`cip` 字段），可以作为一个免鉴权的 IP 回显接口使用。

---

## 1. 单域名查询

### 请求

```
GET https://httpdns.meituan.com/fetch?dm={域名}&appid=10&platform=android&uuid={任意UUID}&networktunnel=1
```

### 参数

| 参数 | 说明 |
|------|------|
| `dm` | 要查询的域名，一次一个 |
| `appid` | 固定 `10`（美团主 App） |
| `platform` | `android` / `ios`，可省略 |
| `uuid` | 设备 UUID，任意 64 位十六进制字符串即可 |
| `networktunnel` | `1` 或 `2`，表示网络栈标识，非必须 |

### 返回

```json
{
  "domain": "httpdnsvip.meituan.com",
  "ipv4": ["103.37.155.60"],
  "ipv6": ["2405:1480:2000:3::5"],
  "ttl": 180,
  "state": "success",
  "cip": "1.2.3.4"
}
```

| 字段 | 说明 |
|------|------|
| `domain` | 查询的域名 |
| `ipv4` / `ipv6` | 解析到的地址列表 |
| `ttl` | 缓存有效期（秒） |
| `state` | `success` / `fail` |
| `cip` | **请求发起者的公网 IP**（由服务端从 TCP 连接取得，原样回显） |

---

## 2. 多域名批量查询

一次请求可查多个域名，用分号分隔。

### 请求

```
GET http://httpdnsmultiapivip.meituan.com/multifetch?dms={域名1};{域名2};{域名3}&appid=10&uuid={UUID}&networktunnel=1
```

### 用法示例

```bash
curl "http://httpdnsmultiapivip.meituan.com/multifetch?dms=apimobile.meituan.com;mapi.meituan.com;wmapi-mt.meituan.com&appid=10&uuid=0000000000000000000000000000000000000000000000000000000000000000"
```

### 返回

```json
{
  "cip": "1.2.3.4",
  "data": {
    "apimobile.meituan.com": {
      "ttl": 60,
      "state": "success",
      "ipv4": ["43.175.253.200", "43.175.252.200"]
    },
    "mapi.meituan.com": {
      "ttl": 60,
      "state": "success",
      "ipv4": ["..."]
    }
  }
}
```

---

## 3. 可查询的常见美团域名

以下域名均可通过 HTTPDNS 查询（从抓包观察到的部分）：

```
apimobile.meituan.com     # 定位等老接口
mapi.meituan.com          # 主业务 API 网关
wmapi-mt.meituan.com      # 外卖 API
scapi.waimai.meituan.com  # 外卖商家 API
mars.meituan.com          # 用户行为上报
portal-portm.meituan.com  # 门户
mrn.meituan.net           # React Native 业务包
ddplus.meituan.net        # 到店业务资源
img.meituan.net           # 图片 CDN
p0.meituan.net / p1.meituan.net   # 图片 CDN
w.meituan.net / layout.meituan.net # 静态资源
bike.meituan.com          # 美团单车
ordercenter.meituan.com   # 订单中心
mtcart.meituan.com        # 购物车
s3plus.meituan.net        # 对象存储

# QUIC 加速节点
mrn-quic.meituan.net
img-quic.meituan.net
w-quic.meituan.net
```

---

## 4. 用途

### 查询自己的出口 IP

最简用法：

```bash
curl -s "https://httpdns.meituan.com/fetch?dm=httpdnsvip.meituan.com&appid=10&uuid=0000000000000000000000000000000000000000000000000000000000000000" | jq -r .cip
```

输出：
```
1.2.3.4
```

和 `ifconfig.me`、`ipv4.ping0.cc` 等效，区别是走美团的国内节点，国内访问速度快。

### 获取美团 CDN 节点实时 IP

如果客户端或爬虫需要绕过本地 DNS 污染、直连美团某个域名，可以通过这个接口拿到美团官方下发的 CDN IP 列表，再自己把 `Host` 头写死请求。

```bash
# 拿 apimobile.meituan.com 的实时 IP
IP=$(curl -s "https://httpdns.meituan.com/fetch?dm=apimobile.meituan.com&appid=10&uuid=0000000000000000000000000000000000000000000000000000000000000000" | jq -r '.ipv4[0]')

# 用拿到的 IP 直连，不走系统 DNS
curl -s --resolve "apimobile.meituan.com:443:${IP}" \
  "https://apimobile.meituan.com/locate/v2/ip/loc?rgeo=true&ip=1.2.3.4"
```

---

## 注意事项

- `cip` 只会回显**请求发起者自身的公网 IP**，不支持查询任意 IP 的信息
- 响应中**不含地理位置字段**，IP 到经纬度需要配合 [美团 IP 定位 API](meituan-ip.md) 使用
- 多域名接口对 `dms` 参数的域名数量没有明确上限，抓包观察到一次请求 20 个域名可正常返回
- 请求频率不宜过高，虽然目前无鉴权但有 IP 级风控
- `uuid` 字段校验很宽，任意 64 位十六进制字符串都接受
