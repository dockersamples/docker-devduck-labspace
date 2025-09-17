# Troubleshooting & Next Steps

Congratulations on completing the Docker DevDuck Multi-Agent Workshop! This final lab covers troubleshooting common issues, debugging techniques, optimization strategies, and exciting next steps for extending your multi-agent system expertise.

!!! info "Learning Focus"
    Master troubleshooting techniques, system optimization, debugging strategies, and explore advanced multi-agent system concepts for continued learning.

## Common Issues & Troubleshooting

### 🐛 Systematic Debugging Approach

#### Debugging Framework

```python
# Create debug_toolkit.py
import docker
import requests
import subprocess
import json
import time
from typing import Dict, List, Optional
import psutil

class DevDuckDebugger:
    def __init__(self):
        self.docker_client = docker.from_env()
        self.base_url = "http://localhost:8000"
        self.debug_results = []
    
    def run_full_diagnostic(self) -> Dict:
        """Run comprehensive system diagnostic."""
        print("🔍 Running Full System Diagnostic")
        print("=" * 40)
        
        diagnostics = {
            "timestamp": time.time(),
            "system_health": self._check_system_health(),
            "docker_status": self._check_docker_status(),
            "container_health": self._check_container_health(),
            "network_connectivity": self._check_network_connectivity(),
            "api_endpoints": self._check_api_endpoints(),
            "resource_usage": self._check_resource_usage(),
            "log_analysis": self._analyze_logs(),
            "configuration_check": self._check_configuration()
        }
        
        # Generate summary
        diagnostics["summary"] = self._generate_diagnostic_summary(diagnostics)
        
        return diagnostics
    
    def _check_system_health(self) -> Dict:
        """Check overall system health."""
        try:
            cpu_percent = psutil.cpu_percent(interval=1)
            memory = psutil.virtual_memory()
            disk = psutil.disk_usage('/')
            
            return {
                "status": "healthy" if cpu_percent < 80 and memory.percent < 85 else "warning",
                "cpu_usage": cpu_percent,
                "memory_usage": memory.percent,
                "disk_usage": disk.percent,
                "available_memory_gb": memory.available / (1024**3)
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _check_docker_status(self) -> Dict:
        """Check Docker daemon and basic functionality."""
        try:
            # Check if Docker is running
            docker_info = self.docker_client.info()
            
            return {
                "status": "healthy",
                "version": docker_info.get('ServerVersion', 'Unknown'),
                "containers_running": docker_info.get('ContainersRunning', 0),
                "images_count": docker_info.get('Images', 0),
                "storage_driver": docker_info.get('Driver', 'Unknown')
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _check_container_health(self) -> Dict:
        """Check health of DevDuck containers."""
        try:
            containers = self.docker_client.containers.list(all=True)
            devduck_containers = [
                c for c in containers 
                if 'devduck' in c.name.lower() or 'cerebras' in c.name.lower()
            ]
            
            container_status = {}
            
            for container in devduck_containers:
                try:
                    stats = container.stats(stream=False)
                    
                    # Calculate CPU usage
                    cpu_usage = self._calculate_cpu_percentage(stats)
                    
                    # Calculate memory usage
                    memory_usage = stats['memory_stats']['usage']
                    memory_limit = stats['memory_stats']['limit']
                    memory_percent = (memory_usage / memory_limit) * 100
                    
                    container_status[container.name] = {
                        "status": container.status,
                        "cpu_usage": cpu_usage,
                        "memory_usage_percent": memory_percent,
                        "memory_usage_mb": memory_usage / (1024*1024),
                        "restart_count": container.attrs['RestartCount'],
                        "health": "healthy" if container.status == 'running' else "unhealthy"
                    }
                    
                except Exception as e:
                    container_status[container.name] = {
                        "status": container.status,
                        "error": str(e),
                        "health": "error"
                    }
            
            overall_health = "healthy" if all(
                status["health"] == "healthy" 
                for status in container_status.values()
            ) else "warning"
            
            return {
                "status": overall_health,
                "containers": container_status,
                "total_containers": len(devduck_containers)
            }
            
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _calculate_cpu_percentage(self, stats: Dict) -> float:
        """Calculate CPU usage percentage from container stats."""
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
    
    def _check_network_connectivity(self) -> Dict:
        """Check network connectivity."""
        connectivity_tests = {
            "docker_network": self._test_docker_network(),
            "internet": self._test_internet_connectivity(),
            "cerebras_api": self._test_cerebras_api_connectivity()
        }
        
        overall_status = "healthy" if all(
            test.get("status") == "healthy" 
            for test in connectivity_tests.values()
        ) else "warning"
        
        return {
            "status": overall_status,
            "tests": connectivity_tests
        }
    
    def _test_docker_network(self) -> Dict:
        """Test Docker network connectivity."""
        try:
            networks = self.docker_client.networks.list()
            devduck_networks = [n for n in networks if 'devduck' in n.name]
            
            return {
                "status": "healthy" if devduck_networks else "warning",
                "networks_found": len(devduck_networks),
                "network_names": [n.name for n in devduck_networks]
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _test_internet_connectivity(self) -> Dict:
        """Test general internet connectivity."""
        try:
            response = requests.get("https://httpbin.org/get", timeout=5)
            return {
                "status": "healthy" if response.status_code == 200 else "warning",
                "response_time": response.elapsed.total_seconds()
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _test_cerebras_api_connectivity(self) -> Dict:
        """Test Cerebras API connectivity."""
        try:
            # Test basic connectivity to Cerebras
            response = requests.get("https://api.cerebras.ai", timeout=10)
            return {
                "status": "healthy" if response.status_code < 500 else "warning",
                "response_code": response.status_code,
                "response_time": response.elapsed.total_seconds()
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _check_api_endpoints(self) -> Dict:
        """Check DevDuck API endpoints."""
        endpoints = {
            "health": "/health",
            "chat": "/chat",
            "metrics": "/metrics"
        }
        
        endpoint_status = {}
        
        for name, path in endpoints.items():
            try:
                if name == "chat":
                    # POST request for chat endpoint
                    response = requests.post(
                        f"{self.base_url}{path}",
                        json={"message": "health check", "conversation_id": "debug-test"},
                        timeout=10
                    )
                else:
                    # GET request for other endpoints
                    response = requests.get(f"{self.base_url}{path}", timeout=5)
                
                endpoint_status[name] = {
                    "status": "healthy" if response.status_code < 400 else "warning",
                    "status_code": response.status_code,
                    "response_time": response.elapsed.total_seconds()
                }
                
            except Exception as e:
                endpoint_status[name] = {
                    "status": "error",
                    "error": str(e)
                }
        
        overall_status = "healthy" if all(
            status.get("status") == "healthy" 
            for status in endpoint_status.values()
        ) else "warning"
        
        return {
            "status": overall_status,
            "endpoints": endpoint_status
        }
    
    def _check_resource_usage(self) -> Dict:
        """Check system resource usage patterns."""
        try:
            # Get container resource usage
            containers = self.docker_client.containers.list()
            devduck_containers = [c for c in containers if 'devduck' in c.name.lower()]
            
            total_memory_usage = 0
            total_cpu_usage = 0
            
            for container in devduck_containers:
                try:
                    stats = container.stats(stream=False)
                    cpu_usage = self._calculate_cpu_percentage(stats)
                    memory_usage = stats['memory_stats']['usage']
                    
                    total_cpu_usage += cpu_usage
                    total_memory_usage += memory_usage
                except:
                    continue
            
            return {
                "status": "healthy",
                "total_memory_usage_mb": total_memory_usage / (1024*1024),
                "total_cpu_usage_percent": total_cpu_usage,
                "container_count": len(devduck_containers)
            }
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _analyze_logs(self) -> Dict:
        """Analyze container logs for issues."""
        try:
            containers = self.docker_client.containers.list()
            devduck_containers = [c for c in containers if 'devduck' in c.name.lower()]
            
            log_analysis = {}
            error_patterns = ['error', 'exception', 'failed', 'timeout', 'connection refused']
            
            for container in devduck_containers:
                try:
                    # Get last 100 lines of logs
                    logs = container.logs(tail=100).decode('utf-8')
                    
                    # Count error patterns
                    error_count = 0
                    found_errors = []
                    
                    for line in logs.split('\n'):
                        line_lower = line.lower()
                        for pattern in error_patterns:
                            if pattern in line_lower:
                                error_count += 1
                                found_errors.append(line.strip())
                                break
                    
                    log_analysis[container.name] = {
                        "error_count": error_count,
                        "recent_errors": found_errors[-5:],  # Last 5 errors
                        "log_lines": len(logs.split('\n')),
                        "status": "healthy" if error_count == 0 else "warning"
                    }
                    
                except Exception as e:
                    log_analysis[container.name] = {
                        "status": "error",
                        "error": str(e)
                    }
            
            overall_status = "healthy" if all(
                analysis.get("status") == "healthy" 
                for analysis in log_analysis.values()
            ) else "warning"
            
            return {
                "status": overall_status,
                "container_logs": log_analysis
            }
            
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _check_configuration(self) -> Dict:
        """Check system configuration."""
        config_checks = {
            "env_vars": self._check_environment_variables(),
            "compose_files": self._check_compose_files(),
            "directories": self._check_directories()
        }
        
        overall_status = "healthy" if all(
            check.get("status") == "healthy" 
            for check in config_checks.values()
        ) else "warning"
        
        return {
            "status": overall_status,
            "checks": config_checks
        }
    
    def _check_environment_variables(self) -> Dict:
        """Check required environment variables."""
        import os
        
        required_vars = ['CEREBRAS_API_KEY']
        optional_vars = ['DEBUG', 'LOG_LEVEL', 'JWT_SECRET_KEY']
        
        missing_required = [var for var in required_vars if not os.getenv(var)]
        present_optional = [var for var in optional_vars if os.getenv(var)]
        
        return {
            "status": "healthy" if not missing_required else "warning",
            "missing_required": missing_required,
            "present_optional": present_optional
        }
    
    def _check_compose_files(self) -> Dict:
        """Check for required compose files."""
        import os
        
        required_files = ['compose.yml', 'docker-compose.yml']
        optional_files = ['monitoring-stack.yml', 'scaling-compose.yml']
        
        present_required = [f for f in required_files if os.path.exists(f)]
        present_optional = [f for f in optional_files if os.path.exists(f)]
        
        return {
            "status": "healthy" if present_required else "warning",
            "present_required": present_required,
            "present_optional": present_optional
        }
    
    def _check_directories(self) -> Dict:
        """Check for required directories."""
        import os
        
        expected_dirs = ['agents', 'docs', 'monitoring']
        present_dirs = [d for d in expected_dirs if os.path.exists(d)]
        
        return {
            "status": "healthy" if 'agents' in present_dirs else "warning",
            "present_directories": present_dirs
        }
    
    def _generate_diagnostic_summary(self, diagnostics: Dict) -> Dict:
        """Generate diagnostic summary with recommendations."""
        issues_found = []
        recommendations = []
        
        # Check each diagnostic section
        for section, data in diagnostics.items():
            if section == "summary":
                continue
                
            status = data.get("status")
            
            if status == "error":
                issues_found.append(f"{section}: {data.get('error', 'Unknown error')}")
                recommendations.append(f"Fix {section} issues before proceeding")
            elif status == "warning":
                issues_found.append(f"{section}: Performance or configuration issues detected")
                recommendations.append(f"Review {section} configuration and optimize")
        
        # Overall health score
        total_sections = len([k for k in diagnostics.keys() if k != "summary"])
        healthy_sections = len([k for k, v in diagnostics.items() 
                              if k != "summary" and v.get("status") == "healthy"])
        
        health_score = (healthy_sections / total_sections) * 100 if total_sections > 0 else 0
        
        return {
            "health_score": health_score,
            "overall_status": "healthy" if health_score >= 90 else "warning" if health_score >= 70 else "critical",
            "issues_found": issues_found,
            "recommendations": recommendations,
            "healthy_sections": healthy_sections,
            "total_sections": total_sections
        }
    
    def print_diagnostic_report(self, diagnostics: Dict):
        """Print formatted diagnostic report."""
        summary = diagnostics.get("summary", {})
        
        print("\n" + "=" * 50)
        print("🏥 SYSTEM DIAGNOSTIC REPORT")
        print("=" * 50)
        
        # Overall health
        health_score = summary.get("health_score", 0)
        overall_status = summary.get("overall_status", "unknown")
        
        status_emoji = {
            "healthy": "✅",
            "warning": "⚠️",
            "critical": "❌",
            "error": "🔥"
        }
        
        print(f"\n{status_emoji.get(overall_status, '❓')} Overall Health: {overall_status.upper()}")
        print(f"📊 Health Score: {health_score:.1f}%")
        print(f"🎯 Healthy Sections: {summary.get('healthy_sections', 0)}/{summary.get('total_sections', 0)}")
        
        # Issues
        issues = summary.get("issues_found", [])
        if issues:
            print(f"\n🚨 Issues Found ({len(issues)}):")
            for i, issue in enumerate(issues, 1):
                print(f"   {i}. {issue}")
        
        # Recommendations
        recommendations = summary.get("recommendations", [])
        if recommendations:
            print(f"\n💡 Recommendations ({len(recommendations)}):")
            for i, rec in enumerate(recommendations, 1):
                print(f"   {i}. {rec}")
        
        # Section details
        print(f"\n📋 Section Details:")
        for section, data in diagnostics.items():
            if section == "summary":
                continue
                
            status = data.get("status", "unknown")
            emoji = status_emoji.get(status, "❓")
            print(f"   {emoji} {section.replace('_', ' ').title()}: {status}")
        
        print("\n" + "=" * 50)

# Usage example
if __name__ == '__main__':
    debugger = DevDuckDebugger()
    diagnostics = debugger.run_full_diagnostic()
    debugger.print_diagnostic_report(diagnostics)
```

