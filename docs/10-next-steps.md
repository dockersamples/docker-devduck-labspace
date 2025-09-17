# Troubleshooting & Next Steps

Congratulations on completing the Docker DevDuck Multi-Agent Workshop! This final lab covers comprehensive troubleshooting techniques, common issues and solutions, performance optimization, and exciting possibilities for extending your multi-agent system.

!!! info "Workshop Completion"
    This section consolidates your learning with practical troubleshooting skills and provides a roadmap for continued development.

## Comprehensive Troubleshooting Guide

### 🔧 Common Issues and Solutions

#### Issue 1: Agent Communication Failures

**Symptoms:**
- Agents not responding to requests
- Timeout errors in logs
- "Agent unavailable" messages

**Diagnostic Steps:**

```bash
# Create comprehensive diagnostics script
cat > diagnose_agents.sh << 'EOF'
#!/bin/bash

echo "🔍 DevDuck Multi-Agent System Diagnostics"
echo "=========================================="

# 1. Check container status
echo "\n📦 Container Status:"
docker compose ps

# 2. Check container logs for errors
echo "\n📋 Recent Error Logs:"
docker compose logs --tail=50 devduck-agent | grep -i "error\|exception\|failed\|timeout"

# 3. Test network connectivity
echo "\n🌐 Network Connectivity:"
docker compose exec devduck-agent ping -c 3 mcp-gateway 2>/dev/null && echo "✅ MCP Gateway reachable" || echo "❌ MCP Gateway unreachable"

# 4. Check API endpoints
echo "\n🔗 API Health Checks:"
curl -s -f http://localhost:8000/health > /dev/null && echo "✅ Main API healthy" || echo "❌ Main API unhealthy"

# 5. Check resource usage
echo "\n💾 Resource Usage:"
docker stats --no-stream --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}\t{{.MemPerc}}"

# 6. Check disk space
echo "\n💿 Disk Space:"
df -h | head -1
df -h | grep -E "(/$|/var|/tmp)"

# 7. Test external API connectivity
echo "\n🌍 External API Connectivity:"
curl -s -I https://api.cerebras.ai > /dev/null && echo "✅ Cerebras API reachable" || echo "❌ Cerebras API unreachable"

# 8. Check environment variables
echo "\n🔑 Environment Configuration:"
docker compose exec devduck-agent env | grep -E "(CEREBRAS|LOCAL_MODEL|DEBUG|LOG_LEVEL)" | sed 's/CEREBRAS_API_KEY=.*/CEREBRAS_API_KEY=***REDACTED***/'

echo "\n📊 Diagnostic Complete!"
EOF

chmod +x diagnose_agents.sh
./diagnose_agents.sh
```

**Solutions:**

```bash
# Fix network issues
docker network prune
docker compose down && docker compose up -d

# Restart individual services
docker compose restart devduck-agent
docker compose restart mcp-gateway

# Check and fix DNS resolution
echo "127.0.0.1 localhost" | docker compose exec -T devduck-agent tee -a /etc/hosts

# Rebuild containers if code changes
docker compose up --build --force-recreate
```

#### Issue 2: Performance Problems

**Symptoms:**
- Slow response times (>30 seconds)
- High memory usage
- Agent timeouts

**Performance Analysis Script:**

