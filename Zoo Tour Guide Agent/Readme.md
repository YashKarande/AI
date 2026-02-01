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

    - Build a secure MCP server that exposes tools to LLMs via authenticated APIs
    - Design an AI agent that consumes remote tools rather than embedding logic locally
    - Demonstrate separation of concerns between reasoning (agent) and tooling (MCP server)
    - Deploy all components as serverless, scalable Cloud Run services
    - Integrate GPU-accelerated model backends for production inference
    - Validate scalability, elasticity, and system behaviour under load

## Impementation:

**Note:** The implementation does not include everything. But this overview is for understanding purposes.

### Deploying a Secure MCP Server on Cloud Run

1. First, I'll create a zoo MCP Server to provide valuable context for improving the use of LLMs with MCP, and set up a zoo MCP server with FastMCP — a standard framework for working with the Model Context Protocol. FastMCP provides a quick way to build MCP servers and clients with Python. This MCP server provides data about animals at a fictional zoo. For simplicity, we store the data in memory. For a production MCP server, you probably want to provide data from sources like databases or APIs.

Create and open a new Dockerfile for deploying to Cloud Run.

**Note:** Add the following zoo MCP server source code to the **server.py** file (mentioned in the folder). Include the **Dockerfile** (mentioned in the folder) to use the uv tool for running the server.pyfile

2. To deploy this, I'll need a docker file and a service account that defines IAM for security. 
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

3. Add the Remote MCP Server to Gemini CLI.

Once deployed, a remote MCP server, you can connect to it using various applications like Google Code Assist or Gemini CLI. In this section, we will establish a connection to your new remote MCP server using Gemini CLI.
This is for testing and Demo purpose.

For Security, we'll use IAM account permission to call the remote MCP server, and GCP credentials and project number in environment variables for use in the Gemini Settings file
    
    gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
        --member=user:$(gcloud config get-value account) \
        --role='roles/run.invoker'

    export PROJECT_NUMBER=$(gcloud projects describe $GOOGLE_CLOUD_PROJECT --format="value(projectNumber)")
    
    export ID_TOKEN=$(gcloud auth print-identity-token)


#### Results:

Start the Gemini CLI in Cloud Shell:
type "gemini" and Enter

<img width="723" height="441" alt="image" src="https://github.com/user-attachments/assets/282076cc-b2df-46db-951b-f1180bdc47f7" />

Have gemini list the MCP tools available to it within its context:
    
    /mcp

<img width="1010" height="444" alt="image" src="https://github.com/user-attachments/assets/e9a2b42c-f476-413e-9b51-256a555febda" />
<img width="1565" height="397" alt="image" src="https://github.com/user-attachments/assets/1e544ae7-9d97-4d4a-b865-ff9761968fe9" />

We have successfully deployed and connected to a secure remote MCP server. Now to phase 2.

### Build and deploy an ADK agent that uses an MCP server on Cloud Run

We will use Agent Development Kit (ADK) to build an AI agent that uses remote tools such as the MCP server created earlier. 

Previously, we created an MCP server that provides data about the animals in a fictional zoo to LLMs, for example when using the Gemini CLI. Now, we are building a tour guide agent for the fictional zoo. The agent will use the same MCP server from the previous implementation to access details about the zoo animals, and also use wikipedia to create the best tour guide experience. 

Finally, we'll deploy the tour guide agent to Google Cloud Run, so it can be accessed by all zoo visitors rather than just running locally.

What we'll do:

    - Implement a tool-using agent with google-adk.
    - Connect an agent to a remote MCP server for its toolset.
    - Deploy a Python application as a serverless container to Cloud Run.
    - Configure secure, service-to-service authentication using IAM roles.
    - Delete Cloud resources to avoid incurring future costs.

The **agent.py** in the folder contains our multi-agent system.

The file contains:
    
    - Imports and Initial Setup
    - The Agent's Capabilities
    - Defines the Specialist Agents
    - The Workflow Agent
    - root agent

1. Imports and Initial Setup
   
This first block brings in all the necessary libraries from the ADK and Google Cloud. It also sets up logging and loads the environment variables from your .env file, which is crucial for accessing your model and server URL.

2. The Agent's Capabilities
   
We define all the capabilities our agent will have, including a custom function to save data, an MCP Tool that connects to our secure MCP server along with a Wikipedia Tool.

