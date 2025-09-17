# Advanced Features & Best Practices

Now that you've mastered the fundamentals of multi-agent systems, it's time to explore advanced features and production-ready best practices. This lab covers security, monitoring, scaling, customization, and enterprise deployment strategies.

!!! info "Learning Focus"
    Advanced system configuration, security hardening, production monitoring, scaling strategies, and enterprise-grade deployment patterns.

## Security & Authentication

### 🔒 Securing Your Multi-Agent System

#### Exercise 1: API Authentication and Authorization

```python
# Create security_manager.py
import hashlib
import hmac
import time
import jwt
from typing import Dict, Optional
import secrets

class MultiAgentSecurityManager:
    def __init__(self, secret_key: str = None):
        self.secret_key = secret_key or secrets.token_urlsafe(32)
        self.api_keys = {}
        self.rate_limits = {}
        self.security_events = []
    
    def generate_api_key(self, user_id: str, permissions: list = None) -> str:
        """Generate API key for a user."""
        permissions = permissions or ['chat', 'health']
        
        api_key = f"ak_{secrets.token_urlsafe(32)}"
        
        self.api_keys[api_key] = {
            'user_id': user_id,
            'permissions': permissions,
            'created_at': time.time(),
            'last_used': None,
            'usage_count': 0
        }
        
        return api_key
    
    def validate_api_key(self, api_key: str, required_permission: str = None) -> tuple:
        """Validate API key and check permissions."""
        if api_key not in self.api_keys:
            self._log_security_event('invalid_api_key', {'key': api_key[:10] + '...'}) 
            return False, "Invalid API key"
        
        key_info = self.api_keys[api_key]
        
        # Check if key is expired (optional, 30 days)
        if time.time() - key_info['created_at'] > 30 * 24 * 3600:
            return False, "API key expired"
        
        # Check permissions
        if required_permission and required_permission not in key_info['permissions']:
            self._log_security_event('insufficient_permissions', {
                'user_id': key_info['user_id'],
                'required': required_permission,
                'available': key_info['permissions']
            })
            return False, "Insufficient permissions"
        
        # Update usage
        key_info['last_used'] = time.time()
        key_info['usage_count'] += 1
        
        return True, key_info
    
    def check_rate_limit(self, user_id: str, limit: int = 100, window: int = 3600) -> bool:
        """Check if user is within rate limits (requests per hour)."""
        current_time = time.time()
        
        if user_id not in self.rate_limits:
            self.rate_limits[user_id] = []
        
        # Remove old requests outside the window
        self.rate_limits[user_id] = [
            timestamp for timestamp in self.rate_limits[user_id] 
            if current_time - timestamp < window
        ]
        
        # Check if under limit
        if len(self.rate_limits[user_id]) >= limit:
            self._log_security_event('rate_limit_exceeded', {
                'user_id': user_id,
                'requests': len(self.rate_limits[user_id]),
                'limit': limit
            })
            return False
        
        # Add current request
        self.rate_limits[user_id].append(current_time)
        return True
    
    def _log_security_event(self, event_type: str, details: dict):
        """Log security events for monitoring."""
        event = {
            'timestamp': time.time(),
            'type': event_type,
            'details': details
        }
        
        self.security_events.append(event)
        
        # Keep only last 1000 events
        if len(self.security_events) > 1000:
            self.security_events = self.security_events[-1000:]
    
    def get_security_report(self) -> dict:
        """Generate security report."""
        now = time.time()
        last_24h = now - 24 * 3600
        
        recent_events = [
            event for event in self.security_events 
            if event['timestamp'] > last_24h
        ]
        
        # Count event types
        event_counts = {}
        for event in recent_events:
            event_type = event['type']
            event_counts[event_type] = event_counts.get(event_type, 0) + 1
        
        return {
            'security_events_24h': len(recent_events),
            'event_breakdown': event_counts,
            'total_api_keys': len(self.api_keys)
        }

# Example usage
if __name__ == '__main__':
    security = MultiAgentSecurityManager()
    
    # Generate API keys for different users
    admin_key = security.generate_api_key('admin_user', ['chat', 'health', 'admin'])
    user_key = security.generate_api_key('regular_user', ['chat', 'health'])
    
    print("🔐 Security Manager Test")
    print("=" * 30)
    
    # Test API key validation
    valid, info = security.validate_api_key(admin_key, 'chat')
    print(f"Admin key validation: {valid}")
    
    # Test rate limiting
    for i in range(5):
        allowed = security.check_rate_limit('test_user', limit=3, window=60)
        print(f"Request {i+1}: {'Allowed' if allowed else 'Rate Limited'}")
    
    # Security report
    report = security.get_security_report()
    print(f"\n📊 Security Report: {report}")
```