```python
# Create performance_analyzer.py
import requests
import time
import statistics
import psutil
import docker
from concurrent.futures import ThreadPoolExecutor, as_completed

class PerformanceAnalyzer:
    def __init__(self):
        self.docker_client = docker.from_env()
        self.base_url = "http://localhost:8000"
        self.results = []
    
    def run_performance_test(self, num_requests=10, concurrent_requests=3):
        """Run comprehensive performance test."""
        print(f"🚀 Running Performance Test: {num_requests} requests, {concurrent_requests} concurrent")
        print("=" * 60)
        
        # Test scenarios
        test_scenarios = [
            {"query": "Hello", "expected_agent": "local", "category": "simple"},
            {"query": "What is Python?", "expected_agent": "local", "category": "simple"},
            {"query": "Explain machine learning concepts in detail", "expected_agent": "cerebras", "category": "complex"},
            {"query": "Design a scalable microservices architecture", "expected_agent": "cerebras", "category": "complex"}
        ]
        
        # System baseline
        baseline_stats = self._get_system_stats()
        print(f"📊 Baseline - CPU: {baseline_stats['cpu']:.1f}%, Memory: {baseline_stats['memory']:.1f}%")
        
        # Run tests
        start_time = time.time()
        
        with ThreadPoolExecutor(max_workers=concurrent_requests) as executor:
            futures = []
            
            for i in range(num_requests):
                scenario = test_scenarios[i % len(test_scenarios)]
                future = executor.submit(self._make_request, scenario, i)
                futures.append(future)
            
            # Collect results
            for future in as_completed(futures):
                result = future.result()
                if result:
                    self.results.append(result)
        
        total_time = time.time() - start_time
        
        # Analyze results
        self._analyze_results(total_time, baseline_stats)
    
    def _make_request(self, scenario, request_id):
        """Make a single request and measure performance."""
        start_time = time.time()
        
        try:
            response = requests.post(
                f"{self.base_url}/chat",
                json={
                    "message": scenario["query"],
                    "conversation_id": f"perf-test-{request_id}"
                },
                timeout=45
            )
            
            duration = time.time() - start_time
            
            if response.status_code == 200:
                response_data = response.json()
                return {
                    "request_id": request_id,
                    "scenario": scenario["category"],
                    "duration": duration,
                    "response_length": len(response_data.get("response", "")),
                    "success": True,
                    "status_code": response.status_code
                }
            else:
                return {
                    "request_id": request_id,
                    "scenario": scenario["category"],
                    "duration": duration,
                    "success": False,
                    "status_code": response.status_code
                }
                
        except Exception as e:
            return {
                "request_id": request_id,
                "scenario": scenario["category"],
                "duration": time.time() - start_time,
                "success": False,
                "error": str(e)
            }
    
    def _get_system_stats(self):
        """Get current system statistics."""
        try:
            return {
                "cpu": psutil.cpu_percent(interval=1),
                "memory": psutil.virtual_memory().percent,
                "disk": psutil.disk_usage('/').percent
            }
        except:
            return {"cpu": 0, "memory": 0, "disk": 0}
    
    def _analyze_results(self, total_time, baseline_stats):
        """Analyze performance test results."""
        if not self.results:
            print("❌ No results to analyze")
            return
        
        # Success rate
        successful = [r for r in self.results if r["success"]]
        success_rate = len(successful) / len(self.results)
        
        # Response times
        response_times = [r["duration"] for r in successful]
        
        if response_times:
            avg_response_time = statistics.mean(response_times)
            median_response_time = statistics.median(response_times)
            min_response_time = min(response_times)
            max_response_time = max(response_times)
            
            # Percentiles
            p90_response_time = statistics.quantiles(response_times, n=10)[8] if len(response_times) >= 10 else max_response_time
            p95_response_time = statistics.quantiles(response_times, n=20)[18] if len(response_times) >= 20 else max_response_time
        else:
            avg_response_time = median_response_time = min_response_time = max_response_time = 0
            p90_response_time = p95_response_time = 0
        
        # Throughput
        throughput = len(self.results) / total_time
        
        # Category breakdown
        category_stats = {}
        for result in successful:
            category = result["scenario"]
            if category not in category_stats:
                category_stats[category] = []
            category_stats[category].append(result["duration"])
        
        # Final system stats
        final_stats = self._get_system_stats()
        
        # Print results
        print("\n📈 Performance Test Results")
        print("=" * 30)
        print(f"Total Requests: {len(self.results)}")
        print(f"Successful Requests: {len(successful)}")
        print(f"Success Rate: {success_rate:.1%}")
        print(f"Total Time: {total_time:.2f}s")
        print(f"Throughput: {throughput:.2f} req/s")
        
        print("\n⏱️ Response Time Statistics:")
        print(f"  Average: {avg_response_time:.2f}s")
        print(f"  Median: {median_response_time:.2f}s")
        print(f"  Min: {min_response_time:.2f}s")
        print(f"  Max: {max_response_time:.2f}s")
        print(f"  90th Percentile: {p90_response_time:.2f}s")
        print(f"  95th Percentile: {p95_response_time:.2f}s")
        
        print("\n📊 Performance by Category:")
        for category, times in category_stats.items():
            avg_time = statistics.mean(times)
            print(f"  {category.title()}: {avg_time:.2f}s avg ({len(times)} requests)")
        
        print("\n💻 System Resource Usage:")
        print(f"  Baseline CPU: {baseline_stats['cpu']:.1f}% → Final: {final_stats['cpu']:.1f}%")
        print(f"  Baseline Memory: {baseline_stats['memory']:.1f}% → Final: {final_stats['memory']:.1f}%")
        
        # Performance recommendations
        print("\n💡 Performance Recommendations:")
        if avg_response_time > 10:
            print("  ⚠️  High average response time - consider scaling up")
        if success_rate < 0.95:
            print("  ⚠️  Low success rate - check error handling")
        if final_stats['cpu'] > 80:
            print("  ⚠️  High CPU usage - consider horizontal scaling")
        if final_stats['memory'] > 85:
            print("  ⚠️  High memory usage - check for memory leaks")
        if avg_response_time < 3 and success_rate > 0.98:
            print("  ✅ Excellent performance - system is well optimized")
    
    def generate_performance_report(self, filename="performance_report.json"):
        """Generate detailed performance report."""
        import json
        
        successful = [r for r in self.results if r["success"]]
        response_times = [r["duration"] for r in successful]
        
        report = {
            "test_summary": {
                "total_requests": len(self.results),
                "successful_requests": len(successful),
                "success_rate": len(successful) / max(1, len(self.results)),
                "test_timestamp": time.time()
            },
            "performance_metrics": {
                "avg_response_time": statistics.mean(response_times) if response_times else 0,
                "median_response_time": statistics.median(response_times) if response_times else 0,
                "min_response_time": min(response_times) if response_times else 0,
                "max_response_time": max(response_times) if response_times else 0
            },
            "detailed_results": self.results,
            "system_info": self._get_system_stats()
        }
        
        with open(filename, 'w') as f:
            json.dump(report, f, indent=2)
        
        print(f"📄 Performance report saved to {filename}")

# Run performance analysis
if __name__ == '__main__':
    analyzer = PerformanceAnalyzer()
    analyzer.run_performance_test(num_requests=20, concurrent_requests=5)
    analyzer.generate_performance_report()
```

