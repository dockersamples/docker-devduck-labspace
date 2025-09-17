# Troubleshooting & Next Steps

Congratulations on completing your journey through the Docker DevDuck Multi-Agent Workshop! This final lab focuses on troubleshooting common issues, debugging techniques, and charting your path forward with multi-agent systems.

!!! info "Workshop Completion"
    By now you've mastered multi-agent system architecture, deployment, monitoring, and optimization. This section helps you handle real-world challenges and continue learning.

## Common Issues & Solutions

### 🔧 Troubleshooting Guide

#### Issue 1: Container Startup Failures

**Symptoms:**
- Containers exit immediately after starting
- "Model too big" errors
- Memory allocation failures
- Port binding conflicts

**Diagnostic Steps:**

```bash
# Check container status and logs
docker compose ps
docker compose logs devduck-agent

# Check system resources
docker system df
docker stats --no-stream

# Verify port availability
netstat -tuln | grep 8000
lsof -i :8000
```

**Solutions:**

```bash
# Create troubleshooting script
cat > troubleshoot-startup.sh << 'EOF'
#!/bin/bash

echo "🔍 DevDuck Startup Troubleshooting"
echo "=================================="

# Check Docker daemon
if ! docker info >/dev/null 2>&1; then
    echo "❌ Docker daemon not running"
    echo "   Solution: Start Docker Desktop or systemctl start docker"
    exit 1
fi

# Check available memory
MEM_AVAILABLE=$(free -m | awk 'NR==2{print $7}')
echo "💾 Available Memory: ${MEM_AVAILABLE}MB"

if [ "$MEM_AVAILABLE" -lt 4000 ]; then
    echo "⚠️  Low memory detected"
    echo "   Recommended: At least 4GB available memory"
    echo "   Solutions:"
    echo "     - Close other applications"
    echo "     - Use smaller model: LOCAL_MODEL_NAME=microsoft/DialoGPT-small"
    echo "     - Increase Docker memory limit"
fi

# Check disk space
DISK_AVAILABLE=$(df -BG . | awk 'NR==2{print $4}' | sed 's/G//')
echo "💿 Available Disk: ${DISK_AVAILABLE}GB"

if [ "$DISK_AVAILABLE" -lt 10 ]; then
    echo "⚠️  Low disk space"
    echo "   Solutions:"
    echo "     - Clean Docker: docker system prune -a"
    echo "     - Remove unused volumes: docker volume prune"
fi

# Check port conflicts
PORT_IN_USE=$(netstat -tuln 2>/dev/null | grep ':8000 ' | wc -l)
if [ "$PORT_IN_USE" -gt 0 ]; then
    echo "⚠️  Port 8000 is already in use"
    echo "   Solutions:"
    echo "     - Stop conflicting service: lsof -ti:8000 | xargs kill"
    echo "     - Use different port: modify compose.yml ports section"
fi

# Check environment variables
if [ -z "$CEREBRAS_API_KEY" ] && [ ! -f ".env" ]; then
    echo "⚠️  No Cerebras API key found"
    echo "   Solutions:"
    echo "     - Create .env file with CEREBRAS_API_KEY=your_key"
    echo "     - Export CEREBRAS_API_KEY=your_key"
fi

# Test container creation
echo "\n🧪 Testing container creation..."
TEST_CONTAINER=$(docker run --rm -d --name devduck-test python:3.11-slim sleep 10)
if [ $? -eq 0 ]; then
    echo "✅ Container creation successful"
    docker stop devduck-test >/dev/null 2>&1
else
    echo "❌ Container creation failed"
    echo "   Check Docker installation and permissions"
fi

echo "\n📋 Quick Fixes:"
echo "1. Restart Docker: systemctl restart docker (Linux) or restart Docker Desktop"
echo "2. Clean system: docker system prune -a && docker volume prune"
echo "3. Use lightweight model: LOCAL_MODEL_NAME=microsoft/DialoGPT-small"
echo "4. Check logs: docker compose logs -f devduck-agent"
echo "5. Rebuild: docker compose up --build --force-recreate"

EOF

chmod +x troubleshoot-startup.sh
./troubleshoot-startup.sh
```

#### Issue 2: Agent Communication Failures

**Symptoms:**
- "Agent unavailable" messages
- Timeouts on requests
- Inconsistent responses
- Network connectivity errors

**Debugging Tools:**

