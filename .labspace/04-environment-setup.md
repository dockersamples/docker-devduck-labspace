# Environment Setup & Deployment

Now that you have the basic system running, let's dive deeper into environment configuration and deployment strategies. This section covers advanced configuration options, deployment patterns, and production considerations.

!!! info "Learning Focus"
    Master environment configuration, Docker Compose orchestration, and deployment best practices.

## Advanced Environment Configuration

### Environment Variables Deep Dive

Let's explore all available configuration options:

#### 🔧 Core System Configuration

```bash
# Application Settings
APP_NAME=devduck-multi-agent
APP_VERSION=1.0.0
ENVIRONMENT=development  # development, staging, production
DEBUG=true
LOG_LEVEL=INFO  # DEBUG, INFO, WARNING, ERROR, CRITICAL

# Network Configuration
HOST=0.0.0.0
PORT=8000
WORKERS=1  # Number of FastAPI workers

# Security Settings
ALLOWED_HOSTS=localhost,127.0.0.1,0.0.0.0
CORS_ORIGINS=http://localhost:3000,http://localhost:8000
API_KEY_HEADER=X-API-Key
```

#### 🧠 AI Model Configuration

```bash
# Cerebras Configuration
CEREBRAS_API_KEY=your_api_key_here
CEREBRAS_BASE_URL=https://api.cerebras.ai/v1
CEREBRAS_CHAT_MODEL=llama3.1-70b
CEREBRAS_MAX_TOKENS=1000
CEREBRAS_TEMPERATURE=0.7
CEREBRAS_TIMEOUT=30

# Local Model Configuration
LOCAL_MODEL_NAME=microsoft/DialoGPT-medium
LOCAL_MODEL_CACHE_DIR=/app/models
LOCAL_MODEL_DEVICE=auto  # auto, cpu, cuda
LOCAL_MAX_LENGTH=100
LOCAL_TEMPERATURE=0.8

# Model Context Protocol (MCP)
MCP_GATEWAY_URL=http://mcp-gateway:3000
MCP_TIMEOUT=15
MCP_RETRY_ATTEMPTS=3
```

#### 📊 Performance & Monitoring

```bash
# Resource Limits
MAX_CONCURRENT_REQUESTS=10
REQUEST_TIMEOUT=60
MAX_CONTENT_SIZE=1048576  # 1MB in bytes

# Caching
CACHE_TYPE=memory  # memory, redis, file
CACHE_TTL=300  # seconds
CACHE_MAX_SIZE=1000  # number of items

# Monitoring
METRICS_ENABLED=true
METRICS_PORT=9090
HEALTH_CHECK_INTERVAL=30
HEALTH_CHECK_TIMEOUT=5
```

### Environment-Specific Configuration

Create different configurations for various environments:

#### Development Environment

```bash
# Create development environment file
cat > .env.development << EOF
# Development Configuration
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=DEBUG
WORKERS=1

# Use smaller/faster models for development
CEREBRAS_CHAT_MODEL=llama3.1-8b
LOCAL_MODEL_NAME=microsoft/DialoGPT-small

# Enable all debugging features
METRICS_ENABLED=true
HEALTH_CHECK_INTERVAL=10
EOF
```

#### Production Environment

```bash
# Create production environment file
cat > .env.production << EOF
# Production Configuration
ENVIRONMENT=production
DEBUG=false
LOG_LEVEL=WARNING
WORKERS=4

# Use production-grade models
CEREBRAS_CHAT_MODEL=llama3.1-70b
LOCAL_MODEL_NAME=microsoft/DialoGPT-large

# Optimize for performance
MAX_CONCURRENT_REQUESTS=50
CACHE_TYPE=redis
CACHE_TTL=600
EOF
```

### Configuration Validation

Create a script to validate your configuration:

```bash
# Create configuration validator
cat > validate_config.py << 'EOF'
#!/usr/bin/env python3
import os
from pathlib import Path

def validate_config():
    """Validate environment configuration."""
    required_vars = [
        'CEREBRAS_API_KEY',
        'CEREBRAS_BASE_URL', 
        'CEREBRAS_CHAT_MODEL'
    ]
    
    missing_vars = []
    for var in required_vars:
        if not os.getenv(var):
            missing_vars.append(var)
    
    if missing_vars:
        print(f"❌ Missing required variables: {', '.join(missing_vars)}")
        return False
    
    # Validate API key format
    api_key = os.getenv('CEREBRAS_API_KEY')
    if api_key and not api_key.startswith('csk-'):
        print("⚠️  API key doesn't match expected format")
    
    print("✅ Configuration validation passed")
    return True

if __name__ == '__main__':
    validate_config()
EOF

# Make it executable
chmod +x validate_config.py

# Run validation
python validate_config.py
```

## Docker Compose Deep Dive

### Complete Docker Compose Configuration

Let's examine and enhance your `compose.yml`:

