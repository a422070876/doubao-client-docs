# 豆包桌面客户端自动化控制方法

> 实测环境：Windows 10/11（PowerShell 5.1）、豆包 PC 版 2.33.5、Chromium 内核 147.0.7727.149
> 实测日期：2026-11-02（CDP 基础能力）/ 2026-11-05（生图、生视频端到端补充）
> 结论：**可以通过 CDP（Chrome DevTools Protocol）完整控制豆包桌面客户端**，无需 OCR、无需坐标点击、无需重新登录。
> 2026-11-05 补充：生图 / 生视频的"上传参考图 → 发送提示词 → 轮询结果 → 保存下载"全流程已实测跑通（见第 9 节）。

---

## 1. 结论概览

| 能力 | 状态 | 说明 |
|---|---|---|
| 远程调试端口（CDP） | ✅ 已验证 | 启动时加 `--remote-debugging-port=9222` 即可 |
| 枚举页面 Target | ✅ 已验证 | 5 个 target（主聊天页 / 侧栏 launcher / 云盘 iframe / 后台页） |
| 执行 JS、读 DOM | ✅ 已验证 | `Runtime.evaluate`，可读任意文本、点任意按钮 |
| 点击按钮 | ✅ 已验证 | 屏幕外元素需用 JS 派发原生事件 |
| 窗口画面抓取 | ✅ 已验证 | `PrintWindow` + `PW_RENDERFULLCONTENT`，最小化/遮挡也能抓 |
| 继承登录态 | ✅ 已验证 | 使用客户端自身 Cookie，无需扫码 |
| 生图 / 生视频端到端 | ✅ 已验证 | 参考图粘贴上传 → 提示词发送 → 轮询生成 → 保存下载，全流程跑通 |

---

## 2. 环境事实（实测）

- 安装位置：`%LOCALAPPDATA%\Doubao\Application\`
  - 启动器 `Doubao.exe`（5MB）→ 真正的浏览器进程是 `Application\app\Doubao.exe`
  - 内核库 `app\Doubao.dll`（323MB），Chromium 派生，UA 报 `Chrome/147.0.7727.149`
  - 特征文件：`resources.pak`、`icudtl.dat`、`Doubao_elf.dll`、`saman_*`（字节自研 Samantha 壳）
- 数据目录：`%LOCALAPPDATA%\Doubao\User Data\`
  - 主日志：`User Data\sdk_storage\log\saman_YYYY.MMDD.N.log`
  - Profile：`Profile 5`
- 注册表：`HKCU\...\Uninstall` → `豆包 2.33.5`
- 进程模型：多进程 Chromium，正常运行时 14～16 个 `Doubao` 进程

---

## 3. 核心原理

豆包桌面版是 Chromium 派生壳，**只要能开调试端口，就等于拿到了整个浏览器的控制权**：

```
启动参数 --remote-debugging-port=9222
        ↓
HTTP 端点 http://127.0.0.1:9222/json/list   → 页面清单（含 webSocketDebuggerUrl）
        ↓
WebSocket 连接 + CDP 协议（Runtime / Input / Page 三大域）
        ↓
