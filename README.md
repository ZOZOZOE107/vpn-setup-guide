# 零基础自建 VPN 完全指南：一台云服务器 + 一个 AI 助手，四协议全搞定

> 适用人群：完全不懂服务器的普通人。全程跟着做即可，预计耗时 1–2 小时，月成本 $0（新用户有 $100 免费额度）。
>
> 本文分为两部分：**上半部是人做的**（注册账号、点几下鼠标），**下半部是给 AI 助手的操作手册**（把它粘贴给 Kimi / Claude / ChatGPT 等支持终端操作的 AI，它会自动帮你装好）。

## ⚠️ 使用前必读

- 本教程仅用于**个人学习与技术研究**。请遵守你所在地区的法律法规，不得用于非法目的，不得向他人提供翻墙服务（在很多地区这是刑事风险）。
- 云服务按量计费，请按教程设置**预算警报**，避免产生意外账单。
- 文中所有 `<尖括号>` 内容请替换成你自己的值；**所有密码请自己重新生成，不要照抄任何示例**。

---

# 上半部：人来做（约 30 分钟）

## 第一步：注册 AWS 账号

1. 打开 https://aws.amazon.com → 创建免费账户
2. 填邮箱、设密码 → 验证邮箱
3. **绑定信用卡**（Visa/Mastercard，国内部分银行发行的外币卡/双币卡可用；会预授权 $1 验证，不会真扣）
4. 完成身份验证（可能要求上传证件或视频认证，按提示来）
5. 选择支持计划：选 **Basic（免费）** 就够了
6. 注册成功后，新账户通常有 **$100 服务抵扣金**（约 6 个月有效），本文整套配置的费用都从抵扣金里出

## 第二步：创建服务器（EC2 实例）

1. 控制台顶部搜索框输入 **EC2**，进入 EC2 Dashboard
2. 右上角确认区域：建议选**亚太地区（东京）**或**美国西部（俄勒冈）**——东京延迟低，俄勒冈最便宜
3. 点「启动实例」，按下面填：
   - **名称**：随便写，比如 `my-vpn`
   - **系统镜像（AMI）**：选 **Amazon Linux 2023**
   - **实例类型**：选 **t3.micro**（1核1G，免费额度内）
   - **密钥对**：选「**在没有密钥对的情况下继续**」（我们用更安全的方式注入密钥，见下）
   - **网络设置**：保持默认（会自动创建安全组）
4. 点「启动实例」，等 1 分钟，状态变绿「正在运行」

## 第三步：开放端口（安全组）

1. 点进实例详情页 → 找到**安全组**（蓝色 `sg-` 开头的链接）点进去
2. 切到「入站规则」标签 → 点「编辑入站规则」，添加：

| 类型 | 协议 | 端口 | 来源 |
|---|---|---|---|
| 自定义 TCP | TCP | 443 | 0.0.0.0/0 |
| 自定义 TCP | TCP | 8443 | 0.0.0.0/0 |
| 自定义 TCP | TCP | 39090 | 0.0.0.0/0 |
| 自定义 UDP | UDP | 39090 | 0.0.0.0/0 |
| 自定义 UDP | UDP | 36712 | 0.0.0.0/0 |
| 自定义 TCP | TCP | 2095 | 我的 IP |
| 自定义 TCP | TCP | 2096 | 我的 IP |

（SSH 22 默认已有，保留。）

## 第四步：记下两个信息

- 实例详情页的 **公有 IPv4 地址**（例如 `13.x.x.x`，动态分配，后面会换成固定的）
- 实例 ID（例如 `i-0xxxxxxxxxxxxx`）

## 第五步：把下半部粘贴给你的 AI 助手

打开 Kimi / Claude Code / 任何支持执行终端命令的 AI 助手，对它说：

> 我想自建一个 VPN，服务器已经开好了。请阅读我发你的部署手册，严格按手册的阶段一步步执行：需要我在 AWS 控制台操作的步骤，请明确告诉我点哪里、选什么，等我确认完成后再继续。完成后把所有账号密码和订阅链接整理给我。

然后把**下半部全文**发给它。它会先要你的服务器 IP，然后全程带你走完。

---

# 下半部：给 AI 助手的操作手册

