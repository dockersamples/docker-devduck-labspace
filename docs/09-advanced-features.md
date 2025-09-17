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
    
    def generate_jwt_token(self, user_id: str, permissions: list, expires_in: int = 3600) -> str:
        """Generate JWT token for session management."""
        payload = {
            'user_id': user_id,
            'permissions': permissions,
            'iat': time.time(),
            'exp': time.time() + expires_in
        }
        
        return jwt.encode(payload, self.secret_key, algorithm='HS256')
    
    def validate_jwt_token(self, token: str) -> tuple:
        """Validate JWT token."""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=['HS256'])
            return True, payload
        except jwt.ExpiredSignatureError:
            return False, "Token expired"
        except jwt.InvalidTokenError:
            return False, "Invalid token"
    
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
        
        # Active API keys
        active_keys = sum(
            1 for key_info in self.api_keys.values()
            if key_info['last_used'] and (now - key_info['last_used']) < 7 * 24 * 3600
        )
        
        return {
            'total_api_keys': len(self.api_keys),
            'active_api_keys': active_keys,
            'security_events_24h': len(recent_events),
            'event_breakdown': event_counts,
            'most_active_users': self._get_most_active_users()
        }
    
    def _get_most_active_users(self, limit: int = 5) -> list:
        """Get most active users by API usage."""
        user_usage = {}
        
        for key_info in self.api_keys.values():
            user_id = key_info['user_id']
            user_usage[user_id] = user_usage.get(user_id, 0) + key_info['usage_count']
        
        # Sort by usage count
        sorted_users = sorted(user_usage.items(), key=lambda x: x[1], reverse=True)
        
        return sorted_users[:limit]

# Example usage and testing
if __name__ == '__main__':
    # Initialize security manager
    security = MultiAgentSecurityManager()
    
    # Generate API keys for different users
    admin_key = security.generate_api_key('admin_user', ['chat', 'health', 'admin'])
    user_key = security.generate_api_key('regular_user', ['chat', 'health'])
    
    print("🔐 Security Manager Test")
    print("=" * 30)
    
    # Test API key validation
    valid, info = security.validate_api_key(admin_key, 'chat')
    print(f"Admin key validation: {valid}")
    print(f"Admin permissions: {info['permissions'] if valid else 'N/A'}")
    
    # Test insufficient permissions
    valid, error = security.validate_api_key(user_key, 'admin')
    print(f"User key admin access: {valid} - {error if not valid else 'OK'}")
    
    # Test rate limiting
    for i in range(5):
        allowed = security.check_rate_limit('test_user', limit=3, window=60)
        print(f"Request {i+1}: {'Allowed' if allowed else 'Rate Limited'}")
    
    # Generate JWT token
    jwt_token = security.generate_jwt_token('test_user', ['chat'], expires_in=3600)
    print(f"\nJWT Token generated: {jwt_token[:50]}...")
    
    # Validate JWT token
    valid, payload = security.validate_jwt_token(jwt_token)
    print(f"JWT validation: {valid}")
    if valid:
        print(f"JWT payload: {payload}")
    
    # Security report
    report = security.get_security_report()
    print("\n📊 Security Report:")
    for key, value in report.items():
        print(f"  {key}: {value}")
```

## Custom Agent Development

### 🛠️ Building Specialized Agents

#### Exercise 2: Code Analysis Agent Example

```python
# Create code_analysis_agent.py
from abc import ABC, abstractmethod
from typing import Dict, List, Optional, Any
import time
import re
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
        self.agent_state = "initialized"
    
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
            self.agent_state = "processing"
            
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
            
            # Add metadata to response
            response.update({
                "agent": self.agent_name,
                "processing_time": processing_time,
                "success": True
            })
            
            self.logger.info(f"Processed request in {processing_time:.2f}s")
            return response
            
        except Exception as e:
            self.metrics["errors"] += 1
            self.logger.error(f"Error processing request: {e}")
            
            return {
                "success": False,
                "error": str(e),
                "agent": self.agent_name,
                "processing_time": time.time() - start_time
            }
        
        finally:
            self.agent_state = "idle"