```yaml
# Enhanced Docker Compose configuration
version: '3.8'

networks:
  devduck-network:
    driver: bridge
    name: devduck-network

volumes:
  model-cache:
    name: devduck-model-cache
  app-logs:
    name: devduck-logs

services:
  # Main DevDuck Agent Service
  devduck-agent:
    build:
      context: ./agents
      dockerfile: Dockerfile
      target: ${BUILD_TARGET:-production}
    image: devduck-agent:${VERSION:-latest}
    container_name: devduck-orchestrator
    restart: unless-stopped
    ports:
      - "${HOST_PORT:-8000}:8000"
    environment:
      - ENVIRONMENT=${ENVIRONMENT:-development}
      - DEBUG=${DEBUG:-false}
      - LOG_LEVEL=${LOG_LEVEL:-INFO}
      - CEREBRAS_API_KEY=${CEREBRAS_API_KEY}
      - CEREBRAS_BASE_URL=${CEREBRAS_BASE_URL:-https://api.cerebras.ai/v1}
      - CEREBRAS_CHAT_MODEL=${CEREBRAS_CHAT_MODEL:-llama3.1-8b}
      - LOCAL_MODEL_CACHE_DIR=/app/models
      - MCP_GATEWAY_URL=http://mcp-gateway:3000
    volumes:
      - model-cache:/app/models
      - app-logs:/app/logs
      - ./agents:/app:ro  # Read-only development mount
    depends_on:
      mcp-gateway:
        condition: service_healthy
    networks:
      - devduck-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '0.5'
          memory: 1G

  # MCP Gateway Service
  mcp-gateway:
    image: docker/mcp-gateway:${MCP_VERSION:-latest}
    container_name: devduck-mcp-gateway
    restart: unless-stopped
    command: >
      /docker-mcp gateway serve 
      --host 0.0.0.0 
      --port 3000
      mcp-gateway-catalog.yaml
    volumes:
      - ./mcp-gateway-catalog.yaml:/mcp-gateway-catalog.yaml:ro
    networks:
      - devduck-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
      start_period: 20s
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.1'
          memory: 256M

  # Optional: Redis for caching (production)
  redis:
    image: redis:7-alpine
    container_name: devduck-redis
    restart: unless-stopped
    command: redis-server --appendonly yes --maxmemory 256mb --maxmemory-policy allkeys-lru
    volumes:
      - redis-data:/data
    networks:
      - devduck-network
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 128M
    profiles:
      - production

  # Optional: Monitoring with Prometheus
  prometheus:
    image: prom/prometheus:latest
    container_name: devduck-prometheus
    restart: unless-stopped
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--web.console.libraries=/etc/prometheus/console_libraries'
      - '--web.console.templates=/etc/prometheus/consoles'
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - devduck-network
    profiles:
      - monitoring

volumes:
  redis-data:
  prometheus-data:
```

### Multi-Environment Deployment

#### Development Deployment

```bash
# Development with hot-reload and debugging
cat > compose.dev.yml << 'EOF'
version: '3.8'
services:
  devduck-agent:
    environment:
      - DEBUG=true
      - LOG_LEVEL=DEBUG
      - ENVIRONMENT=development
    volumes:
      - ./agents:/app  # Enable hot-reload
    ports:
      - "8000:8000"
      - "5678:5678"  # Debugger port
EOF

# Deploy development environment
docker compose -f compose.yml -f compose.dev.yml up --build
```

#### Production Deployment

```bash
# Production with optimizations and monitoring
cat > compose.prod.yml << 'EOF'
version: '3.8'
services:
  devduck-agent:
    environment:
      - DEBUG=false
      - LOG_LEVEL=WARNING
      - ENVIRONMENT=production
      - WORKERS=4
    restart: always
    deploy:
      replicas: 2
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
EOF

# Deploy production environment with profiles
docker compose \
  -f compose.yml \
  -f compose.prod.yml \
  --profile production \
  --profile monitoring \
  up -d
```

## Deployment Strategies

### Rolling Updates

Implement zero-downtime deployments:

```bash
# Create deployment script
cat > deploy.sh << 'EOF'
#!/bin/bash
set -e

echo "🚀 Starting deployment..."

# Build new images
echo "📦 Building new images..."
docker compose build

# Update services one by one
echo "🔄 Updating services..."
docker compose up -d --no-deps devduck-agent

# Wait for health check
echo "🏥 Waiting for health check..."
while ! curl -f http://localhost:8000/health >/dev/null 2>&1; do
  echo "Waiting for service to be healthy..."
  sleep 5
done

echo "✅ Deployment completed successfully!"
EOF

chmod +x deploy.sh
```

### Scaling Services

```bash
# Scale the DevDuck agent horizontally
docker compose up -d --scale devduck-agent=3

# Scale with load balancer (requires additional configuration)
cat > compose.scale.yml << 'EOF'
version: '3.8'
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - devduck-agent
  
  devduck-agent:
    ports: []  # Remove direct port exposure
EOF
```

### Health Monitoring