> 如果你是收到本文的 AI 助手：**先通读全文再动手**。本文已按真实部署验证，标注【坑】处是踩过并解决的坑，务必遵守。

## 0. 角色与交互协议（先读我）

**你的角色**：你是一名 DevOps 工程师，同时是一位耐心的中文向导。用户是不懂技术的普通人。

**执行纪律**：

1. **一次只做一步**，做完一步、验证通过，再做下一步。每步都带「验证门」——看到预期的输出才算过，不过就查附录排错表。
2. **凡是需要用户在 AWS 网页控制台操作的，必须使用「控制台指引」方框里的话术，原样逐条转述给用户**（可以改写得更口语，但步骤顺序、菜单名称、选项名称不得变）。一次最多发 5 小步，等用户回复"好了"或截图确认后再发下一步。
3. **向用户索取的信息只有两样**：① 服务器公有 IPv4 地址；② 实例 ID（固定 IP 阶段才需要）。除此之外所有密码、密钥都由你用脚本自动生成并打印，**绝不要让用户在聊天里发密码**，也不要让用户在控制台手动设置任何密码。
4. 所有服务器上的配置都通过 **§3 的一键脚本**完成，不要自己推导 API 调用、不要改用网页面板点选（面板操作容易漏项且无法复现）。
5. 用户的本机环境是 Windows 或 macOS 都可能：终端命令如果在用户本机执行，优先给两条都适用的写法；SSH 相关命令一律在用户本机执行，服务器上的命令一律通过 `ssh -i <私钥> ec2-user@<IP> '<命令>'` 远程执行。

**部署目标**：一台 VPS 上的 **s-ui 面板**（v1.6.x，sing-box 内核），四个入站：

| 协议 | 端口 | 特点 |
|---|---|---|
| VLESS + Reality | 443/tcp | 抗封锁最强，日常主力 |
| Trojan + Reality | 8443/tcp | 备用 |
| Shadowsocks 2022 | 39090/tcp+udp | 轻量 |
| Hysteria2 | 36712/udp | 弱网/高延迟下通常最快 |

最终交付物：面板地址与管理员密码、订阅地址（普通 / Clash / JSON 三种格式）、四条分享链接、SSH 私钥文件。

---

## 1. 阶段一：建立连接（约 10 分钟）

### 1.1 在你本机生成 SSH 密钥对

```bash
ssh-keygen -t ed25519 -f ./vpn_key -N "" -C "vpn-setup"
cat ./vpn_key.pub
```

验证门：`cat ./vpn_key.pub` 能打印出一行 `ssh-ed25519 AAAA... vpn-setup`。

### 1.2 让用户把公钥注入服务器（控制台操作，按话术转述）

> **发给用户的话术（控制台操作）**：
> 1. 打开 AWS 控制台，顶部搜索框输入 **EC2**，进入 EC2 Dashboard
> 2. 左侧点「实例」，勾选你的实例 → 点上方「连接」按钮
> 3. 选 **EC2 Instance Connect** 标签页 → 点橙色「连接」按钮，会打开一个黑色网页终端
> 4. 把下面这一整行粘贴进去（包含引号），回车，看到 `OK` 就是成功了：
>
> ```bash
> echo "<把刚才打印的公钥整行粘贴到这里>" >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys && echo OK
> ```

验证门：用户回复看到 `OK`。

### 1.3 验证免密登录

```bash
ssh -i ./vpn_key -o StrictHostKeyChecking=accept-new ec2-user@<IP> 'whoami'
```

验证门：打印 `ec2-user`。（Amazon Linux 用户名是 `ec2-user`；若用户选了 Ubuntu 镜像则是 `ubuntu`。sudo 免密。）

### 1.4 系统初始化

```bash
ssh -i ./vpn_key ec2-user@<IP> 'sudo dnf update -y && sudo tee /etc/sysctl.d/99-bbr.conf <<EOF
net.core.default_qdisc = fq
net.ipv4.tcp_congestion_control = bbr
EOF
sudo sysctl --system'
```

验证门：最后打印 `net.ipv4.tcp_congestion_control = bbr`。

---

## 2. 阶段二：安装面板（约 5 分钟）

