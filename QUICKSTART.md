# AGS Tool 快速使用指南

## 安装

```bash
# 基础依赖
pip install tencentcloud-sdk-python>=3.1.32 pydantic

# E2B 集成（可选）
pip install e2b_code_interpreter
```

## 环境配置

```bash
export TENCENTCLOUD_SECRET_ID="your_secret_id"
export TENCENTCLOUD_SECRET_KEY="your_secret_key"
export TENCENTCLOUD_ROLE_ARN="qcs::cam::uin/YOUR_UIN:roleName/YOUR_ROLE"
export E2B_API_KEY="your_e2b_api_key"
```

## 快速开始

### 1. 创建带 envd 挂载的工具

```python
from ags_tool import AGSRuntime

runtime = AGSRuntime(
    region="ap-guangzhou",
    domain="ap-guangzhou.tencentags.com",
    image_registry_type="personal",
    # envd 挂载配置
    mount_name="envd-storage",
    mount_image="ccr.ccs.tencentyun.com/namespace/envd-tools:latest",
    mount_image_registry_type="personal",
    mount_path="/envd",
    image_subpath="/envd",
    mount_readonly=True
)

tool_id = runtime.create_tool(
    tool_name="my-sandbox",
    image="python:3.11",
    cpu="2",
    memory="4Gi"
)
```

### 2. 使用 E2B 创建沙箱实例

```python
# 创建沙箱
sandbox = runtime.create_e2b_sandbox(
    tool_name="my-sandbox",
    timeout=600
)

# 执行命令
runtime.execute_command_in_sandbox(
    sandbox=sandbox,
    command="python --version",
    user="root"
)

# 执行代码
code = """
import pandas as pd
print(pd.__version__)
"""
runtime.execute_code_in_sandbox(
    sandbox=sandbox,
    code=code,
    language="python"
)

# 清理
sandbox.kill()
```

### 3. 文件操作

```python
# 上传文件
with open("local.txt", "r") as f:
    sandbox.files.write("remote.txt", f, user="root")

# 下载文件
content = sandbox.files.read("remote.txt", user="root")

# 列出文件
for item in sandbox.files.list(".", user="root"):
    print(item.name)
```

## 示例文件

- `swe_bench_ags_tool.ipynb` - 基于 ags_tool 的示例与流程
- `swe_bench_demo.ipynb` - 使用 SDK 的对照示例

## 常见问题

### Q1: 工具创建失败

**A:** 检查：
1. 镜像地址是否正确
2. `image_registry_type` 是否匹配（enterprise/personal）
3. `role_arn` 是否有权限访问镜像仓库

### Q2: 实例启动失败

**A:** 检查：
1. 工具是否处于 ACTIVE 状态
2. 资源配额是否充足
3. 探针配置是否合理

### Q3: E2B 连接失败

**A:** 检查：
1. `E2B_API_KEY` 是否正确
2. region 与 domain 是否匹配（`create_e2b_sandbox()` 会自动设置 E2B_DOMAIN）
3. 工具名称是否存在

## 更多信息

- 完整文档：`README.md`
- 完整文档：`README.md`
