# PMOVES.AI Integration Guide for E2B Danger Room

## Integration Complete

The PMOVES.AI integration template has been applied to E2B Danger Room.

## Next Steps

### 1. Customize Environment Variables

Edit the following files with your service-specific values:

- `env.shared` - Base environment configuration
- `env.tier-agent` - AGENT tier specific configuration
- `chit/secrets_manifest_v2.yaml` - Add your service's required secrets

### 2. Update Docker Compose

Add the PMOVES.AI environment anchor to your `docker-compose.yml`:

```yaml
services:
  e2b-danger-room:
    <<: [*env-tier-agent, *pmoves-healthcheck]
    # Your existing service configuration...
```

### 3. Integrate Health Check

Add the health check endpoint to your service:

```python
from pmoves_health import add_custom_check, get_health_status

@app.get("/healthz")
async def health_check():
    return await get_health_status()
```

### 4. Add Service Announcement

Add NATS service announcement to your startup:

```python
from pmoves_announcer import announce_service

@app.on_event("startup")
async def startup():
    await announce_service(
        slug="e2b-danger-room",
        name="E2B Danger Room Sandbox",
        url=f"http://e2b-danger-room:0",
        port=0,
        tier="agent"
    )
```

### 5. Test Integration

```bash
# Verify environment variables loaded
docker compose exec e2b-danger-room env | grep PMOVES
```

## Service Details

- **Name:** E2B Danger Room Sandbox
- **Slug:** e2b-danger-room
- **Tier:** agent (sandbox)
- **Port:** API-based (E2B cloud)
- **Health Check:** N/A
- **NATS Enabled:** False
- **GPU Enabled:** False

## Support

For questions or issues, see the PMOVES.AI documentation.
