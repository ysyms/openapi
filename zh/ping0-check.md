# ping0.cc IP 查询与网络工具 API

来源：[ping0.cc](https://ping0.cc)

ping0.cc 提供 IP 查询、反查域名、网络延迟测试等服务。部分接口可直接 curl 调用，无需鉴权。

---

## 1. 获取当前 IP

### 请求

```
GET ping0.cc
```

### 返回

```
1.2.3.4
```

### 变体

| 接口 | 说明 |
|------|------|
| `curl ping0.cc` | 自动判断 IPv4/IPv6 |
| `curl -4 ping0.cc` 或 `curl ipv4.ping0.cc` | 仅 IPv4 |
| `curl -6 ping0.cc` 或 `curl ipv6.ping0.cc` | 仅 IPv6 |

---

## 2. 获取 IP 详细信息

### 请求

```
GET ping0.cc/geo
```

### 返回

```
1.2.3.4
美国 加利福尼亚州 洛杉矶
AS12345
Example ISP
```

4 行纯文本：IP 地址、地理位置、ASN 编号、组织名称。

### 变体

| 接口 | 说明 |
|------|------|
| `curl ping0.cc/geo` | 自动判断 |
| `curl -4 ping0.cc/geo` 或 `curl ipv4.ping0.cc/geo` | 仅 IPv4 |
| `curl -6 ping0.cc/geo` 或 `curl ipv6.ping0.cc/geo` | 仅 IPv6 |

---

## 3. JSONP 回调

适用于前端页面直接获取访问者 IP 信息。

### 请求

```
GET https://ping0.cc/geo/jsonp/callback
```

### 返回

```javascript
callback('1.2.3.4', '美国 加利福尼亚州 洛杉矶', 'AS12345', 'Example ISP')
```

### 前端用法

```html
<script>
function callback(ip, location, asn, org) {
  console.log(ip, location, asn, org);
}
</script>
<script src="https://ping0.cc/geo/jsonp/callback"></script>
```

---

## 4. 签名档图片

可嵌入论坛签名，自动显示访问者 IP 信息。

| 接口 | 说明 | 格式 |
|------|------|------|
| `https://ping0.cc/img1` | 详细版签名档 | SVG |
| `https://ping0.cc/img2` | 简洁版签名档 | SVG |

### 用法

```html
<img src="https://ping0.cc/img1" />
```

---

## 5. IP 反查域名（网页）

查询某个 IP 或 IP 段绑定过的所有域名（被动 DNS 数据）。

```
https://ping0.cc/revip?ip={IP地址}
https://ping0.cc/revip?ip={IP地址/24}
https://ping0.cc/revip?ip={IP地址}&p={页码}
```

> **注意**：该接口返回 HTML 页面，非 JSON API。数据来源为被动 DNS 数据库。

---

## 6. 网络工具（网页）

以下功能均为网页端操作，非 API 接口。

| 功能 | 地址 |
|------|------|
| IP 详情查询 | `https://ping0.cc/ip/{IP地址}` |
| 反向 DNS 查询 | `https://ping0.cc/reverse` |
| Ping 测试 | `https://ping0.cc/ping` |
| Traceroute | `https://ping0.cc/trace` |
| 延迟测试 | `https://ping0.cc/latency` |
| 端口扫描 | `https://ping0.cc/port` |
| IP 泄漏检测 | `http://ipv4.ping0.cc/ipleak` |
| ASN 变化监控 | `https://ping0.cc/Ip/asnmon` |
| VPS 监控 | `https://ping0.cc/vpsmon/24hour` `5day` `30day` `300day` |

---

## 用法示例

```bash
# 查询当前 IP
curl ping0.cc

# 查询详细信息
curl ping0.cc/geo

# 仅获取 IPv4
curl ipv4.ping0.cc

# PHP 获取 IP
$ip = file_get_contents('http://ipv4.ping0.cc');
```

---

## 注意事项

- 免费 API 返回的是**请求发起者自己的信息**，不支持查询任意 IP
- 查询任意 IP 的详细信息需使用付费接口（¥0.1/次）
- 反查域名等网页功能受 Cloudflare 保护，浏览器访问可能需要过验证码
- 命令行 API（ping0.cc/geo 等）不受 Cloudflare 验证码限制