#### Issue 3: Model Loading Problems

**Symptoms:**
- "Model not found" errors
- Container crashes during startup
- Out of memory errors

**Model Management Script:**

```bash
# Create model_manager.sh
cat > model_manager.sh << 'EOF'
#!/bin/bash

echo "🤖 DevDuck Model Management Tool"
echo "=============================="

# Configuration
MODEL_CACHE_DIR="/tmp/devduck_models"
DOCKER_VOLUME="devduck-model-cache"

# Function to check model status
check_model_status() {
    echo "\n📋 Current Model Status:"
    
    # Check local model cache
    echo "Local Model Cache:"
    if docker volume inspect $DOCKER_VOLUME >/dev/null 2>&1; then
        echo "  ✅ Docker volume exists: $DOCKER_VOLUME"
        docker run --rm -v $DOCKER_VOLUME:/models alpine du -sh /models/* 2>/dev/null || echo "  📁 Volume is empty"
    else
        echo "  ❌ Docker volume missing: $DOCKER_VOLUME"
    fi
    
    # Check available system resources
    echo "\nSystem Resources:"
    echo "  Available Memory: $(free -h | awk '/^Mem:/{print $7}')" 
    echo "  Available Disk: $(df -h / | awk 'NR==2{print $4}')"
    
    # Check model configuration
    echo "\nModel Configuration:"
    docker compose exec devduck-agent env | grep -E "LOCAL_MODEL|MODEL_CACHE" || echo "  ⚠️  No model configuration found"
}

# Function to download models
download_models() {
    echo "\n⬇️  Downloading Models..."
    
    # Create model cache directory
    mkdir -p $MODEL_CACHE_DIR
    
    # Download lightweight models for testing
    cat > download_models.py << 'PYEOF'
import os
from transformers import AutoTokenizer, AutoModelForCausalLM

models_to_download = [
    'microsoft/DialoGPT-small',
    'microsoft/DialoGPT-medium',
    'gpt2'
]

cache_dir = '/tmp/devduck_models'
os.makedirs(cache_dir, exist_ok=True)

for model_name in models_to_download:
    try:
        print(f"Downloading {model_name}...")
        tokenizer = AutoTokenizer.from_pretrained(model_name, cache_dir=cache_dir)
        model = AutoModelForCausalLM.from_pretrained(model_name, cache_dir=cache_dir)
        print(f"✅ {model_name} downloaded successfully")
    except Exception as e:
        print(f"❌ Failed to download {model_name}: {e}")

print("\n📊 Model Download Summary:")
for root, dirs, files in os.walk(cache_dir):
    size = sum(os.path.getsize(os.path.join(root, file)) for file in files)
    if size > 0:
        print(f"  {os.path.basename(root)}: {size / (1024*1024*1024):.2f} GB")
PYEOF
    
    python3 download_models.py
    
    # Copy to Docker volume
    echo "\n📦 Copying models to Docker volume..."
    docker run --rm -v $MODEL_CACHE_DIR:/source -v $DOCKER_VOLUME:/dest alpine cp -r /source/* /dest/
    
    echo "✅ Models downloaded and cached"
}

# Function to optimize models
optimize_models() {
    echo "\n⚡ Optimizing Model Performance..."
    
    # Set optimal model configuration
    cat > .env.model-optimized << 'ENVEOF'
# Optimized model configuration
LOCAL_MODEL_NAME=microsoft/DialoGPT-small
LOCAL_MODEL_DEVICE=auto
LOCAL_MODEL_PRECISION=fp16
LOCAL_MODEL_MAX_LENGTH=512
TORCH_HOME=/app/models
HF_HOME=/app/models
TRANSFORMERS_CACHE=/app/models
ENVEOF
    
    echo "✅ Model optimization configuration created (.env.model-optimized)"
    echo "📝 To apply: cp .env.model-optimized .env && docker compose up --build"
}

# Function to troubleshoot model issues
troubleshoot_models() {
    echo "\n🔧 Model Troubleshooting:"
    
    # Check for common issues
    echo "\n1. Checking model loading errors..."
    docker compose logs devduck-agent | grep -i "model\|loading\|download" | tail -10
    
    echo "\n2. Checking memory usage during model loading..."
    docker stats --no-stream devduck-agent 2>/dev/null || echo "   Container not running"
    
    echo "\n3. Testing model loading directly..."
    docker compose exec devduck-agent python3 -c "
import torch
print(f'PyTorch version: {torch.__version__}')
print(f'CUDA available: {torch.cuda.is_available()}')
print(f'CUDA devices: {torch.cuda.device_count()}')
try:
    from transformers import AutoTokenizer
    tokenizer = AutoTokenizer.from_pretrained('gpt2')
    print('✅ Model loading test successful')
except Exception as e:
    print(f'❌ Model loading test failed: {e}')
" 2>/dev/null || echo "   Cannot test - container not accessible"
    
    echo "\n4. Common Solutions:"
    echo "   - Use smaller models (DialoGPT-small instead of DialoGPT-large)"
    echo "   - Increase Docker memory allocation to 8GB+"
    echo "   - Enable model caching to avoid re-downloading"
    echo "   - Use fp16 precision to reduce memory usage"
    echo "   - Check internet connectivity for model downloads"
}

# Main menu
case "$1" in
    "status")
        check_model_status
        ;;
    "download")
        download_models
        ;;
    "optimize")
        optimize_models
        ;;
    "troubleshoot")
        troubleshoot_models
        ;;
    *)
        echo "Usage: $0 {status|download|optimize|troubleshoot}"
        echo ""
        echo "Commands:"
        echo "  status       - Check current model status"
        echo "  download     - Download and cache models"
        echo "  optimize     - Create optimized model configuration"
        echo "  troubleshoot - Diagnose model loading issues"
        ;;
esac

EOF

chmod +x model_manager.sh

# Test the model manager
echo "🧪 Testing model management:"
./model_manager.sh status
```