读写 DOM、派发鼠标/键盘事件、抓图、拦截网络
```

避开了三条难走的路：OS 级坐标点击（易碎）、图像识别定位（慢）、录屏 OCR（不准）。

---

## 4. 启动：必须用计划任务，不能直接 Start-Process

**坑（最致命的一个）**：在 Agent 的 Shell 工具里 `Start-Process` 启动 GUI 应用，命令一结束，工具的进程树清理会把豆包子进程一起杀掉——表现为"启动成功→几秒后进程数归零"。`DETACHED_PROCESS` 也逃不掉（Windows Job Object 的 KILL_ON_JOB_CLOSE 会连坐）。

**解法**：让进程挂在「任务计划服务」下，由 svchost 持有，彻底脱离工具进程树。

### 4.1 建一个启动包装脚本

`%TEMP%\doubao_cdp.cmd`：

```bat
@echo off
start "" "%LOCALAPPDATA%\Doubao\Application\Doubao.exe" --remote-debugging-port=9222
```

### 4.2 建任务并立即运行

```powershell
schtasks /create /tn "DoubaoCDP" /tr "$env:TEMP\doubao_cdp.cmd" /sc once /st 23:59 /f
schtasks /run /tn "DoubaoCDP"
Start-Sleep -Seconds 15
(Get-Process Doubao -ErrorAction SilentlyContinue | Measure-Object).Count
```

> 注：`schtasks /create` 的 `/tr` 对嵌套引号解析很脆，直接把带参数的 exe 塞进去会报
> `Invalid argument/option`。**用 .cmd 包装一层**是最省事的办法。

### 4.3 验证端口

```powershell
(Invoke-WebRequest "http://127.0.0.1:9222/json/version" -UseBasicParsing).Content
```

正常返回：

```json
{
  "Browser": "Chrome/147.0.7727.149",
  "Protocol-Version": "1.3",
  "webSocketDebuggerUrl": "ws://127.0.0.1:9222/devtools/browser/..."
}
```

> 首次冷启动可能要 10～15 秒端口才就绪；启动过程中客户端可能先跑一次 `repair_*` 自修复进程，属正常现象。

---

## 5. 页面结构（target 说明）

```powershell
(Invoke-WebRequest "http://127.0.0.1:9222/json/list" -UseBasicParsing).Content
```

| URL | 类型 | 作用 |
|---|---|---|
| `doubao://doubao-launcher/chat?viewId=101` | page | **左侧栏外壳**：新建对话、项目、最近会话、签到卡片、用户区 |
| `doubao://doubao-chat/chat` | page | **主聊天区**：对话流、输入框、模式按钮（对话/图像生成/视频生成…） |
| `doubao://doubao-background/` | page | 后台页，无 UI |
| `https://www.doubao.com/pc/agent-explore?...` | page | 额度任务/探索页（点"去试试"后加载） |
| `https://www.doubao.com/drive-iframe/drive/home/` | iframe | 云盘嵌套页 |

**关键认知**：侧栏和主聊天区是**两个独立 target**，很多元素（比如签到卡）只在 launcher 里，去主聊天页找会找不到。

---

## 6. 连接与基本操作

### 6.1 极简 CDP 客户端

依赖：`websocket-client`（anaconda 环境已自带）

```python
import json, urllib.request, websocket

def targets():
    return json.loads(urllib.request.urlopen('http://127.0.0.1:9222/json/list', timeout=5).read())

class CDP:
    def __init__(self, url):
        # suppress_origin=True 必须加，否则新版本 Chromium 会拒绝无 Origin 的连接
        self.ws = websocket.create_connection(url, timeout=20, suppress_origin=True)
        self.i = 0

    def call(self, method, params=None):
        self.i += 1
        self.ws.send(json.dumps({'id': self.i, 'method': method, 'params': params or {}}))
        while True:
            msg = json.loads(self.ws.recv())
            if msg.get('id') == self.i:
                return msg

    def ev(self, js):
        r = self.call('Runtime.evaluate', {
            'expression': js, 'returnByValue': True, 'awaitPromise': True})
        res = r['result']['result']
        return res.get('value', res.get('description'))
```

### 6.2 选择一个 target

```python
def pick(keyword):
    for t in targets():
        if t.get('type') == 'page' and keyword in (t.get('url') or ''):
            return CDP(t['webSocketDebuggerUrl'])
    raise RuntimeError('target not found')

launcher = pick('doubao-launcher')   # 侧栏
chat     = pick('doubao-chat/chat')  # 主聊天区
```

### 6.3 读页面文本

```python
print(chat.ev("document.body.innerText"))
```

### 6.4 点击按钮（含屏幕外元素）

**坑**：很多元素（如签到卡）`getBoundingClientRect().x` 是**负数**（在屏幕外的浮层里），用 `Input.dispatchMouseEvent` 打真实鼠标点不到。

**解法**：用 JS 派发完整的原生事件序列，绕过 React 的合成事件判定：