### 🔧 Common Issues & Solutions

#### Issue Resolution Guide

```bash
# Create issue_resolver.sh
cat > issue_resolver.sh << 'EOF'
#!/bin/bash

echo "🛠️ DevDuck Issue Resolver"
echo "=========================="

# Function to fix common issues
fix_issue() {
    issue_type=$1
    
    case $issue_type in
        "port_conflict")
            echo "🔧 Fixing port conflicts..."
            # Find and stop processes using port 8000
            lsof -ti:8000 | xargs -r kill -9
            echo "✅ Port 8000 freed"
            ;;
            
        "container_restart")
            echo "🔧 Restarting containers..."
            docker compose down
            sleep 5
            docker compose up -d
            echo "✅ Containers restarted"
            ;;
            
        "memory_cleanup")
            echo "🔧 Cleaning up memory..."
            docker system prune -f
            docker volume prune -f
            echo "✅ Memory cleaned up"
            ;;
            
        "model_cache_reset")
            echo "🔧 Resetting model cache..."
            docker compose exec devduck-agent rm -rf /app/models/*
            docker compose restart devduck-agent
            echo "✅ Model cache reset"
            ;;
            
        "permissions_fix")
            echo "🔧 Fixing file permissions..."
            chmod 644 .env*
            chmod 755 *.sh
            chmod -R 755 agents/
            echo "✅ Permissions fixed"
            ;;
            
        "full_reset")
            echo "🔧 Full system reset..."
            docker compose down -v
            docker system prune -af
            docker compose up -d --build
            echo "✅ Full reset completed"
            ;;
            
        *)
            echo "❌ Unknown issue type: $issue_type"
            echo "Available options: port_conflict, container_restart, memory_cleanup, model_cache_reset, permissions_fix, full_reset"
            exit 1
            ;;
    esac
}

# Interactive issue resolution
if [ $# -eq 0 ]; then
    echo "Select an issue to fix:"
    echo "1. Port conflicts"
    echo "2. Container restart"
    echo "3. Memory cleanup"
    echo "4. Model cache reset"
    echo "5. Permissions fix"
    echo "6. Full system reset"
    echo ""
    read -p "Enter choice (1-6): " choice
    
    case $choice in
        1) fix_issue "port_conflict" ;;
        2) fix_issue "container_restart" ;;
        3) fix_issue "memory_cleanup" ;;
        4) fix_issue "model_cache_reset" ;;
        5) fix_issue "permissions_fix" ;;
        6) fix_issue "full_reset" ;;
        *) echo "Invalid choice" && exit 1 ;;
    esac
else
    fix_issue $1
fi

EOF

chmod +x issue_resolver.sh
echo "🛠️ Issue resolver created: ./issue_resolver.sh"
```

