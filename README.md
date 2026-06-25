# 本地密码管理器 — 单文件 SPA 技术架构文档

> 基于 `index.html` 完整代码分析，最后更新：2026-06-25

---

## 1. 架构总览

| 维度 | 说明 |
|---|---|
| 架构模式 | 单文件 SPA（Single-Page Application），所有 HTML/CSS/JavaScript 位于一个文件 |
| 运行模式 | 离线优先（Offline-first），**零外部依赖**，无需构建工具 |
| 存储后端 | File System Access API（首选）+ IndexedDB（文件句柄持久化）+ 手动下载/上传降级 |
| 加密原语 | Web Crypto API（`crypto.subtle`） |
| 兼容性 | Chrome / Edge 推荐，不支持 File System Access API 的浏览器以降级模式运行 |

### 核心流程

```
启动 → 开始屏幕 → 打开/新建文件 → 设置主密码/输入主密码
    → TOTP 验证（可选） → 主界面（侧边栏 + 内容网格）
    → 操作（增删改查） → 防抖加密写入文件
    → 15分钟无操作自动锁定
```

---

## 2. UI 屏幕与组件

### 2.1 开始屏幕（`#startScreen`）

- **功能**：应用入口，提供三个操作路径
- **上次文件提示**（`#lastFileHint`）：从 IndexedDB 恢复文件句柄后显示「继续使用此文件」按钮
- **操作按钮**：
  - 「打开其他密码文件」→ `openVaultFile()`
  - 「新建密码文件」→ `newVaultFile()`
- **使用说明**：嵌入在 start-box 中的 4 条指引

### 2.2 登录屏幕（`#loginScreen`）

包含三个子表单，通过 `display` 切换：

| 表单 ID | 用途 | 关键逻辑 |
|---|---|---|
| `#setupForm` | 首次设置主密码 | 双输入校验、PBKDF2 派生密钥、加密 verify 令牌 |
| `#loginForm` | 主密码解锁 | 反重放验证、失败计数 → 锁定 30s |
| `#totpForm` | 两步验证 | 6 位动态码验证（RFC 6238） |

### 2.3 主界面（`#mainScreen`）

#### 顶部栏（`.topbar`）

- 标题 + 文件标签（`#fileTag`，显示当前文件名）
- 操作区：「+ 添加」按钮、更多菜单（☰）：生成密码 / 访问记录 / 另存备份 / 锁定 / 切换主题

#### 侧边栏（`.sidebar` · 200px 宽）

- 搜索框（实时过滤，防抖 200ms）
- 分组列表：全部 + 用户自定义分组，显示条目计数
- 底部操作：管理分组 / 修改主密码 / 两步验证
- 统计面板：总账号数 + 最近修改时间

#### 内容区（`.content`）

- 自适应网格布局（`grid-template-columns: repeat(auto-fill, minmax(320px, 1fr))`）
- 条目卡片：分组标签、平台名、账号/密码/网址/备注字段、复制/显示/编辑/删除操作
- 虚拟滚动分页：每页 20 条，「加载更多」按钮

### 2.4 模态框

| 模态框 ID | 用途 | 关键内容 |
|---|---|---|
| `#entryModal` | 添加/编辑密码条目 | 分组选择、平台、账号、密码（含强度条+HIBP检查）、网址、备注 |
| `#genModal` | 密码工具 | 双 Tab：密码生成器 + 哈希计算器 |
| `#pwModal` | 修改主密码 | 旧密码验证 + 新密码双输入 → 重加密所有数据 |
| `#totpModal` | 两步验证设置 | 开启（生成密钥→验证码确认）/ 关闭（输入验证码确认） |
| `#groupModal` | 管理分组 | 列表展示、新增、编辑、删除、排序（↑↓）、图标选择器 |
| `#auditModal` | 访问记录 | 最近 20 次登录时间戳 + 浏览器标识 |
| `#confirmModal` | 通用确认框 | 标题 + 消息 + 回调 |

---

## 3. 密码学体系

### 3.1 密钥派生 — `deriveKey()`

```
PBKDF2(HMAC-SHA-256) → AES-256-GCM 密钥
```

