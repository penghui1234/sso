# DolphinScheduler 与 EMR Hue 基于 Keycloak 的 SSO 方案

通过自建 **Keycloak** 作为统一 OIDC 身份源，实现 **Apache DolphinScheduler 3.2.1** 与 **Amazon EMR Hue 4.11** 的单点登录（登录任一系统后，访问另一个自动登录），并在 DolphinScheduler 顶部提供一键跳转 Hue 的入口。

> 环境：AWS us-east-1 / 账户 828414850215。本文 IP、密钥等为示例环境值，落地时请替换为你自己的。

---

## 1. 背景与架构

### 1.1 初始环境
| 组件 | 位置 | 说明 |
|---|---|---|
| DolphinScheduler 3.2.1 | EC2 `i-00fcc919f5b2773e8`（Ubuntu，t3.medium，公网 `3.89.105.183`） | Docker 容器 `apache/dolphinscheduler-standalone-server:3.2.1`，端口 `12345`，路径前缀 `/dolphinscheduler` |
| EMR 集群（含 Hue） | EMR `j-11R1505MDSXUG`，master `i-0559072c85ddfb04a`（Amazon Linux，公网 `3.238.31.232`） | emr-7.12.0；Hue 4.11 端口 `8888`（systemd `hue.service`，配置 `/etc/hue/conf/hue.ini`） |

### 1.2 目标
- 登录一个系统后，访问另一个免再次输密码（真 SSO）。
- DolphinScheduler 界面提供跳转 Hue 的入口。

### 1.3 最终架构
```
                         ┌─────────────────────────────┐
                         │   Keycloak (OIDC IdP)        │
                         │   EC2 i-0441666ff549149b7    │
                         │   http://54.221.153.237:8080 │
                         │   realm: sso                 │
                         └──────────────┬──────────────┘
              authorize/token/userinfo  │  (OIDC)
              ┌─────────────────────────┼───────────────────────────┐
              │                         │                           │
   ┌──────────▼───────────┐   ┌─────────▼──────────┐                │
   │ DolphinScheduler 3.2.1│   │  EMR Hue 4.11      │                │
   │ 3.89.105.183:12345    │   │  3.238.31.232:8888 │                │
   │                       │   │  (标准 OIDC 直连)   │                │
   │  OAuth2(GitHub 方言)  │                                          │
   │     │                 │                                          │
   │     ▼                 │                                          │
   │  token 适配器(:9000)  │── 翻译 token 请求格式 ────────────────────┘
   │  systemd ds-oauth-... │
   └───────────────────────┘
```

### 1.4 为什么 DolphinScheduler 需要一个“适配器”
DolphinScheduler 3.2.1 的内置 OAuth2（`LoginController.loginByAuth2`）是**按 GitHub 协议方言写死的**，与标准 OIDC（Keycloak/Cognito）有三处不兼容：

1. **用户名字段写死取 `login`**：`JSONUtils.getNodeString(userInfoJsonStr, "login")`，而标准 OIDC userInfo 返回 `sub/preferred_username/email`，没有 `login`。
2. **token 请求格式非标准**：把 `client_id/code/grant_type/redirect_uri` 放在 **URL query**、`client_secret` 放在 **JSON body**（`Content-Type: application/json`）；而 OIDC token 端点要求 `application/x-www-form-urlencoded` 表单体。
3. **redirect_uri 追加 `?provider=xxx`**，且前端拼 authorize URL 时**漏带 `scope=openid`**。

> 通用化 OIDC 是 DolphinScheduler GSoC 2025（4.x 之后）才加入的，3.2.1 不具备。Amazon Cognito 还额外强制回调必须 HTTPS（实测 `cannot use the HTTP protocol`），因此本方案改用可关闭 SSL 强制、可加协议映射器的 **Keycloak**。

解决办法（全部不改 DS 源码）：
- 用 Keycloak 的 **protocol mapper** 把 `username` 映射成名为 `login` 的声明 → 解决第 1 点。
- 部署一个**极小的 token 适配器**，把 DS 的 token 请求翻译成标准表单 POST → 解决第 2 点。
- Keycloak 客户端用 **public client + 通配 redirectUri**，并**在前端 bundle 注入 `scope=openid`** → 解决第 3 点。

---

## 2. 组件与关键参数

