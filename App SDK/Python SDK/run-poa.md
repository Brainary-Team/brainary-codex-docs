# 使用 Python SDK 运行 PoA

Python SDK 通过 Codex 的线程接口运行 PoA（Program of Agent）程序。SDK 会启动本地 `codex app-server`，把 `.poa` 包提交到开启了 code mode 的线程，并等待程序执行完成。

## 前置条件

- Python 3.10 或更高版本。
- 支持 `thread/codeMode/exec` 的 Brainary Codex 二进制；`codex` 与 `codex-code-mode-host` 必须放在同一目录。
- 已按[快速开始](../../02-quickstart.md)配置模型、认证信息与 `CODEX_HOME/config.toml`。
- 一个已打包的 `.poa` 文件。SDK 当前不接受包目录；打包方式见[打包与分发](../../08-packaging.md)。

## 安装 SDK

从 PyPI 安装 Python SDK：

```bash
python -m pip install brainary
```
当前还无法提供sdk包，后续需要从github仓库的release中下载

## 运行程序

新建 `run_poa.py`：

```python
from pathlib import Path

from openai_codex import Codex, CodexConfig, Poa, PoaPackage

CODEX_BIN = Path("/path/to/codex-poa/codex") # codex 和 codex-code-mode-host 二进制的保存目录，两者必须保存在同一目录下
POA_FILE = Path("/path/to/my-program.poa")
WORKSPACE = Path("/path/to/workspace")

package = PoaPackage.load(POA_FILE)

with Codex(config=CodexConfig(codex_bin=str(CODEX_BIN))) as codex:
    thread = codex.thread_start(
        cwd=str(WORKSPACE),
        config={"features": {"code_mode": {"enabled": True}}},
    )
    poa = Poa(thread, package)
    result = poa.run()

print("thread_id:", thread.id)
print("output:", result)
```

执行：

```bash
python run_poa.py
```

`Poa` 对象把程序包与运行它的线程绑定在一起。`run()` 返回 Codex 的 `custom_tool_call_output` 对象，其中 `output` 是 PoA 程序通过 `text()`、`image()` 或 `audio()` 产生的结果。

只需临时运行一次时，也可以使用线程提供的简写方法：

```python
result = thread.run_poa("/path/to/my-program.poa")
```

## 注意事项

- `.poa` 必须是 zip 格式，且根目录包含 `manifest.toml`；大小不能超过 64 MiB。
- 必须在创建线程时启用 code mode，否则线程无法运行 PoA。
- SDK 自带的上游 Codex 运行时可能尚未实现 PoA 接口，因此应通过 `CodexConfig(codex_bin=...)` 明确指定 Brainary Codex 二进制。
- 启动 PoA 本身不会触发模型采样，但程序调用子 agent 等工具时可能会调用模型。
- `Poa(thread, package).run()` 与 `thread.run_poa(package)` 等价。
