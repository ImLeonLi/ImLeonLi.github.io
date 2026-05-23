---
layout: post
title: "torchvision 0.11.0 + cuda 11.1 从源码编译指南 (Windows)"
date: 2026-05-20
categories: [torch, torchvision, cuda, windows]
tags: [torch, torchvision, cuda, windows, 编译指南]
---

# torchvision 0.11.0 从源码编译指南 (Windows)

最近在研究 GitHub 上的一个名为 SymPointV2 的项目。该项目依赖 torch 1.10.0 和 CUDA 11.1，对应的 torchvision 版本为 0.11.0。然而，官方并未提供适用于 Windows 的 CUDA 11.1 预编译版本，因此需要从源码自行编译。但，在 Windows 中进行与 torch 和 cuda 的编译，还是不太容易的，现在我分享一下我的经验。

## 源码下载

```powershell
# 方法 1: GitHub 下载 zip （推荐，只下载对应这个标签的源码，比较小也就 20MB 左右）
Invoke-WebRequest -Uri "https://github.com/pytorch/vision/archive/refs/tags/v0.11.0.zip" -OutFile "vision-0.11.0.zip"
Rename-Item "vision-0.11.0" "vision-0.11.0.bak" -Force  # 备份
Expand-Archive -Path "vision-0.11.0.zip" -DestinationPath "."
Rename-Item "vision-0.11.0" -NewName "vision-0.11.0" -Force

# 方法 2: Git 克隆（需要下载所有历史记录，比较慢）
git clone https://github.com/pytorch/vision.git
cd vision
git checkout v0.11.0
```

> **注意**：解压后的目录名可能是 `vision-0.11.0`，如果是 `vision-0.11.0.zip` 解压出来的可能是 `vision-0.11.0` 文件夹，需要确认实际路径。

## 环境概述

| 组件 | 版本/路径 |
|------|----------|
| Python | 3.8.20 |
| torch | 1.10.0+cu111 |
| CUDA | 11.1 |
| Visual Studio | 2019 Community |
| vcpkg | D:\Programs\vcpkg |
| 目标平台 | Windows x64 |

---

> 下面涉及到比较多的路径，根据自己的实际情况进行调整。

## 一、环境准备

### 1.1 安装 Visual Studio 2019

下载并安装 **Visual Studio 2019 Community**，安装时选择：
- **"使用 C++ 的桌面开发"** (Desktop development with C++)
- **"用于 Windows 的 CMake"** (CMake for Windows)

### 1.2 安装 vcpkg

vcpkg 是 Windows 上的 C++ 包管理器，用于安装 libpng、libjpeg 等依赖库。

```powershell
# 克隆到合适的位置
git clone https://github.com/Microsoft/vcpkg.git D:\Programs\vcpkg

# 初始化
cd D:\Programs\vcpkg
.\bootstrap-vcpkg.bat

# 安装图像处理库
.\vcpkg install libpng:x64-windows libjpeg-turbo:x64-windows
```

> **注意**：vcpkg 目录一旦确定就不应随意移动，否则需要重新 bootstrap。

### 1.3 环境检查清单

| 工具 | 检查命令 |
|------|----------|
| Python | `python --version` |
| torch + CUDA | `python -c "import torch; print(torch.cuda.is_available())"` |
| MSVC | 打开 "x64 Native Tools Command Prompt for VS 2019"，输入 `cl` |
| CMake | `cmake --version` |
| vcpkg | `D:\Programs\vcpkg\vcpkg version` |

---

## 二、编译步骤

### 2.1 打开正确的命令提示符

**必须使用 "x64 Native Tools Command Prompt for VS 2019"**，普通 CMD 或 PowerShell 无法使用 MSVC 编译器。

### 2.2 设置环境变量

