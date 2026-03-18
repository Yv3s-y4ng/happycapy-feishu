---
name: happycapy-feishu
description: 为 Claude Code 安装并授权飞书 MCP，让 Claude 直接操作飞书消息、文档、多维表格、日历等。触发词：安装飞书 MCP、配置飞书、接入飞书、飞书 MCP setup、connect feishu
---

# 飞书 MCP 安装向导

用户需要做的事只有两件：
1. 在飞书开放平台创建一个应用，拿到 App ID 和 App Secret
2. 点一个授权链接，把浏览器地址栏的 URL 发回来

其余所有操作由 Claude 自动完成。

---

## 第一步：引导用户创建飞书应用

告知用户按以下步骤操作，完成后把 **App ID** 和 **App Secret** 发给你：

1. 打开 https://open.feishu.cn/app 并登录
2. 右上角点击**创建应用** → **自建应用**，填写任意名称（如 `Claude MCP`）
3. 进入应用后，**凭证与基础信息**页面找到 **App ID** 和 **App Secret**
4. 左侧菜单进入**安全设置** → **重定向 URL** → 添加：`http://localhost:3000/callback`
5. 左侧菜单进入**权限管理**，搜索并开通所需权限（推荐：`im:message`、`docx:document`、`bitable:app`、`calendar:calendar`）
6. 左侧菜单进入**版本管理与发布**，创建版本并发布（内部测试版即可）

---

## 第二步：自动安装配置（收到凭证后 Claude 执行）

### 2.1 注册 MCP 到全局配置

```bash
claude mcp add-json --scope=user lark-mcp '{
  "command": "npx",
  "args": ["-y", "@larksuiteoapi/lark-mcp", "mcp",
           "-a", "<APP_ID>", "-s", "<APP_SECRET>",
           "--oauth", "--token-mode", "user_access_token", "-l", "zh"]
}'
```

### 2.2 修复 headless 环境 token 持久化（替换 keytar 为文件存储）

无桌面 Linux 环境中 keytar 依赖 dbus-launch，导致 token 只存内存、重启即丢失。需覆盖替换：

```bash
# 先触发一次以确保包已下载
npx -y @larksuiteoapi/lark-mcp --version 2>/dev/null || true

KEYTAR_PATH=$(find ~/.local ~/.npm -name "keytar.js" -path "*/keytar/lib/*" 2>/dev/null | grep lark-mcp | head -1)
echo "keytar: $KEYTAR_PATH"
cp "$KEYTAR_PATH" "${KEYTAR_PATH}.bak"
```

然后用以下内容覆盖 `$KEYTAR_PATH`（文件存储版，key 保存在 `~/.lark-mcp-keychain.json`）：

```javascript
const fs = require('fs'), path = require('path'), os = require('os');
const STORE = path.join(os.homedir(), '.lark-mcp-keychain.json');
const load = () => { try { return fs.existsSync(STORE) ? JSON.parse(fs.readFileSync(STORE,'utf8')) : {}; } catch(e){return{};} };
const save = s => fs.writeFileSync(STORE, JSON.stringify(s,null,2), {mode:0o600});
const k = (svc,acc) => `${svc}::${acc}`;
const chk = (v,n) => { if(!v||!v.length) throw new Error(n+' is required.'); };
module.exports = {
  getPassword:(svc,acc)=>{chk(svc,'Service');chk(acc,'Account');return Promise.resolve(load()[k(svc,acc)]||null);},
  setPassword:(svc,acc,pw)=>{chk(svc,'Service');chk(acc,'Account');chk(pw,'Password');const s=load();s[k(svc,acc)]=pw;save(s);return Promise.resolve();},
  deletePassword:(svc,acc)=>{chk(svc,'Service');chk(acc,'Account');const s=load(),existed=k(svc,acc) in s;delete s[k(svc,acc)];if(existed)save(s);return Promise.resolve(existed);},
  findPassword:(svc)=>{chk(svc,'Service');const s=load(),p=`${svc}::`;for(const k of Object.keys(s))if(k.startsWith(p))return Promise.resolve(s[k]);return Promise.resolve(null);},
  findCredentials:(svc)=>{chk(svc,'Service');const s=load(),p=`${svc}::`,r=[];for(const k of Object.keys(s))if(k.startsWith(p))r.push({account:k.slice(p.length),password:s[k]});return Promise.resolve(r);}
};
```

### 2.3 延长 OAuth 超时至 5 分钟

```bash
HANDLER=$(find ~/.local ~/.npm -name "handler-local.js" -path "*lark-mcp*" 2>/dev/null | head -1)
sed -i 's/this\.stopServer(), 60 \* 1000/this.stopServer(), 300 * 1000/g' "$HANDLER"
```

---

## 第三步：OAuth 授权（用户只需做一个操作）

### 3.1 启动 login 进程，获取 code_challenge

```bash
kill $(lsof -ti:3000) 2>/dev/null; sleep 1
nohup npx -y @larksuiteoapi/lark-mcp login \
  -a <APP_ID> -s <APP_SECRET> > /tmp/lark-oauth.log 2>&1 &
sleep 5 && cat /tmp/lark-oauth.log
```

从日志的 Authorization URL 中提取 `code_challenge=` 后面的值。

### 3.2 服务端 curl /authorize（必须执行，存储 PKCE 状态）

> 跳过此步会导致 callback 时报错 `PKCE validation failed: code challenge not found`

```bash
FEISHU_URL=$(curl -s -w "%{redirect_url}" -o /dev/null \
  "http://localhost:3000/authorize?client_id=client_id_for_local_auth&response_type=code&code_challenge=<CODE_CHALLENGE>&code_challenge_method=S256&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Fcallback&state=reauthorize")
echo "$FEISHU_URL"
```

### 3.3 让用户完成授权

把上一步输出的 `https://open.feishu.cn/...` 链接发给用户，**明确告知以下三点**：

> **第一步：** 在浏览器打开上面的链接，用飞书账号登录并点击授权。
>
> **第二步：** 授权完成后，浏览器会自动跳转到一个以 `http://localhost:3000/callback?code=...` 开头的地址，并显示"无法访问此网站"或"连接被拒绝"的错误页面——**这是完全正常的，不要关闭页面。**
>
> **第三步：** 忽略错误提示，直接复制浏览器地址栏的完整 URL（以 `http://localhost:3000/callback` 开头），粘贴发给我。

### 3.4 收到 callback URL 后立即提交

```bash
curl -s "<用户发来的完整 URL>"
# 返回 "success, you can close this page now" 即成功
```

### 3.5 验证

```bash
npx -y @larksuiteoapi/lark-mcp whoami
```

看到 `AccessToken Expired: false` 即完成。重启 Claude Code 后飞书 MCP 工具即可使用。