## Performance Optimization

### ⚡ System Optimization Guide

```python
# Create optimizer.py
import docker
import psutil
import time
from typing import Dict, List, Tuple

class DevDuckOptimizer:
    def __init__(self):
        self.docker_client = docker.from_env()
        self.optimization_results = []
    
    def run_optimization_suite(self) -> Dict:
        """Run comprehensive optimization suite."""
        print("⚡ Running DevDuck Optimization Suite")
        print("=" * 40)
        
        optimizations = {
            "container_optimization": self._optimize_containers(),
            "resource_optimization": self._optimize_resources(),
            "network_optimization": self._optimize_network(),
            "storage_optimization": self._optimize_storage(),
            "performance_tuning": self._tune_performance()
        }
        
        # Calculate overall improvement
        optimizations["summary"] = self._calculate_optimization_impact(optimizations)
        
        return optimizations
    
    def _optimize_containers(self) -> Dict:
        """Optimize container configuration."""
        recommendations = []
        
        try:
            containers = self.docker_client.containers.list()
            devduck_containers = [c for c in containers if 'devduck' in c.name.lower()]
            
            for container in devduck_containers:
                stats = container.stats(stream=False)
                
                # Check memory usage
                memory_usage = stats['memory_stats']['usage']
                memory_limit = stats['memory_stats']['limit']
                memory_percent = (memory_usage / memory_limit) * 100
                
                if memory_percent > 80:
                    recommendations.append(f"Increase memory limit for {container.name}")
                elif memory_percent < 20:
                    recommendations.append(f"Consider reducing memory limit for {container.name}")
                
                # Check CPU usage
                cpu_usage = self._calculate_cpu_percentage(stats)
                if cpu_usage > 80:
                    recommendations.append(f"Consider CPU optimization for {container.name}")
            
            return {
                "status": "completed",
                "containers_analyzed": len(devduck_containers),
                "recommendations": recommendations
            }
            
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _calculate_cpu_percentage(self, stats: Dict) -> float:
        """Calculate CPU usage percentage."""
        try:
            cpu_stats = stats['cpu_stats']
            precpu_stats = stats['precpu_stats']
            
            cpu_usage = cpu_stats['cpu_usage']['total_usage']
            precpu_usage = precpu_stats['cpu_usage']['total_usage']
            
            system_usage = cpu_stats['system_cpu_usage']
            presystem_usage = precpu_stats['system_cpu_usage']
            
            cpu_delta = cpu_usage - precpu_usage
            system_delta = system_usage - presystem_usage
            
            if system_delta > 0 and cpu_delta > 0:
                cpu_num = len(cpu_stats['cpu_usage']['percpu_usage'])
                return (cpu_delta / system_delta) * cpu_num * 100.0
            
            return 0.0
        except:
            return 0.0
    
    def _optimize_resources(self) -> Dict:
        """Optimize system resources."""
        recommendations = []
        
        # Check system resources
        cpu_percent = psutil.cpu_percent(interval=1)
        memory = psutil.virtual_memory()
        disk = psutil.disk_usage('/')
        
        if cpu_percent > 80:
            recommendations.append("High CPU usage detected - consider scaling horizontally")
        
        if memory.percent > 85:
            recommendations.append("High memory usage - consider adding more RAM or optimizing models")
        
        if disk.percent > 80:
            recommendations.append("Low disk space - clean up old models and logs")
        
        # Docker resource optimization
        try:
            docker_info = self.docker_client.info()
            
            if docker_info.get('Images', 0) > 50:
                recommendations.append("Many Docker images present - run 'docker image prune'")
            
            if docker_info.get('Containers', 0) > docker_info.get('ContainersRunning', 0) + 5:
                recommendations.append("Many stopped containers - run 'docker container prune'")
        except:
            pass
        
        return {
            "status": "completed",
            "system_cpu": cpu_percent,
            "system_memory": memory.percent,
            "system_disk": disk.percent,
            "recommendations": recommendations
        }
    
    def _optimize_network(self) -> Dict:
        """Optimize network configuration."""
        recommendations = []
        
        try:
            # Check Docker networks
            networks = self.docker_client.networks.list()
            
            # Check for unused networks
            unused_networks = []
            for network in networks:
                if not network.containers and network.name not in ['bridge', 'host', 'none']:
                    unused_networks.append(network.name)
            
            if unused_networks:
                recommendations.append(f"Remove unused networks: {', '.join(unused_networks)}")
            
            # Check network connectivity
            devduck_networks = [n for n in networks if 'devduck' in n.name]
            if not devduck_networks:
                recommendations.append("Create dedicated DevDuck network for better isolation")
            
            return {
                "status": "completed",
                "total_networks": len(networks),
                "unused_networks": len(unused_networks),
                "recommendations": recommendations
            }
            
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _optimize_storage(self) -> Dict:
        """Optimize storage usage."""
        recommendations = []
        
        try:
            # Check Docker storage usage
            storage_info = self.docker_client.df()
            
            # Check volumes
            volumes = storage_info.get('Volumes', [])
            total_volume_size = sum(v.get('Size', 0) for v in volumes if v.get('Size'))
            
            if total_volume_size > 10 * 1024 * 1024 * 1024:  # > 10GB
                recommendations.append("Large volume usage detected - review model cache size")
            
            # Check images
            images = storage_info.get('Images', [])
            total_image_size = sum(img.get('Size', 0) for img in images if img.get('Size'))
            
            if total_image_size > 5 * 1024 * 1024 * 1024:  # > 5GB
                recommendations.append("Large image storage usage - clean up unused images")
            
            # Check for reclaimable space
            reclaimable = storage_info.get('BuilderSize', 0)
            if reclaimable > 1024 * 1024 * 1024:  # > 1GB
                recommendations.append(f"Reclaimable space: {reclaimable / (1024**3):.1f}GB - run 'docker builder prune'")
            
            return {
                "status": "completed",
                "total_volume_size_gb": total_volume_size / (1024**3),
                "total_image_size_gb": total_image_size / (1024**3),
                "reclaimable_gb": reclaimable / (1024**3),
                "recommendations": recommendations
            }
            
        except Exception as e:
            return {"status": "error", "error": str(e)}
    
    def _tune_performance(self) -> Dict:
        """Tune system performance parameters."""
        recommendations = []
        
        # Check system configuration
        cpu_count = psutil.cpu_count()
        total_memory = psutil.virtual_memory().total
        
        # CPU recommendations
        if cpu_count >= 8:
            recommendations.append("Multi-core system detected - enable parallel processing")
        elif cpu_count <= 2:
            recommendations.append("Limited CPU cores - consider using lighter models")
        
        # Memory recommendations
        memory_gb = total_memory / (1024**3)
        if memory_gb >= 16:
            recommendations.append("Adequate memory available - can use larger models")
        elif memory_gb <= 8:
            recommendations.append("Limited memory - use smaller models and enable memory optimization")
        
        # Docker daemon optimization
        recommendations.extend([
            "Set Docker daemon to use appropriate storage driver",
            "Configure Docker daemon with adequate memory limits",
            "Enable Docker BuildKit for faster builds"
        ])
        
        return {
            "status": "completed",
            "cpu_cores": cpu_count,
            "memory_gb": memory_gb,
            "recommendations": recommendations
        }
    
    def _calculate_optimization_impact(self, optimizations: Dict) -> Dict:
        """Calculate overall optimization impact."""
        total_recommendations = 0
        completed_optimizations = 0
        
        for category, results in optimizations.items():
            if category == "summary":
                continue
                
            if results.get("status") == "completed":
                completed_optimizations += 1
                total_recommendations += len(results.get("recommendations", []))
        
        return {
            "completed_optimizations": completed_optimizations,
            "total_recommendations": total_recommendations,
            "optimization_score": (completed_optimizations / len(optimizations)) * 100 if optimizations else 0,
            "priority_actions": self._get_priority_actions(optimizations)
        }
    
    def _get_priority_actions(self, optimizations: Dict) -> List[str]:
        """Get priority optimization actions."""
        priority_actions = []
        
        # High priority: resource issues
        resource_recs = optimizations.get("resource_optimization", {}).get("recommendations", [])
        for rec in resource_recs:
            if "High" in rec or "Low disk" in rec:
                priority_actions.append(rec)
        
        # Medium priority: container optimization
        container_recs = optimizations.get("container_optimization", {}).get("recommendations", [])
        priority_actions.extend(container_recs[:2])  # Top 2 container recommendations
        
        return priority_actions[:5]  # Top 5 priority actions
    
    def print_optimization_report(self, optimizations: Dict):
        """Print formatted optimization report."""
        summary = optimizations.get("summary", {})
        
        print("\n" + "=" * 50)
        print("⚡ SYSTEM OPTIMIZATION REPORT")
        print("=" * 50)
        
        # Overall score
        score = summary.get("optimization_score", 0)
        print(f"\n📊 Optimization Score: {score:.1f}%")
        print(f"✅ Completed Optimizations: {summary.get('completed_optimizations', 0)}")
        print(f"💡 Total Recommendations: {summary.get('total_recommendations', 0)}")
        
        # Priority actions
        priority_actions = summary.get("priority_actions", [])
        if priority_actions:
            print(f"\n🎯 Priority Actions:")
            for i, action in enumerate(priority_actions, 1):
                print(f"   {i}. {action}")
        
        # Category breakdown
        print(f"\n📋 Optimization Categories:")
        for category, results in optimizations.items():
            if category == "summary":
                continue
                
            status = results.get("status", "unknown")
            rec_count = len(results.get("recommendations", []))
            
            status_emoji = "✅" if status == "completed" else "❌"
            print(f"   {status_emoji} {category.replace('_', ' ').title()}: {rec_count} recommendations")
        
        print("\n" + "=" * 50)

# Usage example
if __name__ == '__main__':
    optimizer = DevDuckOptimizer()
    optimizations = optimizer.run_optimization_suite()
    optimizer.print_optimization_report(optimizations)
```

