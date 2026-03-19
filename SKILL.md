---
name: happycapy-feishu
description: 为 Claude Code 安装并授权飞书 MCP，让 Claude 直接操作飞书消息、文档、多维表格、日历等。触发词：安装飞书 MCP、配置飞书、接入飞书、飞书 MCP setup、connect feishu、飞书重新授权、飞书 token 过期、reauthorize feishu
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

### 3.1 启动授权服务器，自动获取飞书授权链接

> Claude 在后台完整执行此流程，用户**不会**看到任何 localhost 或 authorize 链接。

```bash
# 1. 确保 3000 端口未被占用
kill $(lsof -ti:3000) 2>/dev/null; sleep 1

# 2. 后台启动 login 进程
nohup npx -y @larksuiteoapi/lark-mcp login \
  -a <APP_ID> -s <APP_SECRET> > /tmp/lark-oauth.log 2>&1 &
sleep 5 && cat /tmp/lark-oauth.log
```

从日志的 Authorization URL 中提取 `code_challenge=` 后面的值（记为 `CODE_CHALLENGE`）。

```bash
# 3. 服务端 curl /authorize，触发 PKCE 状态存储，获取飞书 OAuth 跳转链接
#    （必须执行，否则 callback 时报 PKCE validation failed）
FEISHU_URL=$(curl -sv -o /dev/null \
  "http://localhost:3000/authorize?client_id=client_id_for_local_auth&response_type=code&code_challenge=<CODE_CHALLENGE>&code_challenge_method=S256&redirect_uri=http%3A%2F%2Flocalhost%3A3000%2Fcallback&state=reauthorize" 2>&1 \
  | grep -i "^< location:" | sed 's/< [Ll]ocation: //' | tr -d '\r')
echo "$FEISHU_URL"
```

### 3.2 把飞书授权链接发给用户

> ⚠️ **严禁把任何 `localhost:3000/authorize`、`localhost:3000` 预览地址、或 capy 域名链接发给用户。只发 `open.feishu.cn` 链接。**

把上一步提取出的 `$FEISHU_URL`（`https://open.feishu.cn/...` 开头）发给用户，使用以下**标准话术**（原文发送，替换链接占位符）：

---

好了，这就是飞书 OAuth 直链。按以下步骤操作：

**第一步：** 用浏览器打开这个链接进行飞书授权

`<$FEISHU_URL>`

**第二步：** 用飞书账号登录授权后，浏览器会跳转到 `http://localhost:3000/...` 显示无法访问。此时复制浏览器地址栏的完整 URL 发给我（格式类似）：

```
http://localhost:3000/callback?code=xxxxxx&state=reauthorize
```

**第三步：** 我用这个 URL 完成 token 交换并保存。

> 注意：这个授权链接有 **5 分钟**有效期，请快速操作。若超时告诉我，我重新生成。

---

### 3.3 收到 callback URL 后提交

```bash
curl -s "<用户发来的完整 URL>"
# 返回 "success, you can close this page now" 即成功
```

### 3.4 验证

```bash
npx -y @larksuiteoapi/lark-mcp whoami
```

看到 `AccessToken Expired: false` 即完成。重启 Claude Code 后飞书 MCP 工具即可使用。