```cmd
:: 必须设置的环境变量
set DISTUTILS_USE_SDK=1
set MSSdk=1

:: vcpkg 路径
set VCPKG_ROOT=D:\Programs\vcpkg

:: torchvision 库路径（关键！）
:: setup.py 使用 libpng-config 查找库，vcpkg 不提供此工具，需要手动指定
set TORCHVISION_INCLUDE=D:\Programs\vcpkg\installed\x64-windows\include
set TORCHVISION_LIBRARY=D:\Programs\vcpkg\installed\x64-windows\lib

:: 添加 vcpkg 库目录到 LIB 和 INCLUDE
set LIB=D:\Programs\vcpkg\installed\x64-windows\lib;%LIB%
set INCLUDE=D:\Programs\vcpkg\installed\x64-windows\include;%INCLUDE%
```

> **为什么不使用普通命令提示符？**
> 
> 普通命令提示符没有配置 MSVC 环境变量，编译 C++/CUDA 扩展时会找不到编译器或使用错误的架构（32位 vs 64位）。

### 2.3 清理并编译

```cmd
cd d:\Project\uranus\vision-0.11.0

:: 清理之前的编译产物
rmdir /s /q build
rmdir /s /q dist
rmdir /s /q *.egg-info

:: 编译并安装
D:\Project\uranus\SymPointV2\.venv\Scripts\python.exe setup.py install
```

### 2.4 验证安装

```cmd
:: 添加 vcpkg DLL 到 PATH（运行 Python 程序前需要）
set PATH=D:\Programs\vcpkg\installed\x64-windows\bin;%PATH%

:: 测试
python -c "import torch; import torchvision; print(torchvision.__version__)"
```

---

## 三、遇到的问题及解决方案

### 问题 1: 编译架构错误 (win32 vs win-amd64)

**错误表现**：
```
build/temp.win32-cpython-38        ← 编译的是 32 位！
cpython-3.8.20-windows-x86_64-none  ← Python 是 64 位！
error C2446: '==' : no conversion from 'size_t' to 'int'
```

**原因**：使用了普通命令提示符而非 x64 Native Tools Command Prompt。

**解决方案**：使用 "x64 Native Tools Command Prompt for VS 2019"。

---

### 问题 2: PNG/JPEG 库未检测到

**错误表现**：
```
PNG found: False
JPEG found: False
Building torchvision with NVJPEG image support
```

**原因**：torchvision setup.py 使用 `libpng-config`（Linux/Mac 工具）或查找 `pngfix`，但 vcpkg 不提供这些。

**解决方案**：手动通过环境变量指定库路径：
```cmd
set TORCHVISION_INCLUDE=D:\Programs\vcpkg\installed\x64-windows\include
set TORCHVISION_LIBRARY=D:\Programs\vcpkg\installed\x64-windows\lib
```

---

### 问题 3: image.pyd 加载失败 (WinError 127)

**错误表现**：
```
UserWarning: Failed to load image Python extension: [WinError 127] 找不到指定的程序。
```

**原因**：torchvision image 扩展依赖 libpng16.dll、jpeg62.dll 等，但这些 DLL 不在 PATH 中。

**解决方案**：运行程序前添加 DLL 路径到 PATH：
```cmd
set PATH=D:\Programs\vcpkg\installed\x64-windows\bin;%PATH%
```

或者将 DLL 复制到 Python 环境或 torchvision 目录。

---

### 问题 4: 多版本 torchvision 冲突

**错误表现**：导入 torchvision 时版本号不对（如 0.12.0 而非 0.11.0）。

**原因**：之前安装过其他版本的 torchvision。

**解决方案**：删除旧版本：
```powershell
Remove-Item -Recurse -Force "D:\Project\uranus\SymPointV2\.venv\lib\site-packages\torchvision"
Remove-Item -Recurse -Force "D:\Project\uranus\SymPointV2\.venv\lib\site-packages\torchvision-*.dist-info"
Remove-Item -Recurse -Force "D:\Project\uranus\SymPointV2\.venv\lib\site-packages\torchvision-*.egg"
```

---

### 问题 5: DISTUTILS_USE_SDK 未设置警告

**警告信息**：
```
UserWarning: It seems that the VC environment is activated but DISTUTILS_USE_SDK is not set.
This may lead to multiple activations of the VC env. Please set `DISTUTILS_USE_SDK=1` and try again.
```

