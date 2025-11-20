# OkConfigSettings Usage Guide

[← Back to Documentation Home](README.md)

## What is OkConfigSettings?

`OkConfigSettings` is a Pydantic-based configuration system that loads settings from multiple sources in **priority order** (highest to lowest priority):

1. **`env/local.yaml`** - Local development overrides  
2. **`.env` file** - Environment variables from file
3. **Code defaults** - Fallback values in your classes
4. **Environment variables** - System environment  
5. **AWS Parameter Store** - Production configuration

## This Project's Configuration Pattern

### Configuration Class Structure

All configuration classes inherit from `OkConfigSettings`:

```python
# settings.py
from ia_common_libs.ok_config import OkConfigSettings

class KnowledgeBaseSettings(OkConfigSettings):
    knowledge_base_name: list[str]
    knowledge_base_id: list[str]
    minimum_score_for_documents: float = 0.1

class ChatAgentSettings(OkConfigSettings):
    fundation_model: str
    sessions_table: str
    agent_config: AgentConfig = Field(default_factory=AgentConfig)

class ChatBotSettings(OkConfigSettings):
    general: GeneralSettings
    chat: ChatAgentSettings
    kb: KnowledgeBaseSettings
```

### Single Configuration Entry Point

The system uses a cached singleton pattern:

```python
@lru_cache
def get_settings() -> ChatBotSettings:
    return ChatBotSettings()
```

Used throughout the codebase:

```python
# In any service class
self.settings = get_settings()
```

### AWS Parameter Store Integration

AWS parameters follow the naming convention:

```text
/config/{service_name}_{environment}/{parameter_name}

Examples:
/config/mb-api-chatbot-mc_dev/mcp-url
/config/mb-api-chatbot-mc_dev/agent-config
```

These automatically map to Python attributes:

```python
settings.mcp_url        # from /config/.../mcp-url
settings.agent_config   # from /config/.../agent-config
```

### Local Development Override

**Method 1: Create `env/local.yaml`**

```yaml
# env/local.yaml
agent_config:
  MCPServers:
    - "localhost:3000"
fundation_model: "local-model"
```

**Method 2: Use `.env` file**

```bash
# .env file
AGENT_CONFIG='{"MCPServers": ["localhost:3000"]}'
FUNDATION_MODEL=local-model
```

## Using in Other Projects

### Step 1: Install Required Package

```bash
pip install ia-common-libs
```

### Step 2: Create Configuration Structure  

```python
# your_project/config/settings.py
from ia_common_libs.ok_config import OkConfigSettings
from functools import lru_cache

class DatabaseSettings(OkConfigSettings):
    host: str
    port: int = 5432
    username: str
    password: str

class ApiSettings(OkConfigSettings):
    base_url: str
    timeout: int = 30
    api_key: str

class AppSettings(OkConfigSettings):
    database: DatabaseSettings
    api: ApiSettings
    debug: bool = False

@lru_cache
def get_settings() -> AppSettings:
    return AppSettings()
```

### Step 3: Configure AWS Parameter Store

Set parameters following the naming pattern:

```text
/config/{your-service}_{environment}/host
/config/{your-service}_{environment}/port  
/config/{your-service}_{environment}/base_url
/config/{your-service}_{environment}/api_key
```

### Step 4: Use Configuration in Code

```python
# your_service.py
from your_project.config.settings import get_settings

class YourService:
    def __init__(self):
        settings = get_settings()
        self.db_host = settings.database.host
        self.api_url = settings.api.base_url
```

### Step 5: Local Development Setup

Create `env/local.yaml` for local testing:

```yaml
# env/local.yaml
database:
  host: localhost
  username: dev_user
  password: dev_pass
api:
  base_url: http://localhost:8000
  api_key: local-key
debug: true
```

## Configuration Priority System

When multiple sources provide the same setting, this is the priority order:

**Example scenario:**

- **AWS Parameter Store**: `base_url = "https://prod-api.com"`
- **`.env` file**: `BASE_URL=http://localhost:8000`  
- **`env/local.yaml`**: `api: {base_url: "http://dev-api.com"}`

**Result**: `settings.api.base_url = "http://dev-api.com"` ✅

**Why?** Because `env/local.yaml` has the highest priority.

## Key Benefits

- **Environment-agnostic code** - Same code works in dev/staging/production
- **Local development** without AWS dependencies
- **Type safety** through Pydantic validation  
- **Configuration inheritance** for complex nested structures
- **Single source of truth** with clear override hierarchy

## Testing Your Configuration

```python
# test_settings.py
def test_local_config():
    # Create env/local.yaml with test values first
    settings = get_settings()
    assert settings.api.base_url == "http://localhost:8000"
```

## Summary

OkConfigSettings provides **simple, powerful configuration management**:

1. **Inherit from `OkConfigSettings`** for any configuration class
2. **Use `@lru_cache`** for a singleton settings function  
3. **AWS Parameter Store** for production configuration
4. **`env/local.yaml`** for local development overrides
5. **Type-safe access** to all configuration values

This pattern eliminates environment-specific code and enables seamless local development.
