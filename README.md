# PyInstaller Windows Distribution Guide

Package Python scripts into standalone Windows executable files and organize them into foolproof one-click distribution bundles.

## Use Cases

- Distribute Python tools to colleagues without Python installed
- Create portable Windows applications
- Bundle Playwright/Chromium or other dependencies with the exe
- Offline deployment of any Python tool

## Core Workflow

### 1. PyInstaller Packaging

```bash
# Single-file exe
python -m PyInstaller --onefile --name "MyApp" your_script.py

# Embed resource files (Windows uses ; separator)
python -m PyInstaller --onefile --name "MyApp" --add-data "config.html;." your_script.py

# Bundle Playwright/Chromium
python -m PyInstaller --onefile --name "MyApp" --add-data "chromium-1067;chromium-1067" your_script.py
```

### 2. Resource Path Compatibility

```python
import sys, os

def resource_path(relative_path):
    if hasattr(sys, '_MEIPASS'):
        return os.path.join(sys._MEIPASS, relative_path)
    return os.path.join(os.path.abspath("."), relative_path)
```

### 3. BAT Launcher Script

- Must use CRLF line endings (\r\n)
- Folder names must be pure ASCII (avoid Chinese full-width parentheses)

### 4. Distribution Bundle Structure

```
MyTool1.0/
├── start_tool.bat
├── view_report.bat
├── instructions.txt
└── _internal/          (hidden program files)
    ├── app.exe
    ├── chromium-1067/
    └── ...
```

## Common Pitfalls

| Symptom | Root Cause | Fix |
|---------|-----------|-----|
| BAT double-click silent fail | LF line endings | Use newline='\r\n' |
| BAT crash | Chinese full-width brackets | Use _internal |
| exe cannot find Chromium | Browser not bundled | Use --add-data |
| Chinese path read fail | Library compatibility | Copy to tempfile |

## Quality Checklist

- [ ] Test on a computer without Python installed
- [ ] BAT file double-click starts correctly
- [ ] Resource files readable
- [ ] Browser dependencies launch correctly
- [ ] Non-technical users can find the entry point