class CodeAnalysisAgent(BaseCustomAgent):
    """Specialized agent for code analysis tasks."""
    
    def __init__(self):
        super().__init__(
            agent_name="code_analyzer",
            capabilities=["code_review", "complexity_analysis", "bug_detection"]
        )
        
        self.supported_languages = ["python", "javascript", "java", "go", "rust"]
        self.analysis_patterns = {
            "security_issues": [
                r"eval\(", r"exec\(", r"os\.system", 
                r"password\s*=\s*['\"][^'\"]*['\"]"  # Hardcoded passwords
            ],
            "performance_issues": [
                r"for\s+\w+\s+in\s+range\(len\(",  # Inefficient iteration
                r"\+.*\+.*\+.*\+",                    # String concatenation
            ],
            "code_smells": [
                r"def\s+\w+\([^)]*\):\s*pass",       # Empty functions
                r"except:\s*pass",                     # Bare except
            ]
        }
    
    def can_handle(self, request: Dict[str, Any]) -> bool:
        """Check if request is for code analysis."""
        message = request.get("message", "").lower()
        
        # Keywords that indicate code analysis requests
        code_keywords = [
            "review", "analyze", "check", "optimize", "debug", "complexity"
        ]
        
        # Check if code is present
        has_code = any([
            "```" in request.get("message", ""),
            "def " in request.get("message", ""),
            "function " in request.get("message", "")
        ])
        
        has_keywords = any(keyword in message for keyword in code_keywords)
        
        return has_code and has_keywords
    
    def process_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Analyze code and provide insights."""
        message = request.get("message", "")
        code_blocks = self._extract_code_blocks(message)
        
        if not code_blocks:
            return {
                "response": "No code blocks found. Please provide code within ``` blocks."
            }
        
        analysis_results = []
        for i, code_block in enumerate(code_blocks):
            analysis = self._analyze_code(code_block["code"], code_block["language"])
            analysis["block_number"] = i + 1
            analysis_results.append(analysis)
        
        response = self._generate_analysis_response(analysis_results)
        return {"response": response}
    
    def _extract_code_blocks(self, message: str) -> List[Dict[str, str]]:
        """Extract code blocks from message."""
        pattern = r"```(\w*)\n([\s\S]*?)```"
        matches = re.findall(pattern, message)
        
        return [{
            "language": language or "unknown",
            "code": code.strip()
        } for language, code in matches]
    
    def _analyze_code(self, code: str, language: str) -> Dict[str, Any]:
        """Perform detailed code analysis."""
        analysis = {
            "language": language,
            "lines_of_code": len(code.split('\n')),
            "security_issues": [],
            "performance_issues": [],
            "code_smells": [],
            "complexity_score": self._calculate_complexity(code),
            "suggestions": []
        }
        
        # Check for various issues
        for category, patterns in self.analysis_patterns.items():
            issues = self._find_pattern_matches(code, patterns)
            analysis[category].extend(issues)
        
        analysis["suggestions"] = self._generate_suggestions(analysis)
        return analysis
    
    def _calculate_complexity(self, code: str) -> int:
        """Calculate cyclomatic complexity (simplified)."""
        complexity_patterns = [
            r"\bif\b", r"\belif\b", r"\bfor\b", r"\bwhile\b", r"\btry\b"
        ]
        
        complexity = 1  # Base complexity
        for pattern in complexity_patterns:
            matches = re.findall(pattern, code, re.IGNORECASE)
            complexity += len(matches)
        
        return complexity
    
    def _find_pattern_matches(self, code: str, patterns: List[str]) -> List[Dict]:
        """Find pattern matches in code."""
        issues = []
        lines = code.split('\n')
        
        for pattern in patterns:
            for line_num, line in enumerate(lines, 1):
                if re.search(pattern, line):
                    issues.append({
                        "line": line_num,
                        "code": line.strip(),
                        "pattern": pattern
                    })
        return issues
    
    def _generate_suggestions(self, analysis: Dict[str, Any]) -> List[str]:
        """Generate improvement suggestions."""
        suggestions = []
        
        if analysis["security_issues"]:
            suggestions.append("Review security issues: avoid eval(), exec(), hardcoded credentials.")
        
        if analysis["performance_issues"]:
            suggestions.append("Optimize performance: use list comprehensions, avoid string concatenation in loops.")
        
        if analysis["complexity_score"] > 10:
            suggestions.append("High complexity detected. Break down into smaller functions.")
        
        return suggestions
    
    def _generate_analysis_response(self, analysis_results: List[Dict[str, Any]]) -> str:
        """Generate comprehensive analysis response."""
        response_parts = ["## Code Analysis Report\n"]
        
        for analysis in analysis_results:
            block_num = analysis["block_number"]
            response_parts.append(f"### Code Block {block_num} ({analysis['language']})")
            response_parts.append(f"- **Lines of Code**: {analysis['lines_of_code']}")
            response_parts.append(f"- **Complexity Score**: {analysis['complexity_score']}")
            
            # Issues found
            for issue_type in ["security_issues", "performance_issues", "code_smells"]:
                issues = analysis[issue_type]
                if issues:
                    issue_name = issue_type.replace('_', ' ').title()
                    response_parts.append(f"\n**{issue_name}:**")
                    for issue in issues[:3]:  # Limit to first 3
                        response_parts.append(f"- Line {issue['line']}: {issue['code']}")
            
            # Suggestions
            if analysis["suggestions"]:
                response_parts.append("\n**Suggestions:**")
                for suggestion in analysis["suggestions"]:
                    response_parts.append(f"- {suggestion}")
            
            response_parts.append("\n")
        
        return "\n".join(response_parts)

# Test the custom agent
if __name__ == '__main__':
    agent = CodeAnalysisAgent()
    
    # Test request
    test_request = {
        "message": """Please review this Python code:
        