### 🚨 Emergency Recovery Procedures

#### Complete System Reset

```bash
# Create emergency_reset.sh
cat > emergency_reset.sh << 'EOF'
#!/bin/bash

echo "🚨 DevDuck Emergency System Reset"
echo "================================"
echo "⚠️  WARNING: This will destroy all containers, volumes, and cached data!"
read -p "Are you sure you want to continue? (type 'YES' to confirm): " confirmation

if [ "$confirmation" != "YES" ]; then
    echo "❌ Reset cancelled"
    exit 1
fi

echo "\n🗑️  Stopping and removing all containers..."
docker compose down -v --remove-orphans

echo "\n🧹 Cleaning up Docker resources..."
docker system prune -af --volumes

echo "\n📦 Rebuilding from scratch..."
docker compose up --build --force-recreate -d

echo "\n⏳ Waiting for services to start..."
sleep 30

echo "\n🔍 Checking system health..."
docker compose ps
curl -f http://localhost:8000/health && echo "\n✅ System reset successful!" || echo "\n❌ System still having issues"

EOF

chmod +x emergency_reset.sh
```

#### Backup and Restore

```bash
# Create backup_restore.sh
cat > backup_restore.sh << 'EOF'
#!/bin/bash

BACKUP_DIR="./backups/$(date +%Y%m%d_%H%M%S)"

backup_system() {
    echo "💾 Creating system backup..."
    mkdir -p "$BACKUP_DIR"
    
    # Backup configuration
    cp .env compose.yml labspace.yaml "$BACKUP_DIR/" 2>/dev/null || echo "Some config files not found"
    
    # Backup Docker volumes
    docker run --rm -v devduck-model-cache:/data -v "$PWD/$BACKUP_DIR":/backup alpine tar czf /backup/models.tar.gz /data
    
    # Backup custom agents
    cp -r agents "$BACKUP_DIR/" 2>/dev/null || echo "Agents directory not found"
    
    # Backup documentation
    cp -r docs "$BACKUP_DIR/" 2>/dev/null || echo "Docs directory not found"
    
    echo "✅ Backup created: $BACKUP_DIR"
}

restore_system() {
    if [ -z "$1" ]; then
        echo "❌ Please specify backup directory"
        echo "Usage: $0 restore <backup_directory>"
        return 1
    fi
    
    RESTORE_DIR="$1"
    
    if [ ! -d "$RESTORE_DIR" ]; then
        echo "❌ Backup directory not found: $RESTORE_DIR"
        return 1
    fi
    
    echo "🔄 Restoring from backup: $RESTORE_DIR"
    
    # Stop current system
    docker compose down
    
    # Restore configuration
    cp "$RESTORE_DIR"/*.{env,yml,yaml} . 2>/dev/null || echo "Some config files not in backup"
    
    # Restore Docker volumes
    if [ -f "$RESTORE_DIR/models.tar.gz" ]; then
        docker run --rm -v devduck-model-cache:/data -v "$PWD/$RESTORE_DIR":/backup alpine tar xzf /backup/models.tar.gz -C /
    fi
    
    # Restore custom code
    cp -r "$RESTORE_DIR/agents" . 2>/dev/null || echo "No agents backup found"
    cp -r "$RESTORE_DIR/docs" . 2>/dev/null || echo "No docs backup found"
    
    # Start system
    docker compose up -d
    
    echo "✅ System restored from backup"
}

list_backups() {
    echo "📋 Available backups:"
    find ./backups -type d -name "[0-9]*" | sort -r | head -10
}

case "$1" in
    "backup")
        backup_system
        ;;
    "restore")
        restore_system "$2"
        ;;
    "list")
        list_backups
        ;;
    *)
        echo "Usage: $0 {backup|restore|list}"
        echo ""
        echo "Commands:"
        echo "  backup           - Create a full system backup"
        echo "  restore <dir>    - Restore from specified backup directory"
        echo "  list             - List available backups"
        ;;
esac

EOF

chmod +x backup_restore.sh
```

