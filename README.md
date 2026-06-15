# DolphinScheduler 3.2.1 + EMR Hue 基于 Keycloak 的 SSO 部署指南

通过自建 **Keycloak**（OIDC IdP）实现 **Apache DolphinScheduler 3.2.1** 与 **Amazon EMR Hue 4.11** 的单点登录：登录任一系统后访问另一个免再次输密码；并在 DolphinScheduler 提供一键跳转 Hue 的入口。

本文是**可照做的分步指南**，包含 DS 3.2.1 原生 OAuth2 的若干坑及其修复（缺 `scope`、locale cookie 导致 500 等）。

> 文中 IP / 密钥为示例环境值（AWS us-east-1），照做时替换为你自己的。
> 三台机器：DS 主机 `3.89.105.183`、EMR master `3.238.31.232`、Keycloak `54.221.153.237`（本指南新建）。

---

## 目录
1. [架构与原理](#1-架构与原理)
1.5 [改动点与影响面（必读）](#15-改动点与影响面必读)
2. [前置条件](#2-前置条件)
3. [第一步：部署 Keycloak](#3-第一步部署-keycloak)
4. [第二步：配置 Keycloak（realm/用户/客户端/映射器）](#4-第二步配置-keycloakrealm用户客户端映射器)
5. [第三步：部署 DS token 适配器](#5-第三步部署-ds-token-适配器)
6. [第四步：配置 DolphinScheduler 后端](#6-第四步配置-dolphinscheduler-后端)
7. [第五步：修复 DS 前端三个坑](#7-第五步修复-ds-前端三个坑必做)
8. [第六步：配置 EMR Hue](#8-第六步配置-emr-hue)
9. [第七步：验证](#9-第七步验证)
10. [访问方式](#10-访问方式)
11. [回滚](#11-回滚)
12. [安全与生产化建议](#12-安全与生产化建议)
13. [排错 FAQ](#13-排错-faq)

---

## 1. 架构与原理

```
                       ┌──────────────────────────────┐
                       │  Keycloak (OIDC IdP)          │
                       │  http://54.221.153.237:8080   │
                       │  realm: sso                   │
                       └───────────────┬──────────────┘
       authorize(浏览器,公网) / token / userinfo (OIDC)
            ┌──────────────────────────┼─────────────────────────┐
            │                          │                          │
 ┌──────────▼──────────┐      ┌────────▼─────────┐                │
 │ DolphinScheduler     │      │ EMR Hue 4.11     │                │
 │ 3.89.105.183:12345   │      │ 3.238.31.232:8888│ 标准 OIDC 直连  │
 │  原生 OAuth2(GitHub  │      └──────────────────┘                │
 │  方言)               │                                          │
 │     │ token 请求格式不标准                                       │
 │     ▼                                                          │
 │  token 适配器 :9000  │── 翻译 token 请求 → 标准表单 POST ────────┘
 └─────────────────────┘
```

**为什么 DS 需要一个适配器**：DolphinScheduler 3.2.1 的内置 OAuth2 是按 GitHub 协议方言写死的，与标准 OIDC 有三处不兼容，必须分别绕过（本指南均不改 DS 源码）：

| 不兼容点 | 现象 | 解决办法 |
|---|---|---|
| userInfo 用户名字段写死取 `login` | 取不到用户名 → 建用户 NPE | Keycloak 加 protocol mapper：`username` → 声明 `login` |
| token 请求把参数放 URL query、`client_secret` 放 JSON body | Keycloak 报 `Missing form parameter: grant_type` | 部署 token 适配器翻译成标准表单 POST |
| 前端 authorize URL 漏带 `scope=openid` | 签发非 OIDC token → userInfo 返回 **403** → NPE | 前端注入 `scope=openid`（第五步） |

> 另外 DS 前端登录流程会 `DELETE /cookies` 把 `language` cookie 值置为 null，触发 Spring `CookieLocaleResolver` NPE → **登录偶发 500**，也需在第五步修复。
> 通用 OIDC 是 DS GSoC 2025（4.x 之后）才支持，3.2.1 没有。Amazon Cognito 还强制回调必须 HTTPS（`cannot use the HTTP protocol`），故本方案用可关 SSL 强制、可加映射器的 **Keycloak**。

---

## 1.5 改动点与影响面（必读）

本方案对各组件的"碰触面"如下，便于评估风险与审批。核心结论：**DolphinScheduler 只动 api-server，且后端进程代码零改动**。

### 改哪个进程的配置？——只有 api-server
SSO/OAuth2 登录逻辑全在 `dolphinscheduler-api`（`LoginController` / `OAuth2Configuration`），由 **api-server** 进程处理 Web 登录、token 交换、userInfo、建会话。master / worker / alert 与 Web 登录无关。

| 进程 | 是否改动 | 说明 |
|---|---|---|
| **api-server** | ✅ | 改 `application.yaml` 的 oauth2 段；其托管的前端静态资源打补丁 |
| master | ❌ | 不涉及 |
| worker | ❌ | 不涉及 |
| alert | ❌ | 不涉及 |

> 集群部署：只改 **api-server 节点**。standalone（本指南）四进程合一，共用一份 `/opt/dolphinscheduler/conf/application.yaml`。
> 若 UI 由独立前端节点（nginx 托管 `dolphinscheduler-ui`）提供，则第五步的前端补丁打在**前端节点**，而非 api-server。

### 要不要动已有进程的代码？——后端零改动
master/worker/api/alert 的 **Java 后端代码一行未改**。3.2.1 原生 OAuth2 的几处硬编码缺陷全部用"外围"绕过：

| 缺陷（位于 api-server 代码） | 绕过方式 | 动 DS 代码？ | 需重启进程？ |
|---|---|---|---|
| userInfo 用户名写死取 `login` 字段 | Keycloak protocol mapper（IdP 侧） | 否 | 否 |
| token 请求格式非标准（query+JSON body） | 外置 token 适配器（独立 systemd 服务） | 否 | 否 |
| 前端 authorize 漏 `scope=openid` | 改前端静态产物 `ui/assets/*.js` | 否（UI 静态文件） | 否（刷新即可） |
| 前端 `DELETE /cookies` 制造空 `language` cookie → locale NPE | 改前端静态产物（关闭该调用） | 否（UI 静态文件） | 否 |

> 唯一"碰 DS 文件"的是 **api-server 托管的前端静态资源**（`ui/...`），属打包产物而非进程运行代码，改完无需重启进程。
> 如需"代码内正规修复"（去掉适配器+前端补丁），改动点仍只在 api-server：`LoginController.loginByAuth2`（`login` 字段、token 格式）+ 前端 scope/cookie——但需重新编译打包，不在本方案内。

### 配置文件改动点
DS 侧**只有一处**配置文件：api-server `application.yaml` 的 `security.authentication.oauth2`：
- `enable: false → true`
- 新增 `provider.keycloak` 块（authorizationUri / redirectUri / clientId / clientSecret(空) / tokenUri(指向适配器) / userInfoUri / callbackUrl / provider）
- `security.authentication.type` **保持 `PASSWORD`**（保留密码登录兜底，防锁死）
- 改完**需重启 api-server**（standalone 重启容器）

其余改动不属于 DS 配置文件（一并列出评估全貌）：token 适配器（DS 主机新增服务）、前端静态文件（scope/cookie/按钮）、Hue `hue.ini`（EMR master）、Keycloak realm/client/mapper（IdP 侧）。

---
## 2. 前置条件
- DS 以容器 `apache/dolphinscheduler-standalone-server:3.2.1` 运行（端口 12345，路径前缀 `/dolphinscheduler`）。
  - ⚠️ 注意运行的是**容器内** `/opt/dolphinscheduler/conf/application.yaml`，不是宿主机解压包。
- EMR（含 Hue 4.11，端口 8888，systemd `hue.service`，配置 `/etc/hue/conf/hue.ini`）。
- 三台机器两两网络可达（Keycloak 8080 需对浏览器和两台业务机开放）。
- 能在三台机器上执行命令（本文用 AWS SSM；用 SSH 亦可）。
- DS 主机已装 Docker + python3。

---

## 3. 第一步：部署 Keycloak

> DS 主机内存紧张（4G / Xmx4g / 无 swap），**不要**与 DS 同居 Keycloak，单独开一台（如 t3.medium）。

**安全组**（开放 8080）：
```bash
aws ec2 create-security-group --group-name keycloak-sso-sg \
  --description "Keycloak OIDC IdP" --vpc-id <你的VPC>
aws ec2 authorize-security-group-ingress --group-id <sg-id> \
  --ip-permissions IpProtocol=tcp,FromPort=8080,ToPort=8080,IpRanges='[{CidrIp=0.0.0.0/0}]'
```

**实例 user-data**（Amazon Linux 2023，自动装 Docker + 跑 Keycloak）：
```bash
#!/bin/bash
dnf install -y docker
systemctl enable --now docker
mkdir -p /opt/keycloak-data && chmod 777 /opt/keycloak-data
docker run -d --name keycloak --restart always -p 8080:8080 \
  -e KEYCLOAK_ADMIN=admin -e KEYCLOAK_ADMIN_PASSWORD='Adm1n-Keycloak-2026' \
  -e KC_HTTP_ENABLED=true -e KC_HOSTNAME_STRICT=false \
  -v /opt/keycloak-data:/opt/keycloak/data \
  quay.io/keycloak/keycloak:25.0.6 start-dev
```
等待就绪：`curl -s -o /dev/null -w '%{http_code}' http://localhost:8080/realms/master` 返回 `200`。

---

## 4. 第二步：配置 Keycloak（realm/用户/客户端/映射器）

在 Keycloak 容器内用 `kcadm.sh` 执行：
```bash
KC=/opt/keycloak/bin/kcadm.sh
$KC config credentials --server http://localhost:8080 --realm master --user admin --password 'Adm1n-Keycloak-2026'

# realm：关闭 SSL 强制（HTTP POC 必需）
$KC create realms -s realm=sso -s enabled=true -s sslRequired=NONE

# 测试用户
$KC create users -r sso -s username=testuser -s enabled=true \
  -s email=testuser@example.com -s emailVerified=true -s firstName=Test -s lastName=User
$KC set-password -r sso --username testuser --new-password 'Test@12345'

# DolphinScheduler 客户端：public + 通配 redirectUri
DSID=$($KC create clients -r sso -s clientId=dolphinscheduler -s enabled=true \
  -s publicClient=true -s standardFlowEnabled=true -s directAccessGrantsEnabled=true \
  -s 'redirectUris=["http://3.89.105.183:12345/dolphinscheduler/redirect/login/oauth2*","http://3.89.105.183:12345/dolphinscheduler/*"]' \
  -s 'webOrigins=["*"]' -i)

# 关键：把 username 映射成名为 login 的声明（DS 写死读 userInfo 的 "login"）
$KC create clients/$DSID/protocol-mappers/models -r sso \
  -s name=login-mapper -s protocol=openid-connect \
  -s protocolMapper=oidc-usermodel-property-mapper \
  -s 'config={"user.attribute":"username","claim.name":"login","jsonType.label":"String","id.token.claim":"true","access.token.claim":"true","userinfo.token.claim":"true"}'

# Hue 客户端：confidential
HUEID=$($KC create clients -r sso -s clientId=hue -s enabled=true \
  -s publicClient=false -s standardFlowEnabled=true \
  -s 'redirectUris=["http://3.238.31.232:8888/oidc/callback/*","http://3.238.31.232:8888/*"]' \
  -s 'webOrigins=["*"]' -i)
$KC get clients/$HUEID/client-secret -r sso   # 记下 Hue client secret，第六步用
```

OIDC 端点（realm `sso`）：
- authorize: `http://54.221.153.237:8080/realms/sso/protocol/openid-connect/auth`
- token: `.../token`  ·  userinfo: `.../userinfo`  ·  jwks: `.../certs`

> **issuer 一致性**：authorize（浏览器侧，必须公网 IP）与 token/userinfo（后端侧）务必使用**同一 host**（本文统一用公网 `54.221.153.237`），否则 token 的 `iss` 与 userinfo 校验 host 不符会导致 401/403。

---

## 5. 第三步：部署 DS token 适配器

作用：只翻译 **token 端点**——把 DS 的「query 参数 + JSON body」请求转成标准 `application/x-www-form-urlencoded` 表单 POST 给 Keycloak，原样回传。authorize、userInfo 由 DS 直连 Keycloak。

DS 容器是 bridge 网络（网关 `172.17.0.1`），所以适配器跑在**宿主机** `0.0.0.0:9000`，DS 通过 `http://172.17.0.1:9000/token` 访问。

`/opt/ds-oauth-adapter/adapter.py`：
```python
#!/usr/bin/env python3
import json, urllib.parse, urllib.request, urllib.error
from http.server import BaseHTTPRequestHandler, ThreadingHTTPServer

KC_TOKEN_URL = "http://54.221.153.237:8080/realms/sso/protocol/openid-connect/token"
LISTEN = ("0.0.0.0", 9000)
FORWARD_KEYS = ("grant_type", "client_id", "code", "redirect_uri", "refresh_token")

class Handler(BaseHTTPRequestHandler):
    def _send(self, code, body):
        self.send_response(code); self.send_header("Content-Type", "application/json")
        self.send_header("Content-Length", str(len(body))); self.end_headers(); self.wfile.write(body)
    def do_GET(self):
        self._send(200, b'{"status":"ok"}') if self.path.startswith("/health") else self._send(404, b'{}')
    def do_POST(self):
        p = urllib.parse.urlparse(self.path)
        if not p.path.startswith("/token"): return self._send(404, b'{}')
        params = dict(urllib.parse.parse_qsl(p.query, keep_blank_values=True))
        n = int(self.headers.get("Content-Length", 0) or 0); raw = self.rfile.read(n) if n else b""
        cs = ""
        if raw:
            try: cs = (json.loads(raw.decode()) or {}).get("client_secret", "") or ""
            except Exception: cs = dict(urllib.parse.parse_qsl(raw.decode())).get("client_secret", "")
        form = {k: params[k] for k in FORWARD_KEYS if params.get(k)}
        if cs: form["client_secret"] = cs
        req = urllib.request.Request(KC_TOKEN_URL, data=urllib.parse.urlencode(form).encode(),
              headers={"Content-Type": "application/x-www-form-urlencoded", "Accept": "application/json"}, method="POST")
        try:
            with urllib.request.urlopen(req, timeout=30) as r: resp, code = r.read(), r.getcode()
        except urllib.error.HTTPError as e: resp, code = e.read(), e.code
        self._send(code, resp)

if __name__ == "__main__":
    ThreadingHTTPServer(LISTEN, Handler).serve_forever()
```

systemd 单元 `/etc/systemd/system/ds-oauth-adapter.service`：
```ini
[Unit]
Description=DS-Keycloak OAuth2 token adapter
After=network.target
[Service]
ExecStart=/usr/bin/python3 /opt/ds-oauth-adapter/adapter.py
Restart=always
User=root
[Install]
WantedBy=multi-user.target
```
```bash
systemctl daemon-reload && systemctl enable --now ds-oauth-adapter
curl -s http://127.0.0.1:9000/health   # {"status":"ok"}
```

---

## 6. 第四步：配置 DolphinScheduler 后端

编辑**容器内** `/opt/dolphinscheduler/conf/application.yaml`，`security.authentication.oauth2`：
```yaml
    oauth2:
      enable: true            # type 仍保持 PASSWORD，保留密码登录兜底
      provider:
        keycloak:
          authorizationUri: "http://54.221.153.237:8080/realms/sso/protocol/openid-connect/auth"
          redirectUri: "http://3.89.105.183:12345/dolphinscheduler/redirect/login/oauth2"
          clientId: "dolphinscheduler"
          clientSecret: ""                          # public client 留空
          tokenUri: "http://172.17.0.1:9000/token"  # 经适配器
          userInfoUri: "http://54.221.153.237:8080/realms/sso/protocol/openid-connect/userinfo"
          callbackUrl: "http://3.89.105.183:12345/dolphinscheduler/ui/login"
          iconUri: ""
          provider: keycloak
```
应用：
```bash
docker cp app.yaml dolphinscheduler:/opt/dolphinscheduler/conf/application.yaml
docker restart dolphinscheduler   # standalone 启动约 1-2 分钟
# 验证（应返回 keycloak provider）：
curl -s http://localhost:12345/dolphinscheduler/oauth2-provider
```

---

## 7. 第五步：修复 DS 前端三个坑（必做）

DS 3.2.1 的前端编译产物有两个会导致登录失败的问题，且顶部按钮会和菜单重叠。以下都是对**容器内** UI 静态文件的修改，**无需重启**，但浏览器需强制刷新（文件名不变会被缓存）。

> 先定位前端主 bundle 文件名（形如 `index.<hash>.js`）：
> ```bash
> docker exec dolphinscheduler ls /opt/dolphinscheduler/ui/assets/ | grep -E '^index\.[0-9a-f]+\.js$'
> ```
> 下文以 `index.1f78923c.js` 为例。

### 7.1 注入 `scope=openid`（否则 userInfo 403 → 登录失败）
DS 前端拼 authorize URL 时漏了 `scope`。注入：
```bash
F=/opt/dolphinscheduler/ui/assets/index.1f78923c.js
docker exec dolphinscheduler sh -c "cp -n $F ${F}.orig; \
  sed -i 's#response_type=code&redirect_uri=#response_type=code\&scope=openid\&redirect_uri=#g' $F"
```

### 7.2 关闭前端 `DELETE /cookies`（否则偶发登录 500）
DS 前端登录流程调用 `DELETE /cookies`，把 `language` cookie 值置为 null；该空值 cookie 被带到 OAuth 回调时触发 Spring `CookieLocaleResolver` 空指针（`CookieLocaleResolver.java:226`，对 null 值 `indexOf` ）→ 500。把该调用改为空操作：
```bash
F=/opt/dolphinscheduler/ui/assets/index.1f78923c.js
docker exec dolphinscheduler sh -c "cp -n $F ${F}.bak-cookiefix; \
  sed -i 's#function ae(){return f({url:\"/cookies\",method:\"delete\"})}#function ae(){return Promise.resolve()}#' $F"
```
> 注意：`ae` 是压缩后的函数名，不同构建可能不同。先 `grep -o '....url:\"/cookies\",method:\"delete\".....' $F` 找到实际函数名再替换。

### 7.3 Hue 跳转按钮 + language cookie 守护（注入 index.html）
DS 顶栏由编译后的 Vue 渲染，无法直接改标签；在 `index.html` 的 `</body>` 前注入脚本：右下角加悬浮 “Hue ↗” 按钮（放右下角避免与顶部菜单重叠），并守护 `language` cookie 始终为合法值（双保险）。

```html
    <script>
    (function(){
      function getLang(){var m=document.cookie.match(/(?:^|;\s*)language=([^;]*)/);return m?decodeURIComponent(m[1]):'';}
      function ensureLang(){var v=getLang();if(v!=='zh_CN'&&v!=='en_US'){document.cookie='language=zh_CN; path=/; max-age=31536000';}}
      ensureLang();document.addEventListener('DOMContentLoaded',ensureLang);
      window.addEventListener('pagehide',ensureLang);window.addEventListener('beforeunload',ensureLang);setInterval(ensureLang,1000);
    })();
    (function(){
      var HUE_URL='http://3.238.31.232:8888';
      function ensureLink(){if(document.getElementById('hue-quick-link'))return;var a=document.createElement('a');a.id='hue-quick-link';a.href=HUE_URL;a.target='_blank';a.rel='noopener noreferrer';a.title='Open EMR Hue';a.textContent='Hue \u2197';a.style.cssText='position:fixed;bottom:28px;right:28px;z-index:99999;height:40px;line-height:40px;padding:0 18px;background:#1890ff;color:#fff;font-size:14px;font-weight:600;border-radius:20px;text-decoration:none;box-shadow:0 2px 8px rgba(0,0,0,.25);cursor:pointer';a.onmouseover=function(){a.style.background='#40a9ff';};a.onmouseout=function(){a.style.background='#1890ff';};document.body.appendChild(a);}
      function shouldShow(){return !/\/login\b/.test(location.hash+location.pathname);}
      function tick(){var e=document.getElementById('hue-quick-link');if(shouldShow())ensureLink();else if(e)e.remove();}
      window.addEventListener('load',tick);document.addEventListener('DOMContentLoaded',tick);window.addEventListener('hashchange',tick);setInterval(tick,1000);
    })();
    </script>
```
> 落地技巧：先 `docker cp` 取出 index.html，用脚本在 `</body>` 前插入上面这段，再 `docker cp` 回去。**注意编码**：通过 shell/base64 传大文件时，注释里别用中文，避免传输损坏（建议纯 ASCII）。

---

## 8. 第六步：配置 EMR Hue

编辑 master 的 `/etc/hue/conf/hue.ini`：

`[desktop] > [[auth]]`：
```ini
    backend=desktop.auth.backend.OIDCBackend
```
`[desktop] > [[oidc]]`（全部用公网端点，保证 issuer 一致；secret 用第二步记下的）：
```ini
    oidc_rp_client_id=hue
    oidc_rp_client_secret=<第二步的 Hue client secret>
    oidc_op_authorization_endpoint=http://54.221.153.237:8080/realms/sso/protocol/openid-connect/auth
    oidc_op_token_endpoint=http://54.221.153.237:8080/realms/sso/protocol/openid-connect/token
    oidc_op_user_endpoint=http://54.221.153.237:8080/realms/sso/protocol/openid-connect/userinfo
    oidc_op_jwks_endpoint=http://54.221.153.237:8080/realms/sso/protocol/openid-connect/certs
    oidc_rp_sign_algo=RS256
    oidc_verify_ssl=false
    oidc_username_attribute=preferred_username
    login_redirect_url=http://3.238.31.232:8888/oidc/callback/
    logout_redirect_url=http://54.221.153.237:8080/realms/sso/protocol/openid-connect/logout
    login_redirect_url_failure=http://3.238.31.232:8888/hue/oidc_failed/
    create_users_on_login=true
```
应用：`sudo systemctl restart hue`（启动需 1-2 分钟，会跑 migrate）。验证：`curl -s -o /dev/null -w '%{http_code}' http://localhost:8888/` 应为 302（跳转 Keycloak）。

---

## 9. 第七步：验证

后端链路（脚本化模拟浏览器全流程，应得 PASS）：
```
login_form_action_found=yes
AUTH_CODE_OBTAINED=yes
TOKEN_VIA_ADAPTER=OK            # 适配器用真实 code 换到 access_token
USERINFO_login_claim='testuser' # userInfo 含 login
END_TO_END_RESULT=PASS
```
打 DS 真实回调端点（带 `scope`、无坏 cookie）应得：
```
DS_HTTP=302  loc=.../ui/login?sessionId=...&authType=oauth2
```
浏览器：**用无痕窗口**打开 DS → 点 Keycloak 登录 → 输 `testuser/Test@12345` → 进入；再点右下角 “Hue ↗” → 已登录态打开 Hue（SSO 成功）。

---

## 10. 访问方式

| 系统 | 网址 | 说明 |
|---|---|---|
| DolphinScheduler | http://3.89.105.183:12345/dolphinscheduler | Keycloak 登录按钮 + 密码兜底；右下角 “Hue ↗” |
| EMR Hue | http://3.238.31.232:8888 | 自动跳 Keycloak |
| Keycloak 管理台 | http://54.221.153.237:8080/admin | admin / Adm1n-Keycloak-2026 |
| 测试用户 | — | testuser / Test@12345 |

**首次或改完前端后**：用无痕窗口，或清该站点 cookie + 强制刷新（Ctrl+Shift+R），以加载新 JS 并清掉旧的坏 cookie。

---

## 11. 回滚

| 对象 | 备份 | 回滚 |
|---|---|---|
| DS `application.yaml` | 容器内 `.orig` / `.bak-*` | 还原后 `docker restart dolphinscheduler` |
| DS 前端 JS（scope/cookie 补丁） | `index.<hash>.js.orig` / `.bak-cookiefix` | 还原（无需重启） |
| DS `index.html`（按钮/守护） | `index.html.orig` | 还原（无需重启） |
| Hue `hue.ini` | `/etc/hue/conf/hue.ini.bak-*` | 还原后 `sudo systemctl restart hue` |
| token 适配器 | — | `systemctl disable --now ds-oauth-adapter` |

> ⚠️ 容器内文件改动在 `docker restart` 后保留，但容器**重建**会丢失。生产应做成 bind-mount 或自定义镜像固化。

---

## 12. 安全与生产化建议（当前为 POC）
- **全程 HTTP 明文**（含 Keycloak 登录与 token）。生产务必启用 HTTPS（ALB+ACM 或反向代理+证书）。
- Keycloak 为 `start-dev`（H2 内嵌库、开发模式）、8080 对公网开放（含管理台）。生产应：换 PostgreSQL、用 `start`（production 模式）、收紧安全组、改默认 admin 密码、设置合理 `sslRequired`。
- token 适配器处理 token/密钥，建议只监听本机/容器网段，勿对公网暴露 `:9000`。
- DS 主机内存紧张（无 swap），注意 OOM。
- 上述前端补丁是对 3.2.1 的临时绕过；升级到支持通用 OIDC 的 DS 版本后可去掉适配器与前端补丁。
- Hue 用户默认非超级管理员，可在 Keycloak 配 `superuser_group` 并在 hue.ini 设对应项。

---

## 13. 排错 FAQ

**登录后 userInfo 返回 403 / 报错**
→ 前端 authorize URL 缺 `scope=openid`（见 7.1）。检查 `oauth2-provider` 与 Keycloak 客户端 scope。

**登录偶发 500，刷新又能进；日志 `CookieLocaleResolver.java:226` NPE**
→ `language` cookie 被置空（null 值）。执行 7.2 关闭前端 `DELETE /cookies`，并用无痕/清 cookie 去掉已存在的坏 cookie。

**Keycloak 报 `Missing form parameter: grant_type`**
→ DS token 请求格式不标准，未走适配器或 `tokenUri` 配错（应指向 `http://172.17.0.1:9000/token`）。

**userInfo 401 / token 校验失败**
→ authorize 与 token/userinfo 用了不同 host 导致 issuer 不一致。统一用公网 `54.221.153.237`。

**Keycloak 回调 `invalid redirect uri`**
→ 客户端 `redirectUris` 未包含带 `?provider=keycloak` 的回调；用通配 `.../redirect/login/oauth2*`。

**Hue 跳转后报 SSL/证书错**
→ `oidc_verify_ssl=false`；HTTP 环境下 realm `sslRequired=NONE`。
