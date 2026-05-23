---
layout: post
title: "Detectron2 0.6 + cuda 11.1 从源码编译指南 (Windows)"
date: 2026-05-20
categories: [torch, detectron2, cuda, windows]
tags: [torch, detectron2, cuda, windows, 编译指南]
---

# Detectron2 Windows 编译打包手册

本打包文档主要针对 Detectron2 官方没有发布 Windows 平台中 Python 3.8、torch 1.10.x、cuda 11.1 版本的预编译包的情况。

如果使用其他的 torch 和 cuda 版本，可能不需要修改源码，可根据实际情况进行。

问题描述：
- 官方没有提供 Windows 版本的预编译包，导致在 Windows 系统中编译时，需要手动编译。
- 手动编译时，需要安装 CUDA 开发工具包，这会增加额外的依赖。

系统环境：

- Windows 10


Python 环境：

- python 3.8.x
- torch 1.10.x
- cuda 11.1

编译环境：

- Visual Studio 2019

---

## 第零步：确认环境

在开始之前，打开 **cmd**，创建并激活你的 Python 虚拟环境，确认 Python 和 PyTorch 可用：



```bat

:: 创建自己的虚拟环境（路径换成你自己的）
cd /d E:\Temp\rebuild\

:: 创建虚拟环境
uv venv --python 3.8

:: 安装 torch 包（网络不好，可下载包后离线安装）
:: 包下载地址：https://download.pytorch.org/whl/cu111/torch-1.10.0%2Bcu111-cp38-cp38-win_amd64.whl
:: uv pip install E:\软件安装包\torch\torch-1.10.0+cu111-cp38-cp38-win_amd64.whl

:: 在线安装 torch 包
uv pip install torch==1.10.2+cu111 -f https://download.pytorch.org/whl/torch_stable.html

:: 激活虚拟环境
E:\Temp\rebuild\.venv\Scripts\activate

:: 确认 Python
python --version

:: 确认 PyTorch（必须显示 cuda=True）
python -c "import torch; print(f'torch=={torch.__version__}, cuda={torch.cuda.is_available()}')"
```

预期输出：
```txt
Python 3.8.x
torch==1.10.0+cu111, cuda=True
```

> 如果 `cuda=False`，说明 PyTorch 不是 CUDA 版本，编译出的 detectron2 将不含 CUDA 扩展。

---

## 第一步：进入源码目录

```bat
cd /d E:\Temp\detectron2\detectron2
```

> 以下所有命令都在此目录下执行。

---

## 第二步：修改源码（必须）

官方 detectron2 源码在 Windows + PyTorch 1.10.0 环境下**无法直接编译**，需要做以下修改。如果你是从本仓库开始操作的，这些修改已经包含在内，可跳过此步。

需要修改文件 `detectron2/layers/csrc/deformable/deform_conv_cuda_kernel.cu`。

**问题**：该文件 `#include <ATen/cuda/Atomic.cuh>` 中的头文件在 PyTorch 1.10.0 中不存在，会导致编译失败：

```
fatal error C1083: 无法打开包括文件: "ATen/cuda/Atomic.cuh": No such file or directory
```

同时，编译时会报 `atomicAdd` 不接受 `c10::Half` 类型的参数：

```
error: no instance of overloaded function "atomicAdd" matches the argument list
        argument types are: (c10::Half *, c10::Half)
```

**修改**：删除 `#include <ATen/cuda/Atomic.cuh>`，替换为手动实现的 `atomicAdd(c10::Half*)` 函数。

找到第 73-76 行（原始内容）：

```cpp
#include <float.h>
#include <math.h>
#include <stdio.h>
#include <ATen/cuda/Atomic.cuh>

using namespace at;
```

替换为：

```cpp
#include <float.h>
#include <math.h>
#include <stdio.h>
// Provide atomicAdd for c10::Half type when scalar_t is c10::Half
// This must be in global namespace so that the unqualified atomicAdd call works
__device__ inline void atomicAdd(c10::Half* address, c10::Half val) {
    // Implementation using CAS via __half_raw
    unsigned short* addr_as_us = reinterpret_cast<unsigned short*>(address);
    unsigned short old = *addr_as_us;
    unsigned short assumed;
    do {
        assumed = old;
        // Convert to float using __half_raw
        __half_raw old_hr{old};
        __half_raw val_hr{val.x};
        float sum = __half2float(old_hr) + __half2float(val_hr);
        __half_raw new_hr = __float2half(sum);
        old = atomicCAS(addr_as_us, assumed, new_hr.x);
    } while (assumed != old);
}

using namespace at;
```

**完整 diff**：

```diff
 #include <float.h>
 #include <math.h>
 #include <stdio.h>
-#include <ATen/cuda/Atomic.cuh>
+// Provide atomicAdd for c10::Half type when scalar_t is c10::Half
+// This must be in global namespace so that the unqualified atomicAdd call works
+__device__ inline void atomicAdd(c10::Half* address, c10::Half val) {
+    // Implementation using CAS via __half_raw
+    unsigned short* addr_as_us = reinterpret_cast<unsigned short*>(address);
+    unsigned short old = *addr_as_us;
+    unsigned short assumed;
+    do {
+        assumed = old;
+        // Convert to float using __half_raw
+        __half_raw old_hr{old};
+        __half_raw val_hr{val.x};
+        float sum = __half2float(old_hr) + __half2float(val_hr);
+        __half_raw new_hr = __float2half(sum);
+        old = atomicCAS(addr_as_us, assumed, new_hr.x);
+    } while (assumed != old);
+}

using namespace at;
```