**解决方案**：在编译前设置环境变量：
```cmd
set DISTUTILS_USE_SDK=1
set MSSdk=1
```

---

## 四、关键环境变量说明

| 变量名 | 作用 | 设置值 |
|--------|------|--------|
| `DISTUTILS_USE_SDK` | 告知 distutils 使用 SDK 的 Visual Studio 配置 | `1` |
| `MSSdk` | 告知 distutils 使用 Windows SDK | `1` |
| `VCPKG_ROOT` | vcpkg 根目录 | `D:\Programs\vcpkg` |
| `TORCHVISION_INCLUDE` | torchvision 额外的 include 路径 | vcpkg include 目录 |
| `TORCHVISION_LIBRARY` | torchvision 额外的 library 路径 | vcpkg lib 目录 |
| `CUDA_HOME` | CUDA 安装路径 | `D:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.1` |
| `LIB` | 链接器库搜索路径 | 添加 vcpkg lib 目录 |
| `INCLUDE` | 编译器头文件搜索路径 | 添加 vcpkg include 目录 |

---

## 五、永久配置建议

### 5.1 修改虚拟环境激活脚本

编辑 `D:\Project\uranus\SymPointV2\.venv\Scripts\activate.ps1`，在末尾添加：

```powershell
# vcpkg DLL 路径
$env:PATH = "D:\Programs\vcpkg\installed\x64-windows\bin;" + $env:PATH
```

### 5.2 创建编译脚本

在 torchvision 源码目录创建 `build.bat`：

```batch
@echo off
:: 必须使用 x64 Native Tools Command Prompt for VS 2019

set DISTUTILS_USE_SDK=1
set MSSdk=1
set VCPKG_ROOT=D:\Programs\vcpkg
set TORCHVISION_INCLUDE=D:\Programs\vcpkg\installed\x64-windows\include
set TORCHVISION_LIBRARY=D:\Programs\vcpkg\installed\x64-windows\lib
set LIB=D:\Programs\vcpkg\installed\x64-windows\lib;%LIB%
set INCLUDE=D:\Programs\vcpkg\installed\x64-windows\include;%INCLUDE%

rmdir /s /q build
rmdir /s /q dist
rmdir /s /q *.egg-info

D:\Project\uranus\SymPointV2\.venv\Scripts\python.exe setup.py install

echo.
echo 编译完成！如需测试，请运行：
echo   set PATH=D:\Programs\vcpkg\installed\x64-windows\bin;^%%PATH^%%
echo   D:\Project\uranus\SymPointV2\.venv\Scripts\python.exe -c "import torchvision"
pause
```

---

## 六、编译产物位置

编译完成后，torchvision 安装在虚拟环境的 site-packages 中：

```
D:\Project\uranus\SymPointV2\.venv\lib\site-packages\
├── torchvision-0.11.0a0+3864eb4-py3.8-win-amd64.egg\
│   ├── torchvision\
│   │   ├── _C.pyd          # CUDA 扩展 (2.51 MB)
│   │   ├── image.pyd       # 图像处理扩展 (0.12 MB)
│   │   ├── models\
│   │   ├── transforms\
│   │   └── datasets\
```

---

## 七、常见问题排查

| 问题 | 排查命令 |
|------|----------|
| 编译的是 32 位 | 确认使用 x64 Native Tools Command Prompt |
| 库找不到 | 确认 `TORCHVISION_INCLUDE` 和 `TORCHVISION_LIBRARY` 设置正确 |
| DLL 加载失败 | 确认 vcpkg bin 目录在 PATH 中 |
| 导入报错 | 确认只有一份 torchvision，确认版本正确 |

---

## 八、参考链接

- [torchvision 官方编译文档](https://github.com/pytorch/vision)
- [vcpkg 官方文档](https://github.com/Microsoft/vcpkg)
- [Visual Studio Build Tools 下载](https://visualstudio.microsoft.com/downloads/)