```python
# Create agent_diagnostics.py
import requests
import time
import asyncio
import json
from typing import Dict, List

class AgentDiagnostics:
    def __init__(self, base_url="http://localhost:8000"):
        self.base_url = base_url
        self.diagnostic_results = []
    
    async def run_full_diagnostics(self) -> Dict:
        """Run comprehensive agent diagnostics."""
        print("🔬 Running Agent Diagnostics")
        print("=" * 30)
        
        diagnostics = {
            "timestamp": time.time(),
            "tests": {}
        }
        
        # Test suite
        test_functions = [
            ("connectivity", self.test_connectivity),
            ("health_check", self.test_health_endpoint),
            ("basic_chat", self.test_basic_chat),
            ("agent_routing", self.test_agent_routing),
            ("error_handling", self.test_error_handling),
            ("performance", self.test_performance),
            ("concurrent_requests", self.test_concurrent_requests)
        ]
        
        for test_name, test_func in test_functions:
            print(f"\n🧪 Running {test_name} test...")
            try:
                result = await test_func()
                diagnostics["tests"][test_name] = result
                status = "✅ PASS" if result.get("success") else "❌ FAIL"
                print(f"   {status}: {result.get('message', 'No message')}")
            except Exception as e:
                diagnostics["tests"][test_name] = {
                    "success": False,
                    "error": str(e),
                    "message": f"Test failed with exception: {e}"
                }
                print(f"   ❌ ERROR: {e}")
            
            # Small delay between tests
            await asyncio.sleep(1)
        
        # Generate summary
        diagnostics["summary"] = self.generate_summary(diagnostics["tests"])
        
        return diagnostics
    
    async def test_connectivity(self) -> Dict:
        """Test basic network connectivity."""
        try:
            response = requests.get(f"{self.base_url}/", timeout=10)
            return {
                "success": response.status_code in [200, 404],  # 404 is OK if no root route
                "status_code": response.status_code,
                "response_time": response.elapsed.total_seconds(),
                "message": f"Connection established (HTTP {response.status_code})"
            }
        except requests.exceptions.ConnectionError:
            return {
                "success": False,
                "message": "Connection refused - check if service is running",
                "suggestion": "Run 'docker compose ps' to check container status"
            }
        except requests.exceptions.Timeout:
            return {
                "success": False,
                "message": "Connection timeout - service may be overloaded",
                "suggestion": "Check 'docker compose logs' for errors"
            }
    
    async def test_health_endpoint(self) -> Dict:
        """Test health check endpoint."""
        try:
            response = requests.get(f"{self.base_url}/health", timeout=10)
            
            if response.status_code == 200:
                health_data = response.json()
                return {
                    "success": True,
                    "message": f"Health check passed: {health_data.get('status', 'OK')}",
                    "health_data": health_data,
                    "response_time": response.elapsed.total_seconds()
                }
            else:
                return {
                    "success": False,
                    "message": f"Health check failed: HTTP {response.status_code}",
                    "suggestion": "Check application logs for errors"
                }
        
        except Exception as e:
            return {
                "success": False,
                "message": f"Health check error: {e}",
                "suggestion": "Ensure /health endpoint is implemented"
            }
    
    async def test_basic_chat(self) -> Dict:
        """Test basic chat functionality."""
        test_message = {
            "message": "Hello, this is a test message",
            "conversation_id": f"diagnostic-{int(time.time())}"
        }
        
        try:
            start_time = time.time()
            response = requests.post(
                f"{self.base_url}/chat",
                json=test_message,
                timeout=30
            )
            response_time = time.time() - start_time
            
            if response.status_code == 200:
                response_data = response.json()
                return {
                    "success": True,
                    "message": "Basic chat functionality working",
                    "response_time": response_time,
                    "response_length": len(response_data.get("response", ""))
                }
            else:
                return {
                    "success": False,
                    "message": f"Chat request failed: HTTP {response.status_code}",
                    "response_body": response.text[:200]
                }
        
        except Exception as e:
            return {
                "success": False,
                "message": f"Chat test error: {e}",
                "suggestion": "Check if chat endpoint is properly configured"
            }
    
    async def test_agent_routing(self) -> Dict:
        """Test agent routing functionality."""
        routing_tests = [
            {
                "message": "Quick question: what is Python?",
                "expected_agent": "local",
                "description": "Simple query should route to local agent"
            },
            {
                "message": "Design a scalable microservices architecture for an e-commerce platform",
                "expected_agent": "cerebras",
                "description": "Complex query should route to Cerebras agent"
            }
        ]
        
        results = []
        
        for test in routing_tests:
            try:
                start_time = time.time()
                response = requests.post(
                    f"{self.base_url}/chat",
                    json={
                        "message": test["message"],
                        "conversation_id": f"routing-test-{int(time.time())}"
                    },
                    timeout=45
                )
                response_time = time.time() - start_time
                
                # Heuristic to detect which agent was used
                detected_agent = "cerebras" if response_time > 5 or len(response.json().get("response", "")) > 500 else "local"
                
                results.append({
                    "test": test["description"],
                    "expected": test["expected_agent"],
                    "detected": detected_agent,
                    "response_time": response_time,
                    "correct": detected_agent == test["expected_agent"]
                })
                
            except Exception as e:
                results.append({
                    "test": test["description"],
                    "error": str(e)
                })
        
        successful_routes = sum(1 for r in results if r.get("correct", False))
        
        return {
            "success": successful_routes > 0,
            "message": f"Routing tests: {successful_routes}/{len(results)} passed",
            "results": results
        }
    
    async def test_error_handling(self) -> Dict:
        """Test error handling and recovery."""
        error_tests = [
            {
                "test": "malformed_request",
                "data": "invalid json",
                "headers": {"Content-Type": "application/json"}
            },
            {
                "test": "empty_message",
                "data": {"message": "", "conversation_id": "test"}
            },
            {
                "test": "very_long_message",
                "data": {"message": "x" * 10000, "conversation_id": "test"}
            }
        ]
        
        results = []
        
        for test in error_tests:
            try:
                if isinstance(test["data"], str):
                    # Test malformed JSON
                    response = requests.post(
                        f"{self.base_url}/chat",
                        data=test["data"],
                        headers=test.get("headers", {}),
                        timeout=10
                    )
                else:
                    response = requests.post(
                        f"{self.base_url}/chat",
                        json=test["data"],
                        timeout=30
                    )
                
                # Good error handling should return 4xx status codes
                graceful_error = 400 <= response.status_code < 500
                
                results.append({
                    "test": test["test"],
                    "status_code": response.status_code,
                    "graceful_error": graceful_error
                })
                
            except Exception as e:
                results.append({
                    "test": test["test"],
                    "exception": str(e)
                })
        
        graceful_errors = sum(1 for r in results if r.get("graceful_error", False))
        
        return {
            "success": graceful_errors >= len(results) // 2,  # At least half should be graceful
            "message": f"Error handling: {graceful_errors}/{len(results)} tests handled gracefully",
            "results": results
        }
    
    async def test_performance(self) -> Dict:
        """Test basic performance metrics."""
        response_times = []
        
        # Run 5 performance tests
        for i in range(5):
            try:
                start_time = time.time()
                response = requests.post(
                    f"{self.base_url}/chat",
                    json={
                        "message": f"Performance test {i+1}: Hello DevDuck!",
                        "conversation_id": f"perf-test-{i}"
                    },
                    timeout=30
                )
                response_time = time.time() - start_time
                response_times.append(response_time)
                
            except Exception as e:
                response_times.append(30.0)  # Timeout value
        
        if response_times:
            avg_time = sum(response_times) / len(response_times)
            max_time = max(response_times)
            
            return {
                "success": avg_time < 15.0,  # Average should be under 15 seconds
                "message": f"Performance: avg {avg_time:.2f}s, max {max_time:.2f}s",
                "avg_response_time": avg_time,
                "max_response_time": max_time,
                "all_response_times": response_times
            }
        else:
            return {
                "success": False,
                "message": "Performance test failed - no successful responses"
            }
    
    async def test_concurrent_requests(self) -> Dict:
        """Test handling of concurrent requests."""
        import asyncio
        import aiohttp
        
        async def send_request(session, request_id):
            try:
                async with session.post(
                    f"{self.base_url}/chat",
                    json={
                        "message": f"Concurrent test {request_id}",
                        "conversation_id": f"concurrent-{request_id}"
                    },
                    timeout=30
                ) as response:
                    return {
                        "id": request_id,
                        "status": response.status,
                        "success": response.status == 200
                    }
            except Exception as e:
                return {
                    "id": request_id,
                    "error": str(e),
                    "success": False
                }
        
        # Send 5 concurrent requests
        async with aiohttp.ClientSession() as session:
            tasks = [send_request(session, i) for i in range(5)]
            results = await asyncio.gather(*tasks)
        
        successful_requests = sum(1 for r in results if r.get("success", False))
        
        return {
            "success": successful_requests >= 3,  # At least 3 out of 5 should succeed
            "message": f"Concurrent requests: {successful_requests}/5 successful",
            "results": results
        }
    
    def generate_summary(self, test_results: Dict) -> Dict:
        """Generate diagnostic summary."""
        total_tests = len(test_results)
        passed_tests = sum(1 for test in test_results.values() if test.get("success", False))
        
        # Determine overall health
        health_score = passed_tests / total_tests if total_tests > 0 else 0
        
        if health_score >= 0.8:
            overall_status = "healthy"
            status_emoji = "✅"
        elif health_score >= 0.6:
            overall_status = "warning"
            status_emoji = "⚠️"
        else:
            overall_status = "critical"
            status_emoji = "❌"
        
        # Generate recommendations
        recommendations = []
        
        for test_name, result in test_results.items():
            if not result.get("success", True):
                suggestion = result.get("suggestion")
                if suggestion:
                    recommendations.append(f"{test_name}: {suggestion}")
        
        return {
            "overall_status": overall_status,
            "status_emoji": status_emoji,
            "health_score": health_score,
            "tests_passed": passed_tests,
            "total_tests": total_tests,
            "recommendations": recommendations
        }
    
    def print_diagnostic_report(self, diagnostics: Dict):
        """Print formatted diagnostic report."""
        summary = diagnostics["summary"]
        
        print(f"\n{summary['status_emoji']} DIAGNOSTIC REPORT")
        print("=" * 25)
        print(f"Overall Status: {summary['overall_status'].upper()}")
        print(f"Health Score: {summary['health_score']:.1%}")
        print(f"Tests Passed: {summary['tests_passed']}/{summary['total_tests']}")
        
        if summary['recommendations']:
            print("\n🔧 Recommendations:")
            for i, rec in enumerate(summary['recommendations'], 1):
                print(f"  {i}. {rec}")
        
        print("\n📋 Detailed Results:")
        for test_name, result in diagnostics["tests"].items():
            status = "✅" if result.get("success") else "❌"
            print(f"  {status} {test_name}: {result.get('message', 'No message')}")

# Example usage
async def main():
    diagnostics = AgentDiagnostics()
    
    try:
        results = await diagnostics.run_full_diagnostics()
        diagnostics.print_diagnostic_report(results)
        
        # Save results to file
        with open("diagnostic_report.json", "w") as f:
            json.dump(results, f, indent=2)
        
        print("\n💾 Full diagnostic report saved to: diagnostic_report.json")
        
    except KeyboardInterrupt:
        print("\n🛑 Diagnostics interrupted by user")
    except Exception as e:
        print(f"\n❌ Diagnostics failed: {e}")

if __name__ == '__main__':
    asyncio.run(main())
```