### 🔐 Environment Security

```bash
# Create secure configuration script
cat > secure_config.sh << 'EOF'
#!/bin/bash

echo "🔒 Implementing Security Best Practices"
echo "===================================="

# 1. Secure environment variables
echo "📝 Setting up secure environment management..."

cat > .env.secure << 'ENVEOF'
# Security settings
AUTH_ENABLED=true
RATE_LIMITING_ENABLED=true
REQUEST_LOGGING=true

# SSL/TLS Configuration
SSL_VERIFY=true
HSTS_ENABLED=true
CSRF_PROTECTION=true

ENVEOF

# 2. Docker security enhancements
cat > docker-compose.security.yml << 'DOCKEREOF'
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
    deploy:
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
DOCKEREOF

echo "✅ Security configuration complete!"
EOF

chmod +x secure_config.sh
./secure_config.sh
```

## Monitoring & Observability

### 📊 Comprehensive Monitoring Setup

#### Exercise 2: Production Monitoring Stack

```yaml
# Create monitoring-stack.yml
version: '3.8'

networks:
  monitoring:
    driver: bridge
  devduck-network:
    external: true

volumes:
  prometheus-data:
  grafana-data:

services:
  # Prometheus for metrics collection
  prometheus:
    image: prom/prometheus:latest
    container_name: devduck-prometheus
    restart: unless-stopped
    ports:
      - "9090:9090"
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheus-data:/prometheus
    networks:
      - monitoring
      - devduck-network

  # Grafana for visualization
  grafana:
    image: grafana/grafana:latest
    container_name: devduck-grafana
    restart: unless-stopped
    ports:
      - "3001:3000"
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin123}
    volumes:
      - grafana-data:/var/lib/grafana
    networks:
      - monitoring
```

#### Create Monitoring Configuration

```bash
# Create monitoring configuration
mkdir -p monitoring

cat > monitoring/prometheus.yml << 'EOF'
global:
  scrape_interval: 15s

scrape_configs:
  - job_name: 'devduck-agent'
    static_configs:
      - targets: ['devduck-agent:8000']
    metrics_path: '/metrics'
    scrape_interval: 10s
EOF

echo "📊 Monitoring stack configuration created!"
echo "To deploy: docker compose -f monitoring-stack.yml up -d"
echo "Access Grafana at: http://localhost:3001 (admin/admin123)"
```

### 📈 Custom Metrics Implementation

```python
# Create metrics_collector.py
from prometheus_client import Counter, Histogram, Gauge, start_http_server
import time
import threading
import psutil

class DevDuckMetrics:
    def __init__(self, port=9091):
        # Define metrics
        self.request_count = Counter(
            'devduck_requests_total',
            'Total number of requests',
            ['agent', 'status']
        )
        
        self.response_time = Histogram(
            'devduck_response_time_seconds',
            'Response time in seconds',
            ['agent']
        )
        
        self.active_conversations = Gauge(
            'devduck_active_conversations',
            'Number of active conversations'
        )
        
        self.system_memory_usage = Gauge(
            'devduck_system_memory_usage_percent',
            'System memory usage percentage'
        )
        
        # Start metrics server
        start_http_server(port)
        
        # Start background monitoring
        self._start_background_monitoring()
    
    def record_request(self, agent: str, status: str, duration: float):
        """Record a request with timing."""
        self.request_count.labels(agent=agent, status=status).inc()
        self.response_time.labels(agent=agent).observe(duration)
    
    def update_active_conversations(self, count: int):
        """Update active conversation count."""
        self.active_conversations.set(count)
    
    def _start_background_monitoring(self):
        """Start background system monitoring."""
        def monitor():
            while True:
                try:
                    # System metrics
                    memory_percent = psutil.virtual_memory().percent
                    self.system_memory_usage.set(memory_percent)
                except Exception as e:
                    print(f"Monitoring error: {e}")
                
                time.sleep(30)  # Update every 30 seconds
        
        thread = threading.Thread(target=monitor, daemon=True)
        thread.start()

# Usage example
if __name__ == '__main__':
    metrics = DevDuckMetrics(port=9091)
    
    print("📊 DevDuck Metrics Collector Started")
    print("Metrics available at: http://localhost:9091/metrics")
    
    # Keep running
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        print("\nShutting down metrics collector...")
```

