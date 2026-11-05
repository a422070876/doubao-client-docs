# 豆包桌面客户端自动化控制方法

> 实测环境：Windows 10/11（PowerShell 5.1）、豆包 PC 版 2.33.5、Chromium 内核 147.0.7727.149
> 实测日期：2026-11-02
> 结论：**可以通过 CDP（Chrome DevTools Protocol）完整控制豆包桌面客户端**，无需 OCR、无需坐标点击、无需重新登录。

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
| 生图 / 生视频端到端 | ⚠️ 未验证 | 链路预期可行，但要处理异步长任务与结果下载 |

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
start "" "C:\Users\Administrator\AppData\Local\Doubao\Application\Doubao.exe" --remote-debugging-port=9222
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

---

## 9. 未验证部分：生图 / 生视频的端到端自动化

界面上已确认输入框支持切换 **图像生成 / 视频生成** 模式（另有 PPT 生成、帮我写作、录音转写）。

预期实现路径：

1. 在 `doubao-chat/chat` 页点击模式按钮切到「图像生成」/「视频生成」
2. 定位输入框（contenteditable 或 textarea），用 `Input.insertText` / 派发 `input` 事件填提示词
3. 派发 Enter 或点击发送按钮
4. **轮询 DOM**：等待出现图片 `<img>` / 视频 `<video>` 节点，且流式状态结束
5. 从 `<img src>` 或消息数据里取出 URL，用 HTTP 直接下载（CDN 链接通常带签名，无需额外鉴权）

注意事项：

- 生视频耗时可达数十秒到数分钟，轮询超时要比图片宽松
- 结果节点可能是懒加载，需要滚动到可视区才渲染
- 页面改版会使选择器失效，建议用"文本 + role 属性"定位而非 class（class 是 Tailwind 生成的，极不稳定）

---

## 10. 限制与风险

- **安全**：开了 9222 端口意味着**本机任何程序都能完全控制豆包**（含读登录态）。用完应关闭客户端，或改随机端口。
- **稳定性**：依赖客户端版本和 DOM 结构，客户端升级后需要重新校准选择器。
- **合规**：自动化操作属于官方条款的灰色区，低频、模拟人工节奏的使用风险较低；高频批量容易被风控。
- **额度**：签到额度不覆盖创作类功能，生图/视频的消耗需另看会员权益。
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
