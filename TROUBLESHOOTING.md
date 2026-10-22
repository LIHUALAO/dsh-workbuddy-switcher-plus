# 已知问题与排查指南

> 这份文件记录使用中实际遇到的问题和排查过程。
> 有些是上游 bug，有些是环境差异，有些是"看起来像 bug 其实不是"。

---

## 问题 1：国内版账号无法使用（只有 WorkBuddy AI 一个分组）

### 现象

- 模型选择器里只有 **WorkBuddy AI**（国际版）分组
- 没有 **WorkBuddy**（国内版）分组
- 切换到「国内模型」后，请求一直返回 `400 请求被拦截，请重新发送`
- `list_subagent_models` 只显示 `workbuddy-ai` 一条路由

### 排查过程

**第一步：确认账号是好的**

```powershell
# 国内凭据文件
$env:LOCALAPPDATA\CodeBuddyExtension\Data\Public\auth\workbuddy-desktop.info
```

实测结果：
- 文件存在，22:57 更新
- `account.uid = 2ddcffea-6850-4d8b-a631-280cf65a2c99`
- `auth.domain = www.workbuddy.cn`
- `accessToken` 是 `$wbEncrypted` 加密格式（**国内版改版后变成加密的了**）

**第二步：验证能否解密**

用 `WorkBuddy.exe`（国内版）的 `electron_browser_workbuddy_storage` 绑定取 key：

```js
process._linkedBinding('electron_browser_workbuddy_storage').loggerGet()
```

结果：`accessToken` 和 `refreshToken` **都能成功解密**，keyId = `9127dea1b44020a7`。

**第三步：实测 API**

| 账号 | 服务器 | 模型 | 结果 |
|---|---|---|---|
| 国内 158\*\*\*\*0849 | codebuddy.cn | `deepseek-v4.1-flash` | ✅ 200 |
| 国内 158\*\*\*\*0849 | codebuddy.cn | `deepseek-v4.1-flash-sg` | ❌ 400 / 11102 模型不存在 |
| 国内 158\*\*\*\*0849 | workbuddy.ai | `deepseek-v4.1-flash-sg` | ❌ 401 |
| 国际 guge2633 | workbuddy.ai | `deepseek-v4.1-flash-sg` | ✅ 200 |
| 国际 guge2633 | workbuddy.ai | `deepseek-v4.1-flash` | ❌ 429 限流 |

**结论：国内账号完全正常，问题在于 provider 没注册。**

### 根因

`dsh-workbuddy-connect` 只在检测到对应版本的客户端已登录时，才注册该版本的 provider。

当前环境：

```
WORKBUDDY_AI_AUTH_FILE = ...2403697e.json   ← 只有国际版
WORKBUDDY_AUTH_FILE    = （未设置）          ← 国内版从未设置
```

这两个环境变量由 `dsh-workbuddy-switcher` 在切换账号时设置。因为一直用国际账号，`WORKBUDDY_AUTH_FILE` 从未被写过。

而 connect 的国内 provider 需要读到 `WORKBUDDY_AUTH_FILE`（或默认路径 `workbuddy-desktop.info`）才注册。

**重启 DSH 后仍然只有国际版** —— 说明 connect 启动时对国内凭据的检测失败了（可能是加密格式导致判定为"未登录"）。

### 临时绕过

**方法 A：手动设置环境变量**

启动 DSH 前设置：

```powershell
$env:WORKBUDDY_AUTH_FILE = "C:\Users\Administrator\.dsh\workbuddy-accounts\2ddcffea-6850-4d8b-a631-280cf65a2c99.json"
```

然后启动 DSH。这会直接用账号库存档（已解密的 desktop 格式），绕过 connect 对加密文件的检测。

**方法 B：确认客户端真的登录了**

1. 打开 `WorkBuddy.exe`（国内版）
2. 确认界面上是登录状态，不是「登录」按钮
3. **完全退出 DSH**（不是关闭窗口）
4. 重新打开 DSH

### 待验证

- [ ] 手动设置 `WORKBUDDY_AUTH_FILE` 后能否注册成功
- [ ] 是 connect 0.7.9 的 bug 还是环境问题
- [ ] 是否需要向 corrinehu/dsh-workbuddy-connect 提 issue

---

## 问题 2：400「请求被拦截，请重新发送」

### 两种不同的 400

| 错误码 | 含义 | 触发条件 |
|---|---|---|
| `11128` | `first message is not system prompt` | 请求的首条消息不是 system |
| `11102` | `model [xxx] service info not found` | 该服务器上没有这个模型 |

### 11128 的原因

WorkBuddy 要求**每次请求的第一条消息必须是 system prompt**。

实测：

```
国际账号 + workbuddy.ai + 无 system prompt -> 400 / 11128
国际账号 + workbuddy.ai + 有 system prompt -> 200
```

### 11102 的原因

**跨版本用错模型。** 国内版服务器没有 `-sg` 后缀的模型：

```
国内账号 + codebuddy.cn + deepseek-v4.1-flash-sg -> 400 / 11102
国内账号 + codebuddy.cn + deepseek-v4.1-flash    -> 200
```

### 如何区分

看错误消息里的 `code` 字段：

- `11128` → 检查首条消息
- `11102` → 检查模型名和服务器是否匹配