---

## 第三步：加载 MSVC 编译环境

调用 VS 2019 的 `vcvarsall.bat` 加载 64 位编译工具链（`cl.exe`、`link.exe` 等）：

```bat
call "D:\Program Files (x86)\Microsoft Visual Studio\2019\Community\VC\Auxiliary\Build\vcvarsall.bat" amd64
```

> 如果你的 VS 2019 安装在其他位置，替换为实际路径。可用 `vswhere.exe` 查询：
> ```bat
> "C:\Program Files (x86)\Microsoft Visual Studio\Installer\vswhere.exe" -property installationPath
> ```

验证 cl.exe 已加载：
```bat
where cl
```
应输出类似 `D:\Program Files (x86)\...\bin\Hostx64\x64\cl.exe`。

---

## 第四步：设置环境变量

```bat
set DISTUTILS_USE_SDK=1
set "CUDA_HOME=C:\Program Files\NVIDIA GPU Computing Toolkit\CUDA\v11.1"
```

| 变量 | 作用 |
|------|------|
| `DISTUTILS_USE_SDK=1` | **必须设置**，让 PyTorch 的 CUDAExtension 找到 MSVC 编译器 |
| `CUDA_HOME` | 指向 CUDA Toolkit 安装路径，让编译系统找到 `nvcc`、头文件和库文件 |

> `CUDA_HOME` 路径根据你的 CUDA 版本调整。验证路径存在：`dir "%CUDA_HOME%\bin\nvcc.exe"`

---

## 第五步：清理旧编译产物

```bat
if exist build rd /s /q build
if exist dist rd /s /q dist
if exist *.egg-info rd /s /q *.egg-info
```

> 清理可避免旧产物干扰。首次编译可跳过，但建议每次都执行。

---

## 第六步：编译 CUDA 扩展

```bat
python setup.py build
```

这一步会编译 detectron2 的 C++/CUDA 扩展（`_C.pyd`），耗时较长（5-15 分钟）。

**判断编译是否成功**：检查是否生成了 `.pyd` 文件：

```bat
dir /s /b build\*_C*.pyd
```

应输出类似：
```
E:\Temp\detectron2\detectron2\build\lib.win-amd64-cpython-38\detectron2\_C.cp38-win_amd64.pyd
```

> 如果没有 `.pyd` 文件，说明编译失败，**不要继续下一步**。查看上方错误输出排查问题。

---

## 第七步：打包成 Wheel

```bat
pip wheel . --no-build-isolation --wheel-dir dist
```

| 参数 | 含义 |
|------|------|
| `.` | 当前目录（setup.py 所在目录） |
| `--no-build-isolation` | 使用当前已激活的环境（而不是创建临时环境），确保找到已编译的 `.pyd` |
| `--wheel-dir dist` | 输出 wheel 文件到 `dist` 目录 |

查看打包结果：

```bat
dir dist\*.whl
```

应输出类似：
```
detectron2-0.6-cp38-cp38-win_amd64.whl
```

---

## 第八步：安装 Wheel

```bat
pip install dist\detectron2-0.6-cp38-cp38-win_amd64.whl --force-reinstall --no-deps
```

| 参数 | 含义 |
|------|------|
| `--force-reinstall` | 强制重新安装（覆盖已有版本） |
| `--no-deps` | 不安装依赖（依赖应在目标环境中提前装好） |

> wheel 文件名根据你的版本自动生成，用 `dir dist\*.whl` 查看实际文件名。

---

## 第九步：验证安装

```bat
python -c "import detectron2; print('version:', detectron2.__version__)"
python -c "from detectron2.layers import DeformConv; print('CUDA extension OK')"
```

两条命令均无报错即安装成功。

---

## 附：常见问题

### Q1: `ninja: build stopped: subcommand failed`

ninja 只是构建调度器，真正的问题在上方编译器输出中。常见原因：
- MSVC C4xxx 警告 → 可在 `setup.py` 的 `extra_compile_args` 中添加对应 `/wdXXXX` 抑制
- 真正的编译 error → 根据具体错误修复

### Q2: `error: no instance of overloaded function "atomicAdd"` with `c10::Half`

缺少 `atomicAdd(c10::Half*)` 实现。见"第二步 · 修改 1"。

### Q3: `Microsoft Visual C++ 14.0 is required`

未加载 MSVC 环境。确认执行了第三步 `call vcvarsall.bat amd64` 和第四步 `set DISTUTILS_USE_SDK=1`。

### Q4: `torch.cuda.is_available()` 返回 False

PyTorch 未检测到 GPU。检查：显卡驱动、PyTorch 是否为 CUDA 版本、CUDA 版本是否与驱动兼容。

### Q5: `DLL load failed` 导入时报错

`.pyd` 文件与当前 Python/PyTorch 版本不匹配。确保 wheel 是在相同环境下编译的。

### Q6: `fatal error C1083: 无法打开包括文件: "ATen/cuda/Atomic.cuh"`

PyTorch 1.10.0 中不存在该头文件（更高版本才有）。见"第二步 · 修改 1"。

---

## 附：环境信息参考

| 项目 | 版本 |
|------|------|
| PyTorch | 1.10.0+cu111 |
| CUDA Toolkit | 11.1 |
| MSVC | Visual Studio 2019 Community |
| detectron2 | 0.6 |
| Python | 3.8 |