| 参数 | 值 |
|---|---|
| 算法 | PBKDF2 |
| 哈希 | SHA-256 |
| 盐 | 16 字节随机（`crypto.getRandomValues`） |
| 当前迭代次数 | **1,000,000**（`CURRENT_ITERATIONS`） |
| 旧版迭代次数 | 500,000（`LEGACY_ITERATIONS`，v1 兼容） |
| 导出密钥 | `{name: 'AES-GCM', length: 256}` |
| 用途 | `encrypt`, `decrypt` |

旧文件自动迁移：登录时检测 `iterations < CURRENT_ITERATIONS`，用新迭代数重加密所有数据并保存。

### 3.2 加密/解密 — AES-256-GCM

```javascript
async function encryptData(plaintext, key) {
  const iv = crypto.getRandomValues(new Uint8Array(12));
  const ct = await crypto.subtle.encrypt({name:'AES-GCM', iv}, key, ...);
  return {ct: hex(ct), iv: hex(iv)};
}
async function decryptData(ctHex, ivHex, key) {
  return crypto.subtle.decrypt({name:'AES-GCM', iv:hex2buf(ivHex)}, key, hex2buf(ctHex));
}
```

- IV：12 字节随机，每次加密重新生成
- 认证标签：AES-GCM 内嵌，16 字节

### 3.3 压缩加密 — `compressAndEncrypt()` / `decryptAndDecompress()`

```
JSON → gzip 压缩 → AES-256-GCM 加密 → {ct, iv}
```

- 使用 `CompressionStream('gzip')` / `DecompressionStream('gzip')`
- 自动检测 gzip 头部（`0x1F 0x8B`），兼容旧文件未压缩格式

### 3.4 SHA-256 完整性校验

持久化时附上校验和：

```
json + '::HASH::' + SHA-256(json) → compressAndEncrypt()
```

解密时分离校验和，计算实际哈希比对，不匹配则报「数据文件已损坏，请恢复备份」。

---

## 4. 文件格式

### 4.1 版本 2 格式（当前）

```json
{
  "version": 2,
  "iterations": 1000000,
  "salt": "hex-string-16-bytes",
  "verify": { "ct": "hex", "iv": "hex" },
  "data": { "ct": "hex", "iv": "hex" },
  "modified": "2026-06-25T...",
  "backup": true
}
```

| 字段 | 说明 |
|---|---|
| `salt` | PBKDF2 盐值，16 字节 hex |
| `verify` | 加密后的验证令牌 `__VERIFY_OK__`，用于密码校验 |
| `data` | 加密数据 blob，解密后格式：`{"v": [...], "g": [...], "a": [...], "t": "..."} ::HASH:: <sha256>` |
| `iterations` | PBKDF2 迭代次数，支持升级 |
| `backup` | 仅导出备份时存在，防止被误当作主文件打开 |

**加密 blob 内部结构（v2）：**

```javascript
{
  v: vault,          // 密码条目数组
  g: groups_data,    // 分组定义（含 icon）
  a: audit_log,      // 最近 20 条登录记录
  t: totp_secret     // TOTP 密钥（base32）或 null
}
```

所有敏感数据（分组、审计日志、TOTP 密钥）全部加密存储在 `data` 字段内。

### 4.2 版本 1 向后兼容

v1 文件特征：
- 无 `version` 字段或 `version < 2`
- `data` 字段直接是加密的 vault 数组
- 分组和审计日志存储在文件顶层（未加密）

迁移逻辑（`doLogin()`）：

```javascript
if (vaultData.version >= 2) {
  vault = parsed.v; GROUPS_DATA = parsed.g; auditLog = parsed.a; totpSecret = parsed.t;
} else {
  vault = Array.isArray(parsed) ? parsed : [];
  GROUPS_DATA = vaultData.groups || [];
  auditLog = vaultData.auditLog || [];
  totpSecret = null;
}
```

---

## 5. 安全措施

### 5.1 主密码策略

| 规则 | 说明 |
|---|---|
| 最小长度 | 8 字符 |
| 字母要求 | 必须包含至少一个英文字母（`/[a-zA-Z]/`） |
| 数字要求 | 必须包含至少一个数字（`/[0-9]/`） |
| 验证函数 | `validateMasterPw()`，返回错误字符串或 `null` |

### 5.2 PBKDF2 迭代

- 旧文件：500,000 次
- 新文件/升级后：**1,000,000 次**
- 自动升级：登录时检测 `iterations < CURRENT_ITERATIONS`，立即用新迭代重加密

### 5.3 AES-256-GCM