#### Issue 3: Performance Problems

**Symptoms:**
- Slow response times
- High memory usage
- CPU spikes
- System freezing

**Performance Debugging:**

```bash
# Create performance debugging script
cat > debug-performance.sh << 'EOF'
#!/bin/bash

echo "⚡ Performance Debugging Tool"
echo "=============================="

# System resource check
echo "\n🖥️  System Resources:"
echo "CPU Usage:"
top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print "  " 100 - $1"%"}'

echo "Memory Usage:"
free -h | awk 'NR==2{printf "  Used: %s/%s (%.2f%%)\n", $3, $2, $3*100/$2}'

echo "Disk Usage:"
df -h . | awk 'NR==2{printf "  Used: %s/%s (%s)\n", $3, $2, $5}'

# Docker resource usage
echo "\n🐳 Docker Resource Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}\t{{.NetIO}}\t{{.BlockIO}}"

# Container performance analysis
echo "\n📊 Container Performance Analysis:"
for container in $(docker ps --format "{{.Names}}" | grep devduck); do
    echo "\n📦 $container:"
    
    # Get container stats
    STATS=$(docker stats $container --no-stream --format "{{.CPUPerc}} {{.MemUsage}}")
    echo "  Resource Usage: $STATS"
    
    # Check container health
    HEALTH=$(docker inspect $container --format="{{.State.Health.Status}}" 2>/dev/null || echo "no healthcheck")
    echo "  Health: $HEALTH"
    
    # Check restart count
    RESTARTS=$(docker inspect $container --format="{{.RestartCount}}" 2>/dev/null || echo "0")
    echo "  Restarts: $RESTARTS"
    
    # Check log for errors
    ERROR_COUNT=$(docker logs $container --since 1h 2>&1 | grep -i error | wc -l)
    echo "  Recent Errors: $ERROR_COUNT"
done

# Performance recommendations
echo "\n💡 Performance Recommendations:"

# Check available memory
MEM_AVAILABLE=$(free -m | awk 'NR==2{print $7}')
if [ "$MEM_AVAILABLE" -lt 2000 ]; then
    echo "  ⚠️  Low memory: Consider using smaller models or increasing system RAM"
fi

# Check CPU usage
CPU_USAGE=$(top -bn1 | grep "Cpu(s)" | sed "s/.*, *\([0-9.]*\)%* id.*/\1/" | awk '{print 100 - $1}')
if (( $(echo "$CPU_USAGE > 80" | bc -l) )); then
    echo "  ⚠️  High CPU usage: Consider scaling down or optimizing workload"
fi

# Check disk I/O
IO_WAIT=$(top -bn1 | grep "Cpu(s)" | grep -o '[0-9.]*%wa' | sed 's/%wa//')
if (( $(echo "$IO_WAIT > 10" | bc -l) )); then
    echo "  ⚠️  High I/O wait: Check disk performance and model loading"
fi

echo "\n🔧 Quick Performance Fixes:"
echo "1. Restart containers: docker compose restart"
echo "2. Clean Docker cache: docker system prune -f"
echo "3. Use smaller model: LOCAL_MODEL_NAME=microsoft/DialoGPT-small"
echo "4. Increase memory limits in compose.yml"
echo "5. Monitor with: watch docker stats"

EOF

chmod +x debug-performance.sh
./debug-performance.sh
```