```python
def process_data(data):
    result = ""
    for i in range(len(data)):
        result = result + str(data[i]) + ","
    return result

def login(username, password):
    if password == "admin123":  # Security issue!
        return True
    return False
```"""
    }
    
    print("🔍 Testing Code Analysis Agent")
    print("=" * 35)
    
    if agent.can_handle(test_request):
        response = agent.handle_request(test_request)
        print(f"Analysis completed in {response['processing_time']:.2f}s")
        print("\nResponse:")
        print(response['response'])
    else:
        print("Agent cannot handle this request")
```

## Production Deployment

### 🏭 Enterprise-Grade Deployment

#### Exercise 3: Production Configuration

```yaml
# Create production-compose.yml
version: '3.8'

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true
  monitoring:
    driver: bridge

volumes:
  model-cache:
    driver: local
    driver_opts:
      type: none
      device: /opt/devduck/models
      o: bind
  app-logs:
    driver: local
    driver_opts:
      type: none
      device: /var/log/devduck
      o: bind
  ssl-certs:
    driver: local
    driver_opts:
      type: none
      device: /etc/ssl/devduck
      o: bind

services:
  # Production Load Balancer with SSL
  nginx-lb:
    image: nginx:alpine
    container_name: devduck-lb-prod
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/prod-nginx.conf:/etc/nginx/nginx.conf:ro
      - ./nginx/ssl.conf:/etc/nginx/conf.d/ssl.conf:ro
      - ssl-certs:/etc/nginx/ssl:ro
      - app-logs:/var/log/nginx
    environment:
      - NGINX_ENTRYPOINT_QUIET_LOGS=1
    networks:
      - frontend
      - backend
    depends_on:
      - devduck-agent-prod
    healthcheck:
      test: ["CMD", "curl", "-f", "https://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 30s
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  # Production DevDuck Agent Cluster
  devduck-agent-prod:
    build:
      context: ./agents
      dockerfile: Dockerfile.prod
      args:
        - BUILD_ENV=production
    image: devduck-agent:prod
    restart: unless-stopped
    environment:
      - ENVIRONMENT=production
      - DEBUG=false
      - LOG_LEVEL=WARNING
      - WORKERS=4
      - CEREBRAS_API_KEY_FILE=/run/secrets/cerebras_api_key
      - JWT_SECRET_FILE=/run/secrets/jwt_secret
      - DATABASE_URL=postgresql://devduck:${DB_PASSWORD}@postgres-prod:5432/devduck_prod
      - REDIS_URL=redis://redis-prod:6379/0
      - METRICS_ENABLED=true
    volumes:
      - model-cache:/app/models:ro
      - app-logs:/app/logs
    networks:
      - backend
      - monitoring
    secrets:
      - cerebras_api_key
      - jwt_secret
    deploy:
      replicas: 3
      resources:
        limits:
          cpus: '2.0'
          memory: 4G
        reservations:
          cpus: '1.0'
          memory: 2G
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
        window: 120s
      update_config:
        parallelism: 1
        delay: 30s
        failure_action: rollback
        order: start-first
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 60s

  # Production PostgreSQL
  postgres-prod:
    image: postgres:15-alpine
    container_name: devduck-postgres-prod
    restart: unless-stopped
    environment:
      - POSTGRES_DB=devduck_prod
      - POSTGRES_USER=devduck
      - POSTGRES_PASSWORD_FILE=/run/secrets/db_password
      - POSTGRES_INITDB_ARGS=--auth-host=scram-sha-256
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./sql/prod-init.sql:/docker-entrypoint-initdb.d/init.sql:ro
      - app-logs:/var/log/postgresql
    networks:
      - backend
    secrets:
      - db_password
    command: >
      postgres
      -c max_connections=200
      -c shared_buffers=256MB
      -c effective_cache_size=1GB
      -c maintenance_work_mem=64MB
      -c checkpoint_completion_target=0.9
      -c wal_buffers=16MB
      -c default_statistics_target=100
      -c random_page_cost=1.1
      -c effective_io_concurrency=200
      -c work_mem=4MB
      -c min_wal_size=1GB
      -c max_wal_size=4GB
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 1G
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U devduck -d devduck_prod"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s

  # Production Redis with Persistence
  redis-prod:
    image: redis:7-alpine
    container_name: devduck-redis-prod
    restart: unless-stopped
    command: >
      redis-server
      --maxmemory 1gb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --appendfsync everysec
      --auto-aof-rewrite-percentage 100
      --auto-aof-rewrite-min-size 64mb
      --save 900 1
      --save 300 10
      --save 60 10000
    volumes:
      - redis-data:/data
      - app-logs:/var/log/redis
    networks:
      - backend
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 1.5G
        reservations:
          cpus: '0.25'
          memory: 1G
    healthcheck:
      test: ["CMD", "redis-cli", "--raw", "incr", "ping"]
      interval: 10s
      timeout: 3s
      retries: 5
      start_period: 20s

secrets:
  cerebras_api_key:
    file: ./secrets/cerebras_api_key.txt
  jwt_secret:
    file: ./secrets/jwt_secret.txt
  db_password:
    file: ./secrets/db_password.txt

volumes:
  postgres-data:
  redis-data:
```

#### Production Deployment Script

```bash
# Create deploy-production.sh
cat > deploy-production.sh << 'EOF'
#!/bin/bash
set -euo pipefail

# Production deployment script for DevDuck Multi-Agent System
echo "🚀 DevDuck Production Deployment"
echo "================================="

# Configuration
PROJECT_NAME="devduck-prod"
DOMAIN="${DOMAIN:-localhost}"
ENVIRONMENT="production"
BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"

# Colors for output
RED='\033[0;31m'
GREEN='\033[0;32m'
YELLOW='\033[1;33m'
BLUE='\033[0;34m'
NC='\033[0m' # No Color

# Logging function
log() {
    echo -e "${GREEN}[$(date +'%Y-%m-%d %H:%M:%S')]${NC} $1"
}

warn() {
    echo -e "${YELLOW}[WARN]${NC} $1"
}

error() {
    echo -e "${RED}[ERROR]${NC} $1"
    exit 1
}

# Pre-flight checks
log "Running pre-flight checks..."

# Check Docker
if ! command -v docker &> /dev/null; then
    error "Docker is not installed or not in PATH"
fi

if ! docker info &> /dev/null; then
    error "Docker daemon is not running"
fi

# Check Docker Compose
if ! command -v docker-compose &> /dev/null; then
    error "Docker Compose is not installed"
fi

# Check required files
required_files=(
    "production-compose.yml"
    "nginx/prod-nginx.conf"
    "agents/Dockerfile.prod"
)

for file in "${required_files[@]}"; do
    if [[ ! -f "$file" ]]; then
        error "Required file not found: $file"
    fi
done

# Check secrets
log "Checking secrets..."
mkdir -p secrets

if [[ ! -f "secrets/cerebras_api_key.txt" ]]; then
    read -p "Enter Cerebras API key: " -s cerebras_key
    echo "$cerebras_key" > secrets/cerebras_api_key.txt
    chmod 600 secrets/cerebras_api_key.txt
fi

if [[ ! -f "secrets/jwt_secret.txt" ]]; then
    openssl rand -hex 32 > secrets/jwt_secret.txt
    chmod 600 secrets/jwt_secret.txt
fi

if [[ ! -f "secrets/db_password.txt" ]]; then
    openssl rand -base64 32 > secrets/db_password.txt
    chmod 600 secrets/db_password.txt
fi

# SSL Certificates
log "Setting up SSL certificates..."
mkdir -p ssl

if [[ ! -f "ssl/cert.pem" || ! -f "ssl/key.pem" ]]; then
    if [[ "$DOMAIN" == "localhost" ]]; then
        warn "Generating self-signed certificate for localhost"
        openssl req -x509 -newkey rsa:4096 -keyout ssl/key.pem -out ssl/cert.pem -days 365 -nodes \
            -subj "/C=US/ST=CA/L=San Francisco/O=DevDuck/CN=localhost"
    else
        warn "Please ensure SSL certificates are available at ssl/cert.pem and ssl/key.pem"
        warn "For production, use certificates from a trusted CA"
    fi
fi

# Backup existing deployment
if docker-compose -p $PROJECT_NAME -f production-compose.yml ps | grep -q "Up"; then
    log "Creating backup of existing deployment..."
    mkdir -p "$BACKUP_DIR"
    
    # Export current data
    docker-compose -p $PROJECT_NAME -f production-compose.yml exec -T postgres-prod \
        pg_dump -U devduck devduck_prod > "$BACKUP_DIR/database.sql" || warn "Database backup failed"
    
    # Backup volumes
    docker run --rm -v devduck-prod_postgres-data:/data -v "$PWD/$BACKUP_DIR":/backup \
        alpine tar czf /backup/postgres-data.tar.gz -C /data . || warn "Postgres volume backup failed"
    
    docker run --rm -v devduck-prod_redis-data:/data -v "$PWD/$BACKUP_DIR":/backup \
        alpine tar czf /backup/redis-data.tar.gz -C /data . || warn "Redis volume backup failed"
    
    log "Backup completed: $BACKUP_DIR"
fi

# Build production images
log "Building production images..."
docker-compose -p $PROJECT_NAME -f production-compose.yml build --no-cache

# Deploy services
log "Deploying services..."
docker-compose -p $PROJECT_NAME -f production-compose.yml up -d

# Wait for services to be healthy
log "Waiting for services to be healthy..."
sleep 30

# Health checks
log "Running health checks..."
health_check_passed=true

# Check database
if ! docker-compose -p $PROJECT_NAME -f production-compose.yml exec -T postgres-prod \
    pg_isready -U devduck -d devduck_prod; then
    error "Database health check failed"
    health_check_passed=false
fi

# Check Redis
if ! docker-compose -p $PROJECT_NAME -f production-compose.yml exec -T redis-prod \
    redis-cli ping | grep -q PONG; then
    warn "Redis health check failed"
    health_check_passed=false
fi

# Check DevDuck agents
if ! curl -f -s "http://localhost/health" > /dev/null; then
    warn "DevDuck agent health check failed"
    health_check_passed=false
fi

if [[ "$health_check_passed" == "true" ]]; then
    log "✅ All health checks passed!"
else
    warn "Some health checks failed. Check the logs for details."
fi

# Display status
log "Deployment Status:"
docker-compose -p $PROJECT_NAME -f production-compose.yml ps

log "Service URLs:"
echo "  🌐 Application: https://$DOMAIN"
echo "  📊 Health Check: https://$DOMAIN/health"
echo "  📈 Metrics: https://$DOMAIN/metrics (restricted)"

log "Useful Commands:"
echo "  View logs: docker-compose -p $PROJECT_NAME -f production-compose.yml logs -f"
echo "  Scale agents: docker-compose -p $PROJECT_NAME -f production-compose.yml up -d --scale devduck-agent-prod=5"
echo "  Stop services: docker-compose -p $PROJECT_NAME -f production-compose.yml down"
echo "  Backup data: ./backup-production.sh"

log "🎉 Production deployment completed successfully!"
EOF

chmod +x deploy-production.sh
echo "✅ Production deployment script created!"
echo "Run with: ./deploy-production.sh"
```

## Monitoring & Alerting

### 📊 Production Monitoring

#### Exercise 4: Comprehensive Monitoring Setup

```bash
# Create monitoring configuration
mkdir -p monitoring/{prometheus,grafana,alertmanager}

# Prometheus configuration for production
cat > monitoring/prometheus/prometheus.prod.yml << 'EOF'
global:
  scrape_interval: 15s
  evaluation_interval: 15s
  external_labels:
    cluster: 'devduck-prod'
    region: 'us-west-2'

rule_files:
  - "/etc/prometheus/rules/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets:
          - alertmanager:9093
      timeout: 10s
      api_version: v1

scrape_configs:
  # DevDuck Application Metrics
  - job_name: 'devduck-app'
    static_configs:
      - targets: 
          - 'devduck-agent-prod:8000'
    metrics_path: '/metrics'
    scrape_interval: 10s
    scrape_timeout: 5s
    
  # System Metrics
  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
    scrape_interval: 15s

  # Container Metrics
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
    metrics_path: '/metrics'
    scrape_interval: 15s

  # PostgreSQL Metrics
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
    scrape_interval: 15s

  # Redis Metrics  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
    scrape_interval: 15s

  # Nginx Metrics
  - job_name: 'nginx'
    static_configs:
      - targets: ['nginx-exporter:9113']
    scrape_interval: 15s

  # Blackbox Monitoring (External Probes)
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        - https://your-domain.com/health
        - https://your-domain.com/api/status
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: blackbox-exporter:9115
EOF

# Alert Rules
cat > monitoring/prometheus/rules/devduck-alerts.yml << 'EOF'
groups:
- name: devduck.rules
  interval: 30s
  rules:
  # Application Alerts
  - alert: DevDuckHighResponseTime
    expr: histogram_quantile(0.95, rate(devduck_response_time_seconds_bucket[5m])) > 10
    for: 2m
    labels:
      severity: warning
      service: devduck
    annotations:
      summary: "DevDuck high response time"
      description: "DevDuck 95th percentile response time is {{ $value }}s"

  - alert: DevDuckHighErrorRate
    expr: rate(devduck_requests_total{status=~"5.."}[5m]) / rate(devduck_requests_total[5m]) > 0.05
    for: 1m
    labels:
      severity: critical
      service: devduck
    annotations:
      summary: "DevDuck high error rate"
      description: "DevDuck error rate is {{ $value | humanizePercentage }}"

  - alert: DevDuckAgentDown
    expr: up{job="devduck-app"} == 0
    for: 1m
    labels:
      severity: critical
      service: devduck
    annotations:
      summary: "DevDuck agent is down"
      description: "DevDuck agent {{ $labels.instance }} has been down for more than 1 minute"

  - alert: DevDuckLowThroughput
    expr: rate(devduck_requests_total[5m]) < 1
    for: 5m
    labels:
      severity: warning
      service: devduck
    annotations:
      summary: "DevDuck low throughput"
      description: "DevDuck is processing less than 1 request per second"

  # Infrastructure Alerts
  - alert: HighMemoryUsage
    expr: (node_memory_MemTotal_bytes - node_memory_MemAvailable_bytes) / node_memory_MemTotal_bytes > 0.85
    for: 5m
    labels:
      severity: warning
      service: system
    annotations:
      summary: "High memory usage"
      description: "Memory usage is above 85% (current value: {{ $value | humanizePercentage }})"

  - alert: HighCPUUsage
    expr: 100 - (avg by(instance) (irate(node_cpu_seconds_total{mode="idle"}[5m])) * 100) > 80
    for: 5m
    labels:
      severity: warning
      service: system
    annotations:
      summary: "High CPU usage"
      description: "CPU usage is above 80% (current value: {{ $value }}%)"

  - alert: DiskSpaceLow
    expr: (node_filesystem_free_bytes{fstype!="tmpfs"} / node_filesystem_size_bytes{fstype!="tmpfs"}) * 100 < 10
    for: 5m
    labels:
      severity: critical
      service: system
    annotations:
      summary: "Low disk space"
      description: "Disk space is below 10% (current value: {{ $value }}%)"

  # Database Alerts
  - alert: PostgreSQLDown
    expr: pg_up == 0
    for: 1m
    labels:
      severity: critical
      service: database
    annotations:
      summary: "PostgreSQL is down"
      description: "PostgreSQL instance {{ $labels.instance }} is down"

  - alert: PostgreSQLHighConnections
    expr: pg_stat_database_numbackends / pg_settings_max_connections > 0.8
    for: 5m
    labels:
      severity: warning
      service: database
    annotations:
      summary: "PostgreSQL high connections"
      description: "PostgreSQL connection usage is above 80%"

  # Redis Alerts
  - alert: RedisDown
    expr: redis_up == 0
    for: 1m
    labels:
      severity: critical
      service: cache
    annotations:
      summary: "Redis is down"
      description: "Redis instance {{ $labels.instance }} is down"

  - alert: RedisHighMemoryUsage
    expr: redis_memory_used_bytes / redis_memory_max_bytes > 0.9
    for: 5m
    labels:
      severity: warning
      service: cache
    annotations:
      summary: "Redis high memory usage"
      description: "Redis memory usage is above 90%"
EOF

echo "📊 Monitoring configuration created!"
echo "Deploy with: docker-compose -f monitoring-stack.yml up -d"
```

## Performance Optimization

### ⚡ Advanced Optimization Techniques

```python
# Create performance_optimizer.py
import asyncio
import aiohttp
import time
from typing import Dict, List, Optional
import statistics
from concurrent.futures import ThreadPoolExecutor

class PerformanceOptimizer:
    """Advanced performance optimization for multi-agent systems."""
    
    def __init__(self):
        self.metrics_history = []
        self.optimization_actions = []
        self.thresholds = {
            'response_time_p95': 5.0,  # 5 seconds
            'error_rate': 0.02,         # 2%
            'cpu_usage': 80.0,          # 80%
            'memory_usage': 85.0,       # 85%
            'queue_depth': 10           # 10 pending requests
        }
    
    async def analyze_performance(self) -> Dict:
        """Analyze current system performance."""
        tasks = [
            self._measure_response_times(),
            self._check_error_rates(),
            self._monitor_resources(),
            self._analyze_bottlenecks()
        ]
        
        results = await asyncio.gather(*tasks)
        
        performance_analysis = {
            'timestamp': time.time(),
            'response_metrics': results[0],
            'error_metrics': results[1],
            'resource_metrics': results[2],
            'bottleneck_analysis': results[3],
            'optimization_recommendations': []
        }
        
        # Generate recommendations
        recommendations = self._generate_recommendations(performance_analysis)
        performance_analysis['optimization_recommendations'] = recommendations
        
        self.metrics_history.append(performance_analysis)
        
        return performance_analysis
    
    async def _measure_response_times(self) -> Dict:
        """Measure response times across different endpoints."""
        endpoints = [
            '/health',
            '/chat',
            '/metrics'
        ]
        
        response_times = {}
        
        async with aiohttp.ClientSession() as session:
            for endpoint in endpoints:
                times = []
                
                for _ in range(10):  # 10 samples per endpoint
                    start_time = time.time()
                    try:
                        async with session.get(f'http://localhost:8000{endpoint}', timeout=30) as response:
                            duration = time.time() - start_time
                            times.append(duration)
                    except Exception:
                        times.append(30.0)  # Timeout value
                
                if times:
                    response_times[endpoint] = {
                        'mean': statistics.mean(times),
                        'median': statistics.median(times),
                        'p95': statistics.quantiles(times, n=20)[18] if len(times) >= 20 else max(times),
                        'min': min(times),
                        'max': max(times)
                    }
        
        return response_times
    
    async def _check_error_rates(self) -> Dict:
        """Check error rates from application metrics."""
        # Simulate checking metrics endpoint
        try:
            async with aiohttp.ClientSession() as session:
                async with session.get('http://localhost:8000/metrics', timeout=10) as response:
                    if response.status == 200:
                        # Parse metrics (simplified)
                        return {
                            'overall_error_rate': 0.01,  # 1%
                            'local_agent_errors': 0.005,
                            'cerebras_agent_errors': 0.015,
                            'system_errors': 0.002
                        }
        except Exception:
            pass
        
        return {'error': 'Could not retrieve error metrics'}
    
    async def _monitor_resources(self) -> Dict:
        """Monitor system resource usage."""
        # This would integrate with Docker stats or system monitoring
        # Simulated for example purposes
        return {
            'cpu_usage_percent': 75.2,
            'memory_usage_percent': 68.5,
            'disk_usage_percent': 45.0,
            'network_io_mbps': 12.3,
            'container_count': 3,
            'healthy_containers': 3
        }
    
    async def _analyze_bottlenecks(self) -> List[Dict]:
        """Analyze potential system bottlenecks."""
        bottlenecks = []
        
        # Analyze agent response patterns
        recent_metrics = self.metrics_history[-5:] if len(self.metrics_history) >= 5 else self.metrics_history
        
        if recent_metrics:
            avg_response_times = []
            for metric in recent_metrics:
                if 'response_metrics' in metric and '/chat' in metric['response_metrics']:
                    avg_response_times.append(metric['response_metrics']['/chat']['mean'])
            
            if avg_response_times and statistics.mean(avg_response_times) > 8.0:
                bottlenecks.append({
                    'type': 'slow_responses',
                    'severity': 'high',
                    'description': 'Chat endpoint showing consistently slow response times',
                    'suggested_action': 'Scale up agents or optimize model loading'
                })
        
        # Check for resource constraints
        # This would be implemented based on actual resource monitoring
        
        return bottlenecks
    
    def _generate_recommendations(self, analysis: Dict) -> List[str]:
        """Generate optimization recommendations based on analysis."""
        recommendations = []
        
        # Response time recommendations
        response_metrics = analysis.get('response_metrics', {})
        chat_metrics = response_metrics.get('/chat', {})
        
        if chat_metrics.get('p95', 0) > self.thresholds['response_time_p95']:
            recommendations.append(
                "Scale up DevDuck agents - 95th percentile response time exceeds threshold"
            )
        
        if chat_metrics.get('mean', 0) > 3.0:
            recommendations.append(
                "Consider using smaller/faster models for Local Agent"
            )
        
        # Error rate recommendations
        error_metrics = analysis.get('error_metrics', {})
        overall_error_rate = error_metrics.get('overall_error_rate', 0)
        
        if overall_error_rate > self.thresholds['error_rate']:
            recommendations.append(
                "Investigate error sources - error rate exceeds acceptable threshold"
            )
        
        # Resource recommendations
        resource_metrics = analysis.get('resource_metrics', {})
        cpu_usage = resource_metrics.get('cpu_usage_percent', 0)
        memory_usage = resource_metrics.get('memory_usage_percent', 0)
        
        if cpu_usage > self.thresholds['cpu_usage']:
            recommendations.append(
                "High CPU usage detected - consider horizontal scaling"
            )
        
        if memory_usage > self.thresholds['memory_usage']:
            recommendations.append(
                "High memory usage - optimize model caching or increase memory limits"
            )
        
        # Bottleneck recommendations
        bottlenecks = analysis.get('bottleneck_analysis', [])
        for bottleneck in bottlenecks:
            if bottleneck['severity'] == 'high':
                recommendations.append(f"Critical: {bottleneck['suggested_action']}")
        
        return recommendations
    
    async def auto_optimize(self) -> Dict:
        """Automatically apply optimization strategies."""
        analysis = await self.analyze_performance()
        applied_optimizations = []
        
        # Auto-scaling logic (simplified)
        resource_metrics = analysis.get('resource_metrics', {})
        
        if resource_metrics.get('cpu_usage_percent', 0) > 85:
            # Trigger scaling up
            optimization = await self._scale_agents(direction='up')
            applied_optimizations.append(optimization)
        elif resource_metrics.get('cpu_usage_percent', 0) < 30:
            # Trigger scaling down
            optimization = await self._scale_agents(direction='down')
            applied_optimizations.append(optimization)
        
        # Memory optimization
        if resource_metrics.get('memory_usage_percent', 0) > 90:
            optimization = await self._optimize_memory_usage()
            applied_optimizations.append(optimization)
        
        # Cache optimization
        response_metrics = analysis.get('response_metrics', {})
        if response_metrics.get('/chat', {}).get('mean', 0) > 6.0:
            optimization = await self._optimize_caching()
            applied_optimizations.append(optimization)
        
        return {
            'analysis': analysis,
            'applied_optimizations': applied_optimizations,
            'next_analysis_in': 300  # 5 minutes
        }
    
    async def _scale_agents(self, direction: str) -> Dict:
        """Scale agents up or down."""
        # This would integrate with Docker Compose or Kubernetes
        action = f"scale_{direction}"
        
        optimization = {
            'type': 'scaling',
            'action': action,
            'timestamp': time.time(),
            'status': 'simulated',  # In real implementation: 'applied' or 'failed'
            'details': f"Would {action} DevDuck agents based on resource usage"
        }
        
        self.optimization_actions.append(optimization)
        return optimization
    
    async def _optimize_memory_usage(self) -> Dict:
        """Optimize memory usage."""
        optimization = {
            'type': 'memory_optimization',
            'action': 'cache_cleanup',
            'timestamp': time.time(),
            'status': 'simulated',
            'details': 'Would clear model cache and optimize memory allocation'
        }
        
        self.optimization_actions.append(optimization)
        return optimization
    
    async def _optimize_caching(self) -> Dict:
        """Optimize caching strategy."""
        optimization = {
            'type': 'cache_optimization',
            'action': 'increase_cache_size',
            'timestamp': time.time(),
            'status': 'simulated',
            'details': 'Would increase cache TTL and size to improve response times'
        }
        
        self.optimization_actions.append(optimization)
        return optimization
    
    def get_optimization_report(self) -> Dict:
        """Get comprehensive optimization report."""
        recent_analyses = self.metrics_history[-10:] if len(self.metrics_history) >= 10 else self.metrics_history
        recent_optimizations = self.optimization_actions[-5:] if len(self.optimization_actions) >= 5 else self.optimization_actions
        
        # Calculate trends
        response_time_trend = []
        error_rate_trend = []
        
        for analysis in recent_analyses:
            chat_metrics = analysis.get('response_metrics', {}).get('/chat', {})
            if 'mean' in chat_metrics:
                response_time_trend.append(chat_metrics['mean'])
            
            error_metrics = analysis.get('error_metrics', {})
            if 'overall_error_rate' in error_metrics:
                error_rate_trend.append(error_metrics['overall_error_rate'])
        
        return {
            'current_performance': recent_analyses[-1] if recent_analyses else None,
            'performance_trends': {
                'response_time_trend': response_time_trend,
                'error_rate_trend': error_rate_trend,
                'trend_direction': self._calculate_trend(response_time_trend)
            },
            'recent_optimizations': recent_optimizations,
            'optimization_effectiveness': self._calculate_optimization_effectiveness(),
            'recommendations': self._get_priority_recommendations()
        }
    
    def _calculate_trend(self, values: List[float]) -> str:
        """Calculate trend direction."""
        if len(values) < 3:
            return 'insufficient_data'
        
        recent_avg = statistics.mean(values[-3:])
        earlier_avg = statistics.mean(values[:-3]) if len(values) > 3 else values[0]
        
        if recent_avg > earlier_avg * 1.1:
            return 'worsening'
        elif recent_avg < earlier_avg * 0.9:
            return 'improving'
        else:
            return 'stable'
    
    def _calculate_optimization_effectiveness(self) -> str:
        """Calculate how effective recent optimizations have been."""
        if len(self.optimization_actions) < 2 or len(self.metrics_history) < 5:
            return 'insufficient_data'
        
        # Simplified effectiveness calculation
        # In practice, this would compare metrics before and after optimizations
        return 'moderate'  # 'high', 'moderate', 'low'
    
    def _get_priority_recommendations(self) -> List[Dict]:
        """Get prioritized recommendations."""
        if not self.metrics_history:
            return []
        
        latest_analysis = self.metrics_history[-1]
        recommendations = latest_analysis.get('optimization_recommendations', [])
        
        # Prioritize recommendations
        priority_recommendations = []
        
        for rec in recommendations:
            priority = 'medium'
            if 'Critical' in rec or 'high' in rec.lower():
                priority = 'high'
            elif 'consider' in rec.lower() or 'optimize' in rec.lower():
                priority = 'low'
            
            priority_recommendations.append({
                'recommendation': rec,
                'priority': priority,
                'estimated_impact': 'medium'  # This would be calculated based on historical data
            })
        
        # Sort by priority
        priority_order = {'high': 3, 'medium': 2, 'low': 1}
        priority_recommendations.sort(key=lambda x: priority_order[x['priority']], reverse=True)
        
        return priority_recommendations[:5]  # Top 5 recommendations

# Example usage
async def main():
    optimizer = PerformanceOptimizer()
    
    print("⚡ Running Performance Analysis")
    print("=" * 35)
    
    # Run analysis
    analysis = await optimizer.analyze_performance()
    
    print("📊 Performance Analysis Results:")
    print(f"Response Times: {analysis.get('response_metrics', {}).get('/chat', {}).get('mean', 'N/A')}")
    print(f"Error Rate: {analysis.get('error_metrics', {}).get('overall_error_rate', 'N/A')}")
    print(f"CPU Usage: {analysis.get('resource_metrics', {}).get('cpu_usage_percent', 'N/A')}%")
    
    print("\n💡 Recommendations:")
    for i, rec in enumerate(analysis.get('optimization_recommendations', []), 1):
        print(f"  {i}. {rec}")
    
    # Run auto-optimization
    print("\n🔧 Running Auto-Optimization...")
    optimization_result = await optimizer.auto_optimize()
    
    print("Applied Optimizations:")
    for opt in optimization_result['applied_optimizations']:
        print(f"  - {opt['type']}: {opt['details']}")

if __name__ == '__main__':
    asyncio.run(main())
```

## Next Steps

Congratulations! You've completed the advanced features section and mastered:

- ✅ **Security & Authentication**: API keys, JWT tokens, rate limiting, and security monitoring
- ✅ **Custom Agent Development**: Building specialized agents for specific tasks
- ✅ **Production Deployment**: Enterprise-grade deployment with Docker Compose
- ✅ **Monitoring & Alerting**: Comprehensive monitoring with Prometheus and Grafana
- ✅ **Performance Optimization**: Advanced optimization techniques and auto-scaling

In the final section, you'll learn troubleshooting techniques, get guidance on common issues, and discover paths for further learning and development.

Ready for the final lab? Let's wrap up with troubleshooting and next steps! 🏁

---

!!! success "Advanced Features Mastery"
    You now have the knowledge to deploy, secure, monitor, and optimize production-grade multi-agent systems. These skills will serve you well in building enterprise-scale AI applications.