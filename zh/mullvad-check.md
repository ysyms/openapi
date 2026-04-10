# Mullvad 连接检测 API

来源：[mullvad.net/en/check](https://mullvad.net/en/check)

Mullvad 提供的连接检测服务，所有接口无需鉴权，可直接调用。

---

## 1. IP 信息查询

查询访问者或指定 IP 的地理位置和相关信息。

### 请求

```
GET https://am.i.mullvad.net/json
```

### 响应

```json
{
  "ip": "1.2.3.4",
  "country": "Example Country",
  "city": "Example City",
  "longitude": 0.0,
  "latitude": 0.0,
  "mullvad_exit_ip": false,
  "blacklisted": {
    "blacklisted": false,
    "results": []
  },
  "organization": "Example ISP"
}
```

### 其他格式

| 端点 | 返回内容 |
|------|---------|
| `GET /ip` | 纯文本 IP 地址 |
| `GET /country` | 国家名称 |
| `GET /city` | 城市名称 |
| `GET /json` | 完整 JSON |

### 用法示例

```bash
# 查询当前 IP 的完整信息
curl https://am.i.mullvad.net/json

# 仅获取 IP
curl https://am.i.mullvad.net/ip

# 仅获取国家
curl https://am.i.mullvad.net/country
```

---

## 2. DNS 泄漏检测

检测当前网络的 DNS 请求从哪些服务器发出，判断 DNS 是否走了代理。

### 原理

1. 服务端生成随机子域名（UUID），302 重定向到 `{uuid}.dnsleak.*.mullvad.net`
2. 客户端浏览器解析该域名，DNS 请求最终到达 Mullvad 的权威 DNS 服务器
3. Mullvad 记录是哪个 DNS 递归解析器来查询的
4. 客户端请求结果，服务端返回记录到的 DNS 服务器列表

### 请求

```
GET https://am.i.mullvad.net/dnsleak
Accept: application/json
```

> **注意**：必须带 `Accept: application/json` 请求头，否则返回纯文本。

### 响应

```json
[
  {
    "ip": "1.2.3.4",
    "mullvad_dns": false,
    "city": "Example City",
    "country": "Example Country",
    "organization": "Example DNS Provider"
  },
  {
    "ip": "5.6.7.8",
    "mullvad_dns": false,
    "country": "Example Country",
    "organization": "Example DNS Provider 2"
  }
]
```

### 判断规则

- 如果列表中出现你真实 ISP（如中国电信/移动/联通）→ **DNS 泄漏**
- 如果全部是代理节点所在地的 DNS 服务器 → **无泄漏**

### 用法示例

```bash
# 检测 DNS 泄漏
curl -H "Accept: application/json" https://am.i.mullvad.net/dnsleak

# 格式化输出
curl -s -H "Accept: application/json" https://am.i.mullvad.net/dnsleak | jq .
```

---

## 3. WebRTC 泄漏检测

检测浏览器是否通过 WebRTC 暴露真实 IP。**此功能无后端 API，纯前端实现。**

### 原理

浏览器的 `RTCPeerConnection` 在建立连接时会通过 STUN 服务器获取本机 IP 候选地址（ICE Candidate），即使使用了代理/VPN，WebRTC 仍可能暴露真实的本地或公网 IP。

### 实现代码

```javascript
function detectWebRTCLeak() {
  return new Promise((resolve) => {
    const ips = new Set();
    const pc = new RTCPeerConnection({
      iceServers: [{ urls: "stun:stun.l.google.com:19302" }]
    });

    pc.createDataChannel("");
    pc.onicecandidate = (e) => {
      if (!e.candidate) {
        pc.close();
        resolve([...ips]);
        return;
      }
      const match = e.candidate.candidate.match(
        /([0-9]{1,3}(\.[0-9]{1,3}){3}|[a-f0-9]{1,4}(:[a-f0-9]{1,4}){7})/
      );
      if (match) {
        const ip = match[1];
        // 过滤掉内网地址，只关注公网 IP
        if (!ip.startsWith("10.") && !ip.startsWith("192.168.") && !ip.startsWith("172.")) {
          ips.add(ip);
        }
      }
    };

    pc.createOffer()
      .then((offer) => pc.setLocalDescription(offer));

    // 超时兜底
    setTimeout(() => {
      pc.close();
      resolve([...ips]);
    }, 5000);
  });
}

// 使用
detectWebRTCLeak().then((ips) => {
  if (ips.length === 0) {
    console.log("未检测到 WebRTC 泄漏");
  } else {
    console.log("WebRTC 泄漏的 IP:", ips);
  }
});
```

### 判断规则

- 如果检测到的 IP 与代理出口 IP 一致 → **无泄漏**
- 如果检测到非代理的公网 IP → **WebRTC 泄漏**
- 如果没有检测到任何 IP → **WebRTC 已禁用，无泄漏**

---

## 完整检测脚本

以下 Bash 脚本可一键完成 IP 和 DNS 检测（WebRTC 需浏览器环境）：

```bash
#!/bin/bash

echo "=== IP 信息 ==="
curl -s https://am.i.mullvad.net/json | jq .

echo ""
echo "=== DNS 泄漏检测 ==="
curl -s -H "Accept: application/json" https://am.i.mullvad.net/dnsleak | jq .

echo ""
echo "=== WebRTC 检测 ==="
echo "WebRTC 泄漏检测需要在浏览器中执行，无法通过命令行完成。"
echo "请在浏览器控制台中运行上方的 JavaScript 代码，或访问 https://mullvad.net/en/check"
```

---

## 注意事项

- 所有接口返回的是**请求发起者**的信息，不支持查询任意 IP 的 DNS/WebRTC 泄漏
- DNS 泄漏检测依赖 Mullvad 的权威 DNS 服务器，原理上无法自行复制（除非自建权威 DNS）
- 接口无频率限制（截至发现时），但请合理使用