## Advanced Debugging Techniques

### 🔍 Deep Debugging Tools

#### Container Introspection

```bash
# Create debugging toolkit
cat > debug-toolkit.sh << 'EOF'
#!/bin/bash

echo "🛠️  DevDuck Debug Toolkit"
echo "========================"

# Function to debug a specific container
debug_container() {
    local container_name="$1"
    echo "\n🔍 Debugging container: $container_name"
    
    if ! docker ps | grep -q "$container_name"; then
        echo "❌ Container $container_name not found or not running"
        return 1
    fi
    
    echo "\n📋 Container Info:"
    docker inspect "$container_name" --format="{{.State.Status}}: {{.State.StartedAt}}"
    
    echo "\n📊 Resource Usage:"
    docker stats "$container_name" --no-stream --format "CPU: {{.CPUPerc}}, Memory: {{.MemUsage}}"
    
    echo "\n📝 Recent Logs (last 20 lines):"
    docker logs "$container_name" --tail 20 --timestamps
    
    echo "\n🌐 Network Info:"
    docker exec "$container_name" netstat -tuln 2>/dev/null | head -10 || echo "netstat not available"
    
    echo "\n💾 Disk Usage:"
    docker exec "$container_name" df -h 2>/dev/null | head -5 || echo "df not available"
    
    echo "\n🔄 Processes:"
    docker exec "$container_name" ps aux 2>/dev/null | head -10 || echo "ps not available"
    
    echo "\n🔗 Environment Variables:"
    docker exec "$container_name" env | grep -E "(CEREBRAS|LOCAL|DEBUG|LOG)" | sort
}

# Function to test connectivity between containers
test_connectivity() {
    echo "\n🌐 Testing Inter-Container Connectivity"
    echo "======================================"
    
    local source_container="devduck-agent"
    local target_services=("mcp-gateway:3000" "postgres:5432" "redis:6379")
    
    if docker ps | grep -q "$source_container"; then
        for target in "${target_services[@]}"; do
            echo "\n🔍 Testing $source_container -> $target"
            
            # Try to connect
            if docker exec "$source_container" nc -z ${target/:/ } 2>/dev/null; then
                echo "✅ Connection successful"
            else
                echo "❌ Connection failed"
                
                # Additional diagnostics
                host=$(echo $target | cut -d: -f1)
                if docker exec "$source_container" nslookup "$host" 2>/dev/null >/dev/null; then
                    echo "   DNS resolution: ✅"
                else
                    echo "   DNS resolution: ❌"
                fi
            fi
        done
    else
        echo "❌ Source container $source_container not found"
    fi
}

# Function to check API endpoints
test_api_endpoints() {
    echo "\n🚀 Testing API Endpoints"
    echo "========================"
    
    local base_url="http://localhost:8000"
    local endpoints=("/health" "/metrics" "/dev-ui")
    
    for endpoint in "${endpoints[@]}"; do
        echo "\n🔍 Testing $endpoint"
        
        response=$(curl -s -w "HTTPSTATUS:%{http_code};TIME:%{time_total}" "$base_url$endpoint" || echo "HTTPSTATUS:000;TIME:timeout")
        
        http_code=$(echo "$response" | grep -o "HTTPSTATUS:[0-9]*" | cut -d: -f2)
        time_total=$(echo "$response" | grep -o "TIME:[0-9.]*" | cut -d: -f2)
        
        if [ "$http_code" = "200" ]; then
            echo "✅ HTTP $http_code (${time_total}s)"
        elif [ "$http_code" = "000" ]; then
            echo "❌ Connection failed (timeout)"
        else
            echo "⚠️  HTTP $http_code (${time_total}s)"
        fi
    done
}

# Function to analyze logs for common issues
analyze_logs() {
    echo "\n📊 Log Analysis"
    echo "==============="
    
    local container_name="$1"
    
    if ! docker ps | grep -q "$container_name"; then
        echo "❌ Container $container_name not found"
        return 1
    fi
    
    echo "\n🔍 Analyzing logs for $container_name..."
    
    # Get logs from last hour
    local logs=$(docker logs "$container_name" --since 1h 2>&1)
    
    # Count different log levels
    local error_count=$(echo "$logs" | grep -i error | wc -l)
    local warning_count=$(echo "$logs" | grep -i warning | wc -l)
    local info_count=$(echo "$logs" | grep -i info | wc -l)
    
    echo "Log Summary (last hour):"
    echo "  Errors: $error_count"
    echo "  Warnings: $warning_count"
    echo "  Info: $info_count"
    
    if [ "$error_count" -gt 0 ]; then
        echo "\n❌ Recent Errors:"
        echo "$logs" | grep -i error | tail -3
    fi
    
    if [ "$warning_count" -gt 5 ]; then
        echo "\n⚠️  Recent Warnings:"
        echo "$logs" | grep -i warning | tail -3
    fi
    
    # Check for common issues
    if echo "$logs" | grep -qi "out of memory"; then
        echo "\n💾 Memory Issue Detected!"
        echo "   Recommendation: Increase memory limits or use smaller models"
    fi
    
    if echo "$logs" | grep -qi "connection refused"; then
        echo "\n🌐 Connection Issue Detected!"
        echo "   Recommendation: Check service dependencies and networking"
    fi
    
    if echo "$logs" | grep -qi "timeout"; then
        echo "\n⏱️  Timeout Issue Detected!"
        echo "   Recommendation: Increase timeout values or check performance"
    fi
}

# Main execution
case "${1:-all}" in
    "container")
        debug_container "${2:-devduck-agent}"
        ;;
    "connectivity")
        test_connectivity
        ;;
    "api")
        test_api_endpoints
        ;;
    "logs")
        analyze_logs "${2:-devduck-agent}"
        ;;
    "all")
        debug_container "devduck-agent"
        test_connectivity
        test_api_endpoints
        analyze_logs "devduck-agent"
        ;;
    *)
        echo "Usage: $0 [container|connectivity|api|logs|all] [container_name]"
        echo "\nExamples:"
        echo "  $0 container devduck-agent"
        echo "  $0 connectivity"
        echo "  $0 api"
        echo "  $0 logs devduck-agent"
        echo "  $0 all"
        ;;
esac

EOF

chmod +x debug-toolkit.sh
echo "🛠️  Debug toolkit created! Usage: ./debug-toolkit.sh [option]"
```