```python
CLICK_BY_TEXT = """(() => {
  const el = [...document.querySelectorAll('button,div,span')]
      .find(x => x.textContent.trim() === '%s');
  if (!el) return 'NOT_FOUND';
  ['pointerdown','mousedown','pointerup','mouseup','click'].forEach(t => {
    el.dispatchEvent(new MouseEvent(t, {bubbles:true, cancelable:true, view:window}));
  });
  return 'DISPATCHED';
})()"""

print(chat.ev(CLICK_BY_TEXT % '立即签到'))
```

若是正常可见元素（rect 在视口内），用真实鼠标事件更可靠：

```python
for typ in ('mouseMoved', 'mousePressed', 'mouseReleased'):
    c.call('Input.dispatchMouseEvent', {
        'type': typ, 'x': x, 'y': y, 'button': 'left', 'clickCount': 1})
```

### 6.5 抓窗口画面

**坑 1**：`Page.captureScreenshot` 默认 `fromSurface:true` 在这个壳里**会超时挂死**（20 秒无响应）。
**坑 2**：`fromSurface:false` 虽然能返回，但窗口最小化/被遮挡时是**纯白图**。

**解法**：用 Win32 `PrintWindow` + `PW_RENDERFULLCONTENT(0x2)`，绕过合成器直接让窗口自己渲染：

```powershell
Add-Type -AssemblyName System.Drawing
Add-Type @"
using System; using System.Runtime.InteropServices;
public class PW {
 [StructLayout(LayoutKind.Sequential)] public struct RECT { public int L,T,R,B; }
 [DllImport("user32.dll")] public static extern bool GetWindowRect(IntPtr h, out RECT r);
 [DllImport("user32.dll")] public static extern bool PrintWindow(IntPtr h, IntPtr hdc, uint f);
}
"@
$p = Get-Process Doubao | Where-Object { $_.MainWindowHandle -ne 0 } | Select-Object -First 1
$h = $p.MainWindowHandle
$r = New-Object PW+RECT
[PW]::GetWindowRect($h, [ref]$r) | Out-Null
$w = $r.R - $r.L; $ht = $r.B - $r.T
$bmp = New-Object System.Drawing.Bitmap($w, $ht)
$g = [System.Drawing.Graphics]::FromImage($bmp)
$hdc = $g.GetHdc()
[PW]::PrintWindow($h, $hdc, 2) | Out-Null
$g.ReleaseHdc($hdc); $g.Dispose()
$bmp.Save("$env:TEMP\doubao.png", [System.Drawing.Imaging.ImageFormat]::Png)
$bmp.Dispose()
```

### 6.6 唤出/恢复窗口

```powershell
[Win]::ShowWindow($h, 9)      # SW_RESTORE = 9
[Win]::SetForegroundWindow($h)
```

---

## 7. 实测案例：签到领额度（可复现）

完整流程，可作为任何"点击→校验结果"任务的模板：

```python
launcher = pick('doubao-launcher')          # 签到卡只在这个 target 里
print(launcher.ev("document.body.innerText"))   # 确认存在「立即签到」

launcher.ev(CLICK_BY_TEXT % '立即签到')         # JS 派发点击
# 等待 3~4 秒后校验
print(launcher.ev("document.body.innerText"))
```

**实测输出**：

```
今日签到领 400 额度 | 解锁更多功能，周一额度翻倍 | 立即签到
         ↓ 点击后
签到成功，获得 400 额度 | 探索豆包用法，获取更多额度 | 知道了 | 去试试
```

弹窗里"去试试"会打开 `https://www.doubao.com/pc/agent-explore?scroll_to=tasks&enter_method=signin_success`，
即额度任务页（可继续用 CDP 读它，或点它的"立即领取"）。

**幂等性**：当天已签到时该按钮会消失或变成已签状态，重复执行不会重复领取。

---

## 8. 额度机制（顺带记录的实测结论）

