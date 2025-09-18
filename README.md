# Docker DevDuck Multi-Agent Labspace

A comprehensive Docker Labspace for learning to build and deploy multi-agent systems using Docker, Google Agent Development Kit (ADK), and Cerebras AI.

## Overview

This labspace provides hands-on experience with:

- **Multi-Agent System Architecture**: Learn how Docker orchestrates multiple AI agents
- **Cerebras AI Integration**: Connect local models with Cerebras cloud services  
- **Agent Communication**: Master inter-agent messaging and intelligent routing
- **Real-world Applications**: Build Node.js development assistance scenarios
- **Container Orchestration**: Use Docker Compose for agent coordination

## What You'll Build

Throughout this workshop, you'll create a sophisticated multi-agent system featuring:

### 🎼 DevDuck Agent Orchestrator
A central coordinator that manages communication between specialized agents.

### 🤖 Local Agent
Handles local processing tasks, code analysis, and quick responses.

### 🧠 Cerebras Agent  
Leverages Cerebras AI for advanced language processing and complex problem-solving.

### 🌐 Web Interface
A FastAPI-based interface for seamless user interaction with the agent system.

## Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Web Interface │    │  DevDuck Agent  │    │  Cerebras Agent │
│    (FastAPI)    │◄──►│  (Orchestrator) │◄──►│   (Cloud AI)    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
                               │
                               ▼
                       ┌─────────────────┐
                       │   Local Agent   │
                       │ (Local Models)  │
                       └─────────────────┘
```

## Running the Labspace

### Quick Start

To run this labspace, ensure you have Docker installed on your system:

```bash
export CONTENT_REPO_URL=$(git remote get-url origin)
docker compose -f oci://dockersamples/labspace up -y
```

Then open your browser to [http://localhost:3030](http://localhost:3030)

### Development Mode

If you're developing this labspace content:

```bash
CONTENT_PATH=$PWD docker compose -f oci://dockersamples/labspace-content-dev up
```

## Lab Structure

This labspace is organized into 10 comprehensive labs:

1. **Introduction** - Overview of multi-agent systems and workshop goals
2. **Prerequisites & System Overview** - Required setup and architecture understanding
3. **Getting Started** - Repository setup and initial configuration
4. **Environment Setup & Deployment** - Docker deployment and service management
5. **Basic Multi-Agent Interaction** - First interactions with the agent system
6. **Local Agent Tasks** - Working with local processing capabilities
7. **Cerebras Analysis & Intelligence** - Leveraging cloud AI for advanced tasks
8. **Agent Routing & Communication** - Understanding inter-agent messaging
9. **Advanced Features & Best Practices** - Production considerations and optimization
10. **Troubleshooting & Next Steps** - Problem solving and future learning paths

## Key Features

- **Hands-on Learning**: Interactive exercises with real Docker containers
- **Progressive Complexity**: Start simple, build to advanced multi-agent scenarios
- **Real-world Applications**: Node.js development assistance use cases
- **Best Practices**: Learn production-ready deployment patterns
- **Troubleshooting Guides**: Common issues and solutions included

## Learning Outcomes

After completing this labspace, you'll be able to:

- Design and deploy multi-agent systems using Docker
- Integrate local and cloud-based AI models effectively
- Implement agent communication patterns and routing logic
- Build web interfaces for agent interaction
- Apply containerization best practices for AI applications
- Troubleshoot common multi-agent system issues

## Prerequisites

- Basic Docker knowledge
- Familiarity with Python and web APIs
- Understanding of containerization concepts
- Cerebras API account (free tier available)

## File Structure

```
├── labspace.yaml          # Labspace configuration
├── docs/                  # Tutorial markdown files
├── agents/               # Agent implementation code
├── compose.yml           # Docker Compose configuration
├── .env.sample          # Environment template
└── README.md            # This file
```

## Contributing

This labspace is part of the Docker educational content ecosystem. Contributions and improvements are welcome!