### 🔍 Advanced Debugging Tools

```python
# Create advanced_debugger.py
import asyncio
import aiohttp
import time
from typing import Dict, List

class AdvancedDebugger:
    def __init__(self):
        self.base_url = "http://localhost:8000"
        self.debug_sessions = []
    
    async def run_load_test(self, concurrent_requests: int = 10, 
                          total_requests: int = 100) -> Dict:
        """Run advanced load testing."""
        print(f"🚀 Running Load Test: {concurrent_requests} concurrent, {total_requests} total")
        
        async with aiohttp.ClientSession() as session:
            semaphore = asyncio.Semaphore(concurrent_requests)
            
            async def send_request(session, request_id):
                async with semaphore:
                    start_time = time.time()
                    try:
                        async with session.post(
                            f"{self.base_url}/chat",
                            json={
                                "message": f"Load test request {request_id}",
                                "conversation_id": f"load-test-{request_id}"
                            },
                            timeout=30
                        ) as response:
                            duration = time.time() - start_time
                            return {
                                "request_id": request_id,
                                "status_code": response.status,
                                "duration": duration,
                                "success": response.status == 200
                            }
                    except Exception as e:
                        return {
                            "request_id": request_id,
                            "error": str(e),
                            "duration": time.time() - start_time,
                            "success": False
                        }
            
            # Execute all requests
            tasks = [send_request(session, i) for i in range(total_requests)]
            results = await asyncio.gather(*tasks)
            
            # Analyze results
            successful = [r for r in results if r.get("success")]
            failed = [r for r in results if not r.get("success")]
            
            if successful:
                durations = [r["duration"] for r in successful]
                avg_duration = sum(durations) / len(durations)
                min_duration = min(durations)
                max_duration = max(durations)
                
                # Calculate percentiles
                sorted_durations = sorted(durations)
                p50 = sorted_durations[len(sorted_durations)//2]
                p95 = sorted_durations[int(len(sorted_durations)*0.95)]
            else:
                avg_duration = min_duration = max_duration = p50 = p95 = 0
            
            return {
                "total_requests": total_requests,
                "successful_requests": len(successful),
                "failed_requests": len(failed),
                "success_rate": len(successful) / total_requests,
                "avg_duration": avg_duration,
                "min_duration": min_duration,
                "max_duration": max_duration,
                "p50_duration": p50,
                "p95_duration": p95,
                "errors": [r.get("error") for r in failed if r.get("error")]
            }

# Usage example
async def main():
    debugger = AdvancedDebugger()
    results = await debugger.run_load_test(concurrent_requests=5, total_requests=20)
    
    print("\n📊 Load Test Results:")
    print(f"Success Rate: {results['success_rate']:.1%}")
    print(f"Average Duration: {results['avg_duration']:.2f}s")
    print(f"95th Percentile: {results['p95_duration']:.2f}s")
    
    if results['errors']:
        print(f"\nErrors ({len(results['errors'])}):") 
        for error in set(results['errors']):
            print(f"  - {error}")

if __name__ == '__main__':
    asyncio.run(main())
```

