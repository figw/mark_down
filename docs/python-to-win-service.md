要将 PyInstaller 打包的 EXE 程序设置为 Windows 服务并实现开机自启动，可以通过以下两种方法实现：

---

### **方法一：使用 `nssm`（推荐）**
`nssm` (Non-Sucking Service Manager) 是 Windows 下最稳定的服务管理工具，完美支持普通 EXE 转服务。

#### **步骤 1：下载 nssm**
- 官网下载：[https://nssm.cc/download](https://nssm.cc/download)
- 或使用 Chocolatey 安装：
```powershell
choco install nssm
```

#### **步骤 2：安装服务**
```powershell
# 以管理员身份运行 CMD/PowerShell
nssm install "你的服务名"# 例如 MyPythonApp

# 在弹出的 GUI 中设置：
# 1. "Path" → 选择你的 EXE 文件路径（如 D:\app\your_app.exe）
# 2. "Startup directory" → 设置工作目录
# 3. 点击 "Install service"
```

#### **步骤 3：管理服务**
```powershell
# 启动服务
nssm start "你的服务名"

# 设置开机自启（自动延迟启动避免冲突）
sc config "你的服务名" start= delayed-auto

# 其他命令
nssm restart "你的服务名"
nssm remove "你的服务名"
```

---

### **方法二：使用 `pywin32`（代码级集成）**
如果希望直接从 Python 代码实现服务化：

#### **步骤 1：安装依赖**
```powershell
pip install pywin32
```

#### **步骤 2：修改代码**
在原始 Python 脚本中添加服务控制逻辑（示例）：
```python
import win32serviceutil
import win32service
import win32event
import servicemanager
import sys

class MyService(win32serviceutil.ServiceFramework):
_svc_name_ = "MyPythonService"
_svc_display_name_ = "My Python Service"

def __init__(self, args):
super().__init__(args)
self.hWaitStop = win32event.CreateEvent(None, 0, 0, None)

def SvcStop(self):
self.ReportServiceStatus(win32service.SERVICE_STOP_PENDING)
win32event.SetEvent(self.hWaitStop)

def SvcDoRun(self):
# 在这里写你的主程序逻辑
import time
while True:
with open("C:\\log.txt", "a") as f:
f.write("Service running...\n")
time.sleep(5)

if __name__ == '__main__':
if len(sys.argv) == 1:
servicemanager.Initialize()
servicemanager.PrepareToHostSingle(MyService)
servicemanager.StartServiceCtrlDispatcher()
else:
win32serviceutil.HandleCommandLine(MyService)
```

#### **步骤 3：打包并安装服务**
```powershell
# 打包（需保持控制台窗口）
pyinstaller -F --noconsole your_script.py

# 安装服务（管理员权限）
your_script.exe install
your_script.exe start
```

---

### **关键注意事项**
1. **会话隔离问题**
Windows 服务运行在 `Session 0`，无法直接与用户桌面交互。如需 GUI：
- 改用 `任务计划程序`（见方法三）
- 或使用 `winservice` 库的交互式服务支持

2. **路径问题**
服务的工作目录通常是 `C:\Windows\System32`，所有文件路径需用绝对路径：
```python
import os
config_path = os.path.join(os.path.dirname(__file__), "config.ini")
```

3. **日志记录**
服务无法直接打印到控制台，需写入日志文件：
```python
import logging
logging.basicConfig(filename='C:\\service.log', level=logging.INFO)
```

---

### **方法三：任务计划程序（无服务权限时）**
如果无法获得管理员权限，可用计划任务模拟自启动：
1. 按 `Win+R` 输入 `taskschd.msc`
2. 创建任务 → 设置触发器为“登录时”
3. 操作：启动你的 EXE
4. 勾选“使用最高权限运行”

---

### **方法对比**
| 方法| 优点| 缺点|
|------------|----------------------|-----------------------|
| **nssm**| 稳定，支持任意 EXE| 需单独安装工具|
| **pywin32**| 纯 Python 实现| 需修改代码，兼容性较差 |
| **计划任务**| 无需管理员权限| 不是真实服务|

推荐优先使用 **nssm**，这是最可靠的方案。如果遇到权限问题，可尝试计划任务替代。