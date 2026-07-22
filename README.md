<p align="center">
  <h1>py2pydso</h1>
  <a href="https://pypi.org/project/py2pydso/"><img src="https://img.shields.io/pypi/v/py2pydso.svg" alt="PyPI version"></a>
  <a href="https://pypi.org/project/py2pydso/"><img src="https://img.shields.io/badge/Python-3.8~3.14-3776AB?logo=python&logoColor=white" alt="Python"></a>
  <a href="https://github.com/zhenzi0322/py2pydso/blob/main/LICENSE"><img src="https://img.shields.io/pypi/l/py2pydso.svg" alt="License"></a>
</p>

> 将`Python`源文件编译为`.pyd/.so`原生扩展，以便分发和保护源代码。

---

## ✨ Features

- 🔒 **源码保护** — 将 `.py` 编译为 `.pyd`/`.so` 原生扩展，不暴露源码
- 📦 **三种编译模式** — 单文件 / 模块目录 / 完整 wheel 包
- 🗂️ **智能过滤** — 自动保留 `__init__.py` 等元文件，支持自定义排除
- 📝 **类型提示** — 自动生成 `.pyi` 存根文件，保留 IDE 补全体验
- 🌍 **跨平台** — `Windows` (`.pyd`) / `Linux` / `macOS` (`.so`) 全支持

---

## 安装

```bash
pip install py2pydso
```

安装完成后，可通过以下命令验证：

```bash
python -m py2pydso --help
```

### 依赖

`py2pydso` 会自动安装以下 Python 依赖：

| 依赖 | 用途 |
|------|------|
| `Cython` | 将 `.py` 转译为`C`代码并编译为原生扩展 |
| `setuptools` | 驱动编译流程 |
| `mypy` | 通过 stubgen 生成 `.pyi` 类型提示文件 |
| `tomli` | 解析 `pyproject.toml`（Python < 3.11 时自动安装） |

### C 编译器

编译原生扩展需要 C 编译器，请根据平台安装：

- **Windows**: [Visual Studio Build Tools](https://visualstudio.microsoft.com/visual-cpp-build-tools/)（勾选 "C++ build tools"）
- **macOS**: `xcode-select --install`
- **Linux**: `sudo apt install build-essential gcc`

## 使用方式

### 1. 编译单个文件

```bash
# 输出到源文件同目录
python -m py2pydso file -i demo.py
```

输出 `demo.pyd`（或 `demo.so`），位于 `demo.py` 同目录。

```bash
# 指定输出目录
python -m py2pydso file -i demo.py -o output
```

输出 `output/demo.pyd`。

```bash
python -m py2pydso file -i utils/demo.py
```

输出 `utils/demo.pyd`（或 `utils/demo.so`）。

### 2. 编译整个模块目录

```bash
python -m py2pydso module -i utils -o output
```

输入：
```
utils/
  __init__.py
  __main__.py
  tools.py
  config.py
```

输出：
```
output/
  utils/
    __init__.py      ← 原样保留（__开头）
    __main__.py      ← 原样保留（__开头）
    config.py        ← 原样保留（--exclude-files 指定）
    tools.pyd        ← 编译产物
```

- `__`开头的 `.py` 文件（如 `__init__.py`、`__main__.py`）原样保留，不参与编译
- 支持 `--exclude-files` 指定额外排除的文件（可多个）：
  ```bash
  python -m py2pydso module -i utils -o output --exclude-files config.py constants.py
  ```
- 支持子目录递归编译

### 3. 构建完整 wheel 包

> 要求`setup.py`中要使用`ext_modules`。示例如下：

```py
import re
from setuptools import setup, Extension, find_packages
from pathlib import Path

this_directory = Path(__file__).parent
long_description = (this_directory / "README.md").read_text(encoding="utf-8")


def _collect_ext_modules() -> list:
    """收集 loki_service 下所有 .py 文件（排除 __init__.py），声明为 Cython 扩展"""
    pkg_dir = this_directory / "loki_service"
    modules = []
    for py_file in sorted(pkg_dir.rglob("*.py")):
        if py_file.name == "__init__.py":
            continue
        rel = py_file.relative_to(this_directory).with_suffix("")
        module_name = str(rel).replace("/", ".").replace("\\", ".")
        modules.append(Extension(module_name, [str(py_file.relative_to(this_directory))]))
    return modules

# 元数据和依赖统一由 pyproject.toml 管理，setup.py 仅负责 Cython ext_modules
setup(
    version="1.0.0",
    long_description=long_description,
    long_description_content_type="text/markdown",
    packages=find_packages(),
    ext_modules=_collect_ext_modules(),
)
```

```bash
python -m py2pydso package --package-name loki_service --keep-tmp
```

构建保护版本的 wheel 包（`.pyi` 类型提示 + `.pyd/.so` 原生扩展），不暴露源码。

```bash
# 跳过 stubgen，异常时保留临时目录用于调试
python -m py2pydso package --package-name loki_service --skip-stubgen --keep-tmp

# 指定输出目录
python -m py2pydso package --package-name loki_service --output-dir /path/to/wheelhouse
```

输出：`<package_name>-x.x.x-cpYY-cpYY-<platform>.whl`

## 公共参数

| 参数 | 说明 |
|------|------|
| `--skip-stubgen` | 要是带上参数`--skip-stubgen`表示不需要生成`pyi`文件. |
| `--keep-tmp` | 保留临时编译目录，用于调试 |

## 命令行帮助

```bash
python -m py2pydso --help
python -m py2pydso file --help
python -m py2pydso module --help
python -m py2pydso package --help
```