## Learning Resources & Next Steps

### 📚 Recommended Learning Path

#### Beginner to Intermediate (3-6 months)

```markdown
## 🎯 Learning Roadmap: Multi-Agent Systems Mastery

### Phase 1: Foundation Reinforcement (Month 1)
- [ ] **Docker Mastery**
  - Advanced Docker Compose patterns
  - Multi-stage builds and optimization
  - Docker Swarm for production orchestration
  - Security best practices and scanning

- [ ] **Python Advanced Patterns**
  - Async/await and concurrent programming
  - Design patterns for distributed systems
  - Error handling and resilience patterns
  - Performance profiling and optimization

- [ ] **API Design & Development**
  - FastAPI advanced features
  - WebSocket implementation for real-time communication
  - GraphQL for flexible data queries
  - API versioning and documentation

### Phase 2: AI & ML Integration (Month 2-3)
- [ ] **Language Models Deep Dive**
  - Transformer architecture understanding
  - Fine-tuning techniques for specific tasks
  - Prompt engineering advanced techniques
  - Model quantization and optimization

- [ ] **Vector Databases & RAG**
  - Implement RAG (Retrieval Augmented Generation)
  - Vector similarity search with Pinecone/Weaviate
  - Document parsing and chunking strategies
  - Semantic search and embeddings

- [ ] **Multi-Modal AI**
  - Image processing with CLIP and DALL-E
  - Audio processing for voice interfaces
  - Document analysis and OCR integration
  - Video processing capabilities

### Phase 3: Production Systems (Month 4-5)
- [ ] **Kubernetes & Orchestration**
  - Migrate from Docker Compose to Kubernetes
  - Helm charts for application deployment
  - Service mesh with Istio
  - Auto-scaling and resource management

- [ ] **Observability & Monitoring**
  - Distributed tracing with Jaeger
  - Custom metrics and alerting
  - Log aggregation with ELK stack
  - Performance monitoring and APM

- [ ] **Security & Compliance**
  - OAuth 2.0 and OpenID Connect
  - API rate limiting and DDoS protection
  - Data encryption and key management
  - Compliance frameworks (SOC 2, GDPR)

### Phase 4: Advanced Architectures (Month 6)
- [ ] **Event-Driven Architecture**
  - Apache Kafka for message streaming
  - Event sourcing and CQRS patterns
  - Saga pattern for distributed transactions
  - Real-time data processing

- [ ] **Advanced AI Patterns**
  - Multi-agent reinforcement learning
  - Agent communication protocols
  - Distributed AI training
  - Edge AI deployment

- [ ] **Emerging Technologies**
  - WebAssembly for performance
  - Serverless architectures
  - Edge computing integration
  - Quantum computing applications
```

