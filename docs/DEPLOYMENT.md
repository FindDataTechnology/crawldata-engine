# Deployment Guide for crawldata-engine

## Prerequisites

### System Requirements
- Python 3.10+
- 2GB+ RAM recommended
- Network access to data source APIs

### Dependencies
```bash
cd <workspace>/crawldata-engine
uv sync --extra data
```

## Deployment Steps

### 1. Local Development

```bash
# Install in development mode
uv pip install -e .

# Verify installation
python -c "from fd_open_data_mcp.fetch.runner import run_upstream; print('✓ Installed')"
```

### 2. Production Deployment

```bash
# Build package
uv build

# Install from wheel
pip install dist/*.whl

# Or use Docker (if available)
docker build -t crawldata-engine .
docker run -p 8000:8000 crawldata-engine serve
```

### 3. scraw-ops Integration

To integrate with scraw-ops infrastructure:

```bash
# 1. Deploy as a service
cd <workspace>/scraw-ops
# Add crawldata-engine to docker-compose.yml

# 2. Configure environment variables
export FD_OPEN_DATA_MCP_DATABASE_URL="postgresql://..."
export AKSHARE_API_KEY="..."  # if needed
export WIND_API_KEY="..."     # if needed

# 3. Start service
docker-compose up -d crawldata-engine
```

### 4. Monitoring Setup

```bash
# Enable logging
export LOG_LEVEL="INFO"
export LOG_FORMAT="json"

# Health check endpoint
curl http://localhost:8000/health

# Metrics endpoint (if enabled)
curl http://localhost:8000/metrics
```

## Configuration

### Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `FD_OPEN_DATA_MCP_DATABASE_URL` | No | Database URL (defaults to SQLite) |
| `AKSHARE_API_KEY` | Optional | For premium akshare features |
| `WIND_API_KEY` | Optional | For Wind terminal access |
| `EDGAR_IDENTITY` | Yes for SEC | Email for SEC EDGAR API |
| `LOG_LEVEL` | No | Logging level (DEBUG/INFO/WARNING/ERROR) |

### Database Configuration

```bash
# SQLite (default)
export FD_OPEN_DATA_MCP_DATABASE_URL="sqlite:///data/open_data.db"

# PostgreSQL (production)
export FD_OPEN_DATA_MCP_DATABASE_URL="postgresql://user:pass@host:5432/dbname"
```

## Rollback Procedures

If deployment issues occur:

```bash
# 1. Stop current service
docker-compose stop crawldata-engine

# 2. Restore previous version
git checkout <previous-version-tag>
uv pip install -e .

# 3. Restart service
docker-compose start crawldata-engine

# 4. Verify health
curl http://localhost:8000/health
```

## Troubleshooting

### Common Issues

**Issue: "no runner for source X"**
- Solution: Ensure all adapter modules are imported in runner.py
- Check: `python -c "from fd_open_data_mcp.adapters import nbs_gdp"`

**Issue: "FetchError: network timeout"**
- Solution: Increase timeout in adapter implementations
- Check network connectivity to data source APIs

**Issue: "Rate limit exceeded"**
- Solution: Implement caching or reduce request frequency
- Check rate limit policies in docs/DEPLOYMENT.md

## Support

For issues or questions:
- GitHub Issues: https://github.com/FindDataOfficial/crawldata-engine/issues
- Documentation: <workspace>/crawldata-engine/docs/