- 密钥长度：256 位
- 工作模式：GCM（Galois/Counter Mode），提供认证加密
- IV：12 字节，每次随机生成

### 5.4 gzip + SHA-256 完整性校验

- 存储前：`compressAndEncrypt(json + '::HASH::' + SHA-256(json))`
- 读取时：解密 → 分离校验和 → `SHA-256(payload) === storedHash?`
- 不匹配时：拒绝加载，提示恢复备份

### 5.5 TOTP 两步验证（RFC 6238）

| 参数 | 值 |
|---|---|
| 算法 | HMAC-SHA1 |
| 时间步长 | 30 秒 |
| 验证码位数 | 6 位 |
| 密钥长度 | 20 字节随机 → base32 编码 |
| OTP URI | `otpauth://totp/密码管理器:<label>?secret=<base32>&issuer=密码管理器` |

流程：
1. 启用：生成密钥 → 用户输入验证码确认 → 加密保存到文件
2. 登录：密码通过后 → 弹出 TOTP 输入 → `verifyTotp()` 校验
3. 关闭：需输入当前有效验证码确认

### 5.6 HIBP 密码泄露检查

- 算法：k-anonymity 模型（只上传 SHA-1 前 5 位）
- 端点：`https://api.pwnedpasswords.com/range/{prefix}`
- 返回：泄露次数（0 = 安全，>0 = 建议更换，-1 = 网络错误）

### 5.7 登录冷却

- 连续 **3 次** 密码错误 → 锁定 **30 秒**
- 冷却期间所有登录尝试被拒绝，显示剩余等待秒数

### 5.8 自动锁定

- 超时时间：**15 分钟**（`AUTO_LOCK_MS = 15 * 60 * 1000`）
- 触发事件：`mousemove` / `mousedown` / `keydown` / `touchstart` / `scroll`
- 重置节流：连续操作每 500ms 重置一次 deadline
- 倒计时显示：右下角实时显示剩余时间（颜色渐变灰→橙→红）
- 快捷键：`Ctrl+L` 立即锁定

### 5.9 剪贴板安全

- 复制密码时显示顶部警告条（15 秒自动消失）
- 15 秒后自动检查剪贴板内容，若未变更则清除

### 5.10 密码显示安全

- 密码字段 CSS：`user-select: none` 禁止选中复制
- 密码默认隐藏（`••••••••`），点击 👁 显示
- 自动隐藏：显示 **10 秒** 后自动隐藏

### 5.11 审计日志

- 记录最近 **20 条** 成功登录
- 内容：时间戳（ISO 8601）+ 浏览器标识（前 80 字符）
- v2 格式中日志加密存储在 `data` blob 内

---

## 6. 存储系统

### 6.1 File System Access API（首选）

| 操作 | 方法 | API |
|---|---|---|
| 打开 | `openVaultFile()` | `showOpenFilePicker()` |
| 新建 | `newVaultFile()` | `showSaveFilePicker()` |
| 写入 | `saveToFile()` | `fileHandle.createWritable()` |

- 写入锁：`_writeLock` 互斥变量防止并发写入
- 文件类型：`.json`（MIME: `application/json`）

### 6.2 IndexedDB 文件句柄持久化

| 数据库 | `pm_store` |
|---|---|
| 对象仓库 | `handles` |
| 键 | `lastHandle` |
| 用途 | 跨页面刷新恢复文件句柄，实现「继续使用此文件」功能 |

```javascript
async function saveHandleToIDB(handle)  // 打开文件后保存
async function loadHandleFromIDB()       // 启动时恢复
async function clearHandleFromIDB()      // 权限拒绝时清除
```

### 6.3 手动模式（降级）

不支持 File System Access API 的浏览器：
- 打开：创建 `<input type="file">` 读取
- 新建：生成文件供下载
- 保存：创建 Blob + `<a download>` 触发下载

关闭页面时检测 `fileMode === 'manual' && isDirty` 触发 `beforeunload` 警告。

---

## 7. 数据优化

### 7.1 防抖保存

```javascript
function scheduleSave() {
  clearTimeout(_saveTimer);
  isDirty = true;
  _saveTimer = setTimeout(persist, 2000);  // 停止操作 2 秒后保存
}
```

### 7.2 定期自动保存

```javascript
autoSaveTimer = setInterval(() => { if(isDirty) persist(); }, 30000);
```

