# WMPFDebugger-kiss_-
就是根据WMPFDebugger适配新版本


# WMPFDebugger-kiss 适配微信 25715 版本教程

> 目标：让 GUI 版 WMPFDebugger-kiss 能正常 Hook 微信 PC 4.x（WMPF 版本 25715）的小程序，调出 F12 调试。

---

## 一、问题现象

启动调试器后，一直报：

```
[frida] error in find wmpf version
```

或者 hook 抓不到、调试窗口不弹。

### 根因

你这个 fork（WMPFDebugger-kiss）是老版本，存在两个问题：

1. **hook.js 太旧**：只支持 `SceneOffsets` 老格式，25715 已经换成 `CastToJsonHookOffset` + `MiniAppConfigStructOffsets` 新结构。
2. **版本检测逻辑错误**：代码用 `(Get-Process).Path` 取 exe 路径来猜版本号，但你微信的 exe 路径 `...\RadiumWMPF\WeChatAppEx.exe` 里**根本没有 25715 这个数字**，所以永远检测失败。
   - 真正的版本号 `25715` 只在命令行参数 `--flue-runtime-dir="...\RadiumWMPF\25715\extracted\runtime"` 里。

---

## 二、需要改的文件（共 3 处）

项目根目录：`G:\Z-Y\PY\项目\python_AI\test\WMPFDebugger-kiss-main\`

| 序号 | 文件                                | 操作                 |
| ---- | ----------------------------------- | -------------------- |
| 1    | `frida/hook.js`                     | **替换**为官方最新版 |
| 2    | `frida/config/addresses.25715.json` | **新建**             |
| 3    | `dist/index.js` + `src/index.ts`    | **修改版本检测函数** |

---

## 三、具体改法

### 1. 替换 `frida/hook.js`

把官方仓库的最新 hook.js 下载下来，覆盖：

- 下载地址：<https://raw.githubusercontent.com/evi0s/WMPFDebugger/main/frida/hook.js>
- 覆盖目标：`WMPFDebugger-kiss-main\frida\hook.js`

新版和旧版的关键区别：

| 项目      | 旧版（你 fork 自带）      | 新版                                      |
| --------- | ------------------------- | ----------------------------------------- |
| 主模块    | WeChatAppEx.exe           | **flue.dll**（version≥13331）             |
| CDP 过滤  | Interceptor.attach 改内存 | **Interceptor.replace** + CastToJson      |
| 场景偏移  | `SceneOffsets: [a,b,c]`   | **`MiniAppConfigStructOffsets`** 嵌套结构 |
| WebSocket | 不设置                    | 自动写 `ws://localhost:9421`              |

---

### 2. 新建 `frida/config/addresses.25715.json`

> ⚠️ 注意路径：直接放在 `frida/config/` 下，**不要**放到 `frida/config/win32/` 子目录。你这个 kiss 版代码读的是 `frida/config/addresses.<版本号>.json`。

文件内容：

```json
{
    "Version": 25715,
    "LoadStartHookOffset": "0x2D456B0",
    "CDPFilterHookOffset": "0x381C1D0",
    "CastToJsonHookOffset": "0x9EC7870",
    "MiniAppConfigStructOffsets": {
        "LaunchConfigOffsets": [64, 1568, 8],
        "RemoteDebugConfigOffsets": [1504, 16],
        "SceneOffset": 456,
        "WebSocketURLStringOffset": 616,
        "RemoteDebugModeOffset": 724
    }
}
```

三个 Hook 偏移含义：

- `LoadStartHookOffset: 0x2D456B0` → 拦截小程序启动（OnLoadStart），注入调试参数
- `CDPFilterHookOffset: 0x381C1D0` → 绕过 CDP 消息过滤器，放行 DevTools 协议
- `CastToJsonHookOffset: 0x9EC7870` → JSON 转换 Hook（新版替代旧版 SceneOffsets）

---

### 3. 修改版本检测逻辑（dist/index.js + src/index.ts）

这是最关键的一步。两个文件都要改，内容一模一样。

#### 3.1 找到要删的旧代码

在 `src/index.ts` 里找到（大约 151~183 行）：