## Scaling & Performance

### ⚡ Horizontal Scaling Strategies

#### Exercise 3: Load Balancing Setup

```yaml
# Create scaling-compose.yml
version: '3.8'

networks:
  devduck-network:
    driver: bridge
  load-balancer:
    driver: bridge

services:
  # Load balancer
  nginx-lb:
    image: nginx:alpine
    container_name: devduck-load-balancer
    restart: unless-stopped
    ports:
      - "80:80"
    volumes:
      - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
    depends_on:
      - devduck-agent-1
      - devduck-agent-2
    networks:
      - load-balancer
      - devduck-network

  # Multiple DevDuck agent instances
  devduck-agent-1:
    build: ./agents
    container_name: devduck-agent-1
    environment:
      - INSTANCE_ID=agent-1
      - CEREBRAS_API_KEY=${CEREBRAS_API_KEY}
    networks:
      - devduck-network
    deploy:
      resources:
        limits:
          memory: 2G

  devduck-agent-2:
    build: ./agents
    container_name: devduck-agent-2
    environment:
      - INSTANCE_ID=agent-2
      - CEREBRAS_API_KEY=${CEREBRAS_API_KEY}
    networks:
      - devduck-network
    deploy:
      resources:
        limits:
          memory: 2G
```

#### Nginx Configuration for Load Balancing

```bash
# Create nginx configuration
mkdir -p nginx

cat > nginx/nginx.conf << 'EOF'
events {
    worker_connections 1024;
}

http {
    # Upstream servers
    upstream devduck_backend {
        least_conn;
        server devduck-agent-1:8000;
        server devduck-agent-2:8000;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://devduck_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /health {
            proxy_pass http://devduck_backend;
        }
    }
}
EOF

echo "⚖️ Load balancer configuration created!"
```

## Custom Agent Development

### 🛠️ Building Custom Agents

#### Exercise 4: Specialized Agent Creation