#### Intermediate to Expert (6-12 months)

```markdown
### Phase 5: Research & Innovation (Month 7-9)
- [ ] **Research Implementation**
  - Stay current with AI research papers
  - Implement cutting-edge techniques
  - Contribute to open-source projects
  - Write technical blog posts and papers

- [ ] **Custom Model Development**
  - Build domain-specific language models
  - Implement custom training pipelines
  - Distributed training strategies
  - Model serving at scale

- [ ] **Advanced Multi-Agent Systems**
  - Hierarchical multi-agent architectures
  - Agent negotiation and coordination
  - Swarm intelligence implementation
  - Game theory applications

### Phase 6: Leadership & Architecture (Month 10-12)
- [ ] **System Architecture Design**
  - Design patterns for AI systems
  - Scalability and performance optimization
  - Failure mode analysis and recovery
  - Cost optimization strategies

- [ ] **Team Leadership**
  - Technical mentoring and guidance
  - Project management for AI initiatives
  - Stakeholder communication
  - Technical decision making

- [ ] **Business Integration**
  - ROI measurement for AI projects
  - Ethical AI implementation
  - Regulatory compliance
  - Product strategy for AI features
```

### 🔗 Essential Resources

#### Books
1. **"Designing Data-Intensive Applications"** by Martin Kleppmann
2. **"Building Microservices"** by Sam Newman
3. **"The Hundred-Page Machine Learning Book"** by Andriy Burkov
4. **"Multi-Agent Systems: Algorithmic, Game-Theoretic, and Logical Foundations"** by Yoav Shoham
5. **"Patterns of Enterprise Application Architecture"** by Martin Fowler