## Next Steps & Advanced Topics

### 🚀 Extending Your Multi-Agent System

#### Future Enhancement Ideas

```python
# Create enhancement_roadmap.py
class DevDuckEnhancementRoadmap:
    """Roadmap for extending DevDuck capabilities."""
    
    def __init__(self):
        self.enhancements = {
            "immediate": self._get_immediate_enhancements(),
            "short_term": self._get_short_term_enhancements(),
            "long_term": self._get_long_term_enhancements()
        }
    
    def _get_immediate_enhancements(self) -> list:
        """Enhancements you can implement immediately."""
        return [
            {
                "name": "Custom Agent Templates",
                "description": "Create specialized agents for specific domains (DevOps, Data Science, Security)",
                "difficulty": "Medium",
                "time_estimate": "1-2 weeks",
                "skills_needed": ["Python", "Agent Architecture", "Domain Knowledge"]
            },
            {
                "name": "Enhanced Monitoring",
                "description": "Add detailed performance metrics, alerting, and dashboards",
                "difficulty": "Medium",
                "time_estimate": "1 week",
                "skills_needed": ["Prometheus", "Grafana", "Alertmanager"]
            },
            {
                "name": "Configuration Management",
                "description": "Dynamic configuration updates without restarts",
                "difficulty": "Easy",
                "time_estimate": "2-3 days",
                "skills_needed": ["Python", "Configuration Patterns"]
            }
        ]
    
    def _get_short_term_enhancements(self) -> list:
        """Enhancements for next 1-3 months."""
        return [
            {
                "name": "Multi-Modal Support",
                "description": "Add image, audio, and video processing capabilities",
                "difficulty": "Hard",
                "time_estimate": "4-6 weeks",
                "skills_needed": ["Computer Vision", "Audio Processing", "ML Models"]
            },
            {
                "name": "Agent Marketplace",
                "description": "Plugin system for community-contributed agents",
                "difficulty": "Hard",
                "time_estimate": "6-8 weeks",
                "skills_needed": ["Plugin Architecture", "Security", "Package Management"]
            },
            {
                "name": "Workflow Orchestration",
                "description": "Complex multi-step workflows with branching logic",
                "difficulty": "Hard",
                "time_estimate": "4-6 weeks",
                "skills_needed": ["Workflow Engines", "State Management", "Distributed Systems"]
            }
        ]
    
    def _get_long_term_enhancements(self) -> list:
        """Long-term visionary enhancements."""
        return [
            {
                "name": "Autonomous Agent Learning",
                "description": "Agents that learn and improve from interactions",
                "difficulty": "Expert",
                "time_estimate": "3-6 months",
                "skills_needed": ["Machine Learning", "Reinforcement Learning", "Neural Networks"]
            },
            {
                "name": "Distributed Multi-Agent Network",
                "description": "Agents running across multiple machines and cloud providers",
                "difficulty": "Expert",
                "time_estimate": "4-6 months",
                "skills_needed": ["Distributed Systems", "Cloud Architecture", "Network Programming"]
            },
            {
                "name": "Natural Language Programming",
                "description": "Create and modify agents using natural language instructions",
                "difficulty": "Expert",
                "time_estimate": "6-12 months",
                "skills_needed": ["NLP", "Code Generation", "Program Synthesis"]
            }
        ]
    
    def print_roadmap(self):
        """Print the enhancement roadmap."""
        print("🚀 DevDuck Enhancement Roadmap")
        print("=" * 40)
        
        for timeline, enhancements in self.enhancements.items():
            print(f"\n📅 {timeline.upper()} ({len(enhancements)} items):")
            
            for i, enhancement in enumerate(enhancements, 1):
                difficulty_emoji = {
                    "Easy": "🟢",
                    "Medium": "🟡", 
                    "Hard": "🔴",
                    "Expert": "🟣"
                }
                
                emoji = difficulty_emoji.get(enhancement["difficulty"], "⚪")
                print(f"\n   {i}. {emoji} {enhancement['name']}")
                print(f"      📝 {enhancement['description']}")
                print(f"      ⏱️  Time: {enhancement['time_estimate']}")
                print(f"      🎯 Difficulty: {enhancement['difficulty']}")
                print(f"      🛠️  Skills: {', '.join(enhancement['skills_needed'])}")

# Usage
if __name__ == '__main__':
    roadmap = DevDuckEnhancementRoadmap()
    roadmap.print_roadmap()
```