```python
# Create custom_agent_template.py
from abc import ABC, abstractmethod
from typing import Dict, List, Optional, Any
import time
import logging

class BaseCustomAgent(ABC):
    """Base class for creating custom agents."""
    
    def __init__(self, agent_name: str, capabilities: List[str] = None):
        self.agent_name = agent_name
        self.capabilities = capabilities or []
        self.logger = logging.getLogger(f"agent.{agent_name}")
        self.metrics = {
            "requests_handled": 0,
            "total_processing_time": 0,
            "errors": 0
        }
    
    @abstractmethod
    def can_handle(self, request: Dict[str, Any]) -> bool:
        """Determine if this agent can handle the request."""
        pass
    
    @abstractmethod
    def process_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Process the request and return response."""
        pass
    
    def handle_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Main request handler with metrics and error handling."""
        start_time = time.time()
        
        try:
            # Check if agent can handle request
            if not self.can_handle(request):
                return {
                    "success": False,
                    "error": f"Agent {self.agent_name} cannot handle this request",
                    "agent": self.agent_name
                }
            
            # Process the request
            response = self.process_request(request)
            
            # Update metrics
            processing_time = time.time() - start_time
            self.metrics["requests_handled"] += 1
            self.metrics["total_processing_time"] += processing_time
            
            response.update({
                "agent": self.agent_name,
                "processing_time": processing_time,
                "success": True
            })
            
            return response
            
        except Exception as e:
            self.metrics["errors"] += 1
            return {
                "success": False,
                "error": str(e),
                "agent": self.agent_name
            }

# Example: Code Analysis Agent
class CodeAnalysisAgent(BaseCustomAgent):
    """Specialized agent for code analysis tasks."""
    
    def __init__(self):
        super().__init__(
            agent_name="code_analyzer",
            capabilities=["code_review", "complexity_analysis", "bug_detection"]
        )
    
    def can_handle(self, request: Dict[str, Any]) -> bool:
        """Check if request is for code analysis."""
        message = request.get("message", "").lower()
        
        code_keywords = ["review", "analyze", "check", "optimize", "debug"]
        has_code = "```" in request.get("message", "")
        has_keywords = any(keyword in message for keyword in code_keywords)
        
        return has_code and has_keywords
    
    def process_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Analyze code and provide insights."""
        message = request.get("message", "")
        
        # Extract code blocks
        import re
        code_blocks = re.findall(r"```(?:\w*\n)?([\s\S]*?)```", message)
        
        if not code_blocks:
            return {
                "response": "No code blocks found to analyze. Please provide code within ``` blocks."
            }
        
        analysis_results = []
        for code in code_blocks:
            analysis = self._analyze_code(code.strip())
            analysis_results.append(analysis)
        
        response = self._generate_analysis_response(analysis_results)
        return {"response": response}
    
    def _analyze_code(self, code: str) -> Dict[str, Any]:
        """Perform basic code analysis."""
        lines = code.split('\n')
        
        analysis = {
            "lines_of_code": len([line for line in lines if line.strip()]),
            "total_lines": len(lines),
            "has_comments": any('#' in line or '//' in line for line in lines),
            "complexity_score": self._calculate_complexity(code),
            "suggestions": []
        }
        
        # Generate basic suggestions
        if not analysis["has_comments"]:
            analysis["suggestions"].append("Consider adding comments for better code documentation")
        
        if analysis["complexity_score"] > 10:
            analysis["suggestions"].append("High complexity detected - consider breaking down into smaller functions")
        
        return analysis
    
    def _calculate_complexity(self, code: str) -> int:
        """Calculate basic cyclomatic complexity."""
        import re
        
        complexity_patterns = [r"\bif\b", r"\bfor\b", r"\bwhile\b", r"\btry\b"]
        complexity = 1  # Base complexity
        
        for pattern in complexity_patterns:
            matches = re.findall(pattern, code, re.IGNORECASE)
            complexity += len(matches)
        
        return complexity
    
    def _generate_analysis_response(self, analyses: List[Dict]) -> str:
        """Generate analysis report."""
        response_parts = ["## Code Analysis Report\n"]
        
        for i, analysis in enumerate(analyses, 1):
            response_parts.append(f"### Code Block {i}")
            response_parts.append(f"- Lines of Code: {analysis['lines_of_code']}")
            response_parts.append(f"- Complexity Score: {analysis['complexity_score']}")
            response_parts.append(f"- Has Comments: {'Yes' if analysis['has_comments'] else 'No'}")
            
            if analysis["suggestions"]:
                response_parts.append("\n**Suggestions:**")
                for suggestion in analysis["suggestions"]:
                    response_parts.append(f"- {suggestion}")
            
            response_parts.append("\n---\n")
        
        return "\n".join(response_parts)

# Agent Registry
class AgentRegistry:
    """Registry for managing multiple custom agents."""
    
    def __init__(self):
        self.agents: Dict[str, BaseCustomAgent] = {}
    
    def register_agent(self, agent: BaseCustomAgent):
        """Register a new agent."""
        self.agents[agent.agent_name] = agent
    
    def find_capable_agent(self, request: Dict[str, Any]) -> Optional[BaseCustomAgent]:
        """Find an agent capable of handling the request."""
        for agent in self.agents.values():
            if agent.can_handle(request):
                return agent
        return None
    
    def handle_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Route request to appropriate agent."""
        agent = self.find_capable_agent(request)
        
        if agent:
            return agent.handle_request(request)
        else:
            return {
                "success": False,
                "error": "No capable agent found for this request",
                "available_agents": list(self.agents.keys())
            }

# Example usage
if __name__ == '__main__':
    # Create agent registry
    registry = AgentRegistry()
    
    # Register custom agent
    code_agent = CodeAnalysisAgent()
    registry.register_agent(code_agent)
    
    # Test request
    test_request = {
        "message": "Please review this code:\n```python\ndef hello_world():\n    print('Hello, World!')\n```"
    }
    
    response = registry.handle_request(test_request)
    print("🤖 Agent Response:")
    print(response.get("response", response))
```

## Enterprise Features

### 🏢 Multi-Tenant Support

```python
# Create multi_tenant_manager.py
from typing import Dict, List, Optional
import uuid
from dataclasses import dataclass

@dataclass
class Tenant:
    id: str
    name: str
    api_quota: int
    model_access: List[str]
    custom_config: Dict
    is_active: bool = True