## Performance Monitoring Dashboard

### 📊 Real-time System Monitor

```python
# Create monitoring_dashboard.py
import time
import requests
import psutil
import docker
from datetime import datetime
import json
import threading
from collections import deque

class MonitoringDashboard:
    def __init__(self):
        self.docker_client = docker.from_env()
        self.metrics_history = deque(maxlen=100)  # Store last 100 data points
        self.running = False
        self.api_url = "http://localhost:8000"
    
    def collect_system_metrics(self):
        """Collect comprehensive system metrics."""
        try:
            # System metrics
            cpu_percent = psutil.cpu_percent(interval=1)
            memory = psutil.virtual_memory()
            disk = psutil.disk_usage('/')
            
            # Docker container metrics
            container_stats = {}
            try:
                containers = self.docker_client.containers.list(filters={"name": "devduck"})
                for container in containers:
                    stats = container.stats(stream=False)
                    container_stats[container.name] = {
                        'cpu_percent': self._calculate_cpu_percent(stats),
                        'memory_usage': stats['memory_stats']['usage'],
                        'memory_limit': stats['memory_stats']['limit'],
                        'network_rx': stats['networks']['eth0']['rx_bytes'] if 'eth0' in stats.get('networks', {}) else 0,
                        'network_tx': stats['networks']['eth0']['tx_bytes'] if 'eth0' in stats.get('networks', {}) else 0
                    }
            except Exception as e:
                container_stats = {"error": str(e)}
            
            # API health check
            api_healthy = False
            api_response_time = 0
            try:
                start = time.time()
                response = requests.get(f"{self.api_url}/health", timeout=5)
                api_response_time = time.time() - start
                api_healthy = response.status_code == 200
            except:
                pass
            
            # Compile metrics
            metrics = {
                'timestamp': datetime.now().isoformat(),
                'system': {
                    'cpu_percent': cpu_percent,
                    'memory_percent': memory.percent,
                    'memory_used_gb': memory.used / (1024**3),
                    'memory_total_gb': memory.total / (1024**3),
                    'disk_percent': disk.percent,
                    'disk_free_gb': disk.free / (1024**3)
                },
                'containers': container_stats,
                'api': {
                    'healthy': api_healthy,
                    'response_time': api_response_time
                }
            }
            
            return metrics
            
        except Exception as e:
            return {'error': str(e), 'timestamp': datetime.now().isoformat()}
    
    def _calculate_cpu_percent(self, stats):
        """Calculate CPU percentage from Docker stats."""
        try:
            cpu_stats = stats['cpu_stats']
            precpu_stats = stats['precpu_stats']
            
            cpu_usage = cpu_stats['cpu_usage']['total_usage']
            precpu_usage = precpu_stats['cpu_usage']['total_usage']
            
            system_usage = cpu_stats['system_cpu_usage']
            presystem_usage = precpu_stats['system_cpu_usage']
            
            cpu_num = len(cpu_stats['cpu_usage']['percpu_usage'])
            
            cpu_delta = cpu_usage - precpu_usage
            system_delta = system_usage - presystem_usage
            
            if system_delta > 0 and cpu_delta > 0:
                return (cpu_delta / system_delta) * cpu_num * 100.0
            
            return 0.0
        except:
            return 0.0
    
    def start_monitoring(self, interval=10):
        """Start continuous monitoring."""
        self.running = True
        
        def monitor_loop():
            while self.running:
                metrics = self.collect_system_metrics()
                self.metrics_history.append(metrics)
                time.sleep(interval)
        
        self.monitor_thread = threading.Thread(target=monitor_loop, daemon=True)
        self.monitor_thread.start()
        
        print(f"📊 Monitoring started (interval: {interval}s)")
    
    def stop_monitoring(self):
        """Stop monitoring."""
        self.running = False
        print("🛑 Monitoring stopped")
    
    def display_dashboard(self):
        """Display real-time dashboard."""
        import os
        
        while self.running:
            # Clear screen
            os.system('clear' if os.name == 'posix' else 'cls')
            
            if not self.metrics_history:
                print("⏳ Waiting for metrics...")
                time.sleep(2)
                continue
            
            current_metrics = self.metrics_history[-1]
            
            print("🖥️  DevDuck Multi-Agent System Dashboard")
            print("=" * 50)
            print(f"📅 Last Updated: {current_metrics.get('timestamp', 'Unknown')}")
            
            if 'error' in current_metrics:
                print(f"❌ Error: {current_metrics['error']}")
            else:
                # System metrics
                system = current_metrics.get('system', {})
                print(f"\n💻 System Resources:")
                print(f"   CPU: {system.get('cpu_percent', 0):.1f}%")
                print(f"   Memory: {system.get('memory_percent', 0):.1f}% ({system.get('memory_used_gb', 0):.1f}/{system.get('memory_total_gb', 0):.1f} GB)")
                print(f"   Disk: {system.get('disk_percent', 0):.1f}% ({system.get('disk_free_gb', 0):.1f} GB free)")
                
                # Container metrics
                containers = current_metrics.get('containers', {})
                if containers and 'error' not in containers:
                    print(f"\n🐳 Container Status:")
                    for name, stats in containers.items():
                        memory_percent = (stats['memory_usage'] / stats['memory_limit']) * 100 if stats['memory_limit'] > 0 else 0
                        print(f"   {name}:")
                        print(f"     CPU: {stats['cpu_percent']:.1f}%")
                        print(f"     Memory: {memory_percent:.1f}% ({stats['memory_usage']/(1024**3):.2f} GB)")
                        print(f"     Network: ↓{stats['network_rx']/(1024**2):.1f}MB ↑{stats['network_tx']/(1024**2):.1f}MB")
                elif 'error' in containers:
                    print(f"\n🐳 Container Status: ❌ {containers['error']}")
                
                # API status
                api = current_metrics.get('api', {})
                status_icon = "✅" if api.get('healthy') else "❌"
                print(f"\n🔗 API Status: {status_icon} {'Healthy' if api.get('healthy') else 'Unhealthy'}")
                print(f"   Response Time: {api.get('response_time', 0):.3f}s")
                
                # Historical trends (last 10 data points)
                if len(self.metrics_history) >= 2:
                    print(f"\n📈 Trends (last {min(10, len(self.metrics_history))} measurements):")
                    recent_metrics = list(self.metrics_history)[-10:]
                    
                    cpu_values = [m.get('system', {}).get('cpu_percent', 0) for m in recent_metrics if 'system' in m]
                    memory_values = [m.get('system', {}).get('memory_percent', 0) for m in recent_metrics if 'system' in m]
                    
                    if cpu_values:
                        print(f"   CPU: {self._generate_sparkline(cpu_values)} (avg: {sum(cpu_values)/len(cpu_values):.1f}%)")
                    if memory_values:
                        print(f"   Memory: {self._generate_sparkline(memory_values)} (avg: {sum(memory_values)/len(memory_values):.1f}%)")
                
                # Alerts
                alerts = self._generate_alerts(current_metrics)
                if alerts:
                    print(f"\n🚨 Alerts:")
                    for alert in alerts:
                        print(f"   {alert}")
            
            print(f"\n⌨️  Press Ctrl+C to stop monitoring")
            time.sleep(5)
    
    def _generate_sparkline(self, values, width=20):
        """Generate a simple ASCII sparkline."""
        if not values:
            return ""
        
        min_val = min(values)
        max_val = max(values)
        range_val = max_val - min_val if max_val > min_val else 1
        
        chars = ['▁', '▂', '▃', '▄', '▅', '▆', '▇', '█']
        sparkline = ""
        
        for value in values[-width:]:
            normalized = (value - min_val) / range_val
            char_index = min(int(normalized * (len(chars) - 1)), len(chars) - 1)
            sparkline += chars[char_index]
        
        return sparkline
    
    def _generate_alerts(self, metrics):
        """Generate alerts based on current metrics."""
        alerts = []
        
        if 'system' in metrics:
            system = metrics['system']
            
            if system.get('cpu_percent', 0) > 85:
                alerts.append("⚠️  High CPU usage detected")
            if system.get('memory_percent', 0) > 90:
                alerts.append("⚠️  High memory usage detected")
            if system.get('disk_percent', 0) > 85:
                alerts.append("⚠️  Low disk space")
        
        if 'api' in metrics:
            api = metrics['api']
            
            if not api.get('healthy'):
                alerts.append("🚨 API is unhealthy")
            if api.get('response_time', 0) > 5:
                alerts.append("⚠️  Slow API response time")
        
        return alerts
    
    def export_metrics(self, filename=None):
        """Export metrics to JSON file."""
        if not filename:
            filename = f"metrics_export_{datetime.now().strftime('%Y%m%d_%H%M%S')}.json"
        
        with open(filename, 'w') as f:
            json.dump(list(self.metrics_history), f, indent=2)
        
        print(f"📊 Metrics exported to {filename}")

# Main monitoring application
if __name__ == '__main__':
    import signal
    import sys
    
    dashboard = MonitoringDashboard()
    
    def signal_handler(sig, frame):
        print('\n🛑 Stopping dashboard...')
        dashboard.stop_monitoring()
        sys.exit(0)
    
    signal.signal(signal.SIGINT, signal_handler)
    
    # Start monitoring
    dashboard.start_monitoring(interval=5)
    
    try:
        dashboard.display_dashboard()
    except KeyboardInterrupt:
        dashboard.stop_monitoring()
        dashboard.export_metrics()
        print("\n👋 Dashboard stopped")
```

