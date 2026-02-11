# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

AGS Tool is a Python abstraction layer for Tencent Cloud AGS (Agent Sandbox Service). It provides a simplified, Pydantic-based interface for managing sandbox tools, instances, and integrates with E2B for sandbox execution.

**Key value proposition**: Reduces code complexity by ~66% compared to direct SDK usage through semantic methods and dictionary-based configuration.

## Architecture

### Core Components

**src/ags_tool/ags_tool.py** (~826 lines)
- `AGSConfig`: Pydantic model for configuration with environment variable support and auto domain/region matching
- `AGSRuntime`: Main abstraction class providing all AGS operations
  - Tool lifecycle: `create_tool()`, `list_tools()`, `delete_tool()`
  - Instance lifecycle: `start_instance()`, `stop_instance()`, `list_instances()`
  - E2B integration: `create_e2b_sandbox()`, `execute_command_in_sandbox()`, `execute_code_in_sandbox()`, `upload_file_to_sandbox()`
  - Token management: `acquire_token()`, `get_instance_url()`

### Critical Design Patterns

1. **Domain Auto-Configuration** (src/ags_tool/ags_tool.py:92-95)
   - `validate_credentials()` model validator auto-sets domain from region to prevent 401 authentication errors
   - Pattern: `domain = f"{region}.tencentags.com"`

2. **Runtime Storage Mount Override** (src/ags_tool/ags_tool.py:260-296)
   - `create_tool()` accepts `storage_mounts` parameter to override config-based mounts at runtime
   - Enables flexible envd mounting with custom SubPath (e.g., `/usr/bin/envd`)

3. **Code-Interpreter Type Checking** (src/ags_tool/ags_tool.py:654-733)
   - `execute_code_in_sandbox()` checks `hasattr(sandbox, 'run_code')` before execution
   - Only code-interpreter-v1 sandboxes support direct code execution
   - Other sandbox types must use file upload + command execution pattern

4. **E2B Domain Force Override** (src/ags_tool/ags_tool.py:590-592)
   - Always force-sets `os.environ["E2B_DOMAIN"]` to ensure region/domain consistency
   - Critical for preventing cached environment variable issues

## Common Development Tasks

### Running Jupyter Notebooks

```bash
# Install package with E2B support
cd ags-tool
pip install -e ".[e2b]"
pip install jupyter

# Set credentials
export TENCENTCLOUD_SECRET_ID="your_id"
export TENCENTCLOUD_SECRET_KEY="your_key"
export E2B_API_KEY="your_e2b_key"

# Start Jupyter
jupyter notebook

# Open example/swe_bench_ags_tool.ipynb or example/swe_bench_demo.ipynb
```

### Importing ags_tool

```python
# After pip install -e .
from ags_tool import AGSRuntime

# Initialize with auto domain configuration
runtime = AGSRuntime(
    secret_id=os.getenv("TENCENTCLOUD_SECRET_ID"),
    secret_key=os.getenv("TENCENTCLOUD_SECRET_KEY"),
    region="ap-guangzhou"  # domain auto-set to ap-guangzhou.tencentags.com
)
```

### Creating Tools with Custom Storage Mounts

```python
# Runtime override of storage mount configuration
STORAGE_MOUNT = {
    "name": "envd-storage",
    "mount_path": "/mnt/envd",
    "readonly": True,
    "image": "ccr.ccs.tencentyun.com/archerlliu/envd:20260115_201017",
    "image_registry_type": "personal",
    "subpath": "/usr/bin/envd"  # Custom SubPath
}

tool_id = runtime.create_tool(
    tool_name="my-tool",
    image="base:latest",
    storage_mounts=[STORAGE_MOUNT]  # Overrides AGSConfig mount settings
)
```

### Code Execution Patterns