```bash
ssh -i ./vpn_key ec2-user@<IP> 'curl -Ls https://raw.githubusercontent.com/alireza0/s-ui/master/install.sh -o /tmp/sui-install.sh && sudo bash /tmp/sui-install.sh'
```

默认面板端口 **2095**、路径 **`/app/`**。安装输出会显示随机初始用户名和密码，不用记——下一步脚本会重置。

【坑】设置管理员密码要用二进制程序，wrapper 脚本 `s-ui admin` 会进菜单不生效；密码别含 `$`/`#`（多层 shell 引号会吃掉）。下一步的脚本已规避这两点，不要自行改用手动方式。

---

## 3. 阶段三：一键配置（约 2 分钟）

把下面整个脚本保存为 `sui_bootstrap.py`（scp 上传，或 `cat > /tmp/sui_bootstrap.py <<'PYEOF'` 整段粘贴），然后运行：

```bash
scp -i ./vpn_key sui_bootstrap.py ec2-user@<IP>:/tmp/
ssh -i ./vpn_key ec2-user@<IP> 'python3 /tmp/sui_bootstrap.py <IP>'
```

（最后一个参数 `<IP>` 换成服务器公网 IP；可加第二个参数作为订阅名，默认 `vpnuser`。）

脚本行为：自动生成全新管理员密码 → 登录面板 → 创建 API Token → 清空白板 → 建 3 个 TLS 对象 + 4 个入站 + 1 个客户端 → 打印全部交付信息。**幂等，可重复运行**；之后换弹性 IP 直接带上新 IP 重跑即可全量重建。

<details>
<summary>点击展开：sui_bootstrap.py 完整脚本</summary>

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
s-ui 一键配置脚本（在服务器上以普通用户运行，sudo 免密）。
用法:  python3 sui_bootstrap.py <服务器公网IP> [订阅名]
前置:  s-ui 面板已通过 install.sh 安装完成。
行为:  生成全新随机管理员密码 -> 登录 -> 创建 Token -> 清空白板
       -> 建 3 个 TLS 对象 + 4 个入站 + 1 个客户端 -> 打印面板/订阅/链接。