### 📚 Learning Resources & Next Steps

```bash
# Create learning_resources.md
cat > learning_resources.md << 'EOF'
# 📚 DevDuck Learning Resources & Next Steps

## 🎓 Advanced Multi-Agent Systems

### Books & Papers
- "Multi-Agent Systems" by Gerhard Weiss
- "An Introduction to MultiAgent Systems" by Michael Wooldridge  
- "Distributed AI: Theory and Praxis" by various authors
- Research papers on agent communication protocols (FIPA, KQML)

### Online Courses
- "Multi-Agent Systems" (University of Edinburgh - Coursera)
- "Artificial Intelligence Planning" (University of Edinburgh - Coursera)
- "Distributed Systems" (MIT OpenCourseWare)

## 🤖 AI/ML Integration

### Frameworks & Libraries
- **LangChain**: Framework for developing applications with LLMs
- **AutoGen**: Microsoft's multi-agent conversation framework
- **CrewAI**: Framework for orchestrating role-playing, autonomous AI agents
- **Semantic Kernel**: Microsoft's SDK for integrating AI into applications

### Model Deployment
- **Ollama**: Run large language models locally
- **vLLM**: High-throughput LLM serving
- **TensorRT-LLM**: Optimized inference for NVIDIA GPUs
- **Hugging Face Transformers**: State-of-the-art ML models

## 🏗️ Infrastructure & DevOps

### Container Orchestration
- **Kubernetes**: Production-grade container orchestration
- **Docker Swarm**: Docker's native clustering solution
- **Nomad**: HashiCorp's workload orchestrator

### Service Mesh & Communication
- **Istio**: Service mesh for microservices
- **Envoy**: High-performance edge/middle proxy
- **gRPC**: Modern RPC framework
- **Apache Kafka**: Distributed streaming platform

### Monitoring & Observability
- **OpenTelemetry**: Observability framework
- **Jaeger**: Distributed tracing
- **Elastic Stack**: Search and analytics
- **DataDog**: Monitoring and analytics platform

## 🔧 Development Tools

### Testing Frameworks
- **pytest**: Python testing framework
- **Locust**: Load testing tool
- **Chaos Monkey**: Chaos engineering
- **TestContainers**: Integration testing with real dependencies

### Development Environments
- **DevContainers**: Containerized development environments
- **Docker Desktop**: Local development with Docker
- **Tilt**: Development environment for microservices
- **Skaffold**: Build and deploy for Kubernetes

## 🌐 Cloud Platforms

### Container Platforms
- **Google Cloud Run**: Serverless containers
- **AWS Fargate**: Serverless compute for containers
- **Azure Container Instances**: Fast container deployment
- **DigitalOcean App Platform**: Platform-as-a-Service

### AI/ML Platforms
- **Google AI Platform**: End-to-end ML platform
- **AWS SageMaker**: Build, train, and deploy ML models
- **Azure Machine Learning**: Cloud-based ML service
- **Hugging Face Spaces**: Deploy ML applications

## 📖 Recommended Learning Path

### Phase 1: Foundation (Weeks 1-2)
1. Deepen Docker and containerization knowledge
2. Study multi-agent system architectures
3. Learn advanced Python async programming
4. Understand distributed systems concepts

### Phase 2: Specialization (Weeks 3-6)
1. Choose focus area: AI/ML, DevOps, or Architecture
2. Build custom agents for chosen domain
3. Implement advanced monitoring and scaling
4. Study production deployment patterns

### Phase 3: Mastery (Weeks 7-12)
1. Contribute to open-source agent frameworks
2. Design novel multi-agent architectures
3. Implement learning and adaptation mechanisms
4. Build and deploy production systems

## 🎯 Project Ideas

### Beginner Projects
- [ ] **Code Review Agent**: Automated code quality analysis
- [ ] **Documentation Agent**: Generate and maintain documentation
- [ ] **Testing Agent**: Automated test generation and execution
- [ ] **Deployment Agent**: Automate deployment pipelines

### Intermediate Projects
- [ ] **Multi-Cloud Agent**: Deploy across different cloud providers
- [ ] **Security Agent**: Automated security scanning and compliance
- [ ] **Performance Agent**: Monitor and optimize application performance
- [ ] **Data Pipeline Agent**: Automated data processing workflows

### Advanced Projects
- [ ] **Research Agent**: Automated research and knowledge synthesis
- [ ] **Creative Agent**: Generate creative content (writing, design)
- [ ] **Teaching Agent**: Personalized learning and education
- [ ] **Business Intelligence Agent**: Automated business analysis

## 🤝 Community & Contributions

### Open Source Projects
- Contribute to LangChain, AutoGen, or CrewAI
- Build custom Docker images for the community
- Create educational content and tutorials
- Participate in agent-related research

### Professional Development
- Join AI and DevOps communities
- Attend conferences (DockerCon, KubeCon, AI conferences)
- Obtain relevant certifications (Docker, Kubernetes, Cloud)
- Build a portfolio of multi-agent projects

### Stay Updated
- Follow AI research publications
- Subscribe to Docker and Kubernetes newsletters
- Join Discord/Slack communities
- Participate in hackathons and competitions

EOF

echo "📚 Learning resources guide created!"
```

