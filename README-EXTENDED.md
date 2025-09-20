# NordVPN Docker with HTTP API

This is an extended version of the popular [bubuntux/nordvpn](https://github.com/bubuntux/nordvpn) Docker image that adds HTTP API endpoints for external VPN control from other Docker containers. Latest build is at jackiejke/nordvpn:latest

## Features

- **All original NordVPN functionality** from the base `bubuntux/nordvpn` image
- **HTTP API server** running on port 8080 for external control
- **VPN switching** capability from other containers
- **Health and status monitoring** endpoints
- **Country rotation** support (UK, Netherlands, Germany, US)
- **Python-based API server** with proper error handling and logging

## API Endpoints

### Health Check
```bash
GET /health
```
Returns: `{"status":"healthy"}`

### VPN Status
```bash
GET /status
```
Returns: `{"status":"Connected to...", "ip":"xxx.xxx.xxx.xxx"}`

### Switch VPN Country
```bash
POST /switch
Content-Type: application/json

{"country": "uk"}
```
Supported countries: `uk`, `nl`, `de`, `us`, `ca`, `au`

Returns: `{"success": true, "country": "uk", "message": "Connected to..."}`

## Quick Start

### 1. Build the Extended Image

```bash
docker build -f Dockerfile.extended -t nordvpn-with-api .
```

### 2. Run with Docker Compose

```yaml
version: "3.8"

services:
  nordvpn-api:
    build:
      context: .
      dockerfile: Dockerfile.extended
    container_name: nordvpn-api
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      - TOKEN=your_nordvpn_token_here
      - CONNECT=United_States
      - TECHNOLOGY=NordLynx
      - NETWORK=192.168.1.0/24
      - VPN_API_PORT=8080
    ports:
      - "8080:8080"
    sysctls:
      - net.ipv6.conf.all.disable_ipv6=1
    restart: unless-stopped

  # Example service using the VPN connection
  my-app:
    image: my-app:latest
    network_mode: service:nordvpn-api
    depends_on:
      - nordvpn-api
    restart: unless-stopped
```

### 3. Control VPN from Other Containers

From any container that shares the network with the VPN container:

```bash
# Check VPN health
curl http://localhost:8080/health

# Get current VPN status and IP
curl http://localhost:8080/status

# Switch to UK servers
curl -X POST http://localhost:8080/switch \
  -H "Content-Type: application/json" \
  -d '{"country":"uk"}'

# Switch to Netherlands
curl -X POST http://localhost:8080/switch \
  -H "Content-Type: application/json" \
  -d '{"country":"nl"}'
```

## Environment Variables

All original environment variables from `bubuntux/nordvpn` are supported, plus:

- `VPN_API_PORT` - Port for the HTTP API server (default: 8080)

## Use Cases

### 1. Dynamic VPN Switching for Web Scrapers
```python
import requests
import time

def switch_vpn_country(country):
    response = requests.post('http://localhost:8080/switch', 
                           json={'country': country})
    return response.json()

# Rotate through countries for scraping
countries = ['uk', 'nl', 'de', 'us']
for country in countries:
    switch_vpn_country(country)
    time.sleep(5)  # Wait for connection
    # Perform scraping tasks
```

### 2. Load Balancing Across VPN Servers
```bash
#!/bin/bash
# Rotate VPN every hour
while true; do
    curl -X POST http://localhost:8080/switch
    sleep 3600
done
```

### 3. Health Monitoring
```python
import requests

def check_vpn_health():
    try:
        health = requests.get('http://localhost:8080/health', timeout=5)
        status = requests.get('http://localhost:8080/status', timeout=5)
        return health.status_code == 200 and status.status_code == 200
    except:
        return False
```

## Architecture

The extended image adds:

- **Python HTTP API server** (`/usr/bin/vpn_api_server_py`)
- **Bash alternative server** (`/usr/bin/vpn_api_server`) 
- **S6 service** (`/etc/services.d/vpn-api/run`) for automatic startup
- **Additional dependencies**: `python3`, `socat`, `netcat-openbsd`, `curl`

## Logging

API server logs are written to `/tmp/vpn_api.log` inside the container:

```bash
docker exec nordvpn-api tail -f /tmp/vpn_api.log
```

## Security Considerations

- The API server runs as root to access `nordvpn` CLI commands
- No authentication is implemented - secure your network accordingly
- API is designed for internal container-to-container communication
- Consider using Docker secrets for the NordVPN token

## Troubleshooting

### API Not Responding
```bash
# Check if API service is running
docker exec nordvpn-api ps aux | grep vpn_api

# Check API logs
docker exec nordvpn-api cat /tmp/vpn_api.log

# Test API manually
docker exec nordvpn-api curl http://localhost:8080/health
```

### VPN Connection Issues
```bash
# Check NordVPN status
docker exec nordvpn-api nordvpn status

# Check container logs
docker logs nordvpn-api
```

## Contributing

This extended image is built on top of the excellent work by [bubuntux](https://github.com/bubuntux/nordvpn). 

## License

Same license as the original bubuntux/nordvpn project.