### 2.1 Keycloak
- 实例：`i-0441666ff549149b7`，t3.medium，公网 `54.221.153.237`，私网 `172.31.19.154`，端口 `8080`。
- 安全组 `sg-0ebab7034d046a73f`：入站放行 `8080`。
- 容器：`quay.io/keycloak/keycloak:25.0.6`，`start-dev`，数据卷 `/opt/keycloak-data`。
- 管理员：`admin` / `Adm1n-Keycloak-2026`，管理台 `http://54.221.153.237:8080/admin`。
- Realm：`sso`（`sslRequired=NONE`）。
- 测试用户：`testuser` / `Test@12345`。

### 2.2 Keycloak OIDC 端点（realm `sso`）
| 用途 | URL |
|---|---|
| issuer / base | `http://54.221.153.237:8080/realms/sso` |
| authorize | `.../protocol/openid-connect/auth` |
| token | `.../protocol/openid-connect/token` |
| userinfo | `.../protocol/openid-connect/userinfo` |
| jwks | `.../protocol/openid-connect/certs` |
| logout | `.../protocol/openid-connect/logout` |

### 2.3 客户端
| 客户端 | 类型 | 回调 URL | 备注 |
|---|---|---|---|
| `dolphinscheduler` | public | `http://3.89.105.183:12345/dolphinscheduler/redirect/login/oauth2*` | 含 `login` protocol mapper（username→login） |
| `hue` | confidential | `http://3.238.31.232:8888/oidc/callback/*` | secret：`ufQbvE7gGEk6daKDVrzEVAIFmv34VqGt` |

---

## 3. 配置步骤

### 3.1 启动 Keycloak（独立 EC2）
> DolphinScheduler 主机内存紧张（4G 总量、可用 ~360MB、无 swap、DS 已占用约 3.2G），不能同居 Keycloak，故用独立实例。

安全组：
```bash
aws ec2 create-security-group --group-name keycloak-sso-sg \
  --description "Keycloak OIDC IdP" --vpc-id <vpc-id>
aws ec2 authorize-security-group-ingress --group-id <sg-id> \
  --ip-permissions IpProtocol=tcp,FromPort=8080,ToPort=8080,IpRanges='[{CidrIp=0.0.0.0/0}]'
```

实例 user-data（AL2023）：
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

### 3.2 创建 realm / 用户 / 客户端 / 映射器（kcadm，在 Keycloak 容器内）
```bash
KC=/opt/keycloak/bin/kcadm.sh
$KC config credentials --server http://localhost:8080 --realm master --user admin --password 'Adm1n-Keycloak-2026'

# realm（关闭 SSL 强制，便于 HTTP POC）
$KC create realms -s realm=sso -s enabled=true -s sslRequired=NONE

# 测试用户
$KC create users -r sso -s username=testuser -s enabled=true \
  -s email=testuser@example.com -s emailVerified=true -s firstName=Test -s lastName=User
$KC set-password -r sso --username testuser --new-password 'Test@12345'

# DolphinScheduler：public client + 通配 redirectUri
DSID=$($KC create clients -r sso -s clientId=dolphinscheduler -s enabled=true \
  -s publicClient=true -s standardFlowEnabled=true -s directAccessGrantsEnabled=true \
  -s 'redirectUris=["http://3.89.105.183:12345/dolphinscheduler/redirect/login/oauth2*","http://3.89.105.183:12345/dolphinscheduler/*"]' \
  -s 'webOrigins=["*"]' -i)

# 关键：把 username 映射成名为 "login" 的声明（DS 写死读 login）
$KC create clients/$DSID/protocol-mappers/models -r sso \
  -s name=login-mapper -s protocol=openid-connect \
  -s protocolMapper=oidc-usermodel-property-mapper \
  -s 'config={"user.attribute":"username","claim.name":"login","jsonType.label":"String","id.token.claim":"true","access.token.claim":"true","userinfo.token.claim":"true"}'

# Hue：confidential client
HUEID=$($KC create clients -r sso -s clientId=hue -s enabled=true \
  -s publicClient=false -s standardFlowEnabled=true \
  -s 'redirectUris=["http://3.238.31.232:8888/oidc/callback/*","http://3.238.31.232:8888/*"]' \
  -s 'webOrigins=["*"]' -i)
$KC get clients/$HUEID/client-secret -r sso   # 取 Hue client secret
```

### 3.3 DolphinScheduler token 适配器
作用：仅翻译 **token 端点**。接收 DS 的“query 参数 + JSON body”请求，转成标准表单 POST 给 Keycloak，原样回传响应。authorize 与 userinfo 由 DS 直连 Keycloak。

