# Basic Multi-Agent Interaction

Now that your system is properly deployed and configured, it's time to explore how the multi-agent system works in practice. You'll learn to interact with DevDuck and understand how it coordinates between different agents to provide intelligent responses.

!!! info "Learning Objectives"
    - Master the DevDuck web interface
    - Understand agent routing and decision-making
    - Practice basic multi-agent conversations
    - Monitor agent communication patterns

## Understanding the Interface

### 🌐 Web Interface Overview

Let's start by familiarizing yourself with the DevDuck interface:

1. **Open the Interface**: Navigate to [http://localhost:8000/dev-ui/?app=devduck](http://localhost:8000/dev-ui/?app=devduck)

2. **Interface Components**:
   - **Chat Area**: Main conversation interface
   - **Input Field**: Where you type your messages
   - **Agent Indicator**: Shows which agents are active
   - **System Status**: Connection and health indicators

### 🔍 Interface Features

Explore these key features:

#### Message Types
The interface supports various message formats:
- **Text Messages**: Standard conversation
- **Code Blocks**: Formatted code snippets
- **System Messages**: Agent status updates
- **Error Messages**: Troubleshooting information

#### Agent Status Indicators
Look for these visual cues:
- 🟢 **Green**: Agent is online and responding
- 🟡 **Yellow**: Agent is processing
- 🔴 **Red**: Agent is unavailable or erroring
- 🔵 **Blue**: Agent is initializing

## First Interactions

### 🤖 Basic Conversation

Let's start with simple interactions to test the system:

#### Exercise 1: Introduction

**Try this conversation:**

```
You: Hello DevDuck! Can you introduce yourself?
```

**Expected Response Pattern:**
- DevDuck will introduce itself as a multi-agent system
- It should mention its capabilities and available agents
- The response might route to the Local Agent for a quick reply

**Monitor the logs** while you do this:
```bash
# In a separate terminal, watch the logs
docker compose logs -f devduck-agent
```

#### Exercise 2: Agent Awareness

**Ask about the agents:**

```
You: What agents are available in this system?
```

**Expected Response:**
DevDuck should explain:
- Local Agent for quick processing
- Cerebras Agent for complex analysis
- Its own role as orchestrator
- How it decides which agent to use

### 🧠 Understanding Agent Routing

DevDuck makes intelligent decisions about which agent should handle each request. Let's explore this:

#### Exercise 3: Simple vs Complex Queries

**Simple Query (likely routed to Local Agent):**
```
You: What is Python?
```

**Complex Query (likely routed to Cerebras Agent):**
```
You: Can you analyze this algorithm and suggest optimizations?

def fibonacci(n):
    if n <= 1:
        return n
    return fibonacci(n-1) + fibonacci(n-2)
```

**Observation Points:**
- Notice response time differences
- Check which agent handles each request in the logs
- Compare the depth and quality of responses

### 📊 Monitoring Agent Communication

Let's understand how agents communicate internally:

#### Real-time Monitoring Setup

```bash
# Terminal 1: Agent logs
docker compose logs -f devduck-agent

# Terminal 2: Resource monitoring  
watch docker stats

# Terminal 3: Network monitoring
docker compose exec devduck-agent netstat -tuln
```

#### Exercise 4: Multi-step Request

**Send a request that requires multiple agents:**

```
You: I'm building a Node.js application. Can you first explain what Express.js is, and then show me a complete example with error handling?
```

**What to observe:**
1. **Initial Processing**: DevDuck analyzes the request
2. **Agent Routing**: Request may be split between agents
3. **Response Assembly**: Final response combines multiple agent outputs
4. **Resource Usage**: CPU and memory spikes during processing

## Advanced Interaction Patterns

### 🎯 Context Awareness

Test how well the system maintains conversation context:

#### Exercise 5: Conversational Context

**Multi-turn conversation:**

```
Turn 1: "I'm working on a web application with authentication"
Turn 2: "What security best practices should I follow?"
Turn 3: "Can you show me how to implement JWT tokens?"
Turn 4: "What about refresh tokens?"
```

**Observation Points:**
- Does DevDuck remember previous context?
- How does it maintain conversation flow?
- Which agents contribute to different parts?

### 🔄 Agent Fallback Mechanisms

Test what happens when agents are unavailable:

#### Exercise 6: Resilience Testing

**Simulate agent failure:**

```bash
# Temporarily block Cerebras API (simulate network issues)
# Add this to your hosts file temporarily:
echo "127.0.0.1 api.cerebras.ai" >> /etc/hosts
```

**Then try a complex query:**
```
You: Analyze this complex algorithm and provide architectural recommendations for a microservices system.
```

**Observe:**
- How does DevDuck handle the failure?
- Does it fall back to Local Agent?
- What error handling is displayed?

**Clean up:**
```bash
# Remove the hosts entry
sudo sed -i '/api.cerebras.ai/d' /etc/hosts
```

## Practical Use Cases

### 💻 Development Assistance

Let's explore real-world development scenarios:

#### Exercise 7: Code Review

**Submit code for review:**

```
You: Can you review this JavaScript function for bugs and improvements?

function processUserData(users) {
  const results = [];
  for (let i = 0; i < users.length; i++) {
    if (users[i].age > 18) {
      results.push({
        name: users[i].name,
        email: users[i].email.toLowerCase(),
        age: users[i].age
      });
    }
  }
  return results;
}
```

**Expected Agent Behavior:**
- DevDuck might route to Cerebras for detailed analysis
- Local Agent might handle syntax validation
- Combined response with suggestions and improvements

#### Exercise 8: Architecture Discussion

**Ask for architectural guidance:**

```
You: I'm building a real-time chat application that needs to handle 10,000 concurrent users. What architecture would you recommend?
```

**This should trigger:**
- Complex routing to Cerebras Agent
- Detailed architectural analysis
- Multiple considerations and recommendations

### 🧪 Debugging and Troubleshooting

#### Exercise 9: Error Analysis

**Present an error for analysis:**

```
You: I'm getting this error in my Node.js app:

TypeError: Cannot read property 'map' of undefined
    at processItems (/app/utils.js:15:23)
    at UserController.getUsers (/app/controllers/user.js:42:18)

Can you help me debug this?
```

**Agent Response Analysis:**
- How quickly does DevDuck identify the issue?
- Which agent provides the solution?
- Does it offer multiple debugging approaches?

## Performance Analysis

### 📈 Response Time Patterns

Let's analyze response patterns systematically:

#### Create a Test Script

```python
# Create response_time_test.py
import requests
import time
import json

def test_response_times():
    """Test response times for different query types."""
    
    test_cases = [
        {"query": "Hello", "type": "simple", "expected_agent": "local"},
        {"query": "What is JavaScript?", "type": "medium", "expected_agent": "local"},
        {"query": "Design a scalable microservices architecture", "type": "complex", "expected_agent": "cerebras"},
        {"query": "Analyze this algorithm for time complexity", "type": "complex", "expected_agent": "cerebras"}
    ]
    
    results = []
    
    for case in test_cases:
        print(f"Testing: {case['type']} query")
        
        start_time = time.time()
        
        response = requests.post(
            'http://localhost:8000/chat',
            json={
                "message": case['query'],
                "conversation_id": f"test-{int(time.time())}"
            }
        )
        
        duration = time.time() - start_time
        
        results.append({
            "query_type": case['type'],
            "duration": duration,
            "status": response.status_code
        })
        
        print(f"  Duration: {duration:.2f}s")
        time.sleep(1)  # Brief pause between requests
    
    # Summary
    print("\n📊 Response Time Summary:")
    for result in results:
        print(f"  {result['query_type']:8}: {result['duration']:6.2f}s")

if __name__ == '__main__':
    test_response_times()
```

**Run the test:**
```bash
python response_time_test.py
```

### 🔍 Log Analysis

**Create a log analysis script:**

```bash
# Create log_analyzer.sh
cat > log_analyzer.sh << 'EOF'
#!/bin/bash

echo "🔍 DevDuck Log Analysis"
echo "====================="

# Get recent logs
LOGS=$(docker compose logs devduck-agent --tail=100)

# Count agent routing decisions
echo "Agent Routing Summary:"
echo "$LOGS" | grep -i "routing to" | sort | uniq -c | sort -nr

# Response time analysis
echo -e "\nResponse Time Patterns:"
echo "$LOGS" | grep -i "response_time" | tail -10

# Error analysis
echo -e "\nRecent Errors:"
echo "$LOGS" | grep -i "error\|exception\|failed" | tail -5

# Health check status
echo -e "\nHealth Check Status:"
echo "$LOGS" | grep -i "health" | tail -3

EOF

chmod +x log_analyzer.sh
./log_analyzer.sh
```

## Interactive Exercises

### 🎮 Challenge Scenarios

Try these progressively complex scenarios:

#### Challenge 1: Multi-Domain Expertise

```
You: I need help with a full-stack application. Can you explain how to set up authentication with JWT tokens on the backend, and then show me how to handle it in a React frontend?
```

**Success Criteria:**
- Both backend and frontend solutions provided
- Code examples for each layer
- Security considerations mentioned
- Clear separation of concerns

#### Challenge 2: Problem-Solving Chain

```
You: I have a performance issue with my database queries. The app is slow when loading user profiles. Can you help me identify the problem and suggest solutions?
```

**Follow-up questions based on response:**
- "What specific database optimizations would you recommend?"
- "How can I implement caching for this scenario?"
- "Can you show me how to profile the queries?"

#### Challenge 3: Architecture Evolution

```
You: My monolithic application is becoming hard to maintain. Can you guide me through converting it to microservices, step by step?
```

**This should trigger:**
- Detailed architectural analysis
- Step-by-step migration plan
- Risk assessments and mitigation strategies
- Tool and technology recommendations

### 🎯 Success Metrics

Evaluate your system based on:

- **Response Quality**: Accurate, helpful, contextual responses
- **Agent Coordination**: Smooth routing and fallback handling
- **Performance**: Reasonable response times for complexity
- **Reliability**: Consistent behavior across interactions
- **Context Retention**: Maintains conversation flow

## Troubleshooting Common Issues

### 🚨 Common Problems and Solutions

#### Issue: Slow Response Times

**Symptoms:**
- Responses take >30 seconds
- Interface shows "thinking" for extended periods

**Diagnosis:**
```bash
# Check system resources
docker stats

# Check model loading
docker compose logs devduck-agent | grep -i "model"

# Check API connectivity
curl -I https://api.cerebras.ai/v1/health
```

**Solutions:**
- Use smaller/faster models for development
- Increase system RAM allocation
- Check internet connectivity
- Consider caching strategies

#### Issue: Agent Communication Failures

**Symptoms:**
- "Agent unavailable" messages
- Incomplete or error responses
- Service health check failures

**Diagnosis:**
```bash
# Check service health
docker compose ps

# Test inter-service communication
docker compose exec devduck-agent curl -f http://mcp-gateway:3000/health

# Check network connectivity
docker network inspect docker-cerebras-demo_default
```

#### Issue: Memory Issues

**Symptoms:**
- Container restarts
- Out of memory errors
- System freezing

**Solutions:**
```bash
# Increase Docker memory limits
# Edit compose.yml:
# deploy:
#   resources:
#     limits:
#       memory: 8G

# Use lighter models
export LOCAL_MODEL_NAME=microsoft/DialoGPT-small
```

## Next Steps

Congratulations! You've successfully:

- ✅ Mastered basic multi-agent interactions
- ✅ Understood agent routing mechanisms
- ✅ Monitored system performance
- ✅ Handled various use cases and scenarios
- ✅ Troubleshot common issues

In the next section, you'll dive deeper into local agent capabilities and learn how to optimize local processing for specific tasks.

Ready to explore local agent specialization? Let's continue! 🚀

---

!!! tip "Pro Tip"
    Keep your logs open while testing. Understanding the agent communication patterns will help you optimize your multi-agent system for specific use cases.