## Future Enhancement Ideas

### 🚀 Advanced Features to Explore

1. **AI Model Fine-tuning**
   - Custom training on domain-specific data
   - Model distillation for performance
   - Federated learning approaches

2. **Advanced Agent Capabilities**
   - Multi-modal agents (text + images + audio)
   - Specialized domain agents (medical, legal, financial)
   - Self-improving agents with feedback loops

3. **Enterprise Integrations**
   - Slack/Teams bot integration
   - CRM system integration
   - Document management system connectors
   - API gateway integration

4. **Scalability Enhancements**
   - Kubernetes operators for auto-scaling
   - Edge deployment for reduced latency
   - Multi-region deployment
   - Cost optimization algorithms

5. **Security Improvements**
   - Zero-trust architecture
   - End-to-end encryption
   - Audit logging and compliance
   - Privacy-preserving techniques

### 🎓 Learning Resources

**Recommended Reading:**
- "Multi-Agent Systems" by Gerhard Weiss
- "Docker Deep Dive" by Nigel Poulton
- "Designing Data-Intensive Applications" by Martin Kleppmann
- "Building Microservices" by Sam Newman

**Online Courses:**
- Docker and Kubernetes certification paths
- Machine Learning Operations (MLOps) courses
- Cloud architecture certifications
- AI/ML specializations on Coursera/edX