- 部署在 DS 主机（python3 标准库，无依赖），systemd 服务 `ds-oauth-adapter`，监听 `0.0.0.0:9000`。
- DS 容器（bridge 网络，网关 `172.17.0.1`）通过 `http://172.17.0.1:9000/token` 访问。
- 适配器上游指向 Keycloak `token` 端点（用公网 `54.221.153.237`，保证 issuer 与 authorize 一致）。

`/opt/ds-oauth-adapter/adapter.py`（核心逻辑）：
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
```

### 3.4 DolphinScheduler OAuth2 配置
编辑**容器内** `/opt/dolphinscheduler/conf/application.yaml`（注意不是宿主机的解压包，运行的是容器），`security.authentication.oauth2`：
```yaml
    oauth2:
      enable: true            # 附加 OAuth2 登录；type 仍为 PASSWORD，保留密码登录兜底
      provider:
        keycloak:
          authorizationUri: "http://54.221.153.237:8080/realms/sso/protocol/openid-connect/auth"
          redirectUri: "http://3.89.105.183:12345/dolphinscheduler/redirect/login/oauth2"
          clientId: "dolphinscheduler"
          clientSecret: ""                         # public client，留空
          tokenUri: "http://172.17.0.1:9000/token" # 经适配器
          userInfoUri: "http://54.221.153.237:8080/realms/sso/protocol/openid-connect/userinfo"
          callbackUrl: "http://3.89.105.183:12345/dolphinscheduler/ui/login"
          iconUri: ""
          provider: keycloak
```
应用：`docker cp` 回容器后 `docker restart dolphinscheduler`。验证：
```bash
curl -s http://localhost:12345/dolphinscheduler/oauth2-provider   # 应返回 keycloak provider
```

### 3.5 前端注入 `scope=openid`（关键修复）
DS 3.2.1 前端拼 authorize URL 时漏带 `scope`，导致 Keycloak 签发非 OIDC token、userInfo 返回 403。在前端 bundle 注入：
```bash
F=/opt/dolphinscheduler/ui/assets/index.<hash>.js
docker exec dolphinscheduler sh -c "cp -n $F ${F}.orig; \
  sed -i 's#response_type=code&redirect_uri=#response_type=code\&scope=openid\&redirect_uri=#g' $F"
```
（静态文件，无需重启；浏览器需强制刷新 Ctrl+Shift+R 以加载新 JS。）

### 3.6 DolphinScheduler 顶部 Hue 跳转入口
DS 顶栏由编译后的 Vue 渲染，无法直接改标签。在容器内 `/opt/dolphinscheduler/ui/index.html` 注入一段脚本，在右上角加固定悬浮按钮 “Hue ↗”，点击新标签打开 Hue：
```html
<script>
(function () {
  var HUE_URL = 'http://3.238.31.232:8888';
  function ensureLink() {
    if (document.getElementById('hue-quick-link')) return;
    var a = document.createElement('a');
    a.id='hue-quick-link'; a.href=HUE_URL; a.target='_blank'; a.rel='noopener'; a.textContent='Hue \u2197';
    a.style.cssText='position:fixed;top:14px;right:200px;z-index:99999;height:34px;line-height:34px;padding:0 16px;background:#1890ff;color:#fff;font-weight:600;border-radius:17px;text-decoration:none;box-shadow:0 2px 6px rgba(0,0,0,.2)';
    document.body.appendChild(a);
  }
  function tick(){ if(!/\/login\b/.test(location.hash+location.pathname)) ensureLink();
    else { var e=document.getElementById('hue-quick-link'); if(e) e.remove(); } }
  window.addEventListener('load',tick); window.addEventListener('hashchange',tick); setInterval(tick,1000);
})();
</script>
```

### 3.7 EMR Hue OIDC 配置
编辑 master 上 `/etc/hue/conf/hue.ini`：
- `[desktop] [[auth]]`：`backend=desktop.auth.backend.OIDCBackend`
- `[desktop] [[oidc]]` 段（全部用公网端点，保证 issuer 一致）：
```ini
    oidc_rp_client_id=hue
    oidc_rp_client_secret=ufQbvE7gGEk6daKDVrzEVAIFmv34VqGt
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
应用：`sudo systemctl restart hue`（启动需 1-2 分钟，会跑 migrate）。

