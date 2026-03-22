# Environment Variables Guidelines

This document defines a design based on **server-only environment variables** for Python applications (FastAPI / CLI), ensuring secrets are managed securely. The deployment environment assumes **Docker / cloud deployment**, leveraging platform-specific secret management features.

---

## 1. Management Policy (Core Principles)

## 1. Enforce Server-Only

* `os.environ` and `pydantic-settings` are used **only in server-side code**
* Never expose secrets in API responses
* When a client needs data, **return only the minimum required via API endpoint** — "minimum" means: only public identifiers (e.g., payment Client ID) and non-sensitive configuration. Never return secrets, internal IDs, or data the client does not directly need.

---

## 2. .env Files for Local Only

* `.env`: Local use only (excluded from Git)
* `.env.example`: List only the required key names (no values)
* Never use `.env` files in production — inject from cloud secrets or environment

---

## 3. Application Code Separation

* Consolidate all external service configurations in `app/core/config.py`
* **Validate with pydantic-settings**, outputting clear error messages on missing values

```python
## app/core/config.py - pydantic-settings validation pattern (recommended)
from pydantic_settings import BaseSettings, SettingsConfigDict


class Settings(BaseSettings):
    model_config = SettingsConfigDict(
        env_file=".env",
        env_file_encoding="utf-8",
        case_sensitive=True,
    )

    # {AI_SERVICE}_API_KEY: AI API key used by the project
    AI_API_KEY: str
    OAUTH_CLIENT_ID: str
    DATABASE_URL: str

    # Optional with defaults
    APP_ENV: str = "development"
    LOG_LEVEL: str = "INFO"
    DEBUG: bool = False


## Validate at import time (fail immediately on error)
settings = Settings()
```

---

## 2. Primary Environment Variables

```python
## Database (SQLAlchemy)
DATABASE_URL=
DATABASE_POOL_SIZE=10

## Authentication (OAuth / JWT)
SECRET_KEY=
OAUTH_CLIENT_ID=
OAUTH_CLIENT_SECRET=

## {Payment Service} (e.g., PayPal, Stripe, etc.)
{PAYMENT_SERVICE}_CLIENT_ID=
{PAYMENT_SERVICE}_CLIENT_SECRET=
{PAYMENT_SERVICE}_WEBHOOK_SECRET=

## {AI Service} (e.g., OpenAI, Gemini, etc.)
{AI_SERVICE}_API_KEY=

## Application
APP_ENV=
APP_URL=
LOG_LEVEL=
ALLOWED_ORIGINS=
```

---

## 3. Development Environment (Local)

## Use .env

* DB = SQLite connection string
* Use sandbox keys / OAuth keys for testing
* `.env` must **never be committed to Git**

---

## 4. Production Environment (Docker / Cloud)

## Use Platform Secret Management

* Inject environment variables via Docker secrets, Kubernetes secrets, or cloud provider secret management (AWS SSM, GCP Secret Manager, etc.)
* **Secrets rotation** support recommended
* Never bake secrets into Docker images

### Docker Compose Example

```yaml
## docker-compose.yml
services:
  app:
    build: .
    env_file:
      - .env.production  # Not committed to Git
    environment:
      - DATABASE_URL=postgresql://user:pass@db:5432/app
    secrets:
      - payment_secret

secrets:
  payment_secret:
    file: ./secrets/payment_secret.txt
```

### Additional Policies

* Separate variables for staging / production
* Rotate Secrets (periodic key rotation)

---

## 5. Secure Secret Usage Rules

## 1. Never Expose Secrets in API Responses

* Example: Returning a payment service's Client Secret in an endpoint response → **absolutely prohibited**

## 2. Provide Only Minimum Information via API Endpoints

* Only Public Keys may be exposed (e.g., payment service's Client ID)

## 3. Keep Webhook Secrets Server-Only

* Configure in cloud secret management
* Hardcoding in code is prohibited

## 4. Never Output Keys in Logs

* `print(os.environ["X"])` or `logger.info(settings.SECRET_KEY)` is also prohibited

---

## 6. Using pydantic-settings Effectively

### Nested Settings and Validation

```python
from pydantic import field_validator
from pydantic_settings import BaseSettings


class DatabaseSettings(BaseSettings):
    url: str
    pool_size: int = 10
    max_overflow: int = 20

    model_config = SettingsConfigDict(env_prefix="DATABASE_")


class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=True)

    SECRET_KEY: str
    APP_ENV: str = "development"

    @field_validator("SECRET_KEY")
    @classmethod
    def secret_key_must_be_strong(cls, v: str) -> str:
        if len(v) < 32:
            raise ValueError("SECRET_KEY must be at least 32 characters")
        return v
```

### Accessing Settings via Dependency Injection

```python
## app/core/config.py
from functools import lru_cache


@lru_cache
def get_settings() -> Settings:
    return Settings()


## app/api/routes/example.py
from fastapi import Depends
from app.core.config import Settings, get_settings


@router.get("/api/info")
async def get_info(settings: Settings = Depends(get_settings)):
    return {"app_env": settings.APP_ENV}
```

---

## 7. Secret and Production Key Rotation Strategy

* Secret rotation is a mandatory security measure
* Document the following as operational rules:

  1. Who updates the keys
  2. When they are updated (e.g., quarterly)
  3. How they are updated (rotate in cloud secret manager, redeploy containers)

---

## 8. Directory Structure (Environment Variable Related)

```text
app/
  core/
    config.py       # pydantic-settings loader + validation
  api/
    routes/...
  features/
    billing/
    calendar/
.env                # Local only, Git-ignored
.env.example        # Key names only, committed
```

---

## 9. Improvements and Operational Notes

### Leveraging pydantic-settings Features

* Use `SettingsConfigDict` for env file path, encoding, and prefix configuration
* Use `field_validator` for custom validation (e.g., URL format, minimum length)
* Use nested `BaseSettings` with `env_prefix` to group related variables

### Reviewing Environment Variables (Section 2)

* Clearly distinguish public and secret variables in the list
* Classify with comments (e.g., `# Public`, `# Secret`)

---

## 10. Summary

* **All secrets are operated as server-only**
* Type-safe env loading via `pydantic-settings`
* **Use platform-specific secret management in production** (Docker secrets, cloud secret managers)
* `.env` is local only, never included in Git
* Webhooks and API keys are processed securely on the server side
* Key rotation is clearly documented as operational rules
* Fail-fast design ensures immediate process termination on missing environment variables