**Communities:**
- Docker Community Slack
- Kubernetes Community
- AI/ML Reddit communities
- Local Docker and DevOps meetups

### 🛠️ Development Environment Setup

```bash
# Create development environment setup
cat > setup_dev_environment.sh << 'EOF'
#!/bin/bash

echo "🛠️ Setting up DevDuck Development Environment"
echo "=============================================="

# Install development tools
echo "📦 Installing development tools..."

# Python development
pip install -U pip
pip install black flake8 pytest pytest-cov mypy
pip install jupyter notebook ipython

# Docker development
echo "🐳 Setting up Docker development tools..."
docker plugin install store/sumologic/docker-output-driver:1.0.0 || echo "Plugin already installed"

# Pre-commit hooks
echo "🎣 Setting up pre-commit hooks..."
pip install pre-commit

cat > .pre-commit-config.yaml << 'PRECOMMIT'
repos:
-   repo: https://github.com/psf/black
    rev: 22.3.0
    hooks:
    -   id: black
        language_version: python3
-   repo: https://github.com/pycqa/flake8
    rev: 4.0.1
    hooks:
    -   id: flake8
        args: [--max-line-length=88, --extend-ignore=E203]
-   repo: https://github.com/pre-commit/mirrors-mypy
    rev: v0.950
    hooks:
    -   id: mypy
        additional_dependencies: [types-requests]
PRECOMMIT

pre-commit install

# Testing framework
echo "🧪 Setting up testing framework..."
mkdir -p tests

cat > tests/test_agents.py << 'PYTEST'
import pytest
import requests
import time

def test_health_endpoint():
    """Test that the health endpoint is accessible."""
    response = requests.get('http://localhost:8000/health')
    assert response.status_code == 200

def test_basic_chat():
    """Test basic chat functionality."""
    response = requests.post(
        'http://localhost:8000/chat',
        json={
            'message': 'Hello test',
            'conversation_id': 'test-123'
        }
    )
    assert response.status_code == 200
    data = response.json()
    assert 'response' in data

def test_agent_routing():
    """Test that different query types route appropriately."""
    test_cases = [
        {'message': 'Hello', 'expected_fast': True},
        {'message': 'Design a complex distributed system architecture', 'expected_fast': False}
    ]
    
    for case in test_cases:
        start_time = time.time()
        response = requests.post(
            'http://localhost:8000/chat',
            json={
                'message': case['message'],
                'conversation_id': f'routing-test-{int(time.time())}'
            }
        )
        duration = time.time() - start_time
        
        assert response.status_code == 200
        
        if case['expected_fast']:
            assert duration < 5, f"Expected fast response, got {duration:.2f}s"
        # Note: We don't test slow responses in CI as they may timeout

PYTEST

# Documentation tools
echo "📚 Setting up documentation tools..."
pip install mkdocs mkdocs-material mkdocs-mermaid2-plugin

# Development compose override
echo "🔧 Creating development compose override..."
cat > compose.dev.yml << 'DEVCOMPOSE'
version: '3.8'
services:
  devduck-agent:
    build:
      context: ./agents
      target: development
    environment:
      - DEBUG=true
      - LOG_LEVEL=DEBUG
      - RELOAD=true
    volumes:
      - ./agents:/app:ro
      - /app/__pycache__
    ports:
      - "8000:8000"
      - "5678:5678"  # Debugger port
DEVCOMPOSE

echo "✅ Development environment setup complete!"
echo ""
echo "📋 Next steps:"
echo "   1. Run tests: pytest tests/"
echo "   2. Start development server: docker compose -f compose.yml -f compose.dev.yml up"
echo "   3. Format code: black agents/"
echo "   4. Check types: mypy agents/"
echo "   5. Build docs: mkdocs serve"

EOF

chmod +x setup_dev_environment.sh
```