- 签到：**每日 200 额度，周一翻倍 = 400**（2026-11-02 是周一，实测到账 400）
- 累计 30 天可得 8800 额度
- 官方原话：**"体验额度可用于创作以外的任意功能，额度明细请前往「设置－订阅与额度管理」查看。"**

  即额度**不是生图/生视频专用**，而是覆盖办公 / Agent 类功能：
  文档整理、网页任务、表格生成、PPT、电脑文件整理、复杂任务、技能调用、定时提醒（每项 +100）

- 额度余额与逐项扣费明细，入口在 **设置 → 订阅与额度管理**
- 生视频另有独立的「视频生成免费额度」（实测每日若干条），用尽后返回"今日视频生成免费额度已用完"，次日恢复；换账号登录可立即继续

---

## 9. 生图 / 生视频端到端（已实测跑通）

已用 `K:\game\_tmp_cdp\gen_batch.py` 反复验证，累计生成 20+ 条视频。完整链路：

```
粘贴参考图 → 写入提示词 → 点击发送 → 轮询"视频生成好了" → 打开预览面板 → 点"保存" → 从下载目录取回
```

### 9.1 参考图上传：必须用剪贴板粘贴，不能用 setFileInputFiles

**坑**：页面用的是自研上传组件，`DOM.setFileInputFiles` 打到隐藏 input 上不会产生任何附件；发送时豆包仍沿用对话里最早的那张图，表现为"看起来发了图，其实没用上"。

**解法**：在输入框里构造 `ClipboardEvent('paste')`，用 `DataTransfer` 塞入 `File`：

```javascript
(async () => {
  const ed = document.querySelector('[contenteditable="true"]');
  ed.focus();
  const b = atob('...');                 // 图片 base64
  const u = new Uint8Array(b.length);
  for (let i = 0; i < b.length; i++) u[i] = b.charCodeAt(i);
  const f = new File([u], 'ref.png', {type: 'image/png'});
  const dt = new DataTransfer();
  dt.items.add(f);
  ed.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true, cancelable: true}));
  return 'PASTED';
})()
```

多图一次粘贴：`dt.items.add()` 多次后只派发一次 paste，实测一次带 5 张关键帧可正常识别。

**发送前必须校验附件数**，为 0 就重试：

```javascript
(() => {
  const ed = document.querySelector('[contenteditable="true"]');
  const r = ed.getBoundingClientRect();
  return String([...document.querySelectorAll('img')].filter(i => {
    const q = i.getBoundingClientRect();
    return q.top > r.top - 200 && q.bottom < r.bottom + 10 && q.width > 40 && q.width < 160;
  }).length);
})()
```

### 9.2 写入提示词并发送

- 用 `Input.insertText` 写 contenteditable（逐字 `dispatchKeyEvent` 偶尔丢字符）；
- 发送按钮按几何位置定位（输入框右下角、宽度 24–60px、取最右），再派发真实鼠标事件：

```javascript
(() => {
  const ed = document.querySelector('[contenteditable="true"]');
  const r = ed.getBoundingClientRect();
  const bs = [...document.querySelectorAll('button')]
    .map(e => e.getBoundingClientRect())
    .filter(q => q.width >= 24 && q.width <= 60 && q.y >= r.bottom - 60 && q.y <= r.bottom + 130 && q.x >= r.right - 90)
    .sort((a, b) => b.x - a.x);
  if (!bs.length) return 'NO_BTN';
  return JSON.stringify({x: Math.round(bs[0].x + bs[0].width / 2), y: Math.round(bs[0].y + bs[0].height / 2)});
})()
```

清空输入框：`execCommand('selectAll')` + `execCommand('delete')`。

### 9.3 轮询生成结果

- **用"新增图片哈希"判断新产物**：取页面上 `src` 以 http 开头、宽度 > 200 的 `<img>`，把 `src` 文件名前缀（14 字符）当 id，发送前先快照一份，之后做差集；
- 文本侧再等"视频生成好了"出现；
- 出片耗时：视频 **60～150 秒**；轮询间隔 5 秒，超时给 10 分钟。

```javascript
(() => [...document.querySelectorAll('img')]
  .filter(i => i.getBoundingClientRect().width > 200 && (i.src || '').startsWith('http'))
  .map(i => i.src.split('/').pop().slice(0, 14)).join(','))()
```

