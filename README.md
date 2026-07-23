# WHWMY - What Happen With My Internet

> Windows 桌面网络检测工具，一次性诊断代理状态、网络连通性与网线接入状态，并提供一键修复入口。

## 功能概览

| 模块 | 检测内容 | 修复操作 |
|------|----------|----------|
| 代理状态 | 读取注册表 `ProxyEnable` / `ProxyServer` | 关闭代理（UAC 提权）、查看详情 |
| 网络连通性 | 百度连通检测（默认）、Google 连通检测（可选） | - |
| 网线接入 | 物理有线网卡 isup 状态 | - |

## 界面预览

```
┌──────────────────────────────────────────┐
│  WHWMY - 网络诊断工具                    │
├──────────────────────────────────────────┤
│  代理状态                                │
│  ● 代理未开启，状态正常                  │
│  [关闭代理]  [查看详情]                  │
├──────────────────────────────────────────┤
│  网络连通性                              │
│  ● 百度连通正常 (200)                    │
│  ● Google 连通正常 (200)                 │
│  [I'm not in CN]                         │
├──────────────────────────────────────────┤
│  网线接入状态                            │
│  ● 网线已接入 (Realtek PCIe GbE)         │
├──────────────────────────────────────────┤
│                              [重新检测]  │
└──────────────────────────────────────────┘
```

- Win10 扁平化风格，卡片式布局
- 状态圆点：灰色=检测中，绿色=正常，红色=异常
- 所有网络请求在后台线程执行，界面不卡顿

## 环境要求

- Windows 10 / 11
- Python 3.8+

## 安装

```bash
cd WHWMY
pip install -r requirements.txt
```

依赖清单：

```
requests>=2.28.0
psutil>=5.9.0
```

## 运行

```bash
python whwmy.py
```

> 无需管理员权限启动。点击「关闭代理」时会自动弹出 UAC 提权窗口。

## 功能详解

### 1. 代理检测

读取 Windows 注册表：

```
HKEY_CURRENT_USER\Software\Microsoft\Windows\CurrentVersion\Internet Settings
```

- `ProxyEnable == 1`：显示红色警告 + 代理地址
- `ProxyEnable == 0`：显示绿色「状态正常」

**关闭代理**：通过 `shell32.ShellExecuteW(None, "runas", ...)` 启动提权子进程，将 `ProxyEnable` 设为 0，并调用 `InternetSetOptionW` 通知系统刷新设置。

**查看详情**：执行 `rundll32.exe shell32.dll,Control_RunDLL inetcpl.cpl,,4` 打开 Internet 属性 - 连接 - 局域网设置。

### 2. 网络连通性

- **百度检测**（自动）：`requests.get("https://www.baidu.com", timeout=5)`，状态码 200 即正常
- **Google 检测**（手动）：点击「I'm not in CN」按钮触发，`requests.get("https://www.google.com", timeout=5)`

### 3. 网线接入状态

使用 `psutil.net_if_stats()` 遍历网卡，过滤虚拟/无线网卡（VMware、VirtualBox、Bluetooth、Wi-Fi、Tunnel 等），优先匹配以太网关键字（Ethernet、Realtek、Intel 等）。兜底策略：有流量数据的非虚拟网卡也视为已接入。

### 4. 多线程

所有检测在 `threading.Thread(daemon=True)` 后台执行，通过 `root.after(0, callback)` 回到主线程更新 UI，避免界面卡死。

## 项目结构

```
WHWMY/
├── whwmy.py           # 主程序（GUI + 检测逻辑）
└── requirements.txt   # Python 依赖
```

## 技术栈

| 层面 | 技术 |
|------|------|
| GUI | tkinter（Win10 扁平化自定义控件） |
| 网络请求 | requests |
| 系统信息 | psutil |
| 注册表操作 | winreg |
| UAC 提权 | ctypes.windll.shell32.ShellExecuteW |
| 设置刷新 | ctypes.windll.wininet.InternetSetOptionW |

## 使用场景

- 突然断网，快速定位是代理、DNS 还是网线问题
- 开了 VPN 忘关，导致部分网站异常
- 网线松动或网卡被禁用，排查物理连接
- 检测当前网络是否能访问国内/国外站点