#### Online Courses
1. **Multi-Agent Systems** (University of Edinburgh - Coursera)
2. **Advanced Machine Learning Specialization** (HSE University - Coursera)
3. **Microservices Patterns** (Chris Richardson)
4. **Kubernetes for Developers** (Cloud Native Computing Foundation)
5. **System Design Interview** (Educative.io)

#### Communities & Forums
1. **AI/ML Communities**: Hugging Face, Papers with Code, /r/MachineLearning
2. **DevOps Communities**: DevOps.com, CNCF Slack, Docker Community
3. **Multi-Agent Systems**: AAMAS Conference, Multi-Agent.org
4. **General Tech**: Stack Overflow, Hacker News, Dev.to

#### Tools & Platforms to Explore
1. **Orchestration**: Kubernetes, Docker Swarm, Nomad
2. **AI Platforms**: MLflow, Kubeflow, Weights & Biases
3. **Monitoring**: Prometheus, Grafana, Datadog, New Relic
4. **Message Queues**: Apache Kafka, RabbitMQ, Redis Streams
5. **Databases**: PostgreSQL, MongoDB, Vector DBs (Pinecone, Weaviate)

### 🚀 Project Ideas for Practice

#### Beginner Projects
1. **Multi-Language Code Assistant**
   - Support multiple programming languages
   - Code generation, review, and optimization
   - Integration with VS Code extension

2. **Smart Document Processor**
   - PDF/Word document analysis
   - Automatic summarization and Q&A
   - Multi-format output generation

3. **Customer Service Agent System**
   - Multi-channel support (chat, email, voice)
   - Sentiment analysis and routing
   - Knowledge base integration

#### Intermediate Projects
1. **Autonomous DevOps Agent**
   - Infrastructure monitoring and self-healing
   - Automated deployment and rollback
   - Performance optimization recommendations

2. **Multi-Modal Content Creator**
   - Text-to-image and image-to-text processing
   - Video content analysis and generation
   - Social media content optimization

3. **Distributed AI Training Platform**
   - Multi-node training coordination
   - Resource allocation and scheduling
   - Model versioning and deployment

#### Advanced Projects
1. **Autonomous Research Assistant**
   - Scientific paper analysis and summarization
   - Hypothesis generation and testing
   - Experimental design suggestions