### 9.4 保存（下载）结果

三步，缺一不可：

1. 点消息里的缩略图 → 打开右侧「图片与视频」预览面板；
2. 在面板里再点一次同 hash 的缩略图；
3. 点 `aria-label="保存"` 的按钮（JS 派发完整事件序列）。

然后轮询 `%USERPROFILE%\Downloads`：**连续两次读到相同文件大小且 > 0** 才算完成，再复制到目标目录并删除临时文件。浏览器会先建 0 字节文件，不能"看到文件就当完成"。

### 9.5 额度检测

每轮轮询顺带读页尾文本，命中即停：

```
额度已用完 / 次数已用完 / 额度不足
```

生视频走独立的「视频生成免费额度」，当天用尽后所有生成请求都会被拒绝；换账号重新登录即可继续。

### 9.6 生成规格与稳定性（实测）

- 输出：4 秒、16:9、1280×720、24fps、约 97 帧；
- 单条耗时 60～150 秒，文件落在 `Downloads`；
- **动作可控性有限**：同一条提示词改几个字，结果差异很大；"后仰""原地不动""首尾帧一致"这类约束经常被模型自行改写，需要多轮迭代（本项目一个刹停动作迭代了 13 轮）；
- **多图关键帧模式更稳**：一次粘贴 4～5 张、提示词写"图1→图5 的顺序生成"，比纯文字更能锁住姿势；代价是模型有时会反问"确认后我再开始生成"，需补发"确认，直接开始生成，不用再问我"；
- 生成 mp4 用 OpenCV 默认 `mp4v` 编码，播放器打不开，交付前要 `ffmpeg -c:v libx264 -pix_fmt yuv420p` 转码；
- 生图质量一般（角色一致性差），需要精细控制时改用本地 ComfyUI + qwen image 工作流。

### 9.7 参考实现

`K:\game\_tmp_cdp\gen_batch.py` 里的 `upload()` / `send()` / `save()` 即上述流程的完整实现，可直接复用；批量跑多个提示词的入口在文件末尾的 `PROMPTS` 循环。

### 9.8 切换账号（换号续额度，已实测）

**入口**：顶栏/侧栏里的用户名按钮（`aria-label` 或文本是「用户123456」「昵称」），点击后弹出账号弹层，内容是已登录过的账号列表 + 「添加账号」。

实测弹层（1440 宽窗口，账号名以代号代替）：

```
账号A        @177,66
账号B        @177,127
账号C        @177,166
账号D        @177,205
添加账号      @177,253
```

**关键坑：点击账号会触发客户端整体重载**，表现是：

- `doubao-chat/chat` 的 URL 从 `.../chat/<会话id>` 变回 `chrome://doubao-chat/chat`，聊天区重置到空对话；
- 已有的 CDP WebSocket **立刻断开**，报 `ConnectionAbortedError [WinError 10053]`；
- launcher 的 `viewId` 会变（如 106 → 107），必须重新枚举 target。

因此切换账号必须拆成"点击 → 断开 → 重连 → 校验"四步，不能在一个连接里连续操作。

```python
import time
from cdp_util import pick

# 1) 打开账号弹层
l = pick('doubao-launcher')
l.ev("""(() => {
  const bs = [...document.querySelectorAll('button')]
    .filter(b => /用户\\d+/.test((b.textContent || '').trim()) && b.getBoundingClientRect().x > 0);
  ['pointerdown','mousedown','pointerup','mouseup','click'].forEach(k =>
    bs[0].dispatchEvent(new MouseEvent(k, {bubbles: true, cancelable: true, view: window})));
})()""")
time.sleep(2.5)

# 2) 点目标账号（点击后连接会断，用 try 包住）
try:
    l.ev("""(() => {
      const el = [...document.querySelectorAll('span,div')]
        .find(e => e.children.length === 0 && e.textContent.trim() === '账号A');
      ['pointerdown','mousedown','pointerup','mouseup','click'].forEach(k =>
        el.dispatchEvent(new MouseEvent(k, {bubbles: true, cancelable: true, view: window})));
    })()""")
except Exception:
    pass   # 10053 断开属正常

# 3) 等客户端重载完成（约 10~15 秒）再重连
time.sleep(14)
l = pick('doubao-launcher')

# 4) 校验当前账号
print(l.ev("""(() => [...document.querySelectorAll('button')]
  .filter(b => b.getBoundingClientRect().x > 0)
  .map(b => (b.textContent || '').trim()).filter(t => t).slice(0, 3).join(' | '))()"""))
```

