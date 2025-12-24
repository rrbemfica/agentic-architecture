# Agentic Architecture

Reference architecture for building secure AI Agents with Amazon Bedrock AgentCore.

## Overview

This repository contains documentation for implementing a secure agentic architecture using:

- **Amazon Bedrock AgentCore** - Runtime and Gateway for Agents and MCPs
- **MCP (Model Context Protocol)** - Agent to Tools communication
- **A2A (Agent-to-Agent)** - Direct agent communication via JSON-RPC 2.0
- **OAuth 2.0 (Cognito M2M)** - Machine-to-machine authentication

## Documentation

- [AGENTCORE_ARCHITECTURE.md](AGENTCORE_ARCHITECTURE.md) - Complete architecture guide covering:
  - Authentication flows (User, M2M, External OAuth)
  - MCP communication via AgentCore Gateway
  - A2A communication patterns (internal, cross-domain, external)
  - Context propagation strategies
  - EKS vs AgentCore Runtime deployment options

## Key Principles

- All MCP traffic routes through AgentCore Gateway
- A2A communication is always direct (no Gateway)
- External A2A exposure requires Axway API Gateway
- 100% private communication via VPC Endpoints

## License

MIT
