# PUA — 使用与权限配置说明

本仓库是一个**模板**：workflow 用「**代码仓（只读） + 你的私有 Gist**」模型——
你的 TG 会话只从 Secret 读取、不入仓库；域名归属手机号(`domain_phones`)加密后存进**你自己的私有 Gist**。

---

## 1. 数据流向（先看懂）

| 数据 | 存哪 | 谁有 |
|---|---|---|
| 代码（`ppua_renew.py` 等） | 开发者代码仓 | 只读（`PRIVATE_REPO_TOKEN`） |
| TG 会话（telethon session） | **只从 Secret `TG_LOGIN_BATCH` 读取**，不落任何仓库/Gist | 只有你 |
| 域名↔归属手机号（`domain_phones`） | **你自己的私有 Gist**，且被 `STATE_ENCRYPT_KEY` 加密 | 只有你（有你的 GIST_PAT 的人） |
| 账号密码 / TG 注册表 | 你触发仓的 Secrets | 只有你 |

> 每个人跑 = 从**代码仓**拉代码执行；会话用 secret 的；归属手机号回写**自己私有 gist**。
> 开发者不接触你的会话/手机号；你也不碰别人的。

---

## 2. 你要准备的（三次配置）

### 2.1 私有 Gist（持久化 domain_phones）
1. 打开 https://gist.github.com → **Create secret gist**（必须私有）；
2. 文件名写 **`domain_phones.json`**，内容先填 `{}`，保存；
3. 地址形如 `https://gist.github.com/你的用户名/abcdef0123456789` → 末尾 `abcdef0123456789` 就是 **GIST_ID**。

### 2.2 两个 Token / 一个密钥
生成位置：
- PAT：右上头像 → **Settings → Developer settings → Personal access tokens**
- Gist 只能用 **classic** token：`Tokens (classic)` → Generate new token → 勾选 **`gist`** scope

| 配置项 | 类型 | 生成/取值 | 用途 |
|---|---|---|---|
| `GIST_PAT` | classic PAT（`gist` scope） | classic token 勾 `gist` | 代码读写你的私有 gist |
| `PRIVATE_REPO_TOKEN` | Fine-grained PAT，只读 | 开发者发给你 / 开发者配 | 检出代码仓（只读） |
| `STATE_ENCRYPT_KEY` | Fernet key（44 位 base64） | 见 2.3 | 加密 gist 里的 domain_phones |

### 2.3 生成 STATE_ENCRYPT_KEY（推荐，自己生成自己保管）
```bash
# 零依赖一行：32 个随机字节 base64url → 合法 Fernet key（等效 Fernet.generate_key()）
python3 -c "import base64, os; print(base64.urlsafe_b64encode(os.urandom(32)).decode())"
```
- 输出的串（形如 `UPVUvzQyddLIFBnNIHE=…`）整行存为 Secret `STATE_ENCRYPT_KEY`；
- **同一人保持同一个 key**；换 key 会让旧 gist 内容无法解密；
- 不配置 = 明文存入 gist（代码会告警），不推荐。

### 2.4 触发仓（fork 本仓库，或拷贝 workflow）
1. fork 本仓库（或新建仓库把 `.github/workflows/ppua-AutoRenew.yml` 放进）；
2. 在触发仓配 Secrets / Variables（见下）。

---

## 3. 触发仓配置

路径：触发仓 → **Settings → Secrets and variables → Actions**

### Secrets
| Secret | 填什么 |
|---|---|
| `PRIVATE_REPO_TOKEN` | 开发者发的只读 PAT |
| `GIST_ID` | 你私有 gist 的 id（§2.1） |
| `GIST_PAT` | classic PAT（`gist` scope）（§2.2） |
| `STATE_ENCRYPT_KEY` | Fernet key（§2.3） |
| `NICUA_BATCH_XIONG2` | 你的 nic.ua 账号（格式见代码仓 README） |
| `TG_LOGIN_BATCH_XIONG2` | 你的 TG 会话注册表（`setup_tg_session.py` 输出，含 session 串） |
| `PROXY_CONTENT`（可选） | 代理 URL，留空=直连 |
| `DASHBOARD_URL` / `DASHBOARD_PASSWORD`（可选） | 域名监控（阶段 E） |
| `RENEWHELPER_URL` / `RENEWHELPER_PASSWORD`（可选） | RenewHelper（阶段 F） |

### Variables
| Variable | 填什么 |
|---|---|
| `SOCKS_PORT`（可选） | 默认 `10808` |
| `APP_TIMEZONE`（可选） | 默认 `Asia/Shanghai` |
| `GATE_BEFORE`（可选） | 默认 `none`（跑完全程）；人工校对再设 `order_submit` |
| `COLLECT_TG_FOR_DOMAIN` | 首次先用 `true` 学一次，跑完改回 `false` |
| `DASHBOARD_LABEL` / `RENEWHELPER_LABEL`（可选） | 报告标签 |

---

## 4. 权限矩阵

| 配置 | 开发者代码仓 | 你的 Gist | 你的会话 |
|---|---|---|---|
| `PRIVATE_REPO_TOKEN` | 只读代码 | 不可见 | × |
| `GIST_PAT` | 不可见 | 读写自己的 gist | × |
| `TG_LOGIN_BATCH`(Secret) | × | × | 只在你的触发仓 Secrets |

- `PRIVATE_REPO_TOKEN` 即使被转给别人：只能看代码，写不动开发者仓、也碰不到你的 Gist。
- 你的 Gist / Secrets 只有你有；domain_phones 落盘即加密（`enc:`），**拿到文件也不是明文**。

---

## 5. 首次跑通清单
1. 建私有 Gist(`domain_phones.json`=`{}`) → 记下 GIST_ID；
2. 生成 `STATE_ENCRYPT_KEY`；创建 classic PAT(`gist`) 与 PRIVATE_REPO_TOKEN；
3. 触发仓复制 workflow，配好 Secrets（PRIVATE_REPO_TOKEN / GIST_ID / GIST_PAT / STATE_ENCRYPT_KEY / NICUA_BATCH / TG_LOGIN_BATCH）和 Variables；
4. 触发仓 → **Actions → PPUA AutoRenew → Run workflow**（或等 cron）；
5. 第一次建议 `COLLECT_TG_FOR_DOMAIN=true` 学全手机号；确认后改回 `false`；
6. `GATE_BEFORE` 默认 `none` 跑完；不确定先设 `order_submit` 校验一次。

---

## 6. 安全提醒
- **Gist 必须私有**；`GIST_PAT` 也要只写你自己的 gist。
- 会话在 Secrets，不进仓库/Gist；domain_phones 进 Gist 前按 `STATE_ENCRYPT_KEY` 加密。
- 换 `STATE_ENCRYPT_KEY` 会让旧 Gist 内容解不开；谨慎轮换。