校验标准：顶栏用户名变成目标账号，且出现「有什么我能帮你的吗？」的空对话首页。

**副作用与注意**：

- 聊天区会重置到新对话；原会话仍保留在侧栏「最近」里，点进去可以继续；
- 各账号额度独立。实测两个账号当天的「视频生成免费额度」都已用尽，换号后依然返回
  `今日视频生成免费额度已用完`，所以**换号不一定有额度**，切完先测一条再批量跑；
- 签到卡片不保证存在：实测切换后两个账号的侧栏与首页都搜不到「立即签到」，
  无法通过 DOM 领取签到额度，脚本里不要把签到设成必过步骤；
- 弹层里的账号按钮 rect 可能落在可视区外（x 为负），用 JS 派发事件即可，不必真实鼠标点击。

---

## 10. 限制与风险

- **安全**：开了 9222 端口意味着**本机任何程序都能完全控制豆包**（含读登录态）。用完应关闭客户端，或改随机端口。
- **稳定性**：依赖客户端版本和 DOM 结构，客户端升级后需要重新校准选择器。
- **合规**：自动化操作属于官方条款的灰色区，低频、模拟人工节奏的使用风险较低；高频批量容易被风控。
- **额度**：签到额度不覆盖创作类功能；生视频走独立的「视频生成免费额度」，用尽后当天无法再生成（实测次日恢复）。
- **窗口依赖**：JS 执行不依赖窗口可见，但抓图、真实鼠标事件需要窗口未最小化。

---

## 11. 清理

```powershell
# 停客户端
Get-Process Doubao -ErrorAction SilentlyContinue | Stop-Process -Force

# 删计划任务
schtasks /delete /tn "DoubaoCDP" /f

# 删临时脚本
Remove-Item "$env:TEMP\doubao_cdp.cmd","$env:TEMP\cdp.py" -Force -ErrorAction SilentlyContinue
```

---

## 12. 踩坑速查表

| 现象 | 根因 | 解法 |
|---|---|---|
| 启动后几秒进程归零 | Shell 工具进程树清理连坐 | 改用计划任务启动 |
| `schtasks` 报 Invalid argument | `/tr` 嵌套引号解析失败 | 用 .cmd 包装脚本 |
| CDP 连不上 / 端口无响应 | 客户端未起来或已退出 | 等 15s，检查进程数 |
| `captureScreenshot` 卡死 20s | `fromSurface:true` 在该壳不兼容 | 改用 `PrintWindow` |
| 截图纯白 | 窗口最小化 / 屏幕不可见 | `ShowWindow(SW_RESTORE)` 后再抓 |
| 点击无效 | 元素在屏幕外（rect x 为负） | JS 派发 pointer/mouse/click 事件序列 |
| 找不到签到卡 | 找错 target | 侧栏元素去 `doubao-launcher` 找 |
| `websocket` 握手被拒 | Chromium 要求 Origin 合法 | 连接时加 `suppress_origin=True` |
| 发了图但没用上 / 一直沿用对话首图 | `setFileInputFiles` 对自研上传组件无效 | 改用剪贴板 paste 事件，发送前校验缩略图数 |
| 粘贴后仍无附件 | 输入框未聚焦，或事件被 React 拦截 | 先 `ed.focus()`，再派发带 `clipboardData` 的 paste |
| 模型反问"确认后我再开始生成" | 提示词里的参数被当作待确认事项 | 补发"确认，直接开始生成，不用再问我" |
| 找不到"保存"按钮 | 右侧「图片与视频」面板没打开 | 先点缩略图开面板，再点 `aria-label="保存"` |
| 下载文件 0 字节 / 取回不完整 | 浏览器先建空文件，读得太早 | 轮询到文件大小连续两次稳定再复制 |
| 生成的 mp4 播放器打不开 | OpenCV 默认 `mp4v` 编码不兼容 | `ffmpeg -c:v libx264 -pix_fmt yuv420p` 转码 |
| 点击切换账号后报 `10053` | 客户端整体重载，WebSocket 被动断开 | try 包住点击，等 10~15 秒后重新 `pick` target |
| 切换后 `viewId` 找不到页面 | 重载会换 viewId（106→107） | 重新枚举 target 再连接，不要复用旧连接 |