---

## 4. 访问与使用

| 系统 | 地址 | 说明 |
|---|---|---|
| DolphinScheduler | `http://3.89.105.183:12345/dolphinscheduler` | 登录页有 Keycloak 登录按钮（也保留密码登录）；右上角 “Hue ↗” 跳转 |
| EMR Hue | `http://3.238.31.232:8888` | 自动跳转 Keycloak |
| Keycloak 管理台 | `http://54.221.153.237:8080/admin` | `admin` / `Adm1n-Keycloak-2026` |
| 测试用户 | — | `testuser` / `Test@12345` |

**SSO 体验**：先登任意一个（Keycloak 输一次密码），再访问另一个时浏览器携带 Keycloak 会话，直接放行、无需再次登录。

---

## 5. 验证

DolphinScheduler 服务端链路（脚本化模拟浏览器全流程）结果：
```
login_form_action_found=yes
post_creds_http=302 redirect_to=.../redirect/login/oauth2?provider=keycloak&state=...
AUTH_CODE_OBTAINED=yes
TOKEN_VIA_ADAPTER=OK                # 适配器用真实 code 换到 access_token
USERINFO_login_claim='testuser'     # userInfo 返回 login（DS 据此建用户/登录）
END_TO_END_RESULT=PASS
```
打 DS 真实回调端点 `/dolphinscheduler/redirect/login/oauth2`（带 scope）：
```
DS_HTTP=302
DS_REDIRECT_LOCATION=.../ui/login?sessionId=...&authType=oauth2
RESULT=DS_LOGIN_SUCCESS
```
Hue：访问 `/` → 302 `/oidc/authenticate/` → 302 跳转 Keycloak authorize（`client_id=hue`）。

---

## 6. 回滚

| 对象 | 备份 | 回滚 |
|---|---|---|
| DS `application.yaml` | `.orig` / `.bak-*` / `.bak2`（容器内 `/opt/dolphinscheduler/conf/`） | 还原后 `docker restart dolphinscheduler` |
| DS `index.html`（Hue 按钮） | `index.html.orig` | 还原（无需重启） |
| DS 前端 JS（scope 注入） | `index.<hash>.js.orig` | 还原（无需重启） |
| Hue `hue.ini` | `/etc/hue/conf/hue.ini.bak-*` | 还原后 `sudo systemctl restart hue` |
| token 适配器 | — | `systemctl disable --now ds-oauth-adapter` |

---

## 7. 安全与生产化建议（当前为 POC）

- **全程 HTTP 明文**（含 Keycloak 登录、token 交换）。生产务必启用 **HTTPS**（ALB + ACM 或反向代理 + 证书）。
- Keycloak 为 `start-dev`（H2 内嵌库、开发模式），`8080` 对公网开放（含管理台）。生产应：换持久化数据库（PostgreSQL）、用 `start`（production 模式）、收紧安全组来源、修改默认 admin 密码、为 realm 设置合理 `sslRequired`。
- DolphinScheduler 主机内存紧张（可用约 360MB、无 swap），留意 OOM；适配器与 DS 同机，注意资源。
- token 适配器处理 token/密钥，建议仅监听本机/容器网段，勿对公网暴露 `:9000`。
- 容器内文件改动（DS `application.yaml`/`index.html`/JS）在 `docker restart` 后保留，但容器**重建**会丢失；生产应做成 bind-mount 或自定义镜像固化。
- Hue 用户默认非超级管理员，可在 Keycloak 配 `superuser_group` 并在 hue.ini 设置对应映射。

---

## 附：关键定位结论（排错备忘）

- DS 实际运行的是 **Docker 容器**，宿主机 `/opt/apache-dolphinscheduler-3.2.1-bin/...` 是未运行的解压包，改它无效。
- Cognito 不可行的两个硬约束：① 回调强制 HTTPS（`cannot use the HTTP protocol`）；② DS 3.2.1 OAuth2 是 GitHub 方言，token 格式/`login` 字段不兼容。故选 Keycloak。
- 登录 403 根因：DS 前端 authorize URL **缺 `scope=openid`** → 非 OIDC token → userInfo 403 → NPE。注入 scope 后修复。
- issuer 一致性：authorize/token/userinfo 必须用**同一 host**（本方案统一用公网 `54.221.153.237`），否则 token 的 `iss` 与 userinfo 校验 host 不符导致 401/403。