---

## 问题 3：国际版账号 429 限流

### 现象

```
429 / code 6004
usage exceeds frequency limit, but don't worry, your usage will reset at ...
```

### 关键：这是**单模型**限流，不是账号被封

| 模型 | 状态 |
|---|---|
| `deepseek-v4.1-flash` | ❌ 429 |
| `deepseek-v4.1-flash-sg` | ✅ 200 |
| `glm-5.3-flash` | ✅ 200 |
| `hy3` | ✅ 200 |

**同一个账号，有的模型限流、有的正常。**

所以看到 429 不要以为账号废了，换个模型就好。

### 与「账号被封」的区别

| 错误 | 含义 | 恢复 |
|---|---|---|
| `429 / 6004` | 单模型频率限制 | 等到 reset 时间，或换模型 |
| `403 / 11140` | **账号被限制** | 需要申诉，短期无法恢复 |

本插件的**禁用功能**专门处理后者：检测到 403/11140 后自动禁用该账号并改投其他账号。

---

## 问题 4：两个 WorkBuddy 版本互相覆盖

### 现象

打开国内版，启动的却是国际版。

### 三层原因

**第一层：装在同一个目录**

```
D:\Programs\WorkBuddy\
├── WorkBuddy.exe      5.8.3  国内版
├── WorkBuddyAI.exe    5.8.4  国际版
└── resources\app.asar         ← 只有一个
```

**第二层：注册表卸载项指向同一目录**

国际版更新器读自己的卸载记录，发现「我在这里」，每次更新都装这里，覆盖国内版。

**第三层（最隐蔽）：更新缓存错位**

```
%TEMP%\workbuddy-update-x64\ 里放的是国际版的安装包
```

国内版启动时读到它，以为有新版本，退出去装 —— 装完又是国际版。**死循环。**

### 解法

两个版本必须装在不同目录，且注册表卸载项分别指向各自目录：

```
WorkBuddy 5.8.3     → %LOCALAPPDATA%\Programs\WorkBuddy\WorkBuddy.exe
WorkBuddy AI 5.8.4  → %LOCALAPPDATA%\Programs\WorkBuddyAI\WorkBuddyAI.exe
```

---

## 问题 5：启动后 0.17 秒退出，退出码 0

### 现象

双击图标，进程一闪就没了。不写日志，退出码 0。

### 排查过的（全排除）

- 单实例锁 / 命名互斥体
- IPC 管道
- 权限 / 提权要求
- 内存不足
- Image File Execution Options 劫持
- AppInit_DLLs 注入

### 真实原因

**它在自动更新。** 更新器接管了启动流程，装完才起主程序。

等更新完成就能打开。**这不是故障。**

---

## 问题 6：`file:` 安装不会自动同步

### 现象

改了源码，重启 DSH，行为没变。

### 原因

pnpm 对同一个 `file:` 路径重复 `add` 只会说 `Already up to date`，**不会重新复制**。

### 解法

改完源码后手动同步：

```powershell
Copy-Item "源码\lib\*" "$env:USERPROFILE\.dsh\profiles\desktop\node_modules\dsh-workbuddy-switcher-plus\lib\" -Force
```

或者 `remove` + `add` 重装。

---

## 问题 7：改代码后白屏（面板空白）

### 现象

插件列表里能看到名字，点开什么都没有。

### 原因（这次的）

用脚本删函数时，`lastIndexOf('/**')` 找注释起点，结果**把前一个函数一起切掉了**。

被删的 `checkin` 还挂在 `return` 对象上：

```js
return {
  ...
  checkin,        // ← 指向不存在的变量
  ...
};
```

渲染时抛 `ReferenceError`，整个组件崩溃。

### 为什么难以发现

| 检查项 | 结果 |
|---|---|
| 语法检查 | ✅ 通过 |
| 模块加载 | ✅ 通过 |
| factory 执行 | ✅ 通过 |
| 括号平衡 | ✅ 通过 |
| 文案键完整性 | ✅ 通过 |
| 样式键完整性 | ✅ 通过 |

**全部通过，但代码是错的。**

### 正确的排查方法

**拿官方原版做逐函数结构对比**，而不是逐个组件猜。

```
原版     : load, run, refreshModels, checkin, exportBundle, importBundleFile
损坏版   : load, run, refreshModels,          exportBundle, importBundleFile
                     ↑ 少了 checkin
```

### 教训

删函数要用**精确边界匹配**，删完立刻做「return 键 vs 定义」的交叉校验。

---

## 快速诊断清单

遇到问题时按顺序查：

1. **看错误码**
   - `429 / 6004` → 单模型限流，换模型
   - `403 / 11140` → 账号被封，禁用并换号
   - `400 / 11128` → 首条不是 system prompt
   - `400 / 11102` → 模型和服务器不匹配

2. **确认账号状态**
   ```powershell
   Get-Content "$env:USERPROFILE\.dsh\workbuddy-switcher.json" -Raw
   ```

3. **确认 provider 注册**
   看模型选择器有几个分组（WorkBuddy / WorkBuddy AI）

4. **确认凭据可解密**
   国内版和国际版用的是不同的 keyblob，分别在各自的 exe 里

5. **确认改动已同步**
   `file:` 安装需要手动 `Copy-Item`