### 🏆 Workshop Completion Certificate

```bash
# Create completion_certificate.sh
cat > completion_certificate.sh << 'EOF'
#!/bin/bash

echo "🎓 Generating DevDuck Multi-Agent Workshop Certificate"
echo "===================================================="

# Get user information
read -p "Enter your name: " user_name
read -p "Enter completion date (YYYY-MM-DD) [$(date +%Y-%m-%d)]: " completion_date
completion_date=${completion_date:-$(date +%Y-%m-%d)}

# Create certificate
cat > certificate.txt << CERT_EOF

================================================================================
                    🎓 CERTIFICATE OF COMPLETION 🎓
================================================================================

                        Docker DevDuck Multi-Agent Workshop

                    This certifies that

                           ${user_name}

                has successfully completed the comprehensive
              Docker DevDuck Multi-Agent Systems Workshop

                        Completion Date: ${completion_date}

    Topics Mastered:
    ✅ Multi-Agent System Architecture
    ✅ Docker Container Orchestration  
    ✅ Cerebras AI Integration
    ✅ Agent Communication & Routing
    ✅ Local vs Cloud Agent Optimization
    ✅ Production Deployment & Scaling
    ✅ Security & Monitoring Implementation
    ✅ Advanced Troubleshooting & Debugging
    ✅ Custom Agent Development
    ✅ Performance Optimization

    Skills Acquired:
    🛠️  Multi-agent system design and implementation
    🏗️  Production-ready container orchestration
    🤖 AI agent development and optimization
    📊 System monitoring and observability
    🔒 Security hardening and best practices
    🚀 Automated deployment and scaling
    🐛 Advanced debugging and troubleshooting
    ⚡ Performance tuning and optimization

                        Congratulations on this achievement!

                    You are now equipped to build and deploy
                  production-grade multi-agent systems using
                    Docker, Cerebras AI, and modern DevOps
                              best practices.

              Next Steps: Continue learning, build amazing projects,
                     and contribute to the community!

================================================================================
                    Workshop by: Docker DevDuck Community
                    Certificate ID: DDW-$(date +%Y%m%d)-$(echo "$user_name" | tr ' ' '-' | tr '[:upper:]' '[:lower:]')
================================================================================

CERT_EOF

# Display certificate
cat certificate.txt

echo ""
echo "🎉 Congratulations! Your certificate has been saved to certificate.txt"
echo "📤 Share your achievement on social media with #DockerDevDuck #MultiAgent"
echo "🚀 Ready for your next multi-agent adventure!"

EOF

chmod +x completion_certificate.sh
echo "🏆 Certificate generator created: ./completion_certificate.sh"
```

