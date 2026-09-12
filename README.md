# PPUA

自动完成 **NIC.UA `.pp.ua` 域名**的免费续期 + 激活 + 监控同步，支持**多账号**，可在 GitHub Actions 上定时无人值守运行。

需求与完整设计见 [REQUIREMENTS.md](./REQUIREMENTS.md)。

---

## 运行流程

```
[P] 代理出口（可选）→ [A] NIC.UA 登录 → 检查域名列表（Renew 可用 / 已有 "Activation is required"）
    → 待激活域名【跳过下单，直接激活】；待续期域名走完整下单 → 获取 Payer 联系人
    → 校验手机号在 /en/my/contacts → [G] TG 预检门（用 Payer 手机号探活并比对）
    → 加入购物车（校验 0.00₴）→ 选择已保存联系人 → 提交免费订单
    → "Activation is required" → [B] @ppuabot 激活（提取验证码）
    → [C] apu.drs.ua（隐形 Turnstile 等按钮启用后提交）
    → [D] 校验（Activation is required 消失 且 到期日 +≥360 天）
    → [E] 域名监控同步（可选）→ [F] RenewHelper 同步（可选）
    → TG 报告（见下）
```

> 已存在 `Activation is required` 的域名（人工或上次流程已完成下单）会被识别出来，直接执行
> 激活（B/C/D），不再重复下单。

---

## 环境变量配置（总清单）

在仓库 **Settings → Secrets and variables → Actions** 配置；**Secrets** 存敏感值，**Variables** 存可公开配置。