2. **Smart City Management System**
   - Traffic optimization with multiple agents
   - Resource allocation (energy, water)
   - Predictive maintenance scheduling

3. **Financial Trading Agent Network**
   - Market analysis and prediction
   - Risk assessment and portfolio management
   - Real-time trading decision making

## Contribution & Community

### 🤝 How to Contribute

This workshop is open-source and welcomes contributions:

#### Ways to Contribute
1. **Bug Reports**: Found an issue? Report it on GitHub
2. **Feature Requests**: Suggest new agents or capabilities
3. **Documentation**: Improve tutorials and examples
4. **Code Contributions**: Submit pull requests with improvements
5. **Community Support**: Help others in discussions

#### Contribution Guidelines
```bash
# Fork the repository
git fork https://github.com/ajeetraina/docker-cerebras-labspace

# Create a feature branch
git checkout -b feature/your-feature-name

# Make your changes
# - Add comprehensive tests
# - Update documentation
# - Follow coding standards

# Commit with descriptive messages
git commit -m "Add: New specialized agent for data analysis"

# Push and create pull request
git push origin feature/your-feature-name
```

### 🌟 Success Stories

Share your success stories and use cases:

- **Enterprise Deployments**: How you used DevDuck in production
- **Educational Use**: Teaching multi-agent systems
- **Research Applications**: Academic or commercial research
- **Creative Projects**: Unique applications and innovations

### 📬 Stay Connected

- **GitHub**: Follow the repository for updates
- **Docker Hub**: Check for new container images
- **Community Discussions**: Participate in GitHub Discussions
- **Social Media**: Share your projects with #DevDuckMultiAgent

## Workshop Completion Certificate

### 🏆 Congratulations!

You have successfully completed the **Docker DevDuck Multi-Agent Workshop**!

#### What You've Accomplished:
- ✅ **System Architecture**: Designed and deployed multi-agent systems
- ✅ **Container Orchestration**: Mastered Docker Compose for AI applications
- ✅ **Agent Development**: Built local and cloud-based AI agents
- ✅ **Inter-Agent Communication**: Implemented routing and coordination
- ✅ **Production Deployment**: Applied enterprise-grade deployment practices
- ✅ **Monitoring & Optimization**: Set up comprehensive observability
- ✅ **Security Implementation**: Applied security best practices
- ✅ **Performance Tuning**: Optimized system performance and scalability
- ✅ **Troubleshooting Skills**: Developed debugging and problem-solving abilities
- ✅ **Advanced Features**: Explored cutting-edge multi-agent capabilities

#### Skills Gained:
- Multi-agent system design and architecture
- Docker containerization for AI applications
- API development and integration
- Cloud AI service integration (Cerebras)
- Production monitoring and alerting
- Performance optimization techniques
- Security implementation and best practices
- Troubleshooting and debugging methodologies

### 🎯 Your Next Challenge

Choose your next adventure:

1. **Build a Production System**: Deploy DevDuck for a real-world use case
2. **Contribute to Open Source**: Improve the workshop or create new agents
3. **Research and Innovation**: Implement cutting-edge AI research
4. **Teaching and Mentoring**: Share your knowledge with others
5. **Enterprise Integration**: Apply multi-agent patterns in your organization

### 📜 Certificate Information

```
🏆 CERTIFICATE OF COMPLETION

Docker DevDuck Multi-Agent Workshop

This certifies that you have successfully completed
the comprehensive multi-agent systems workshop,
demonstrating proficiency in:

• Multi-Agent System Architecture & Design
• Docker Containerization for AI Applications  
• Production Deployment & Monitoring
• Performance Optimization & Security
• Advanced Troubleshooting & Debugging

Completed: [Current Date]
Workshop Version: 1.0
Instructor: Ajeet S Raina, Docker Captain

"The future belongs to those who can orchestrate
intelligence across distributed systems."
```

---

## Final Words

Thank you for completing the Docker DevDuck Multi-Agent Workshop! You've gained valuable skills in building, deploying, and managing sophisticated AI systems using modern containerization practices.

The field of multi-agent systems is rapidly evolving, and the foundation you've built here will serve you well as you continue to innovate and build the next generation of intelligent applications.

Remember: The best way to master these concepts is through continuous practice and experimentation. Keep building, keep learning, and don't hesitate to contribute back to the community.

**Happy Building!** 🚀🐋🤖

---

!!! success "Workshop Complete! 🎉"
    You are now equipped with the knowledge and skills to build production-grade multi-agent systems. The journey doesn't end here – it's just the beginning of your adventure in intelligent system orchestration!