```typescript
const getWeChatAppExPath = (pid: number): string | null => {
    try {
        const output = execSync(
            `powershell -NoProfile -Command "(Get-Process -Id ${pid}).Path"`,
            { encoding: "utf8", timeout: 10000 }
        );
        const trimmed = output.trim();
        return trimmed.length > 0 ? trimmed : null;
    } catch (e) {
        return null;
    }
};

const frida_server = async () => {
    const localDevice = await frida.getLocalDevice();
    const processes = await localDevice.enumerateProcesses();
    const wmpfProcesses = processes.filter(process => process.name === "WeChatAppEx.exe");

    if (wmpfProcesses.length === 0) {
        throw new Error("[frida] WeChatAppEx.exe process not found");
    }

    const wmpfProcess = wmpfProcesses[0];
    const wmpfPid = wmpfProcess.pid;
    const wmpfProcessPath = getWeChatAppExPath(wmpfPid);
    const wmpfVersionMatch = wmpfProcessPath ? wmpfProcessPath.match(/\d+/g) : "";
    const wmpfVersion = wmpfVersionMatch ? new Number(wmpfVersionMatch.pop()) : 0;
    if (wmpfVersion === 0) {
        throw new Error("[frida] error in find wmpf version");
        return;
    }

    const session = await localDevice.attach(Number(wmpfPid));
```

在 `dist/index.js` 里找到对应段（编译后的 JS，逻辑一样）：

```javascript
const getWeChatAppExPath = (pid) => {
    try {
        const output = (0, node_child_process_1.execSync)(`powershell -NoProfile -Command "(Get-Process -Id ${pid}).Path"`, { encoding: "utf8", timeout: 10000 });
        const trimmed = output.trim();
        return trimmed.length > 0 ? trimmed : null;
    }
    catch (e) {
        return null;
    }
};
const frida_server = async () => {
    const localDevice = await frida.getLocalDevice();
    const processes = await localDevice.enumerateProcesses();
    const wmpfProcesses = processes.filter(process => process.name === "WeChatAppEx.exe");
    if (wmpfProcesses.length === 0) {
        throw new Error("[frida] WeChatAppEx.exe process not found");
    }
    const wmpfProcess = wmpfProcesses[0];
    const wmpfPid = wmpfProcess.pid;
    const wmpfProcessPath = getWeChatAppExPath(wmpfPid);
    const wmpfVersionMatch = wmpfProcessPath ? wmpfProcessPath.match(/\d+/g) : "";
    const wmpfVersion = wmpfVersionMatch ? new Number(wmpfVersionMatch.pop()) : 0;
    if (wmpfVersion === 0) {
        throw new Error("[frida] error in find wmpf version");
        return;
    }
    const session = await localDevice.attach(Number(wmpfPid));
```

#### 3.2 用新代码替换

**`src/index.ts` 里替换成：**

```typescript
const getWeChatAppExVersion = (): number => {
    try {
        const output = execSync(
            `powershell -NoProfile -Command "Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'WeChatAppEx.exe' } | ForEach-Object { $_.CommandLine } | Where-Object { $_ -like '*--flue-runtime-dir*' } | Select-Object -First 1"`,
            { encoding: "utf8", timeout: 10000 }
        );
        const cmdline = output.trim();
        const flueMatch = cmdline.match(/--flue-runtime-dir="([^"]+)"/);
        const dir = flueMatch ? flueMatch[1] : cmdline;
        const nums = dir.match(/\d+/g);
        if (nums && nums.length > 0) {
            return Number(nums[nums.length - 1]);
        }
        return 0;
    } catch (e) {
        return 0;
    }
};

const getMainWeChatAppExPid = (): number | null => {
    try {
        const output = execSync(
            `powershell -NoProfile -Command "Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'WeChatAppEx.exe' -and $_.ParentProcessId -ne 0 } | Group-Object ParentProcessId | Sort-Object Count -Descending | Select-Object -First 1 | ForEach-Object { $_.Name }"`,
            { encoding: "utf8", timeout: 10000 }
        );
        const pid = Number(output.trim());
        return pid > 0 ? pid : null;
    } catch (e) {
        return null;
    }
};

const frida_server = async () => {
    const localDevice = await frida.getLocalDevice();
    const processes = await localDevice.enumerateProcesses();
    const wmpfProcesses = processes.filter(process => process.name === "WeChatAppEx.exe");

    if (wmpfProcesses.length === 0) {
        throw new Error("[frida] WeChatAppEx.exe process not found");
    }

    // 版本号从命令行参数 --flue-runtime-dir 提取（新版微信版本号只出现在运行时目录里，exe 路径没有）
    const wmpfVersion = getWeChatAppExVersion();
    if (wmpfVersion === 0) {
        throw new Error("[frida] error in find wmpf version");
        return;
    }

    // 选择主进程（browser process，加载了 flue.dll）：被最多子进程作为父进程的那个
    const mainPid = getMainWeChatAppExPid();
    const wmpfPid = mainPid !== null ? mainPid : wmpfProcesses[0].pid;

    const session = await localDevice.attach(Number(wmpfPid));
    // ... 后面的代码保持不变
```

