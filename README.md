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
docker compose -f oci://dockersamples/docker-devduck-labspace up -y
```

Then open your browser to [http://localhost:3030](http://localhost:3030)

### Development Mode

If you're developing this labspace content:

```bash
CONTENT_PATH=$PWD docker compose -f oci://dockersamples/labspace-content-dev up
```