| 类型 | 变量 | 是否必填 | 说明 |
|---|---|---|---|
| **Secret** | `NICUA_BATCH` | ✅ 必填 | 多账号：`email,password[,tg_token,tg_chat][#白名单|...][!黑名单|...]`；详见 NICUA_BATCH 示例 |
| **Secret** | `TG_LOGIN_BATCH` | ✅ 必填 | TG 用户会话+每手机号通知：`phone,notify_bot_token,notify_chat_id,api_id,api_hash,session`；`setup_tg_session.py` 输出 |
| **Secret** | `PRIVATE_REPO_TOKEN` | ✅ 必填 | **只读** Fine-grained PAT，检出开发者私有代码仓；创建位置/权限见下方旧章节（只读即可，不再用于回写 state） |
| **Secret** | `GIST_ID` | ✅ 必填 | 你的私有 gist 的 id（存 domain_phones）；创建方法见下方 |
| **Secret** | `GIST_PAT` | ✅ 必填 | **classic PAT（gist scope）**，代码用它读写你的私有 gist |
| **Secret** | `STATE_ENCRYPT_KEY` | ✅ 推荐 | 状态加密 **Fernet key**；生成方法见下方；一共44个字符 |
| **Secret** | `PROXY_CONTENT` | 可选 | 代理 URL（推荐填写）；留空=直连 |
| **Secret** | `DASHBOARD_URL` | ✅ 阶段E启用时必填 | 域名监控地址（开源：[CF-Domain-AutoCheck](https://github.com/decadefaiz/CF-Domain-AutoCheck)） |
| **Secret** | `DASHBOARD_PASSWORD` | ✅ 阶段E启用时必填 | 域名监控密码 |
| **Secret** | `RENEWHELPER_URL` | ✅ 阶段F启用时必填 | RenewHelper 地址（开源：[renewhelper](https://github.com/ieax/renewhelper)） |
| **Secret** | `RENEWHELPER_PASSWORD` | ✅ 阶段F启用时必填 | RenewHelper 密码 |
| **Variable** | `DASHBOARD_SYNC_ENABLED` | 可选 | `true` 启用阶段 E（workflow 默认 `true`） |
| **Variable** | `DASHBOARD_LABEL` | 可选 | 报告标签（阶段 E），默认 `域名监控同步` |
| **Variable** | `DASHBOARD_GROUP` | 可选 | 域名监控写入的 `registrar` 分组名，默认 `nic.ua` |
| **Variable** | `RENEWHELPER_SYNC_ENABLED` | 可选 | `true` 启用阶段 F（workflow 默认 `true`） |
| **Variable** | `RENEWHELPER_LABEL` | 可选 | 报告标签（阶段 F），默认 `RenewHelper同步` |
| **Variable** | `RENEWHELPER_TAG_DOMAIN` | 可选 | RenewHelper 新增服务时把域名写进 `tags`（标签=域名），默认 `true` |
| **Variable** | `RENEWHELPER_GROUP` | 可选 | RenewHelper 新增服务的分组标签（tags 表达），默认 `域名` |
| **Variable** | `SOCKS_PORT` | 可选 | 本地 socks5 端口，默认 `10808` |
| **Variable** | `GATE_BEFORE` | 可选 | 下单流程「门栓」：**默认 `none`（关闭，跑完全程）**。传 step_id 可在该步前 快照+退出 方便校对：`renew_click` / `cart_clean` / `cart_continue` / `contact_select` / `order_submit` / `tg_activation` / `apu_submit`；设 `off` 或 `none` = 关闭（正常跑完） |
| **Variable** | `APP_TIMEZONE` | 可选 | 报告时区，默认 `Asia/Shanghai` |

> 通知分流：报告按**归属手机号**发 → `TG_LOGIN_BATCH` 该手机号行的 `notify_bot_token/notify_chat_id`（缺省回退 `NICUA_BATCH` 账号级）。

### 生成 STATE_ENCRYPT_KEY（推荐做法）

每个使用者**自己生成、自己保管**（还有 Gist 的人各自不同 key；同一人保持同一个 key，别乱换，换 key 会让旧 state/Gist 无法解密）。生成只用标准库，无需额外依赖：

```bash
# 零依赖一行：32 个随机字节 base64url → 合法 Fernet key（等效 Fernet.generate_key()）
python3 -c "import base64, os; print(base64.urlsafe_b64encode(os.urandom(32)).decode())"
```

把输出的 **44 位 urlsafe base64** 串（形如 `UPVUvzQyddLIFBnNIHE=…`）整行填进仓库 Secret `STATE_ENCRYPT_KEY`。
不配置 = state/Gist 明文落盘（代码会打印告警），不推荐。

### GIST_ID / GIST_PAT 怎么生成（持久化域名归属手机号用）

**GIST_ID** = 你的私有 Gist 的 id（就是它 URL 末尾那串）。拿到它：

1. 浏览器登录 GitHub；
2. 打开 https://gist.github.com → 右上 **Create secret gist**（记住：必须选 **secret**（私有），别选 public）；
3. 文件名 `Filename including extension…` 填 **`domain_phones.json`**，内容先填 `{}`，保存；
4. 保存后地址栏形如：
   `https://gist.github.com/你的用户名/abcdef0123456789abcdef`
   → 末尾这段 `abcdef0123456789abcdef` 就是 **`GIST_ID`**，把它填进仓库 Secret `GIST_ID`（或 Variable）。

**GIST_PAT** = 一个只写好你 gist 的 **classic PAT**（Gist 不支持 Fine-grained token）：

1. 右上头像 → **Settings** → 拉到最左菜单 **Developer settings**；
2. **Personal access tokens → Tokens (classic) → Generate new token (classic)**；
3. Select scopes 里勾选 **`gist`**（把 gist 权限单独勾上即可）；
4. 生成后立刻复制一次（只显示这一次），整段填进仓库 Secret `GIST_PAT`。

> 一句话：`GIST_ID` 告诉代码「读写哪个 gist」，`GIST_PAT` 是访问它的钥匙；两者都要是你自己的。
> Gist 内容会先经 `STATE_ENCRYPT_KEY` 加密（`enc:`），拿到 gist 也看不到明文手机号。

### Gist 读写流程（domain_phones 怎么往返）

`domain_phones`(**域名↔激活归属手机号**)的持久化**全程走 GitHub API，不落地本地文件**：

```
运行中 load_domain_phones(email)  = GET  /gists/{GIST_ID}
                                    → files['domain_phones.json'].content
                                    → 解密(enc:) → 内存 dict → 取该账号映射
账号收尾 flush_pending_domain_phones = 合并本次学习进内存 dict
                                    → 整份 PATCH /gists/{GIST_ID}
                                    → content = STATE_ENCRYPT_KEY 加密后的新 JSON
```

- **单一数据源**：gist 就是唯一真值，本地 `state/`（`STATE_DIR`）只在**未配置 Gist 时**做本地兜底，不产生文件分叉；
- 读：一次 `GET` 全量进内存；写：整个 gist 重建后一次 `PATCH`；
- 内容始终 `enc:` 加密存储；多账号映射共用一个 gist（按账号邮箱分键），每账号只在收尾时写回。

> ⚠️ **并发注意**：Gist 的 `PATCH` 是**整文件替换**。若你同时跑多个 workflow 且**共用同一个 `GIST_ID`**，
> 后写的会整份覆盖先写的（丢更新）。代码已做「**写前重读 + 写后校验告警**」缓解，但**并发流请各自用不同的
> `GIST_ID`**（一个并发写入流 = 一个私有 gist）做物理隔离；同一仓库多个 workflow 也可用相同
> `concurrency.group` 串行化。

> REQUIREMENTS 对应章节：§「状态持久化（Gist）」。

### 必填

| Secret | 说明 |
|---|---|
| `NICUA_BATCH` | **多账号**。每行 `email,password[,tg_notify_token,tg_notify_chat][#白名单|...][!黑名单|...]`。<br>`#`=账号级白名单（只处理这些域名）、`!`=账号级黑名单（排除这些），名单内多个域名用 **`|` 竖线** 分隔、均可选可单用；账号级优先，未配回退全局名单；多行=多账号；**名单同时约束「续期下单」与「待激活」两个阶段**（Activation is required 的域名同样先过名单，黑名单/不在白名单的待激活域名不会被执行激活）；示例见下 |
| `TG_LOGIN_BATCH` | **TG 用户会话 + 每手机号通知** 注册表（`setup_tg_session.py` 输出格式）：`phone,notify_bot_token,notify_chat_id,api_id,api_hash,session`，分号/换行分隔多账号。**session 必须先手动跑一次 `setup_tg_session.py` 生成并填入**（`state/tg_<phone>.session` 是运行时自动回写的持久化文件，不是初次配置方式）；**该行的 notify_bot_token/notify_chat_id 用于给该手机号所有者发报告**（同一账号挂了别人的域名时，通知正确分流） |
| `PRIVATE_REPO_TOKEN` | ⚠️ 当前用途：**只读**检出开发者私有代码仓（不再回写 state）。持久化已改为 私有 Gist(`GIST_ID`/`GIST_PAT`) + `STATE_ENCRYPT_KEY` 加密，见上方 [环境变量配置总清单](#环境变量配置总清单) 与新章节 |

> **报告通知发给谁**：按**域名归属手机号**取 `TG_LOGIN_BATCH` 对应行的 `notify_bot_token/notify_chat_id`；
> 该行没配，才回退用 `NICUA_BATCH` 账号级 `tg_notify_token/tg_notify_chat`。这样同一 NIC.UA 账号下的
> 别人域名，报告会发给各自归属者，而不是都发给自己。

#### TG 相关变量从哪来（`TG_LOGIN_BATCH`）

**`TG_LOGIN_BATCH`** 每行 6 个字段，逐字段说明：

| 字段 | 说明 | 怎么弄 |
|---|---|---|
| `phone` | 用来激活域名的 **TG 账号手机号**（就是 NIC.UA 里该域名 Payer 那个电话，带 `+` 国码） | 直接用你自己的 TG 手机号 |
| `notify_bot_token` | 给"该手机号所有者"发报告的 **bot token**（可留空=回退账号级） | Telegram 里找 **@BotFather** → `/newbot` → 起名 → 复制它给的 `123456:AA...` |
| `notify_chat_id` | 收报告的目标 **chat id**（可留空=回退账号级） | 给自己发：找 **@userinfobot** 发任意消息拿纯数字 ID；给群/频道发：把 bot 拉进去 |
| `api_id` | **Telegram 开发者 App 凭证**（纯数字） | 打开 https://my.telegram.org → 登录 → **API development tools** → 填 App 名称 → 得到 `api_id` / `api_hash`。**同一个人的所有手机号可共用一套** |
| `api_hash` | 同上，32 位十六进制串 | 上面同一个页面 |
| `session` | **用户登录态**（免验证码） | **必须**手动本地跑一次 `setup_tg_session.py` 生成，把输出的 session 字符串填进本字段 |

> 注意：这里的 creds 是「**用户会话**」（激活必须用用户身份和 @ppuabot 对话），**不是 bot token**。
> `state/` 里的会话文件由代码内置密钥加密，无需额外配置。

**完整初始化流程（连起来）**：
```bash
# 1) my.telegram.org 拿 api_id / api_hash；@BotFather 拿 bot_token；@userinfobot 拿 chat_id
# 2) 手动在本地生成 TG 会话（按提示输验证码）——这一步必做，程序拿不到 session 无法登录 TG
TG_LOGIN_BATCH='+15551234567,123456:AA...bot_token,123456789,<api_id>,<api_hash>, ' \
  python3 setup_tg_session.py
# 3) 把输出的完整一行（含 session）存进 Secret TG_LOGIN_BATCH，session 字段不能为空
```

#### `PRIVATE_REPO_TOKEN` 怎么创建 / 需要什么权限

作用：workflow 用它**检出私有代码仓**并把 `state/` 加密状态**回写**回私有仓。创建于 GitHub 的 **Fine-grained personal access token**：

- 入口：**https://github.com/settings/personal-access-tokens/new**（Settings → Developer settings → Personal access tokens → Fine-grained tokens → Generate new token）
- 关键配置项：
  - **Resource owner**：选你自己的账号/组织（首次创建需邮件/浏览器确认批准一次）
  - **Expiration**：选 **No expiration（永不过期）**，一劳永逸，不用定期重生成
  - **Repository access** → **Only select repositories** → 勾选 `jyucoeng/ppua-auto-private`（只需私有仓，别多授权的库）
  - **Permissions** → **Repository permissions** → `Contents` 选 **Read and write**（检出代码 + 回写 `state/` 都需要写权限）
  - 其它保持默认即可（`Metadata: Read` 会随 Contents 自动带上）
- 创建后把 token 复制，存进**公开仓** Secret `PRIVATE_REPO_TOKEN`（不是私有仓！workflow 在公开仓读取 secrets）

> ⚠️ 老式 Classic PAT 也行（scope 勾 `repo`），但 Fine-grained 只授权单个私有仓、权限更小更安全，推荐用它。

#### NICUA_BATCH 配置示例（含账号级黑白名单）

```bash
# 基本账号：处理该账号全部可续期域名
a@gmail.com,Pass1

# 只处理白名单里的多个域名（名单内用 | 竖线分隔，别用逗号/分号）
b@gmail.com,Pass2#example1.pp.ua|example2.pp.ua|example3.pp.ua

# 排除某些域名（黑名单单独用最省事）
c@gmail.com,Pass3!retire-me.pp.ua|old2.pp.ua

# 全配：白名单 + 黑名单 + 通知 bot
# 生效结果：白名单 {example1, example2, keep} − 黑名单 {example2, skip}
#          = 实际只处理 example1、keep
d@gmail.com,Pass4,token4,chat4#example1.pp.ua|example2.pp.ua|keep.pp.ua!example2.pp.ua|skip.pp.ua
```

- 一句话语义：**白名单=入围，黑名单=出局**；都有时 = 先按白名单锁定，再剔掉黑名单里的
- 白/黑名单可在同一账号行里**同时出现**，也可只用其一
- 域名筛选**只按账号级** `#白名单` / `!黑名单`（`.gitignore` 里没有全局名单了）
- **名单内多个域名用 `|` 竖线分隔**，别用逗号（逗号是字段分隔符）、也别用分号（分号是账号分隔符）；
  大小写不敏感
- **`;` 留给多账号**：`NICUA_BATCH` 多账号用换行分隔、`TG_LOGIN_BATCH` 多账号用分号/换行；
  名单用 `|` 后，`;` 不会与域名名单歧义
- **密码可含 `#` / `!`（及 `;`）**：解析上「按逗号分字段，前两个字段必然是 email、密码」；
  名单段只在行尾。取**最后一个 `#`** 作白名单切点（`#` 之后到行尾只会是名单内容）——
  只有当其后整体是一串含点域名时才生效；否则该 `#` 一律视为密码的一部分。
  例：`a@gmail.com,Pa#ss!word` → 密码就是 `Pa#ss!word`（无名单）；
  `b@gmail.com,Pa2#a.pp.ua|b.pp.ua` → 密码 `Pa2` + 白名单 `{a.pp.ua, b.pp.ua}`。

#### 白名单 + 黑名单同时出现时，怎么算最终处理哪些？

```
候选域名 = 该账号下「可续期 + 待激活」的全部域名

候选：      [ example1, example2, example3, keep, skip ]
白名单：#example1|example2|keep        → 入围 { example1, example2, keep }
黑名单：!example2|skip                 → 从中再排除 { example2, skip }
最终处理：                              → 仅 { example1, keep }
```

三步口诀：**① 候选 → ② 用白名单圈定（没配白名单=全量）→ ③ 用黑名单剔除（没配黑名单=不剔）**。
用代码说就是：`候选 ∩ 白名单(若有) − 黑名单(若有)`。

> 这套过滤对**「待续期」和「待激活」两种域名一视同仁**：已下单但还挂着 `Activation is required` 的域名同样先过名单——黑名单命中、或不在白名单内的待激活域名会被跳过，**不会被执行激活**；想放行待激活的写法见黑/白名单配置（把该域名加入白名单或从黑名单移除）。

### 代理（可选但推荐）

| Secret | 说明 |
|---|---|
| `PROXY_CONTENT` | 代理 URL，支持 `vless/vmess/trojan/ss/hysteria2/tuic/anytls/socks5`。留空 = 直连 |

### 阶段 E / F（开关=仓库 Variables，非 Secret）

| 位置 | 变量 | 说明 |
|---|---|---|
| Variables | `DASHBOARD_SYNC_ENABLED` | `true` 启用「域名监控」同步（阶段 E） |
| Secret | `DASHBOARD_URL` | 例 `https://dashboard.example.com` |
| Secret | `DASHBOARD_PASSWORD` | 独立密码 |
| Variables | `DASHBOARD_LABEL` | 报告标签，默认 `域名监控同步`（如 "域名监控"） |
| Variables | `DASHBOARD_GROUP` | 域名监控里的分组名（写入 `registrar` 字段），默认 `nic.ua` |
| Variables | `RENEWHELPER_SYNC_ENABLED` | `true` 启用「RenewHelper 通知」同步（阶段 F） |
| Secret | `RENEWHELPER_URL` | 例 `https://renewhelper.example.com/` |
| Secret | `RENEWHELPER_PASSWORD` | 独立密码 |
| Variables | `RENEWHELPER_LABEL` | 报告标签，默认 `RenewHelper同步` |

> 开关 / 时区不是机密，放 **仓库 Variables**（Settings → Secrets and variables → Variables），
> 协作者可直接改而不用碰 Secrets。

### 其它

| 位置 | 变量 | 说明 |
|---|---|---|
| Variables | `SOCKS_PORT` | 本地 socks5 端口，默认 `10808` |
| Variables | `APP_TIMEZONE` | 报告"运行时间(北京时间)"所用时区，默认 `Asia/Shanghai` |
| Variables | `GATE_BEFORE` | 下单流程「门栓」，默认 `none`（关闭，正常跑完）；设 step_id 可在该步前停+退出；`off`/`none`=正常跑完；其它停点见上方总清单 |

---

## 公开仓需要设置的权限（yml 所在仓库 = `jyucoeng/ppua-auto-public`）

> workflow 在**公开仓**的 Actions 上运行，所以以下几项全部在公开仓设置/生效：

| # | 设置项 | 位置 | 值 / 说明 |
|---|---|---|---|
| 1 | **Workflow permissions** | Settings → Actions → **General → Workflow permissions** | 选 **Read and write permissions**。默认新仓库是只读（Read repository contents），会导致「提交执行时间到公开仓」一步 push 失败。直达：`https://github.com/jyucoeng/ppua-auto-public/settings/actions` |
| 2 | **yml 内 GITHUB_TOKEN 权限** | `ppua-AutoRenew.yml` 的 `permissions:` 块 | 已内置 `contents: write` + `actions: write`（无需手动配） |
| 3 | **Secrets** | Settings → Secrets and variables → Actions → **Secrets** | `NICUA_BATCH` / `TG_LOGIN_BATCH` / `PRIVATE_REPO_TOKEN` 等（见[总清单](#环境变量配置总清单)）。`PRIVATE_REPO_TOKEN` 必须是 Fine-grained PAT，授权私有仓 `ppua-auto-private` 的 `Contents: Read and write` |
| 4 | **Variables** | Settings → Secrets and variables → Actions → **Variables** | `DASHBOARD_SYNC_ENABLED`、`RENEWHELPER_SYNC_ENABLED`、`SOCKS_PORT`、`APP_TIMEZONE` 等非机密配置 |
| 5 | **检出私有仓** | yml 内 `actions/checkout` | 用 `PRIVATE_REPO_TOKEN` 检出 `jyucoeng/ppua-auto-private`，无需给公开仓开其它私有仓权限 |

> **私有仓 `ppua-auto-private` 不用配 Actions**：它不跑 workflow，只被公开仓检出 + 回写 `state/`（通过 `PRIVATE_REPO_TOKEN`）。

## 首次使用步骤

1. **创建私有代码仓**：`PRIVATE_REPO_TOKEN`
   创建方法见上文（Fine-grained PAT，仅授权该私有仓，Contents Read+write），token 配置在**公开仓** Secret。
2. **设置公开仓权限**：按上方「[公开仓需要设置的权限](#公开仓需要设置的权限yml-所在仓库--jyucoengppua-auto-public)」清单配置
   （Workflow permissions → Read and write、Secrets、Variables；主要放在 `ppua-auto-public`）。
3. **初始化 TG 会话（一次性，本地执行，必做）**：
   ```bash
   TG_LOGIN_BATCH='+15551234567,,,123456,abcdefgh, ' python3 setup_tg_session.py
   # 按提示输入验证码；输出完整 TG_LOGIN_BATCH 行（含补全后的 session）
   ```
   把输出的完整行**存入 `TG_LOGIN_BATCH`**（session 字段不能留空，否则程序缺少 TG 会话无法预检/激活）。
4. **（可选）确认代理**：`PROXY_CONTENT` 填一个可用节点。
5. **手动触发一次**：Actions 里 `Run workflow` 直接运行（已去掉 dry_run/心跳干跑模式），观察日志确认
   代理/NIC.UA 登录/TG 会话/E/F 连通。

---

## 手动触发（workflow_dispatch）

定时：默认**每月 1 日 00:43 UTC**（`.pp.ua` 按年度续期，月频足够）。可改 `ppua-AutoRenew.yml` 的 `cron`；
也可随时用 `workflow_dispatch` 手动触发。

---

## 报告模板（TG 通知）

```text
【单域名单条】
🎮 PPUA 续期报告
🕐 运行时间: 2026-09-11 12:31:57 (Asia/Shanghai)
🕐 运行时间: 2026-09-11 04:31:57 (UTC)
🌐 IP 信息: 185.156.53.*** (UA) [✅ 代理]
👤 账号: z******t@g***l.com          ← 域名也掩码，TG 不会自动加邮箱链接

🖥️ 域名: example1.pp.ua
✈️ tg账号: +4475****9202
⏰ 到期时间(前): 2027-01-02
⏰ 到期时间(后): 2028-01-02
✅ 续期成功
❌ 续期失败 / 原因：{errorReason}   (失败时)
📊 域名监控同步✅更新              (阶段 E 启用时，已压缩)
📊 RenewHelper同步✅新增            (阶段 F 启用时，已压缩)
```

> **多域名合并**：同一通知渠道下多个域名（成功/失败）处理后合并成 1~N 条，头一致 +
> `📦 本次处理 N 个域名（成功 X / 失败 Y）`，逐域名按块列出；超长自动拆条，标题带进度 `（合并 1/2）`。
>
> **域名列表截图**：每个账号处理完后单独发一张 Pillow 绘制的 PNG 长图
> （账号置顶 + 域名/到期逐行，域名再多也整张显示）。
>
> **成功判定**：`apu.drs.ua` 出现 `Done! …activated and ready to use` 即视为成功；
> 新到期日在该账号全部域名处理完后批量回读（60s 首等 + 至多 3 轮×60s，只读到期日，
> 忽略 Activation 标签滞后），读不到用「旧到期+365」预期兜底，不误判失败。

> **续期规则**：pp.ua 免费续期是**在原到期日基础上 +365 天**（不是"今天 +365"）。
> 例如原本 `02 Jan 2027`，激活/续期成功后 → `02 Jan 2028`；跨闰年实际可能 +366 天，
> 因此代码以「到期日增长 ≥360 天」确认新到期日（见 §7 阶段 D）。

> 手机号掩码显示的是**本次激活用到的 TG 手机号**（`get_me().phone`，为空时用注册表兜底），
> 可顺带校验本次用的手机号是否与 Payer 一致。

---

## 架构要点（详见 REQUIREMENTS.md）

- **代理前置**：登录 nic.ua 前先起代理，全部动作 A~F 都走它（规避数据中心 IP 风控）。
- **TG 预检门**：下单前用 Payer 手机号探活 TG 会话（`get_me()` + 手机号比对；`get_me().phone` 偶尔
  为空时用 `TG_LOGIN_BATCH` 注册表手机号兜底），失败即通知并终止，不产生任何订单副作用。
- **归属学习 + 自动切换**：若 `@ppuabot` 回复 "can be activated by X only, but you act as Y"，
  即获权威归属手机号 X → 记入 `state/domain_phones.json`（下次预检直接用），并在本次运行里
  **自动换成 X 的会话重试激活**。
- **待激活域名**：行上已有 "Activation is required" 的域名（人工/上次已完成下单）会被识别，
  跳过下单直接激活。
- **会话自治**：Telethon 连接按需各建一次——TG 预检门探活（`tg_probe`）与激活对话（`_activate_flow`，
  @ppuabot 提取验证码）各建立/断开一次；会话本身无 TTL，靠月度运行保持活跃，失效时配 `TG_LOGIN_BATCH`
  里该手机号行重登（账号级事件才可能需人工重预授权）。
- **状态回写**：TG 会话/归属/去重**加密后放私有代码仓 `state/`**（只 2 个仓），每次运行 git 回写；
  `concurrency` + rebase 重试防并发。
- **审计**：每个关键检查点用 `snapshot()` 同时保存 **HTML dump + 截图**（含 apu 提交后下一页），
  随 `upload-artifact` 打包归档；失败必推 TG。

## 已知边界（诚实声明）

- nic.ua / apu.drs.ua 的**选择器可能随改版失效**：全部集中在 `ppua_renew.py` 顶部 `SEL` 与 URL 常量，
  改一行适配；建议观察月度运行日志提前暴露。
- `@ppuabot` 为第三方 bot，提示语变化会导致激活失败（预检门会提前报警；"can be activated by X only"
  提示还会被用作归属学习+自动切换）。
- **apu.drs.ua 有 Cloudflare Turnstile（隐形模式）**：Continue 初始 disabled，等 Turnstile 后台自动
  验证通过、按钮解除 disabled 后才点击提交（不再用 `uc_gui_click_captcha`）；提交后检测
  `Done! ... activated and ready to use` 判定激活成功。
- **联系人归档**：若 Payer 联系人被归档，详情页显示「Restore from archive」——脚本会自动点击恢复并
  重新打开详情页继续流程；「Archive this contact」=正常状态，无需处理。
- pp.ua 免费续期政策不受控；金额校验前置会在"收费化"时停止并告警。
- `expiry_before`（到期时间前）字段当前为尽力求取，读不到显示 `-`；报告不再输出「续期时间」行。

---

## 说明

需求、论证、验证清单见 [REQUIREMENTS.md](./REQUIREMENTS.md)（16 章）。实现基于框架
`ppua_renew.py`（seleniumbase + telethon + requests）与 `ppua-AutoRenew.yml` 骨架，按文档 §16 映射改造。