class MultiTenantManager:
    def __init__(self):
        self.tenants: Dict[str, Tenant] = {}
        self.tenant_usage: Dict[str, Dict] = {}
    
    def create_tenant(self, name: str, api_quota: int = 1000, 
                     model_access: List[str] = None) -> str:
        """Create a new tenant."""
        tenant_id = str(uuid.uuid4())
        
        self.tenants[tenant_id] = Tenant(
            id=tenant_id,
            name=name,
            api_quota=api_quota,
            model_access=model_access or ['local'],
            custom_config={}
        )
        
        self.tenant_usage[tenant_id] = {
            'requests_made': 0,
            'tokens_used': 0,
            'last_request': None
        }
        
        return tenant_id
    
    def get_tenant(self, tenant_id: str) -> Optional[Tenant]:
        """Get tenant by ID."""
        return self.tenants.get(tenant_id)
    
    def check_quota(self, tenant_id: str) -> bool:
        """Check if tenant is within quota limits."""
        tenant = self.get_tenant(tenant_id)
        if not tenant or not tenant.is_active:
            return False
        
        usage = self.tenant_usage.get(tenant_id, {})
        return usage.get('requests_made', 0) < tenant.api_quota
    
    def record_usage(self, tenant_id: str, tokens_used: int = 0):
        """Record tenant usage."""
        if tenant_id in self.tenant_usage:
            self.tenant_usage[tenant_id]['requests_made'] += 1
            self.tenant_usage[tenant_id]['tokens_used'] += tokens_used
            self.tenant_usage[tenant_id]['last_request'] = time.time()

# Example usage
if __name__ == '__main__':
    manager = MultiTenantManager()
    
    # Create tenants
    tenant1 = manager.create_tenant("Company A", api_quota=5000, model_access=['local', 'cerebras'])
    tenant2 = manager.create_tenant("Company B", api_quota=1000, model_access=['local'])
    
    print(f"Created tenants: {tenant1}, {tenant2}")
    
    # Check quotas
    print(f"Tenant 1 quota OK: {manager.check_quota(tenant1)}")
    print(f"Tenant 2 quota OK: {manager.check_quota(tenant2)}")
```

## Deployment Automation

### 🚀 CI/CD Pipeline

```yaml
# Create .github/workflows/deploy.yml
name: Deploy DevDuck Multi-Agent System

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r agents/requirements.txt
          pip install pytest
      
      - name: Run tests
        run: |
          pytest tests/ -v
      
      - name: Security scan
        run: |
          pip install bandit
          bandit -r agents/ -f json -o security-report.json
  
  build:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v2
      
      - name: Login to Container Registry
        uses: docker/login-action@v2
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v4
        with:
          context: ./agents
          push: true
          tags: |
            ghcr.io/${{ github.repository }}/devduck-agent:latest
            ghcr.io/${{ github.repository }}/devduck-agent:${{ github.sha }}
  
  deploy:
    needs: build
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v3
      
      - name: Deploy to production
        run: |
          echo "Deploying to production environment"
          # Add your deployment commands here
```

### 📦 Deployment Script

```bash
# Create deploy.sh
cat > deploy.sh << 'EOF'
#!/bin/bash
set -e

echo "🚀 DevDuck Multi-Agent System Deployment"
echo "======================================"

# Configuration
ENVIRONMENT=${1:-production}
VERSION=${2:-latest}

echo "Environment: $ENVIRONMENT"
echo "Version: $VERSION"

# Pre-deployment checks
echo "🔍 Running pre-deployment checks..."

# Check Docker
if ! command -v docker &> /dev/null; then
    echo "❌ Docker is not installed"
    exit 1
fi

# Check environment variables
if [ -z "$CEREBRAS_API_KEY" ]; then
    echo "❌ CEREBRAS_API_KEY not set"
    exit 1
fi

echo "✅ Pre-deployment checks passed"

# Backup current deployment (if exists)
echo "💾 Creating backup..."
if docker compose ps | grep -q "devduck-agent"; then
    docker compose logs > "backup-$(date +%Y%m%d_%H%M%S).log"
    echo "✅ Logs backed up"
fi

# Pull latest images
echo "📥 Pulling latest images..."
docker compose pull

# Deploy new version
echo "🚀 Deploying new version..."
docker compose up -d --remove-orphans

# Health check
echo "🏥 Running health checks..."
for i in {1..30}; do
    if curl -f http://localhost:8000/health >/dev/null 2>&1; then
        echo "✅ Health check passed"
        break
    fi
    echo "⏳ Waiting for service to be ready... ($i/30)"
    sleep 10
done

if [ $i -eq 30 ]; then
    echo "❌ Health check failed after 5 minutes"
    echo "🔄 Rolling back..."
    docker compose logs
    exit 1
