# Advanced Features & Best Practices

Now that you've mastered the fundamentals of multi-agent systems, it's time to explore advanced features and production-ready best practices. This lab covers security, monitoring, scaling, customization, and enterprise deployment strategies.

!!! info "Learning Focus"
    Advanced system configuration, security hardening, production monitoring, scaling strategies, and enterprise-grade deployment patterns.

## Security & Authentication

### 🔐 Securing Your Multi-Agent System

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

#### Exercise 2: Code Analysis Agent

```python
# Create code_analysis_agent.py
from abc import ABC, abstractmethod
from typing import Dict, List, Optional, Any
import time
import logging
import re

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
    
    def get_agent_info(self) -> Dict[str, Any]:
        """Get agent information and status."""
        avg_processing_time = (
            self.metrics["total_processing_time"] / self.metrics["requests_handled"]
            if self.metrics["requests_handled"] > 0 else 0
        )
        
        return {
            "name": self.agent_name,
            "capabilities": self.capabilities,
            "state": self.agent_state,
            "metrics": self.metrics,
            "avg_processing_time": avg_processing_time,
            "error_rate": self.metrics["errors"] / max(1, self.metrics["requests_handled"])
        }

class CodeAnalysisAgent(BaseCustomAgent):
    """Specialized agent for code analysis tasks."""
    
    def __init__(self):
        super().__init__(
            agent_name="code_analyzer",
            capabilities=["code_review", "complexity_analysis", "bug_detection", "optimization_suggestions"]
        )
        
        # Initialize code analysis tools
        self.supported_languages = ["python", "javascript", "java", "go", "rust"]
        self.analysis_patterns = self._load_analysis_patterns()
    
    def _load_analysis_patterns(self) -> Dict[str, List[str]]:
        """Load code analysis patterns."""
        return {
            "security_issues": [
                r"eval\(",
                r"exec\(",
                r"os\.system",
                r"subprocess\.call",
                r"input\(.*password",
                r"password\s*=\s*['\"][^'\"]*['\"]"  # Hardcoded passwords
            ],
            "performance_issues": [
                r"for\s+\w+\s+in\s+range\(len\(",  # Inefficient iteration
                r"\+.*\+.*\+.*\+",                    # String concatenation
                r"time\.sleep\(\d+\)",                 # Blocking sleep
            ],
            "code_smells": [
                r"def\s+\w+\([^)]*\):\s*pass",       # Empty functions
                r"except:\s*pass",                     # Bare except
                r"import\s+\*",                        # Wildcard imports
            ]
        }
    
    def can_handle(self, request: Dict[str, Any]) -> bool:
        """Check if request is for code analysis."""
        message = request.get("message", "").lower()
        
        # Keywords that indicate code analysis requests
        code_keywords = [
            "review", "analyze", "check", "optimize", "debug", "complexity",
            "performance", "security", "bugs", "code quality"
        ]
        
        # Check if code is present
        has_code = any([
            "```" in request.get("message", ""),
            "def " in request.get("message", ""),
            "function " in request.get("message", ""),
            "class " in request.get("message", "")
        ])
        
        # Check for code analysis keywords
        has_keywords = any(keyword in message for keyword in code_keywords)
        
        return has_code and has_keywords
    
    def process_request(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Analyze code and provide insights."""
        message = request.get("message", "")
        code_blocks = self._extract_code_blocks(message)
        
        if not code_blocks:
            return {
                "response": "No code blocks found to analyze. Please provide code within ``` blocks."
            }
        
        analysis_results = []
        
        for i, code_block in enumerate(code_blocks):
            language = code_block.get("language", "unknown")
            code = code_block.get("code", "")
            
            # Perform analysis
            analysis = self._analyze_code(code, language)
            analysis["block_number"] = i + 1
            analysis_results.append(analysis)
        
        # Generate comprehensive response
        response = self._generate_analysis_response(analysis_results)
        
        return {"response": response}
    
    def _extract_code_blocks(self, message: str) -> List[Dict[str, str]]:
        """Extract code blocks from message."""
        # Pattern for code blocks with language specification
        pattern = r"```(\w*)\n([\s\S]*?)```"
        matches = re.findall(pattern, message)
        
        code_blocks = []
        for language, code in matches:
            code_blocks.append({
                "language": language or "unknown",
                "code": code.strip()
            })
        
        return code_blocks
    
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
        
        # Generate suggestions based on findings
        analysis["suggestions"] = self._generate_suggestions(analysis)
        
        return analysis
    
    def _calculate_complexity(self, code: str) -> int:
        """Calculate cyclomatic complexity (simplified)."""
        # Count decision points
        complexity_patterns = [
            r"\bif\b", r"\belif\b", r"\belse\b", r"\bfor\b", 
            r"\bwhile\b", r"\btry\b", r"\bexcept\b", r"\band\b", r"\bor\b"
        ]
        
        complexity = 1  # Base complexity
        for pattern in complexity_patterns:
            matches = re.findall(pattern, code, re.IGNORECASE)
            complexity += len(matches)
        
        return complexity
    
    def _find_pattern_matches(self, code: str, patterns: List[str]) -> List[Dict[str, str]]:
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
        """Generate improvement suggestions based on analysis."""
        suggestions = []
        
        # Security suggestions
        if analysis["security_issues"]:
            suggestions.append("Consider reviewing security issues: avoid using eval(), exec(), and hardcoded credentials.")
        
        # Performance suggestions
        if analysis["performance_issues"]:
            suggestions.append("Optimize performance: use list comprehensions, avoid string concatenation in loops.")
        
        # Complexity suggestions
        if analysis["complexity_score"] > 10:
            suggestions.append("High complexity detected. Consider breaking down functions into smaller, more focused methods.")
        
        # Code smell suggestions
        if analysis["code_smells"]:
            suggestions.append("Address code smells: avoid empty functions, bare except clauses, and wildcard imports.")
        
        return suggestions
    
    def _generate_analysis_response(self, analysis_results: List[Dict[str, Any]]) -> str:
        """Generate comprehensive analysis response."""
        response_parts = ["## Code Analysis Report\n"]
        
        for analysis in analysis_results:
            block_num = analysis["block_number"]
            response_parts.append(f"### Code Block {block_num} ({analysis['language']})\n")
            
            # Basic metrics
            response_parts.append(f"- **Lines of Code**: {analysis['lines_of_code']}")
            response_parts.append(f"- **Complexity Score**: {analysis['complexity_score']}")
            
            # Issues found
            for issue_type in ["security_issues", "performance_issues", "code_smells"]:
                issues = analysis[issue_type]
                if issues:
                    issue_name = issue_type.replace('_', ' ').title()
                    response_parts.append(f"\n**{issue_name}:**")
                    for issue in issues[:3]:  # Limit to first 3 issues
                        response_parts.append(f"- Line {issue['line']}: {issue['code']}")
            
            # Suggestions
            if analysis["suggestions"]:
                response_parts.append("\n**Suggestions:**")
                for suggestion in analysis["suggestions"]:
                    response_parts.append(f"- {suggestion}")
            
            response_parts.append("\n---\n")
        
        return "\n".join(response_parts)

# Agent Registry for managing custom agents
class AgentRegistry:
    """Registry for managing multiple custom agents."""
    
    def __init__(self):
        self.agents: Dict[str, BaseCustomAgent] = {}
        self.logger = logging.getLogger("agent_registry")
    
    def register_agent(self, agent: BaseCustomAgent):
        """Register a new agent."""
        self.agents[agent.agent_name] = agent
        self.logger.info(f"Registered agent: {agent.agent_name}")
    
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
    
    def get_registry_status(self) -> Dict[str, Any]:
        """Get status of all registered agents."""
        return {
            "total_agents": len(self.agents),
            "agents": {name: agent.get_agent_info() for name, agent in self.agents.items()}
        }

# Example usage
if __name__ == '__main__':
    # Create agent registry
    registry = AgentRegistry()
    
    # Register custom agents
    code_agent = CodeAnalysisAgent()
    registry.register_agent(code_agent)
    
    # Test request
    test_request = {
        "message": "Please review this Python code:\n```python\ndef calculate_total(items):\n    total = 0\n    for i in range(len(items)):\n        total = total + items[i].price\n    return total\n```"
    }
    
    print("🛠️ Testing Custom Code Analysis Agent")
    print("=" * 45)
    
    response = registry.handle_request(test_request)
    
    if response["success"]:
        print("✅ Analysis completed successfully:")
        print(response["response"])
    else:
        print(f"❌ Analysis failed: {response['error']}")
    
    # Show registry status
    status = registry.get_registry_status()
    print(f"\n📊 Registry Status: {status['total_agents']} agents registered")
    for name, info in status['agents'].items():
        print(f"  {name}: {info['metrics']['requests_handled']} requests handled")
```

## Enterprise Integration

### 🏢 Production Deployment Patterns

#### Exercise 3: Kubernetes Deployment

```yaml
# Create kubernetes/namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: devduck-multi-agent
  labels:
    name: devduck-multi-agent
    environment: production
---
# ConfigMap for application configuration
apiVersion: v1
kind: ConfigMap
metadata:
  name: devduck-config
  namespace: devduck-multi-agent
data:
  LOG_LEVEL: "INFO"
  ENVIRONMENT: "production"
  DEBUG: "false"
  WORKERS: "4"
  MAX_CONCURRENT_REQUESTS: "100"
  REQUEST_TIMEOUT: "60"
  CACHE_TTL: "300"
  METRICS_ENABLED: "true"
---
# Secret for sensitive data
apiVersion: v1
kind: Secret
metadata:
  name: devduck-secrets
  namespace: devduck-multi-agent
type: Opaque
stringData:
  CEREBRAS_API_KEY: "your-api-key-here"
  JWT_SECRET_KEY: "your-jwt-secret-here"
  DATABASE_URL: "postgresql://user:pass@db:5432/devduck"
---
# DevDuck Agent Deployment
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devduck-agent
  namespace: devduck-multi-agent
  labels:
    app: devduck-agent
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: devduck-agent
  template:
    metadata:
      labels:
        app: devduck-agent
        version: v1
    spec:
      serviceAccountName: devduck-service-account
      containers:
      - name: devduck-agent
        image: devduck-agent:1.0.0
        imagePullPolicy: Always
        ports:
        - containerPort: 8000
          name: http
        - containerPort: 9090
          name: metrics
        env:
        - name: CEREBRAS_API_KEY
          valueFrom:
            secretKeyRef:
              name: devduck-secrets
              key: CEREBRAS_API_KEY
        - name: JWT_SECRET_KEY
          valueFrom:
            secretKeyRef:
              name: devduck-secrets
              key: JWT_SECRET_KEY
        envFrom:
        - configMapRef:
            name: devduck-config
        resources:
          requests:
            memory: "1Gi"
            cpu: "500m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        readinessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 30
        volumeMounts:
        - name: model-cache
          mountPath: /app/models
        - name: tmp
          mountPath: /tmp
      volumes:
      - name: model-cache
        persistentVolumeClaim:
          claimName: model-cache-pvc
      - name: tmp
        emptyDir: {}
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - weight: 100
            podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - devduck-agent
              topologyKey: kubernetes.io/hostname
---
# Service for DevDuck Agent
apiVersion: v1
kind: Service
metadata:
  name: devduck-agent-service
  namespace: devduck-multi-agent
  labels:
    app: devduck-agent
    service: devduck-agent
spec:
  ports:
  - port: 80
    targetPort: 8000
    protocol: TCP
    name: http
  - port: 9090
    targetPort: 9090
    protocol: TCP
    name: metrics
  selector:
    app: devduck-agent
---
# Ingress for external access
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: devduck-ingress
  namespace: devduck-multi-agent
  annotations:
    kubernetes.io/ingress.class: nginx
    cert-manager.io/cluster-issuer: letsencrypt-prod
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/rate-limit-window: "1m"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  tls:
  - hosts:
    - devduck.yourdomain.com
    secretName: devduck-tls
  rules:
  - host: devduck.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: devduck-agent-service
            port:
              number: 80
---
# Horizontal Pod Autoscaler
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: devduck-agent-hpa
  namespace: devduck-multi-agent
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: devduck-agent
  minReplicas: 2
  maxReplicas: 20
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  - type: Resource
    resource:
      name: memory
      target:
        type: Utilization
        averageUtilization: 80
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
      - type: Pods
        value: 2
        periodSeconds: 60
    scaleUp:
      stabilizationWindowSeconds: 60
      policies:
      - type: Pods
        value: 4
        periodSeconds: 60
---
# PersistentVolumeClaim for model storage
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: model-cache-pvc
  namespace: devduck-multi-agent
spec:
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Gi
  storageClassName: fast-ssd
```

#### Deployment Script

```bash
# Create deploy-to-k8s.sh
cat > deploy-to-k8s.sh << 'EOF'
#!/bin/bash
set -e

echo "🚀 Deploying DevDuck Multi-Agent System to Kubernetes"
echo "===================================================="

# Configuration
NAMESPACE="devduck-multi-agent"
APP_VERSION=${APP_VERSION:-"1.0.0"}
DOCKER_REGISTRY=${DOCKER_REGISTRY:-"your-registry.com"}

# Check prerequisites
echo "🔍 Checking prerequisites..."
if ! command -v kubectl &> /dev/null; then
    echo "❌ kubectl is not installed"
    exit 1
fi

if ! command -v docker &> /dev/null; then
    echo "❌ Docker is not installed"
    exit 1
fi

# Build and push Docker image
echo "🏗️  Building Docker image..."
docker build -t devduck-agent:${APP_VERSION} ./agents
docker tag devduck-agent:${APP_VERSION} ${DOCKER_REGISTRY}/devduck-agent:${APP_VERSION}

echo "📤 Pushing to registry..."
docker push ${DOCKER_REGISTRY}/devduck-agent:${APP_VERSION}

# Update image in Kubernetes manifests
echo "📝 Updating Kubernetes manifests..."
sed -i "s|image: devduck-agent:.*|image: ${DOCKER_REGISTRY}/devduck-agent:${APP_VERSION}|g" kubernetes/namespace.yaml

# Apply Kubernetes manifests
echo "☸️  Applying Kubernetes manifests..."
kubectl apply -f kubernetes/namespace.yaml

# Wait for deployment
echo "⏳ Waiting for deployment to be ready..."
kubectl wait --for=condition=available --timeout=300s deployment/devduck-agent -n ${NAMESPACE}

# Check status
echo "📊 Checking deployment status..."
kubectl get pods -n ${NAMESPACE} -l app=devduck-agent
kubectl get services -n ${NAMESPACE}
kubectl get ingress -n ${NAMESPACE}

# Get external URL
INGRESS_IP=$(kubectl get ingress devduck-ingress -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
if [ -n "$INGRESS_IP" ]; then
    echo "🌐 Application available at: https://${INGRESS_IP}"
else
    echo "🌐 Check ingress configuration for external access"
fi

echo "✅ Deployment completed successfully!"
echo "📋 Next steps:"
echo "   1. Configure DNS for your domain"
echo "   2. Set up monitoring and alerting"
echo "   3. Configure backup and disaster recovery"
echo "   4. Review security settings"

EOF

chmod +x deploy-to-k8s.sh
```

## Performance Optimization

### ⚡ Advanced Caching Strategies

```python
# Create advanced_cache.py
import redis
import pickle
import hashlib
import time
import json
from typing import Any, Optional, Dict
from dataclasses import dataclass

@dataclass
class CacheStats:
    hits: int = 0
    misses: int = 0
    total_requests: int = 0
    
    @property
    def hit_rate(self) -> float:
        return self.hits / max(1, self.total_requests)

class AdvancedCacheManager:
    def __init__(self, redis_url: str = "redis://localhost:6379/0"):
        self.redis_client = redis.from_url(redis_url)
        self.stats = CacheStats()
        self.default_ttl = 300  # 5 minutes
        
        # Cache prefixes for different types
        self.prefixes = {
            "conversation": "conv:",
            "agent_response": "resp:",
            "model_output": "model:",
            "user_session": "sess:",
            "system_config": "config:"
        }
    
    def _generate_key(self, cache_type: str, identifier: str) -> str:
        """Generate cache key with prefix."""
        prefix = self.prefixes.get(cache_type, "misc:")
        # Hash long identifiers
        if len(identifier) > 100:
            identifier = hashlib.sha256(identifier.encode()).hexdigest()
        return f"{prefix}{identifier}"
    
    def get(self, cache_type: str, identifier: str) -> Optional[Any]:
        """Get item from cache."""
        key = self._generate_key(cache_type, identifier)
        self.stats.total_requests += 1
        
        try:
            cached_data = self.redis_client.get(key)
            if cached_data:
                self.stats.hits += 1
                return pickle.loads(cached_data)
            else:
                self.stats.misses += 1
                return None
        except Exception as e:
            print(f"Cache get error: {e}")
            self.stats.misses += 1
            return None
    
    def set(self, cache_type: str, identifier: str, value: Any, ttl: Optional[int] = None) -> bool:
        """Set item in cache."""
        key = self._generate_key(cache_type, identifier)
        ttl = ttl or self.default_ttl
        
        try:
            serialized_data = pickle.dumps(value)
            return self.redis_client.setex(key, ttl, serialized_data)
        except Exception as e:
            print(f"Cache set error: {e}")
            return False
    
    def delete(self, cache_type: str, identifier: str) -> bool:
        """Delete item from cache."""
        key = self._generate_key(cache_type, identifier)
        
        try:
            return bool(self.redis_client.delete(key))
        except Exception as e:
            print(f"Cache delete error: {e}")
            return False
    
    def exists(self, cache_type: str, identifier: str) -> bool:
        """Check if item exists in cache."""
        key = self._generate_key(cache_type, identifier)
        
        try:
            return bool(self.redis_client.exists(key))
        except Exception as e:
            print(f"Cache exists error: {e}")
            return False
    
    def cache_conversation_response(self, conversation_id: str, query: str, response: str, agent: str, ttl: int = 1800):
        """Cache conversation response with metadata."""
        cache_data = {
            "response": response,
            "agent": agent,
            "timestamp": time.time(),
            "query_hash": hashlib.md5(query.encode()).hexdigest()
        }
        
        identifier = f"{conversation_id}:{cache_data['query_hash']}"
        return self.set("agent_response", identifier, cache_data, ttl)
    
    def get_cached_response(self, conversation_id: str, query: str) -> Optional[Dict]:
        """Get cached response for conversation and query."""
        query_hash = hashlib.md5(query.encode()).hexdigest()
        identifier = f"{conversation_id}:{query_hash}"
        
        return self.get("agent_response", identifier)
    
    def invalidate_conversation(self, conversation_id: str):
        """Invalidate all cache entries for a conversation."""
        pattern = self._generate_key("agent_response", f"{conversation_id}:*")
        
        try:
            keys = self.redis_client.keys(pattern)
            if keys:
                self.redis_client.delete(*keys)
        except Exception as e:
            print(f"Cache invalidation error: {e}")
    
    def get_cache_stats(self) -> Dict[str, Any]:
        """Get comprehensive cache statistics."""
        try:
            # Redis info
            redis_info = self.redis_client.info('memory')
            redis_stats = self.redis_client.info('stats')
            
            # Count keys by type
            key_counts = {}
            for cache_type, prefix in self.prefixes.items():
                pattern = f"{prefix}*"
                keys = self.redis_client.keys(pattern)
                key_counts[cache_type] = len(keys)
            
            return {
                "application_stats": {
                    "hits": self.stats.hits,
                    "misses": self.stats.misses,
                    "total_requests": self.stats.total_requests,
                    "hit_rate": self.stats.hit_rate
                },
                "redis_stats": {
                    "used_memory": redis_info.get('used_memory_human', 'Unknown'),
                    "connected_clients": redis_stats.get('connected_clients', 0),
                    "total_commands_processed": redis_stats.get('total_commands_processed', 0),
                    "keyspace_hits": redis_stats.get('keyspace_hits', 0),
                    "keyspace_misses": redis_stats.get('keyspace_misses', 0)
                },
                "key_distribution": key_counts,
                "total_keys": sum(key_counts.values())
            }
        except Exception as e:
            print(f"Error getting cache stats: {e}")
            return {"error": str(e)}
    
    def cleanup_expired_keys(self) -> int:
        """Cleanup expired keys (manual cleanup for testing)."""
        try:
            # This is a simplified cleanup - Redis handles expiration automatically
            # But we can manually clean up keys that should have expired
            cleaned = 0
            for cache_type, prefix in self.prefixes.items():
                pattern = f"{prefix}*"
                keys = self.redis_client.keys(pattern)
                
                for key in keys:
                    ttl = self.redis_client.ttl(key)
                    if ttl == -2:  # Key doesn't exist
                        cleaned += 1
            
            return cleaned
        except Exception as e:
            print(f"Cleanup error: {e}")
            return 0

# Cache decorator for automatic caching
def cache_result(cache_manager: AdvancedCacheManager, cache_type: str = "misc", ttl: int = 300):
    def decorator(func):
        def wrapper(*args, **kwargs):
            # Generate cache key from function name and arguments
            func_signature = f"{func.__name__}:{str(args)}:{str(sorted(kwargs.items()))}"
            cache_key = hashlib.md5(func_signature.encode()).hexdigest()
            
            # Try to get from cache first
            cached_result = cache_manager.get(cache_type, cache_key)
            if cached_result is not None:
                return cached_result
            
            # Execute function and cache result
            result = func(*args, **kwargs)
            cache_manager.set(cache_type, cache_key, result, ttl)
            
            return result
        return wrapper
    return decorator

# Example usage
if __name__ == '__main__':
    # Initialize cache manager
    cache = AdvancedCacheManager()
    
    # Test basic caching
    print("🗄️ Testing Advanced Cache Manager")
    print("=" * 40)
    
    # Cache a conversation response
    cache.cache_conversation_response(
        "conv_123", 
        "What is Docker?", 
        "Docker is a containerization platform...", 
        "local_agent"
    )
    
    # Retrieve cached response
    cached = cache.get_cached_response("conv_123", "What is Docker?")
    if cached:
        print(f"✅ Retrieved cached response from {cached['agent']}")
        print(f"   Response: {cached['response'][:50]}...")
    
    # Test decorator
    @cache_result(cache, "computation", ttl=600)
    def expensive_computation(n):
        print(f"Computing factorial of {n}...")
        result = 1
        for i in range(1, n + 1):
            result *= i
        return result
    
    # First call - will compute and cache
    result1 = expensive_computation(10)
    print(f"First call result: {result1}")
    
    # Second call - will use cache
    result2 = expensive_computation(10)
    print(f"Second call result: {result2}")
    
    # Show cache statistics
    stats = cache.get_cache_stats()
    print("\n📊 Cache Statistics:")
    print(json.dumps(stats, indent=2))
```

## Next Steps

Congratulations! You've mastered advanced DevDuck features:

- ✅ **Security Implementation**: API authentication, rate limiting, and security monitoring
- ✅ **Custom Agent Development**: Built specialized agents for specific tasks
- ✅ **Enterprise Deployment**: Kubernetes deployment with auto-scaling
- ✅ **Performance Optimization**: Advanced caching and monitoring strategies
- ✅ **Production Best Practices**: Security hardening and operational excellence

In the final section, you'll learn comprehensive troubleshooting techniques, common issues and solutions, and explore next steps for extending your multi-agent system.

Ready for the final challenge? Let's master troubleshooting and future planning! 🚀🔧

---

!!! success "Advanced Mastery Achieved"
    You're now equipped with enterprise-grade knowledge for deploying and managing sophisticated multi-agent systems. These patterns will serve you well in production environments.