**`dist/index.js` 里替换成（注意是编译后的 JS 写法）：**

```javascript
const getWeChatAppExVersion = () => {
    try {
        const output = (0, node_child_process_1.execSync)(`powershell -NoProfile -Command "Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'WeChatAppEx.exe' } | ForEach-Object { $_.CommandLine } | Where-Object { $_ -like '*--flue-runtime-dir*' } | Select-Object -First 1"`, { encoding: "utf8", timeout: 10000 });
        const cmdline = output.trim();
        const flueMatch = cmdline.match(/--flue-runtime-dir="([^"]+)"/);
        const dir = flueMatch ? flueMatch[1] : cmdline;
        const nums = dir.match(/\d+/g);
        if (nums && nums.length > 0) {
            return Number(nums[nums.length - 1]);
        }
        return 0;
    }
    catch (e) {
        return 0;
    }
};
const getMainWeChatAppExPid = () => {
    try {
        const output = (0, node_child_process_1.execSync)(`powershell -NoProfile -Command "Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'WeChatAppEx.exe' -and $_.ParentProcessId -ne 0 } | Group-Object ParentProcessId | Sort-Object Count -Descending | Select-Object -First 1 | ForEach-Object { $_.Name }"`, { encoding: "utf8", timeout: 10000 });
        const pid = Number(output.trim());
        return pid > 0 ? pid : null;
    }
    catch (e) {
        return null;
    }
};
const frida_server = async () => {
    const localDevice = await frida.getLocalDevice();
    const processes = await localDevice.enumerateProcesses();
    const wmpfProcesses = processes.filter(process => process.name === "WeChatAppEx.exe");
    if (wmpfProcesses.length === 0) {
        throw new Error("[frida] WeChatAppEx.exe process not found");
    }
    // 版本号从命令行参数 --flue-runtime-dir 提取
    const wmpfVersion = getWeChatAppExVersion();
    if (wmpfVersion === 0) {
        throw new Error("[frida] error in find wmpf version");
        return;
    }
    // 选择主进程（加载了 flue.dll 的 browser process）
    const mainPid = getMainWeChatAppExPid();
    const wmpfPid = mainPid !== null ? mainPid : wmpfProcesses[0].pid;
    const session = await localDevice.attach(Number(wmpfPid));
    // ... 后面保持不变
```

> **两个文件必须都改**：Electron 实际跑的是 `dist/index.js`，只改 `src/index.ts` 不生效。

---

## 四、验证步骤

1. **先开微信，打开任意一个小程序**（让 WeChatAppEx.exe 进程跑起来）

2. 打开 PowerShell，进入项目目录：

   ```powershell
   cd G:\Z-Y\PY\项目\python_AI\test\WMPFDebugger-kiss-main
   node dist/index.js miniapp
   ```

3. 看到以下输出就说明成功：

   ```
   [server] debug server running on ws://localhost:9421
   [server] proxy server running on ws://localhost:62000
   [frida] script loaded, WMPF version: 25715, pid: 26012
   [frida] you can now open any miniapps
   ```

4. 然后关闭当前小程序再重新打开，调试窗口（Chrome DevTools）会自动弹出。

---

## 五、原理说明（为什么这样改）

| 问题           | 旧做法                                      | 新做法                                                   |
| -------------- | ------------------------------------------- | -------------------------------------------------------- |
| 取版本号       | `(Get-Process).Path` → exe 路径（无版本号） | 读 `--flue-runtime-dir` 命令行参数 → 里面有 `\25715\`    |
| 选 attach 进程 | 随便枚举第一个 WeChatAppEx.exe              | 选子进程最多的父进程（主 browser 进程，加载了 flue.dll） |
| 主模块         | WeChatAppEx.exe                             | flue.dll（新版微信小程序运行时换了 DLL）                 |

---

## 六、常见问题

**Q: 还是报 `error in find wmpf version`？**

- 确认微信里至少打开过一个小程序（WeChatAppEx.exe 进程存在）

- 确认 PowerShell 命令能跑通：

  ```powershell
  Get-CimInstance Win32_Process | Where-Object { $_.Name -eq 'WeChatAppEx.exe' } | Select-Object -First 1 CommandLine
  ```

**Q: 报 `version config not found: xxxx`？**

- 把打印出来的版本号，在 `frida/config/` 下建对应的 `addresses.<版本号>.json`。

**Q: 注入成功但调试窗口不弹？**

- 必须**先关小程序再重新打开**（不能刷新），hook 是在小程序启动时触发的。

---

> 仅用于本地学习调试，禁止用于非法抓包或篡改第三方小程序。