```python
# For code-interpreter-v1 sandboxes (direct execution supported)
sandbox = runtime.create_e2b_sandbox(tool_name="code-interpreter-v1", timeout=600)
runtime.execute_code_in_sandbox(sandbox, "print('Hello')")

# For custom sandboxes (SWE-Bench, etc.) - use file upload pattern
import tempfile
import os

with tempfile.NamedTemporaryFile(mode='w', delete=False, suffix='.py') as f:
    f.write("print('Hello')")
    temp_path = f.name

runtime.upload_file_to_sandbox(sandbox, temp_path, "script.py")
runtime.execute_command_in_sandbox(sandbox, "python script.py")
os.unlink(temp_path)
```

## Key Technical Constraints

### Sandbox Type Restrictions

- **code-interpreter-v1**: Supports `sandbox.run_code()` method for direct code execution
- **Custom sandboxes**: Do NOT support `run_code()` - must use file upload + command execution
- `execute_code_in_sandbox()` will raise `AttributeError` with helpful message if used on unsupported sandbox

### Region/Domain Consistency

- Region and domain MUST match for E2B authentication to work
- AGSConfig automatically sets domain from region if not explicitly provided
- Common regions:
  - `ap-guangzhou` → `ap-guangzhou.tencentags.com`
  - `ap-chongqing` → `ap-chongqing.tencentags.com`
  - `ap-shanghai` → `ap-shanghai.tencentags.com`

### Module Caching in Jupyter

If src/ags_tool/ags_tool.py is modified, Jupyter notebooks require kernel restart to load changes:
- Click "Kernel" → "Restart Kernel"
- Or use: `%load_ext autoreload` + `%autoreload 2`

## Documentation Files

- **README.md**: Complete API reference and usage examples
- **example/swe_bench_ags_tool.ipynb**: Example notebook using ags_tool abstraction
- **example/swe_bench_demo.ipynb**: Original example using SDK directly

## Common Error Patterns

### 401 Authentication Error

**Symptom**: `Response 401` when creating E2B sandbox

**Causes**:
1. Region/domain mismatch - Fixed by auto domain configuration in v1.1.1
2. Incorrect E2B_API_KEY - Set in notebook Cell 4 or environment variable
3. Tool not yet ACTIVE - Wait for tool activation before creating sandbox

**Debug**:
```python
print(f"Region: {runtime._config.region}")
print(f"Domain: {runtime._config.domain}")
print(f"E2B_API_KEY: {os.getenv('E2B_API_KEY')[:10]}...")
```

### AttributeError: sandbox has no attribute 'run_code'

**Symptom**: Error when calling `execute_code_in_sandbox()` on custom sandbox

**Cause**: Attempting direct code execution on non-code-interpreter sandbox

**Fix**: Use file upload + command execution pattern (see Code Execution Patterns above)

### Image Pull Failed

**Symptom**: Tool creation fails with image pull error

**Causes**:
1. Incorrect `image_registry_type` (should be "personal" for CCR images)
2. Missing or incorrect `role_arn` for image access
3. Typo in image reference

## Version History

- **v1.4** (2026-02-10): Migrated to src layout (`src/ags_tool/`) to prevent Python namespace package shadowing when the clone directory is named `ags_tool`
- **v1.3** (2026-02-10): Converted to installable Python package (`pip install -e .`), merged QUICKSTART.md into README.md
- **v1.2** (2026-02-05): Code-interpreter type checking + file rename to ags_tool.py
- **v1.1.1** (2026-02-05): E2B domain auto-configuration fix
- **v1.1**: E2B integration + storage_mounts parameter + upload_file_to_sandbox()
- **v1.0**: Initial release with basic AGS operations

## Conventions

### File Naming
- Python modules: Use underscores (e.g., `ags_tool.py`, not `ags-tool.py`)
- Notebook generation: Scripts named `generate_*_notebook.py`

### Configuration
- Credentials: Always prefer environment variables over hardcoded values
- Region: Default to `ap-guangzhou` unless specified
- Probe settings: Use reasonable defaults (60s ready timeout, 2s period)

### Error Handling
- SDK exceptions wrapped with context-rich error messages
- Type checking at method entry with helpful AttributeError messages
- Emoji prefixes in console output (🚀 creating, ✅ success, ❌ error, 📋 info)
