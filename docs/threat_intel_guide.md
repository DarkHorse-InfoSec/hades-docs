# HADES Cloud Threat Intelligence Guide

## Overview

HADES integrates with cloud-based threat intelligence providers to enrich scan results with external reputation data. When enabled, file hashes, URLs, and IP addresses extracted during scanning are looked up against multiple threat intelligence services to provide consensus verdicts on whether artifacts are malicious, suspicious, or clean.

The threat intelligence system supports:
- **Multi-provider lookups** with consensus scoring across VirusTotal, AbuseIPDB, OTX AlienVault, and MalwareBazaar
- **Automatic IOC type detection** (file hash, URL, IP address, domain)
- **Result caching** with configurable TTL to reduce API calls and enable offline operation
- **Scan result enrichment** that augments existing HADES scan output with threat intel data
- **REST API endpoints** for programmatic access
- **Offline mode** for air-gapped or restricted environments using cached data only

## Getting API Keys

### VirusTotal

1. Create a free account at [https://www.virustotal.com](https://www.virustotal.com)
2. Navigate to your profile and find the API key section
3. Copy your API key (free tier allows 4 requests/minute, 500 requests/day)
4. Add the key to `cli/hades_config.json` under `threat_intel.providers.virustotal.api_key`

### AbuseIPDB

1. Create a free account at [https://www.abuseipdb.com](https://www.abuseipdb.com)
2. Go to your account dashboard and generate an API key
3. Free tier allows 1,000 checks/day
4. Add the key to `cli/hades_config.json` under `threat_intel.providers.abuseipdb.api_key`

### OTX AlienVault

1. Create a free account at [https://otx.alienvault.com](https://otx.alienvault.com)
2. Navigate to Settings > API Integration
3. Copy your OTX API key
4. Add the key to `cli/hades_config.json` under `threat_intel.providers.otx.api_key`

### MalwareBazaar

MalwareBazaar by abuse.ch is a free service that does not require an API key. It is enabled by default and queries the MalwareBazaar database for known malware samples by hash.

## Configuration

All threat intelligence settings are stored in `cli/hades_config.json` under the `threat_intel` section:

```json
{
  "threat_intel": {
    "enabled": false,
    "providers": {
      "virustotal": {
        "api_key": "YOUR_VT_API_KEY",
        "enabled": true
      },
      "abuseipdb": {
        "api_key": "YOUR_ABUSEIPDB_KEY",
        "enabled": true
      },
      "otx": {
        "api_key": "YOUR_OTX_KEY",
        "enabled": true
      },
      "malwarebazaar": {
        "enabled": true
      }
    },
    "cache_ttl_hours": 24,
    "timeout_seconds": 10,
    "offline_mode": false
  }
}
```

| Setting             | Default | Description                                              |
|---------------------|---------|----------------------------------------------------------|
| `enabled`           | `false` | Master toggle for threat intel features                  |
| `cache_ttl_hours`   | `24`    | How long cached results remain valid (in hours)          |
| `timeout_seconds`   | `10`    | Per-provider API request timeout                         |
| `offline_mode`      | `false` | When true, only return cached results (no API calls)     |

## CLI Usage

### Scan with Threat Intel Enrichment

Enrich scan results with cloud threat intelligence data:

```bash
python core/hades_enhanced_cli.py -r --threat-intel /path/to/evidence/
```

Use specific providers only:

```bash
python core/hades_enhanced_cli.py -r --threat-intel --ti-providers virustotal malwarebazaar /path/to/evidence/
```

### Standalone IOC Lookup

Look up a file hash:

```bash
python core/hades_enhanced_cli.py --ti-lookup d41d8cd98f00b204e9800998ecf8427e
```

Look up an IP address:

```bash
python core/hades_enhanced_cli.py --ti-lookup 192.168.1.100
```

Look up a URL:

```bash
python core/hades_enhanced_cli.py --ti-lookup "https://suspicious-domain.example.com/payload"
```

Use specific providers for a lookup:

```bash
python core/hades_enhanced_cli.py --ti-lookup 8.8.8.8 --ti-providers abuseipdb otx
```

### Offline Mode

Use only cached results without making any API calls:

```bash
python core/hades_enhanced_cli.py --ti-lookup d41d8cd98f00b204e9800998ecf8427e --ti-offline
```

```bash
python core/hades_enhanced_cli.py -r --threat-intel --ti-offline /path/to/evidence/
```

### Cache Statistics

View the current state of the threat intel cache:

```bash
python core/hades_enhanced_cli.py --ti-cache-stats
```

## REST API Endpoints

When the HADES API server is running, three threat intelligence endpoints are available:

### GET /api/v1/threat-intel/lookup

Look up an IOC against configured providers.

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/threat-intel/lookup?ioc=d41d8cd98f00b204e9800998ecf8427e"
```

Optional query parameter `providers` (comma-separated) to restrict which providers are queried:

```bash
curl -H "X-API-Key: your-api-key-here" \
     "http://localhost:8666/api/v1/threat-intel/lookup?ioc=8.8.8.8&providers=abuseipdb,otx"
```

### GET /api/v1/threat-intel/cache/stats

Retrieve cache statistics:

```bash
curl -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/cache/stats
```

### DELETE /api/v1/threat-intel/cache

Flush the entire threat intel cache:

```bash
curl -X DELETE -H "X-API-Key: your-api-key-here" \
     http://localhost:8666/api/v1/threat-intel/cache
```

## Cache Management

The threat intelligence system uses an in-memory cache backed by SQLite to store lookup results. This reduces API calls, improves performance, and enables offline operation.

- **TTL**: Cached entries expire after `cache_ttl_hours` (default: 24 hours). Expired entries are automatically refreshed on the next lookup.
- **Statistics**: Use `--ti-cache-stats` or the `/api/v1/threat-intel/cache/stats` endpoint to view hit/miss ratios and entry counts.
- **Flushing**: Use the `DELETE /api/v1/threat-intel/cache` endpoint to clear all cached entries. This is useful after updating provider API keys or when you suspect stale data.
- **Offline mode**: When `--ti-offline` is set, only cached results are returned. No API calls are made to external providers.

## Privacy Considerations

When using cloud threat intelligence, HADES sends the following data to external services:

| Data Type     | Sent To                          | Description                                          |
|---------------|----------------------------------|------------------------------------------------------|
| File hashes   | VirusTotal, MalwareBazaar, OTX   | SHA-256, SHA-1, and MD5 hashes of scanned files      |
| IP addresses  | AbuseIPDB, VirusTotal, OTX       | IP addresses found in metadata or provided for lookup |
| URLs          | VirusTotal, OTX                  | URLs found in metadata or provided for lookup         |
| Domains       | VirusTotal, OTX                  | Domain names extracted from URLs                     |

**HADES never sends file contents to any external service.** Only hashes and extracted indicators are transmitted. This ensures that sensitive file data remains on your system.

For environments where no external communication is permitted:
1. Set `offline_mode: true` in the config to prevent all API calls
2. Use `--ti-offline` on the command line
3. Pre-populate the cache by running lookups in a connected environment, then transfer the cache database

## Troubleshooting

### "Cloud threat intelligence module not available"

The `cloud_threat_intel.py` module could not be imported. Ensure it exists in the `core/` directory and that the `requests` Python package is installed:

```bash
pip install requests
```

### No results from providers

- Verify API keys are correctly set in `cli/hades_config.json`
- Check that the provider is enabled (`"enabled": true`)
- Ensure network connectivity to the provider APIs
- Check the timeout setting; increase `timeout_seconds` if on a slow connection
- Review logs for HTTP error codes (403 = invalid key, 429 = rate limited)

### Rate limiting

Free-tier API keys have rate limits. If you encounter 429 errors:
- Reduce the number of concurrent lookups
- Increase `cache_ttl_hours` to reduce repeat lookups
- Use `--ti-providers` to limit which providers are queried
- Consider upgrading to a paid API tier for high-volume use

### Cache not working

- Ensure the HADES process has write permissions to the working directory (cache is stored as a SQLite database)
- Check `--ti-cache-stats` to verify the cache is being populated
- If the cache database is corrupted, delete it and let HADES recreate it on next run
