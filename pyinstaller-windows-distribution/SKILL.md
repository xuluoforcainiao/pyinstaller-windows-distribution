---
name: pyinstaller-windows-distribution
description: Package Python scripts into standalone Windows executable files and organize them into foolproof one-click distribution bundles. Use when the user needs to distribute a Python tool to colleagues who don't have Python installed, when creating portable Windows apps from Python scripts, when bundling Playwright/Chromium or other dependencies with an exe, when doing "功能本地化部署" or "线下部署" of any Python tool, or when the user mentions "打包成exe"、"分发"、"一键启动"、"同事没装Python"、"做成可执行文件"、"本地化部署"、"线下部署"、"Portable app"、"Windows deployment".
---

# 功能本地化（线下）部署 —— Python 脚本打包为 Windows 可执行分发包

将 Python 脚本打包为同事双击即可运行的 Windows exe，并整理成清晰的分发包结构。

## 核心流程

### 1. 用 PyInstaller 打包 exe

```bash
# 基础打包（单文件）
python -m PyInstaller --onefile --name "程序名称" your_script.py

# 打包并嵌入资源文件（Windows 用 ; 分隔）
python -m PyInstaller --onefile --name "程序名称" --add-data "config.html;." your_script.py

# 打包 Playwright/Chromium 等浏览器
python -m PyInstaller --onefile --name "程序名称" --add-data "chromium-1067;chromium-1067" your_script.py
```

**关键参数：**
- `--onefile`：打包成单个 exe 文件
- `--name`：exe 文件名
- `--add-data "源路径;目标路径"`：将额外文件/文件夹嵌入 exe（**Windows 用 `;`，Mac/Linux 用 `:`**）
- `--windowed`：无控制台窗口（GUI 程序用）
- `--icon=app.ico`：自定义图标

### 2. 在代码中读取嵌入资源

PyInstaller 运行时会把资源解压到临时目录，用 `sys._MEIPASS` 获取路径：

```python
import sys, os

def resource_path(relative_path):
    """获取资源路径，兼容开发模式和 PyInstaller 模式"""
    if hasattr(sys, '_MEIPASS'):
        return os.path.join(sys._MEIPASS, relative_path)
    return os.path.join(os.path.abspath("."), relative_path)

CONFIG_HTML = resource_path("config.html")
```

### 3. 制作 bat 启动脚本

bat 文件是 Windows 用户最熟悉的入口。必须注意以下坑：

**a) 必须用 CRLF 换行（`\r\n`）**

用 Python 的 Write 工具直接写 bat 会得到 LF 换行，cmd.exe 会静默失败。正确写法：

```python
bat_content = "@echo off\r\nchcp 65001 >nul\r\nset \"PROG_DIR=%~dp0_internal\"\r\n\"%PROG_DIR%\\程序.exe\"\r\npause\r\n"
with open("启动.bat", "w", encoding="utf-8", newline="\r\n") as f:
    f.write(bat_content)
```

**b) 文件夹名必须是纯 ASCII**

中文全角括号 `（）` 会被 cmd.exe 当作语法字符解析，导致 bat 静默失败。

- ❌ `程序文件（勿删）`
- ✅ `_internal`
- ✅ `app_files`

**c) 标准 bat 模板**

```bat
@echo off
chcp 65001 >nul
set "PROG_DIR=%~dp0_internal"
set "PLAYWRIGHT_BROWSERS_PATH=%PROG_DIR%"
"%PROG_DIR%\程序.exe"
echo.
echo 程序已退出。
pause
```

### 4. 整理分发包文件夹结构

目标：用户打开文件夹后，**一眼就知道该点哪个**。

```
我的工具1.0/
├── 启动工具.bat          ← 清晰入口 1
├── 查看报表.bat          ← 清晰入口 2
├── 运行说明.txt          ← 清晰入口 3
└── _internal/            ← 隐藏的程序文件
    ├── 程序.exe
    ├── chromium-1067/    ← 浏览器等依赖
    └── ...
```

**隐藏 _internal 文件夹：**

```bat
attrib +h "_internal"
```

### 5. 常见踩坑清单

| 问题现象 | 根因 | 解决 |
|---------|------|------|
| 双击 bat 无任何反应 | bat 文件是 LF 换行 | 用 `newline='\r\n'` 写入 |
| 双击 bat 闪退/报错 | 文件夹名含中文全角括号 `（）` | 改为纯 ASCII 如 `_internal` |
| exe 找不到 Chromium | Playwright 浏览器未打包 | `--add-data` 嵌入 + `find_chromium()` 多路径查找 |
| exe 运行时找不到 config.html | 资源路径未用 `sys._MEIPASS` | 用 `resource_path()` 包装 |
| exe 复制时提示 "设备忙" | 旧 exe 进程仍在运行 | `taskkill /f /im 程序名.exe` |
| 路径含中文导致读取失败 | 某些库对中文路径支持差 | 把资源文件复制到 `tempfile.gettempdir()` 再读取 |

### 6. 带 Playwright 的完整打包示例

```python
# find_chromium.py — 多路径查找浏览器
def find_chromium():
    import sys, os
    # 1. PyInstaller 临时目录
    if hasattr(sys, '_MEIPASS'):
        bundled = os.path.join(sys._MEIPASS, "chromium-1067", "chrome-win", "chrome.exe")
        if os.path.exists(bundled):
            return bundled
    # 2. exe 同级目录
    app_dir = os.path.dirname(sys.executable if getattr(sys, 'frozen', False) else __file__)
    portable = os.path.join(app_dir, "chromium-1067", "chrome-win", "chrome.exe")
    if os.path.exists(portable):
        return portable
    # 3. 系统安装的 Playwright
    candidates = [
        os.path.expandvars(r"%LOCALAPPDATA%\ms-playwright\chromium-1067\chrome-win\chrome.exe"),
    ]
    for path in candidates:
        if os.path.exists(path):
            return path
    return None

# main.py 中使用
from playwright.sync_api import sync_playwright

with sync_playwright() as p:
    chromium_path = find_chromium()
    browser = p.chromium.launch(
        headless=False,
        executable_path=chromium_path,
        args=["--start-maximized"]
    )
```

打包命令：
```bash
python -m PyInstaller --onefile --name "我的工具" \
  --add-data "config.html;." \
  --add-data "chromium-1067;chromium-1067" \
  main.py
```

## 质量检查清单

打包完成后验证：

- [ ] 在未安装 Python 的电脑上测试运行
- [ ] 验证 bat 文件双击能正常启动
- [ ] 验证资源文件（html、配置文件）能被正确读取
- [ ] 如有浏览器依赖，验证浏览器能正常启动
- [ ] 文件夹结构清晰，非技术人员能一眼找到入口