fi

# Cleanup old images
echo "🧹 Cleaning up..."
docker image prune -f

echo "✅ Deployment completed successfully!"
echo "🌐 Service available at: http://localhost:8000"
EOF

chmod +x deploy.sh
echo "📦 Deployment script created: ./deploy.sh"
```

## Best Practices Summary

### ✅ Production Readiness Checklist

```bash
# Create production_checklist.sh
cat > production_checklist.sh << 'EOF'
#!/bin/bash

echo "📋 Production Readiness Checklist"
echo "================================"

CHECKS_PASSED=0
TOTAL_CHECKS=10

# Check 1: Environment variables
echo "1. Checking environment variables..."
if [ -n "$CEREBRAS_API_KEY" ] && [ -n "$JWT_SECRET_KEY" ]; then
    echo "   ✅ Required environment variables set"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Missing required environment variables"
fi

# Check 2: Security configuration
echo "2. Checking security configuration..."
if [ -f ".env.secure" ] && [ -f "docker-compose.security.yml" ]; then
    echo "   ✅ Security configurations present"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Security configurations missing"
fi

# Check 3: Monitoring setup
echo "3. Checking monitoring setup..."
if [ -f "monitoring-stack.yml" ] && [ -d "monitoring" ]; then
    echo "   ✅ Monitoring stack configured"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Monitoring stack not configured"
fi

# Check 4: Resource limits
echo "4. Checking resource limits..."
if grep -q "deploy:" compose.yml && grep -q "limits:" compose.yml; then
    echo "   ✅ Resource limits configured"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Resource limits not set"
fi

# Check 5: Health checks
echo "5. Checking health check endpoints..."
if curl -f http://localhost:8000/health >/dev/null 2>&1; then
    echo "   ✅ Health check endpoint responding"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Health check endpoint not responding"
fi

# Check 6: Backup strategy
echo "6. Checking backup strategy..."
if [ -f "backup.sh" ]; then
    echo "   ✅ Backup script present"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Backup strategy not implemented"
fi

# Check 7: Load balancing
echo "7. Checking load balancing setup..."
if [ -f "nginx/nginx.conf" ] || [ -f "scaling-compose.yml" ]; then
    echo "   ✅ Load balancing configured"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Load balancing not configured"
fi

# Check 8: SSL/TLS
echo "8. Checking SSL/TLS configuration..."
if [ -d "ssl" ] || grep -q "ssl" nginx/nginx.conf 2>/dev/null; then
    echo "   ✅ SSL/TLS configuration present"
    ((CHECKS_PASSED++))
else
    echo "   ❌ SSL/TLS not configured"
fi

# Check 9: Log management
echo "9. Checking log management..."
if grep -q "logging:" compose.yml || [ -d "logs" ]; then
    echo "   ✅ Log management configured"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Log management not configured"
fi

# Check 10: Documentation
echo "10. Checking documentation..."
if [ -f "README.md" ] && [ -d "docs" ]; then
    echo "   ✅ Documentation present"
    ((CHECKS_PASSED++))
else
    echo "   ❌ Documentation incomplete"
fi

# Summary
echo ""
echo "Summary: $CHECKS_PASSED/$TOTAL_CHECKS checks passed"

if [ $CHECKS_PASSED -eq $TOTAL_CHECKS ]; then
    echo "🎉 System is production ready!"
    exit 0
elif [ $CHECKS_PASSED -gt 7 ]; then
    echo "⚠️  System is mostly ready, address remaining issues"
    exit 0
else
    echo "❌ System is not production ready"
    exit 1
fi
EOF

chmod +x production_checklist.sh
```

## Next Steps

Excellent! You've now mastered advanced multi-agent system features:

- ✅ Security implementation with authentication and authorization
- ✅ Comprehensive monitoring and observability
- ✅ Horizontal scaling and load balancing strategies
- ✅ Custom agent development framework
- ✅ Multi-tenant support for enterprise deployments
- ✅ CI/CD pipeline automation
- ✅ Production deployment best practices

In the final section, you'll learn troubleshooting techniques, common issues resolution, and explore next steps for extending your multi-agent system.

Ready for the final lab on troubleshooting and advanced topics? Let's complete your journey! 🏁

---

!!! success "Advanced Features Mastered"
    You now have all the tools and knowledge needed to deploy, secure, monitor, and scale production-ready multi-agent systems. These advanced features will serve you well in real-world deployments!