```bash
# Create comprehensive health check script
cat > health_check.sh << 'EOF'
#!/bin/bash

# Check container health
echo "🔍 Checking container health..."
docker compose ps

# Check service endpoints
echo "🌐 Checking service endpoints..."
services=("http://localhost:8000/health")

for service in "${services[@]}"; do
  if curl -f "$service" >/dev/null 2>&1; then
    echo "✅ $service - OK"
  else
    echo "❌ $service - FAILED"
  fi
done

# Check resource usage
echo "📊 Resource usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"
EOF

chmod +x health_check.sh
./health_check.sh
```

## Production Considerations

### Security Hardening

```bash
# Create security-focused compose override
cat > compose.security.yml << 'EOF'
version: '3.8'
services:
  devduck-agent:
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
    user: "1000:1000"
EOF
```

### Backup and Recovery

```bash
# Create backup script
cat > backup.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"
mkdir -p "$BACKUP_DIR"

# Backup volumes
docker run --rm -v devduck-model-cache:/data -v "$PWD/$BACKUP_DIR":/backup alpine tar czf /backup/models.tar.gz /data

# Backup configuration
cp .env compose.yml "$BACKUP_DIR/"

echo "✅ Backup completed: $BACKUP_DIR"
EOF

chmod +x backup.sh
```

### Performance Optimization

```bash
# Create performance tuning script
cat > optimize.sh << 'EOF'
#!/bin/bash

# Optimize Docker daemon
echo "🔧 Optimizing Docker settings..."

# Clean up unused resources
docker system prune -f

# Pre-pull images
docker compose pull

# Warm up model cache
docker compose exec devduck-agent python -c "import torch; torch.cuda.is_available()"

echo "✅ Optimization completed"
EOF

chmod +x optimize.sh
```

## Verification and Testing

### Complete System Test

```bash
# Create comprehensive test suite
cat > test_deployment.py << 'EOF'
#!/usr/bin/env python3
import requests
import time
import json

def test_health_endpoint():
    """Test system health endpoint."""
    response = requests.get('http://localhost:8000/health')
    assert response.status_code == 200, f"Health check failed: {response.status_code}"
    print("✅ Health endpoint working")

def test_agent_interaction():
    """Test basic agent interaction."""
    payload = {
        "message": "Hello DevDuck, please introduce yourself",
        "conversation_id": "test-123"
    }
    
    response = requests.post(
        'http://localhost:8000/chat',
        json=payload,
        headers={'Content-Type': 'application/json'}
    )
    
    assert response.status_code == 200, f"Chat failed: {response.status_code}"
    data = response.json()
    assert 'response' in data, "No response field in output"
    print("✅ Agent interaction working")

def test_performance():
    """Test basic performance metrics."""
    start_time = time.time()
    
    response = requests.get('http://localhost:8000/health')
    
    duration = time.time() - start_time
    assert duration < 1.0, f"Response too slow: {duration:.2f}s"
    print(f"✅ Performance acceptable: {duration:.3f}s")

if __name__ == '__main__':
    print("🧪 Running deployment tests...")
    test_health_endpoint()
    test_agent_interaction()
    test_performance()
    print("🎉 All tests passed!")
EOF

# Run the tests
python test_deployment.py
```

### Load Testing

```bash
# Install and run load testing
pip install httpx asyncio

cat > load_test.py << 'EOF'
#!/usr/bin/env python3
import asyncio
import httpx
import time

async def send_request(client, session_id):
    """Send a single request."""
    payload = {
        "message": f"Load test message {session_id}",
        "conversation_id": f"load-test-{session_id}"
    }
    
    try:
        response = await client.post(
            'http://localhost:8000/chat',
            json=payload,
            timeout=30.0
        )
        return response.status_code == 200
    except:
        return False

async def load_test(concurrent_requests=10, total_requests=100):
    """Run load test."""
    async with httpx.AsyncClient() as client:
        semaphore = asyncio.Semaphore(concurrent_requests)
        
        async def bounded_request(session_id):
            async with semaphore:
                return await send_request(client, session_id)
        
        start_time = time.time()
        
        tasks = [bounded_request(i) for i in range(total_requests)]
        results = await asyncio.gather(*tasks)
        
        duration = time.time() - start_time
        success_count = sum(results)
        
        print(f"📊 Load Test Results:")
        print(f"   Total Requests: {total_requests}")
        print(f"   Successful: {success_count}")
        print(f"   Failed: {total_requests - success_count}")
        print(f"   Duration: {duration:.2f}s")
        print(f"   Requests/sec: {total_requests/duration:.2f}")

if __name__ == '__main__':
    asyncio.run(load_test())
EOF

python load_test.py
```

## Next Steps

Excellent! You now have a production-ready deployment with:

- ✅ Comprehensive environment configuration
- ✅ Multi-environment deployment strategies
- ✅ Health monitoring and testing
- ✅ Performance optimization
- ✅ Security hardening

In the next section, you'll start interacting with your multi-agent system and learn how the agents communicate and collaborate.

Ready to explore multi-agent interactions? Let's dive in! 🚀