## Final Workshop Summary

### 🎯 What You've Accomplished

Throughout this comprehensive workshop, you have:

1. **🏗️ Built a Complete Multi-Agent System**
   - Deployed DevDuck orchestrator with local and cloud agents
   - Integrated Cerebras AI for advanced reasoning capabilities
   - Implemented intelligent request routing and load balancing

2. **🔧 Mastered Production Operations**
   - Configured monitoring, logging, and alerting systems
   - Implemented security best practices and authentication
   - Set up automated deployment and scaling strategies

3. **🚀 Developed Advanced Skills**
   - Custom agent development and specialization
   - Performance optimization and troubleshooting
   - Multi-tenant and enterprise deployment patterns

### 🌟 Key Takeaways

- **Multi-agent systems** provide powerful capabilities for complex problem-solving
- **Docker containerization** enables scalable, portable agent deployments
- **Intelligent routing** maximizes efficiency by matching agents to appropriate tasks
- **Production readiness** requires comprehensive monitoring, security, and automation
- **Continuous learning** and optimization are essential for system success

### 🎊 Congratulations!

You've successfully completed one of the most comprehensive multi-agent system workshops available. You now possess the knowledge and skills to:

- Design and implement sophisticated multi-agent architectures
- Deploy production-ready systems with Docker and modern DevOps practices
- Integrate cutting-edge AI technologies like Cerebras for enhanced capabilities
- Optimize performance, ensure security, and maintain reliable operations
- Troubleshoot complex issues and continuously improve your systems

The future of AI is multi-agent, and you're now equipped to be part of that future!

---

**🚀 Ready to build the next generation of intelligent systems? The journey starts now!**

!!! success "Workshop Complete!"
    You've mastered multi-agent systems with Docker and Cerebras AI. Use the troubleshooting tools, optimization techniques, and learning resources to continue your journey in building intelligent, scalable systems. The future is multi-agent, and you're ready to shape it!