进入主界面时启动，锁定/返回时清除。

### 7.3 gzip 压缩

加密前使用 `CompressionStream('gzip')` 压缩 JSON，典型场景可减少 60-80% 存储体积。

### 7.4 虚拟滚动

| 参数 | 值 |
|---|---|
| 每页大小 | 20 条（`PAGE_SIZE`） |
| 加载方式 | 「加载更多」按钮，递增渲染页数 |
| 重置条件 | 搜索条件或分组变化时重置 `renderPage = 0` |

### 7.5 删除撤销

- 删除后 5 秒内可撤销
- Toast 内嵌「撤销」链接 → `undoDelete()` 恢复条目

---

## 8. 工具功能

### 8.1 密码生成器

- 长度范围：**8 ~ 100 位**（自定义滑块）
- 字符类型：大写字母 / 小写字母 / 数字 / 符号（复选框切换）
- 随机源：`crypto.getRandomValues(new Uint32Array(len))`

### 8.2 哈希计算器

| 算法 | 实现方式 |
|---|---|
| SHA-256 | `crypto.subtle.digest('SHA-256', ...)` |
| SHA-512 | `crypto.subtle.digest('SHA-512', ...)` |
| SHA-1 | `crypto.subtle.digest('SHA-1', ...)` |
| MD5 | 纯 JavaScript 实现 |

### 8.3 密码强度指示器

评分维度（满分 7 分）：

| 条件 | 分数 |
|---|---|
| 长度 ≥ 8 | +1 |
| 长度 ≥ 12 | +1 |
| 长度 ≥ 16 | +1 |
| 包含小写字母 | +1 |
| 包含大写字母 | +1 |
| 包含数字 | +1 |
| 包含特殊字符 | +1 |

颜色映射：0-1 灰 / 2 红 / 3-4 黄 / 5-7 绿。

---

## 9. 主题系统

| 特性 | 说明 |
|---|---|
| 默认 | 跟随系统偏好（`prefers-color-scheme: dark`） |
| 切换 | 按钮切换，`localStorage` 持久化（`pm_theme`） |
| 实现 | HTML `data-theme` 属性 + CSS 自定义属性 |

```css
:root { --bg: #f0f2f5; --card: #fff; ... }
html[data-theme="dark"] { --bg: #1a1a2e; --card: #16213e; ... }
```

---

## 10. 安全边缘用例清单

| 场景 | 保护措施 |
|---|---|
| 密码被选中 | CSS `user-select: none` |
| 密码从 DOM 读取 | 明文仅存内存 `vault`，DOM 显示 `••••••••` |
| 剪贴板遗留密码 | 15 秒后自动清除 |
| 暴力破解 | 3 次失败锁 30 秒 |
| 闲置会话 | 15 分钟自动锁定 |
| 数据损坏 | SHA-256 校验和验证 |
| TOTP 关闭 | 必须输入有效验证码确认 |
| 主密码修改 | 需先验证旧密码 |
| 旧文件降级攻击 | iterations 字段检查 + 自动升级 |
| 手动模式数据丢失 | `beforeunload` 保存提醒 |
| 并发写入 | `_writeLock` 互斥锁 |

---

## 附录：关键常量汇总

| 常量 | 值 | 说明 |
|---|---|---|
| `FILE_FORMAT_VERSION` | 2 | 当前文件格式版本 |
| `CURRENT_ITERATIONS` | 1,000,000 | PBKDF2 当前迭代次数 |
| `LEGACY_ITERATIONS` | 500,000 | PBKDF2 旧版迭代次数 |
| `AUTO_LOCK_MS` | 900,000 (15 min) | 自动锁定超时 |
| `PAGE_SIZE` | 20 | 虚拟滚动每页条数 |
| 加密 IV 长度 | 12 字节 | AES-GCM 初始化向量 |
| 盐长度 | 16 字节 | PBKDF2 盐值 |
| 密码隐藏超时 | 10,000 ms | 显示后自动隐藏 |
| 剪贴板清除超时 | 15,000 ms | 复制后自动清除 |
| 防抖保存延迟 | 2,000 ms | 防抖写入等待时间 |
| 自动保存间隔 | 30,000 ms | 兜底自动保存 |
| 审计日志上限 | 20 条 | 最多保留登录记录数 |
| 登录锁定超时 | 30,000 ms | 3 次失败后锁定时间 |