Three Tools explained:

    - add_prompt_to_state
    This tool remembers what a zoo visitor asks. When a visitor asks, "Where are the lions?", this tool saves that specific question into the agent's memory so the other agents in the workflow know what to research.
    
    How: It's a Python function that writes the visitor's prompt into the shared tool_context.state dictionary. This tool context represents the agent's short-term memory for a single conversation. Data saved to the state by one agent can be read by the next agent in the workflow.
    
    - MCPToolset
    This is used to connect the tour guide agent to the zoo MCP server created initially. This server has special tools for looking up specific information about our animals, like their name, age, and enclosure.
    
    How: It securely connects to the zoo's private server URL. It uses get_id_token to automatically get a secure "keycard" (a service account ID token) to prove its identity and gain access.
    
    - LangchainTool
    This gives the tour guide agent general world knowledge. When a visitor asks a question that isn't in the zoo's database, like "What do lions eat in the wild?", this tool lets the agent look up the answer on Wikipedia.

    How: It acts as an adapter, allowing our agent to use the pre-built WikipediaQueryRun tool from the LangChain library.

Resources:

    - MCP Toolset (https://google.github.io/adk-docs/tools-custom/mcp-tools/)
    - Function Tools (https://google.github.io/adk-docs/tools-custom/function-tools/)
    - State (https://google.github.io/adk-docs/sessions/state/)
    
3. The Specialist Agents
   
we will define the researcher agent and response formatter agent.

    - The researcher agent is the "brain" of our operation. This agent takes the user's prompt from the shared State, examines its powerful tools (the Zoo's MCP Server Tool and the Wikipedia Tool), and decides which ones to use to find the answer.
    - The response formatter agent's role is presentation. It doesn't use any tools to find new information. Instead, it takes the raw data gathered by the Researcher agent (passed via the State) and uses the LLM's language skills to transform it into a friendly, conversational response.

4. The Workflow Agent
   
It's a SequentialAgent, a special type of agent that doesn't think for itself. Its only job is to run a list of sub_agents (the researcher and formatter) in a fixed sequence, automatically passing the shared memory from one to the next.

5. Root Agent

The ADK framework uses as the starting point for all new conversations. Its primary role is to orchestrate the overall process. It acts as the initial controller, managing the first turn of the conversation.

#### Execution:

Grant the service account the Vertex AI User role, which gives it permission to make predictions and call Google's models.

    # Grant the "Vertex AI User" role to your service account
    gcloud projects add-iam-policy-binding $PROJECT_ID \
      --member="serviceAccount:$SERVICE_ACCOUNT" \
      --role="roles/aiplatform.user"
      
You will use the adk deploy cloud_run command, a convenient tool that automates the entire deployment workflow. This single command packages your code, builds a container image, pushes it to Artifact Registry, and launches the service on Cloud Run, making it accessible on the web.

    # Run the deployment command
    uvx --from google-adk \
    adk deploy cloud_run \
      --project=$PROJECT_ID \
      --region=europe-west1 \
      --service_name=zoo-tour-guide \
      --with_ui \
      . \
      -- \
      --labels=dev-tutorial=codelab-adk \
      --service-account=$SERVICE_ACCOUNT

#### Results:

<img width="1432" height="1294" alt="image" src="https://github.com/user-attachments/assets/53e48e53-1f1d-4dd1-b560-630da9460d84" />

#### WorkFlow Explained:

1. The Zoo Greeter (The Welcome Desk)

    The entire process begins with the greeter agent.
    
    Its Job: To start the conversation. Its instruction is to greet the user and ask what animal they would like to learn about.
    
    Its Tool: When the user replies, the Greeter uses its add_prompt_to_state tool to capture their exact words (e.g., "tell me about the lions") and save them in the system's memory.
    
    The Handoff: After saving the prompt, it immediately passes control to its sub-agent, the tour_guide_workflow.

2. The Comprehensive Researcher (The Super-Researcher)

    This is the first step in the main workflow and the "brain" of the operation. Instead of a large team, you now have a single, highly-skilled agent that can access all the available information.
    
    Its Job: To analyze the user's question and form an intelligent plan. It uses the language model's powerful tool use capability to decide if it needs:
    
    Internal data from the zoo's records (via the MCP Server).
    General knowledge from the web (via the Wikipedia API).
    Or, for complex questions, both.
    Its Action: It executes the necessary tool(s) to gather all the required raw data. For example, if asked "How old are our lions and what do they eat in the wild?", it will call the MCP server for the ages and the Wikipedia tool for the diet information.

3. The Response Formatter (The Presenter)

    Once the Comprehensive Researcher has gathered all the facts, this is the final agent to run.
    
    Its Job: To act as the friendly voice of the Zoo Tour Guide. It takes the raw data (which could be from one or both sources) and polishes it.
    
    Its Action: It synthesizes all the information into a single, cohesive, and engaging answer. Following its instructions, it first presents the specific zoo information and then adds the interesting general facts.

#### The Final Result: The text generated by this agent is the complete, detailed answer that the user sees in the chat window.