特性:  幂等，可重复运行；换 IP 后带新 IP 重跑即可全量重建。
"""
import base64, http.cookiejar, json, secrets, subprocess, sys, time
import urllib.parse, urllib.request, uuid

if len(sys.argv) < 2:
    sys.exit("用法: python3 sui_bootstrap.py <服务器公网IP> [订阅名]")
HOST = sys.argv[1]
NAME = sys.argv[2] if len(sys.argv) > 2 else "vpnuser"

BASE = "http://localhost:2095/app"
SUB = "http://localhost:2096/sub"


def die(msg):
    sys.exit("[失败] " + msg)


# ---------- 0. 设置管理员密码 ----------
# 【坑】必须用二进制程序设置；wrapper 脚本 `s-ui admin` 会进菜单不生效。
# 【坑】密码不能含 $ #（多层 shell 引号会吃掉）；token_urlsafe 只产字母数字和 - _，天然安全。
ADMIN_PASS = secrets.token_urlsafe(12)
subprocess.run(["sudo", "/usr/local/s-ui/sui", "admin",
                "-username", "admin", "-password", ADMIN_PASS], check=True)
subprocess.run(["sudo", "systemctl", "restart", "s-ui"], check=True)

# ---------- 1. 登录拿 session ----------
# 【坑】必须带 Origin 头否则 403；参数是 form 不是 JSON。
cj = http.cookiejar.CookieJar()
opener = urllib.request.build_opener(urllib.request.HTTPCookieProcessor(cj))


def post_form(url, form):
    req = urllib.request.Request(
        url, data=urllib.parse.urlencode(form).encode(),
        headers={"Content-Type": "application/x-www-form-urlencoded",
                 "Origin": "http://localhost:2095"})
    with opener.open(req, timeout=15) as r:
        return json.loads(r.read().decode())


out = None
for _ in range(10):  # 面板重启需要时间，重试登录
    try:
        out = post_form(BASE + "/api/login", {"user": "admin", "pass": ADMIN_PASS})
        if out.get("success"):
            break
    except Exception:
        pass
    time.sleep(3)
if not out or not out.get("success"):
    die("面板登录失败，请确认 s-ui 已安装: journalctl -u s-ui -n 50 --no-pager")

# ---------- 2. 创建 API Token ----------
# 【坑】接口失败也返回 HTTP 200，必须检查响应 JSON 的 success 字段。
out = post_form(BASE + "/api/addToken", {"expiry": 0, "desc": "bootstrap"})
if not out.get("success"):
    die("创建 API Token 失败: %s" % out.get("msg"))


def find_token(x):
    """在 addToken 响应里递归找一个长得像 token 的长字符串。"""
    if isinstance(x, str) and len(x) >= 20 and x.isalnum():
        return x
    if isinstance(x, dict):
        for v in x.values():
            t = find_token(v)
            if t:
                return t
    if isinstance(x, list):
        for v in x:
            t = find_token(v)
            if t:
                return t
    return None


TOKEN = find_token(out.get("obj")) or find_token(out)
if not TOKEN:
    die("未能从 addToken 响应中解析出 token，原始响应: %s" % json.dumps(out, ensure_ascii=False))


def req(path, form=None):
    data, headers = None, {"Token": TOKEN}
    if form is not None:
        data = urllib.parse.urlencode(form).encode()
        headers["Content-Type"] = "application/x-www-form-urlencoded"
    r = urllib.request.Request(BASE + path, data=data, headers=headers)
    with urllib.request.urlopen(r, timeout=15) as resp:
        return json.loads(resp.read().decode())


def save(obj, action, data, check=True):
    out = req("/apiv2/save", {"object": obj, "action": action, "data": json.dumps(data)})
    if check and not out.get("success"):
        die("save %s/%s 失败: %s" % (obj, action, out.get("msg")))
    return out


def tls_by_name():
    return {t["name"]: t["id"] for t in req("/apiv2/tls")["obj"]["tls"]}


# ---------- 3. 清空白板 ----------
# 【坑】inbounds 删除 data 是 tag 的 JSON 字符串（传 id 会静默无效）。
# 【坑】保存入站时核心实时绑定端口，删除后 sleep 3 秒再重建，否则 address already in use。
for i in list(req("/apiv2/inbounds")["obj"]["inbounds"]):
    save("inbounds", "del", i["tag"])
time.sleep(3)
for _name, tid in list(tls_by_name().items()):
    req("/apiv2/save", {"object": "tls", "action": "del", "data": str(tid)})

# ---------- 4. 生成全部凭据 ----------
vless_uuid = str(uuid.uuid4())
trojan_pass = secrets.token_urlsafe(16)
hy2_pass = secrets.token_urlsafe(16)
ss_server_key = base64.b64encode(secrets.token_bytes(16)).decode()  # 16字节，带 padding
ss_user_key = base64.b64encode(secrets.token_bytes(16)).decode()


def reality_keys():
    d = {}
    for line in req("/apiv2/keypairs?k=reality")["obj"]:
        if ": " in line:
            k, v = line.split(": ", 1)
            d[k] = v.strip()
    return d["PrivateKey"], d["PublicKey"]


def tls_pems():
    # 返回 (私钥PEM, 证书PEM)。响应是 PEM 行数组，先私钥块后证书块。
    lines = [l for l in req("/apiv2/keypairs?k=tls")["obj"] if l.strip()]
    blocks, cur = [], []
    for l in lines:
        cur.append(l)
        if l.strip().endswith("-----") and "END" in l:
            blocks.append("\n".join(cur) + "\n")
            cur = []
    return blocks[0], blocks[1]


v_priv, v_pub = reality_keys()
t_priv, t_pub = reality_keys()
v_sid = secrets.token_hex(8)
t_sid = secrets.token_hex(8)
hy2_key, hy2_cert = tls_pems()

# ---------- 5. 创建 3 个 TLS 对象 ----------
save("tls", "new", {
    "name": "Reality-VLESS",
    "server": {"enabled": True, "server_name": "www.microsoft.com", "alpn": ["h2", "http/1.1"],
               "reality": {"enabled": True,
                           "handshake": {"server": "www.microsoft.com", "server_port": 443},
                           "private_key": v_priv, "short_id": [v_sid]}},
    "client": {"enabled": True, "server_name": "www.microsoft.com", "alpn": ["h2", "http/1.1"],
               "utls": {"enabled": True, "fingerprint": "chrome"},
               "reality": {"enabled": True, "public_key": v_pub, "short_id": v_sid}},
})
save("tls", "new", {
    "name": "Reality-Trojan",
    "server": {"enabled": True, "server_name": "www.amazon.com", "alpn": ["h2", "http/1.1"],
               "reality": {"enabled": True,
                           "handshake": {"server": "www.amazon.com", "server_port": 443},
                           "private_key": t_priv, "short_id": [t_sid]}},
    "client": {"enabled": True, "server_name": "www.amazon.com", "alpn": ["h2", "http/1.1"],
               "utls": {"enabled": True, "fingerprint": "chrome"},
               "reality": {"enabled": True, "public_key": t_pub, "short_id": t_sid}},
})
save("tls", "new", {
    "name": "SelfSigned-Hysteria2",
    "server": {"enabled": True, "server_name": "www.bing.com", "alpn": ["h3"],
               "certificate": [hy2_cert], "key": [hy2_key]},
    "client": {"enabled": True, "server_name": "www.bing.com", "alpn": ["h3"], "insecure": True},
})
tids = tls_by_name()
tid_v, tid_t, tid_h = tids["Reality-VLESS"], tids["Reality-Trojan"], tids["SelfSigned-Hysteria2"]


def reality_client_tls(sni, pub, sid):
    return {"enabled": True, "server_name": sni, "alpn": ["h2", "http/1.1"],
            "utls": {"enabled": True, "fingerprint": "chrome"},
            "reality": {"enabled": True, "public_key": pub, "short_id": sid}}


# ---------- 6. 创建 4 个入站 ----------
# 【坑】save 响应返回受影响对象的完整列表，取 id 必须按 tag 匹配，不能取 list[0]。
# 【坑】shadowsocks 不要传 network 字段（默认 tcp+udp），传任何值都可能报 unknown network。
ids = []


def add_inbound(payload):
    out = save("inbounds", "new", payload)
    objs = out["obj"]
    lst = objs["inbounds"] if isinstance(objs, dict) else objs
    for item in lst:
        if item.get("tag") == payload["tag"]:
            ids.append(item["id"])
            return
    die("入站 %s 创建后未在响应列表中找到" % payload["tag"])


add_inbound({
    "type": "vless", "tag": "VLESS-Reality", "tls_id": tid_v,
    "listen": "::", "listen_port": 443,
    "users": [{"name": NAME, "uuid": vless_uuid, "flow": "xtls-rprx-vision"}],
    "addrs": [{"server": HOST, "server_port": 443, "remark": ""}],
    "out_json": {"type": "vless", "server": HOST, "server_port": 443, "uuid": vless_uuid,
                 "flow": "xtls-rprx-vision",
                 "tls": reality_client_tls("www.microsoft.com", v_pub, v_sid)},
})
add_inbound({
    "type": "trojan", "tag": "Trojan-Reality", "tls_id": tid_t,
    "listen": "::", "listen_port": 8443,
    "users": [{"name": NAME, "password": trojan_pass}],
    "addrs": [{"server": HOST, "server_port": 8443, "remark": ""}],
    "out_json": {"type": "trojan", "server": HOST, "server_port": 8443, "password": trojan_pass,
                 "tls": reality_client_tls("www.amazon.com", t_pub, t_sid)},
})
add_inbound({
    "type": "shadowsocks", "tag": "SS2022", "tls_id": 0,
    "listen": "::", "listen_port": 39090,
    "method": "2022-blake3-aes-128-gcm", "password": ss_server_key,
    "addrs": [{"server": HOST, "server_port": 39090, "remark": ""}],
    "out_json": {"type": "shadowsocks", "server": HOST, "server_port": 39090,
                 "method": "2022-blake3-aes-128-gcm",
                 "password": ss_server_key + ":" + ss_user_key},
})
add_inbound({
    "type": "hysteria2", "tag": "Hysteria2", "tls_id": tid_h,
    "listen": "::", "listen_port": 36712,
    "users": [{"name": NAME, "password": hy2_pass}],
    "addrs": [{"server": HOST, "server_port": 36712, "remark": ""}],
    "out_json": {"type": "hysteria2", "server": HOST, "server_port": 36712, "password": hy2_pass,
                 "tls": {"enabled": True, "server_name": "www.bing.com", "alpn": ["h3"],
                         "insecure": True}},
})

# ---------- 7. 创建客户端（订阅） ----------
# 【坑】clients 新建必须带 "links": []，否则 unexpected end of JSON input。
# 【坑】clients 删除 data 是 id 的 JSON 数字（如 "1"）。
for c in req("/apiv2/clients")["obj"]["clients"] or []:
    req("/apiv2/save", {"object": "clients", "action": "del", "data": str(c["id"])})

save("clients", "new", {
    "enable": True, "name": NAME, "remark": NAME, "inbounds": sorted(ids),
    "volume": 0, "expiry": 0, "links": [],
    "config": {
        "vless": {"name": NAME, "uuid": vless_uuid, "flow": "xtls-rprx-vision"},
        "trojan": {"name": NAME, "password": trojan_pass},
        "shadowsocks16": {"name": NAME, "password": ss_user_key},
        "hysteria2": {"name": NAME, "password": hy2_pass},
    },
})

# ---------- 8. 取回订阅链接并打印交付信息 ----------
time.sleep(2)
with urllib.request.urlopen(SUB + "/" + NAME, timeout=15) as resp:
    links = base64.b64decode(resp.read().decode()).decode().splitlines()

print(json.dumps({
    "host": HOST,
    "panel": "http://%s:2095/app/  用户名 admin  密码 %s" % (HOST, ADMIN_PASS),
    "subscription_plain": "http://%s:2096/sub/%s" % (HOST, NAME),
    "subscription_clash": "http://%s:2096/sub/%s?format=clash" % (HOST, NAME),
    "subscription_json": "http://%s:2096/sub/%s?format=json" % (HOST, NAME),
    "links": links,
}, indent=2, ensure_ascii=False))
print("\n[提醒] 以上密码和链接只显示这一次，请完整复制保存。面板登录后建议立即修改密码。")
```

</details>

验证门：脚本最后打印一段 JSON，包含 `panel`、`subscription_clash` 和 4 条 `links`。缺任何一项都算失败，查附录排错表。

---

## 4. 阶段四：验证（约 5 分钟）

### 4.1 服务器侧验证

```bash
ssh -i ./vpn_key ec2-user@<IP> 'ss -tlnp | grep -E ":(443|8443|39090|2095|2096)" && ss -ulnp | grep -E ":(39090|36712)" && journalctl -u s-ui -n 20 --no-pager | grep -i error; echo EXIT=$?'
```

验证门：5 个 TCP 端口（443/8443/39090/2095/2096）+ 2 个 UDP 端口（39090/36712）都在监听；journalctl 无 error。

### 4.2 外部连通性验证（分布式探测）

【坑】**不要用用户本机 nc/curl 判断连通性**——若用户开着 TUN 模式的代理工具，本地 TCP 连接会被虚拟网卡假接受，全部显示成功。用 check-host.net 分布式外部探测：

```bash
R=$(curl -s -H "Accept: application/json" "https://check-host.net/check-tcp?host=<IP>:443&max_nodes=5")
sleep 12
curl -s -H "Accept: application/json" "https://check-host.net/check-result/$(echo $R | python3 -c 'import sys,json;print(json.load(sys.stdin)["request_id"])')"
```

验证门：返回的节点结果里多数为 `"ok"`。全 timeout 但用户本机通 → 用户开着代理，以本结果为准。

### 4.3 用户侧验证（让用户操作，按话术转述）

> **发给用户的话术（客户端操作）**：
> 1. 下载 **Clash Verge**（Windows/Mac 都有，GitHub 搜 clash-verge-releases 或官网 clash-verge.dev）
> 2. 打开 Clash Verge → 左侧「订阅」→ 顶部输入框粘贴你的 **Clash 订阅地址**（就是带 `?format=clash` 的那个）→ 点「导入」
> 3. 回到「代理」页，能看到 4 个节点（VLESS / Trojan / Shadowsocks / Hysteria2），随便点一个，打开浏览器访问 google.com 试试
> 4. 四个都点一遍，哪个快用哪个（一般 Hysteria2 在弱网下最快，VLESS-Reality 最稳）
>
> 如果导入时报 invalid yaml，检查订阅地址末尾有没有 `?format=clash`（Clash 系客户端必须用这个格式）。

---

## 5. 固定 IP（强烈建议，约 5 分钟）

EC2 默认公网 IP **重启即变**，IP 一变订阅地址就失效。建议换弹性 IP：

> **发给用户的话术（控制台操作）**：
> 1. 控制台顶部搜索框输入 **弹性 IP**（Elastic IP），进入
> 2. 点「分配弹性 IP 地址」→ 网络边界组保持默认 → 「分配」
> 3. 勾选刚分配的 IP → 点「操作」→「关联弹性 IP 地址」
> 4. 实例那里选你的实例 →「关联」
> 5. 把新的 IP 告诉我（实例详情页的"公有 IPv4 地址"已经变成新 IP 了）

注意：**未关联实例的弹性 IP 会持续计费**（约 $0.005/小时），不用就释放。

拿到新 IP 后，你只需要重跑阶段三（`python3 /tmp/sui_bootstrap.py <新IP>`），脚本会自动全量重建并打印新的订阅地址。然后：

> **发给用户的话术（更新订阅）**：你的订阅地址变了，在 Clash Verge 里把旧订阅删掉，重新导入我发你的新地址就行。

---

## 6. 交付清单（向用户交付并提醒记录）

- [ ] 面板地址 + 管理员账号密码（提醒用户登录后改掉）
- [ ] 订阅地址（普通版 + `?format=clash` 版 + `?format=json` 版）
- [ ] 四条分享链接（存成文件交给用户）
- [ ] SSH 私钥文件 `./vpn_key`（丢失只能重新注入公钥）
- [ ] 提醒用户：AWS 控制台建**预算警报**（控制台搜 Budgets → 创建预算 →「零支出预算」→ 填邮箱，超出免费额度会邮件提醒）
- [ ] 提醒用户：$100 抵扣金有效期约 6 个月；实例约 $8–9/月；流量按量计费
- [ ] 提醒法律合规：仅限个人学习使用

---

## 7. 排错速查

| 症状 | 原因 |
|---|---|
| 登录 403 | 没加 `Origin` 头（脚本已处理；手动调试 curl 时必须加） |
| wrong user or password | 密码特殊字符被 shell 吃掉；或 admin 命令走了 wrapper 没生效 |
| save 返回 200 但 success=false | 正常，读 msg 字段 |
| address already in use | 旧入站没删掉（del 传了 id 而非 tag） |
| unexpected end of JSON input | clients 缺 `links: []` |
| unknown network | shadowsocks 传了 network 字段 |
| 订阅导入报 invalid yaml | Clash 系客户端要用 `?format=clash` |
| 全球探测全 timeout 但本机通 | 用户本机 TUN 代理假连通，或安全组规则加在了错误的安全组上 |
| 网页终端粘贴公钥报错 | 引号没配对；公钥必须整行完整粘贴进双引号之间 |
| 所有节点昨天还好好的、今天全部连不上 | 九成是 IP 被墙了，见下节「IP 被墙急救」 |

---

## 8. IP 被墙急救（所有节点突然连不上）

**症状**：昨天还用得好好的，今天四个协议全部连不上，连面板网页也打不开。服务器本身没坏（SSH 依然能登录），重启、重装客户端都没用。

**原理**：IP 跑代理流量久了会被防火墙列入黑名单，AWS 等大机房的 IP 段被重点关照。这是正常现象，不是你的配置错了。一台 IP 的寿命从几天到几个月不等。

**判断**（让 AI 帮你做）：用海外网络探测服务器端口（如 check-host.net 的分布式检测）——海外通、国内不通 = 被墙实锤。

**复活方案二选一**：

- **方案 A：换弹性 IP**。控制台 → 弹性 IP → 释放旧的 → 分配新的 → 关联实例 → 把 §3 的一键脚本带上新 IP 重跑 → 客户端导入新订阅。适合旧实例还想要的情况
- **方案 B：新开实例（推荐，新 IP 天然干净）**。按上半部第二步重开一台（15 分钟），安全组照抄第三步，把下半部再跑一遍，满血复活。旧实例直接终止

**善后（都在烧钱，别忘）**：

- 旧实例：「终止」或至少「停止」
- 旧弹性 IP：**必须手动「释放」**——弹性 IP 不和实例关联也持续计费（$0.005/小时）
- 新实例如果不绑弹性 IP，重启后 IP 会变、订阅失效；介意就绑一个新的（绑之前先用几天确认新 IP 没被墙）

---

## 9. 扩展：多国家节点

单台服务器出口只能是一个国家（物理限制）。想要新加坡/香港/美国多个节点：

- **AWS 内部**：按上半部在别的区域再开一台实例（新加坡 `ap-southeast-1`；香港 `ap-east-1` 需先在账户设置里启用；美国 `us-west-2` 最便宜），本手册下半部原样再跑一遍即可
- **其他商家**（ often 更便宜）：RackNerd（racknerd.com，年付约 $10–20）、Cloudcone（cloudcone.com，按小时计费约 $4–5/月）、Vultr（vultr.com，$5–6/月）。注意这些商家的镜像选 Ubuntu 22.04，SSH 用户名是 `root` 而非 `ec2-user`，且安全组概念叫「防火墙/Firewall」，本手册第三步的端口表照抄即可

客户端里多个订阅并存，随时切换国家。

---

## 附录：s-ui API 细节参考（手动排错时用）

官方 Wiki：`https://github.com/alireza0/s-ui/wiki/API-Documentation`。前端 API `/app/api/*`（session 认证），REST API `/app/apiv2/*`（Token 认证）。

### 登录（必须带 Origin 头，否则 403；参数是 form 不是 JSON）

```bash
curl -s -c /tmp/cj -X POST http://localhost:2095/app/api/login \
  -H "Origin: http://localhost:2095" \
  -d "user=admin&pass=<密码>"
```

### 创建 API Token

```bash
curl -s -b /tmp/cj -X POST http://localhost:2095/app/api/addToken \
  -H "Origin: http://localhost:2095" -d "expiry=0&desc=setup"
```

之后所有请求带 `-H "Token: <token>"`。**失败也返回 HTTP 200**，必须检查响应 JSON 的 `success` 字段。

### save 端点 `POST /app/apiv2/save`（表单：object / action / data）

- `inbounds` 删除：data 是 **tag 的 JSON 字符串**（`"VLESS-Reality"`），传 id 会静默无效
- `clients` 删除：data 是 **id 的 JSON 数字**（`1`）
- `clients` 新建必须带 `"links": []`
- shadowsocks 入站**不要传 network 字段**（默认 tcp+udp）
- 保存入站时核心实时绑定端口——重建前先删旧的，间隔 sleep 3 秒
- save 响应返回的是**受影响对象的完整列表**，取 id 必须按 `tag`/`name` 匹配，不能取 `list[0]`
- 列表接口 `GET /apiv2/inbounds` 不返回 `out_json` 和用户凭据；`GET /apiv2/inbounds?id=N` 才返回。凭据可从 out_json 读回（ss2022 的 password 是 `serverKey:userKey` 格式）

### 密钥与证书生成

- `GET /apiv2/keypairs?k=reality` → x25519 密钥对（Reality 用）
- `GET /apiv2/keypairs?k=tls` → 自签证书（Hysteria2 用），返回 PEM 行数组，**先私钥块后证书块**

### 订阅格式

- `http://<IP>:2096/sub/<name>` → base64 链接列表（V2rayN/NekoBox）
- 加 `?format=clash` → Clash 配置（**Clash Verge 必须用这个**，默认格式报 invalid yaml）
- 加 `?format=json` → sing-box JSON

### 配置对象说明

- **TLS 对象**（object=tls）：`server` = 拼进入站的 sing-box TLS JSON；`client` = 生成订阅链接用的客户端 TLS JSON。面板自动把 server 的 `enabled/server_name/alpn` 和随机一个 reality `short_id` 合并进 client
- **入站对象**：固定字段 `id/type/tag/tls_id/addrs/out_json`，其余原样透传给 sing-box
- **客户端对象**：`name` 即订阅 ID（URL 最后一段，注意唯一性和私密性）；`volume=0` 不限流量
