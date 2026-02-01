# Zoo Tour Guide Agent

## Abstract:

This project demonstrates an end-to-end, production-ready architecture for building, securing, and deploying AI agents on Google Cloud. It showcases how Model Context Protocol (MCP) servers, Agent Development Kit (ADK) agents, and GPU-accelerated model backends can be composed into a scalable, secure, and modular system.

The project is implemented in three stages:
Deploying a secure MCP server on Cloud Run
Building and deploying an ADK-based agent that consumes the MCP server
Transitioning from prototype to production by deploying a GPU-accelerated ADK agent with autoscaling and load testing

The final outcome is a cloud-native, authenticated, service-to-service AI system suitable for real-world workloads.

## Problem Statement:

Large Language Models (LLMs) are powerful but inherently limited by their lack of direct access to external tools, real-time data, and domain-specific services. Embedding tools directly into monolithic applications creates tight coupling, poor scalability, and security risks.

Moving from a local AI prototype to a secure, scalable, production-grade deployment introduces challenges such as:
Secure tool exposure and authentication
Separation of reasoning and tooling layers
Service-to-service authorization
Deployment and scaling of AI agents
Integrating GPU-backed model inference
Observability and performance validation under load

This project addresses these challenges by designing a modular, cloud-native AI architecture using MCP, ADK, and Cloud Run.

## Objectives:

The primary goals of this project are to:
Build a secure MCP server that exposes tools to LLMs via authenticated APIs
Design an AI agent that consumes remote tools rather than embedding logic locally
Demonstrate separation of concerns between reasoning (agent) and tooling (MCP server)
Deploy all components as serverless, scalable Cloud Run services
Integrate GPU-accelerated model backends for production inference
Validate scalability, elasticity, and system behaviour under load

## Impementation:

Note: The implementation does not include everything. But this is for public understanding.

1. Deploying a Secure MCP Server on Cloud Run

First, I'll create a zoo MCP Server to provide valuable context for improving the use of LLMs with MCP, set up a zoo MCP server with FastMCP — a standard framework for working with the Model Context Protocol. FastMCP provides a quick way to build MCP servers and clients with Python. This MCP server provides data about animals at a fictional zoo. For simplicity, we store the data in memory. For a production MCP server, you probably want to provide data from sources like databases or APIs.

Add the following zoo MCP server source code to the server.py file (mentioned in the folder).

To deploy this, I'll need a docker file and a service account that defines IAM for security. 
The simply run command:
cd ~/mcp-on-cloudrun
gcloud run deploy zoo-mcp-server \
    --service-account=mcp-server-sa@$GOOGLE_CLOUD_PROJECT.iam.gserviceaccount.com \
    --no-allow-unauthenticated \
    --region=europe-west1 \
    --source=. \
    --labels=dev-tutorial=codelab-mcp

Here, --no-allow-unauthenticated flag to require authentication. This is important for security reasons. If you don't require authentication, anyone can call your MCP server and potentially cause damage to your system.

Since it is your first time deploying to Cloud Run from source code, you will see:
Deploying from source requires an Artifact Registry Docker repository to store built containers. A repository named
[cloud-run-source-deploy] in region [europe-west1] will be created.

