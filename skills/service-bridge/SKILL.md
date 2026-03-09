---
name: service-bridge
description: "Use when creating or editing Kodexa service bridges — YAML definitions for external API proxy endpoints with centralized authentication, header configuration, and caching settings"
---

# Kodexa Service Bridge Authoring

## Overview

Service bridges create proxy endpoints in Kodexa that forward requests to external APIs. They centralize authentication, manage headers, and provide caching — so modules and assistants can access external services without embedding credentials.

## When to Use

- Connecting to external REST APIs from modules or assistants
- Centralizing API authentication (API keys, OAuth tokens)
- Setting up proxy endpoints for third-party services
- Configuring request caching for external API calls

## Interactive Wizard

1. **External service** — What API are you connecting to? (ERP, CRM, lookup service, AI provider)
2. **Authentication** — How does it authenticate? (API key header, Bearer token, basic auth)
3. **Endpoints** — What endpoints do you need? (single or multiple paths)
4. **Caching** — Should responses be cached? (duration, cache key)

## Service Bridge Structure

```yaml
slug: my-bridge                        # Required: unique identifier
orgSlug: my-org                        # Required: organization slug
name: "My External API"               # Required: display name
type: serviceBridge                    # Required: must be "serviceBridge"
description: "Connects to external API"

metadata:
  # Target API
  baseUrl: "https://api.example.com"   # Base URL of external API

  # Authentication headers
  headers:
    Authorization: "Bearer ${secrets.API_TOKEN}"  # Use org secrets
    Content-Type: "application/json"
    X-API-Key: "${secrets.API_KEY}"

  # Caching configuration
  caching:
    enabled: true
    ttlSeconds: 300                    # Cache duration (5 minutes)
    cacheKeyHeaders:                   # Headers included in cache key
      - "X-Request-Id"

  # Endpoint mappings
  endpoints:
    - path: "/lookup"                  # Local path exposed by bridge
      targetPath: "/v2/lookup"         # Remote path on target API
      method: GET                      # HTTP method
    - path: "/submit"
      targetPath: "/v2/submit"
      method: POST
```

## Authentication Patterns

### API Key Header
```yaml
metadata:
  headers:
    X-API-Key: "${secrets.EXTERNAL_API_KEY}"
```

### Bearer Token
```yaml
metadata:
  headers:
    Authorization: "Bearer ${secrets.OAUTH_TOKEN}"
```

### Basic Auth
```yaml
metadata:
  headers:
    Authorization: "Basic ${secrets.BASIC_AUTH_ENCODED}"
```

### Custom Headers
```yaml
metadata:
  headers:
    X-Client-Id: "${secrets.CLIENT_ID}"
    X-Client-Secret: "${secrets.CLIENT_SECRET}"
    X-Tenant-Id: "my-tenant"
```

**Note**: Use organization secrets (`${secrets.SECRET_NAME}`) for credentials. Never hardcode sensitive values in the YAML.

## Usage from Modules

```python
# In a Python module, use the bridge endpoint
import requests

def infer(document, pipeline_context):
    # Call external API through the bridge
    response = pipeline_context.http_client.get(
        f"/api/service-bridges/{bridge_slug}/lookup",
        params={"query": document.uuid}
    )

    result = response.json()
    # Use external data in processing
    return document
```

## Usage from Data Forms (Bridge API)

```javascript
// In a Data Form V2 script
const result = await kodexa.http.get('/api/service-bridges/my-bridge/lookup?id=123');
kodexa.log.debug('External lookup result:', result);
```

## Complete Example

```yaml
slug: vendor-lookup
orgSlug: acme-corp
name: "Vendor Lookup Service"
type: serviceBridge
description: "Connects to vendor management system for real-time vendor data"

metadata:
  baseUrl: "https://vendors.acme-corp.com/api/v2"

  headers:
    Authorization: "Bearer ${secrets.VENDOR_API_TOKEN}"
    Content-Type: "application/json"
    Accept: "application/json"
    X-Source: "kodexa-platform"

  caching:
    enabled: true
    ttlSeconds: 600

  endpoints:
    - path: "/vendors"
      targetPath: "/vendors"
      method: GET
    - path: "/vendors/validate"
      targetPath: "/vendors/validate"
      method: POST
```

## Common Mistakes

| Mistake | Fix |
|---------|-----|
| Hardcoded credentials | Always use `${secrets.SECRET_NAME}` |
| Missing Content-Type header | Include appropriate content type for the API |
| Cache TTL too long for volatile data | Reduce TTL or disable caching for real-time lookups |
| Wrong `type` field | Must be `serviceBridge` (not `service-bridge`) |
| Secrets not created in org | Create secrets in Organization > Secrets before referencing |