## Workshop Completion

### 🎉 Congratulations!

You've successfully completed the Docker DevDuck Multi-Agent Workshop! You've gained expertise in:

#### ✅ **Core Competencies Mastered**

1. **Multi-Agent System Architecture**
   - Understanding agent roles and responsibilities
   - Designing communication patterns
   - Implementing orchestration logic

2. **Docker Containerization**
   - Container orchestration with Docker Compose
   - Volume management and networking
   - Production deployment strategies

3. **AI Integration**
   - Local model management and optimization
   - Cloud AI service integration (Cerebras)
   - Intelligent request routing

4. **Production Operations**
   - Monitoring and observability
   - Security implementation
   - Performance optimization
   - Troubleshooting and debugging

5. **Enterprise Deployment**
   - Kubernetes deployment
   - Auto-scaling strategies
   - High availability configuration
   - Disaster recovery planning

#### 📊 **Workshop Statistics**
- **10 comprehensive labs** completed
- **50+ hands-on exercises** mastered  
- **20+ production tools** implemented
- **100+ code examples** explored
- **Enterprise-grade skills** developed

### 🎯 **Next Steps for Continued Learning**

1. **Immediate Actions** (Next 1-2 weeks)
   - Deploy your system to a cloud provider
   - Implement monitoring and alerting
   - Add custom agents for your specific use cases
   - Share your experience with the community

2. **Short-term Goals** (Next 1-3 months)
   - Integrate with enterprise systems (Slack, CRM, etc.)
   - Implement advanced security features
   - Optimize for production workloads
   - Contribute to open-source projects

3. **Long-term Vision** (3+ months)
   - Build specialized domain agents
   - Explore multi-modal AI capabilities
   - Develop agent marketplace
   - Lead AI transformation in your organization

### 🌟 **Community and Support**

- **Share Your Success**: Post about your workshop completion on LinkedIn/Twitter with #DockerDevDuck
- **Join Communities**: Engage with Docker, AI/ML, and DevOps communities
- **Contribute Back**: Help others by answering questions and sharing knowledge
- **Keep Learning**: Stay updated with latest developments in AI and containerization

### 📜 **Workshop Certificate**

You've earned expertise equivalent to:
- Docker Certified Associate level knowledge
- MLOps intermediate proficiency
- Multi-agent systems specialist understanding
- Production deployment competency

---

## 🚀 **Ready to Build the Future with Multi-Agent Systems!**

The skills you've developed in this workshop position you at the forefront of AI and containerization technology. Multi-agent systems represent the future of intelligent applications, and you now have the expertise to build, deploy, and scale them effectively.

Remember: The journey of learning never ends. Stay curious, keep experimenting, and continue building amazing things!

**Happy Building! 🐳🤖✨**

---

!!! success "🎓 Workshop Mastery Achieved!"
    You are now equipped with enterprise-grade skills for building and deploying sophisticated multi-agent systems. Use this knowledge to drive innovation and create intelligent solutions that make a real impact.