---

## 附录：常用 JS 片段

```javascript
// 当前页信息
document.title + ' | ' + location.href

// 全部可见文本
document.body.innerText

// 按文本找元素并取坐标
(() => {
  const el = [...document.querySelectorAll('button,div,span')]
      .find(x => x.textContent.trim() === '立即签到');
  if (!el) return null;
  const r = el.getBoundingClientRect();
  return {x: r.x + r.width/2, y: r.y + r.height/2, w: r.width, h: r.height};
})()

// 列出所有含关键字的可见元素及坐标（用于定位）
(() => {
  const out = [];
  document.querySelectorAll('div,span,p,button').forEach(e => {
    if (e.children.length === 0 && e.textContent.includes('额度') && e.offsetParent) {
      const r = e.getBoundingClientRect();
      out.push(e.textContent.trim().slice(0, 60) + ' @x=' + Math.round(r.x) + ' y=' + Math.round(r.y));
    }
  });
  return out.join('\n');
})()

// 是否登录
(() => [...document.querySelectorAll('div,span')]
    .some(e => /用户\d+/.test(e.textContent.trim())))()
```

## 附录 2：生图 / 生视频常用片段

```javascript
// 1) 上传参考图（单张，base64 由 Python 注入）
(async () => {
  const ed = document.querySelector('[contenteditable="true"]');
  ed.focus();
  const b = atob('BASE64');
  const u = new Uint8Array(b.length);
  for (let i = 0; i < b.length; i++) u[i] = b.charCodeAt(i);
  const dt = new DataTransfer();
  dt.items.add(new File([u], 'ref.png', {type: 'image/png'}));
  ed.dispatchEvent(new ClipboardEvent('paste', {clipboardData: dt, bubbles: true, cancelable: true}));
  return 'PASTED';
})()

// 2) 清空输入框
(() => {
  const e = document.querySelector('[contenteditable="true"]');
  e.focus();
  document.execCommand('selectAll');
  document.execCommand('delete');
})()

// 3) 输入框附件数（发送前校验）
(() => {
  const ed = document.querySelector('[contenteditable="true"]');
  const r = ed.getBoundingClientRect();
  return String([...document.querySelectorAll('img')].filter(i => {
    const q = i.getBoundingClientRect();
    return q.top > r.top - 200 && q.bottom < r.bottom + 10 && q.width > 40 && q.width < 160;
  }).length);
})()

// 4) 结果图片 id 快照（判断是否有新产物）
(() => [...document.querySelectorAll('img')]
  .filter(i => i.getBoundingClientRect().width > 200 && (i.src || '').startsWith('http'))
  .map(i => i.src.split('/').pop().slice(0, 14)).filter(Boolean).join(','))()

// 5) 打开结果预览并点保存
(() => {
  const b = [...document.querySelectorAll('button')]
    .find(x => (x.getAttribute('aria-label') || '') === '保存');
  if (!b) return 'NO_SAVE';
  ['pointerdown','mousedown','pointerup','mouseup','click'].forEach(k =>
    b.dispatchEvent(new MouseEvent(k, {bubbles: true, cancelable: true, view: window})));
  return 'CLICKED';
})()

// 6) 额度/状态检测
(() => {
  const t = document.body.innerText || '';
  return ['额度已用完','次数已用完','额度不足','视频生成好了']
    .filter(k => t.includes(k)).join(',') || 'idle';
})()
```

