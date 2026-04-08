# Zoo Guide Agent - APAC Hackathon 2026

The Zoo Guide Agent is an AI-powered orchestrator built using the **Google ADK (Agent Development Kit)** and **Vertex AI**. It functions as a digital docent, managing animal habitats, scheduling maintenance, and providing educational insights through the **Model Context Protocol (MCP)** and Google Cloud Datastore.

## 🦒 Features
- **Habitat Orchestration**: Schedule feedings and health check-ups for zoo animals.
- **Educational Knowledge Base**: Save and retrieve species facts and enclosure notes.
- **Multi-Agent Design**: Features a 'Zoo Greeter' root agent that routes requests to a 'Habitat Coordinator' for database actions.
- **Natural Language Processing**: Converts complex user requests into structured Datastore entities.

## 🛠️ Tech Stack
- **AI Core**: Google Gemini 1.5 Pro
- **Orchestration**: Google ADK (Agent Development Kit)
- **Framework**: FastAPI / Uvicorn
- **Database**: Google Cloud Datastore (NoSQL)
- **Environment**: Google Cloud Run

## 📋 Prerequisites
1. A Google Cloud Project with **Datastore Mode** enabled.
2. Python 3.10 or higher.
3. Service Account permissions for Datastore and Cloud Run.

## ⚙️ Installation & Setup

1. **Initialize Workspace:**
   ```bash
   mkdir zoo_guide_agent && cd zoo_guide_agent

 2   terminal, enable the APIs:
 gcloud services enable \
  run.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com \
  aiplatform.googleapis.com \
  compute.googleapis.com
 3.requirements.txt file
 google-adk==1.14.0
langchain-community==0.3.27
wikipedia==1.4.0
4..env file
import os
import logging
import google.cloud.logging
from dotenv import load_dotenv

from google.adk import Agent
from google.adk.agents import SequentialAgent
from google.adk.tools.tool_context import ToolContext
from google.adk.tools.langchain_tool import LangchainTool

from langchain_community.tools import WikipediaQueryRun
from langchain_community.utilities import WikipediaAPIWrapper

import google.auth
import google.auth.transport.requests
import google.oauth2.id_token

# --- Setup Logging and Environment ---

cloud_logging_client = google.cloud.logging.Client()
cloud_logging_client.setup_logging()

load_dotenv()

model_name = os.getenv("MODEL")

# Greet user and save their prompt

def add_prompt_to_state(
    tool_context: ToolContext, prompt: str
) -> dict[str, str]:
    """Saves the user's initial prompt to the state."""
    tool_context.state["PROMPT"] = prompt
    logging.info(f"[State updated] Added to PROMPT: {prompt}")
    return {"status": "success"}

# Configuring the Wikipedia Tool
wikipedia_tool = LangchainTool(
    tool=WikipediaQueryRun(api_wrapper=WikipediaAPIWrapper())
)

# 1. Researcher Agent
comprehensive_researcher = Agent(
    name="comprehensive_researcher",
    model=model_name,
    description="The primary researcher that can access both internal zoo data and external knowledge from Wikipedia.",
    instruction="""
    You are a helpful research assistant. Your goal is to fully answer the user's PROMPT.
    You have access to two tools:
    1. A tool for getting specific data about animals AT OUR ZOO (names, ages, locations).
    2. A tool for searching Wikipedia for general knowledge (facts, lifespan, diet, habitat).

    First, analyze the user's PROMPT.
    - If the prompt can be answered by only one tool, use that tool.
    - If the prompt is complex and requires information from both the zoo's database AND Wikipedia,
        you MUST use both tools to gather all necessary information.
    - Synthesize the results from the tool(s) you use into preliminary data outputs.

    PROMPT:
    { PROMPT }
    """,
    tools=[
        wikipedia_tool
    ],
    output_key="research_data" # A key to store the combined findings
)

# 2. Response Formatter Agent
response_formatter = Agent(
    name="response_formatter",
    model=model_name,
    description="Synthesizes all information into a friendly, readable response.",
    instruction="""
    You are the friendly voice of the Zoo Tour Guide. Your task is to take the
    RESEARCH_DATA and present it to the user in a complete and helpful answer.

    - First, present the specific information from the zoo (like names, ages, and where to find them).
    - Then, add the interesting general facts from the research.
    - If some information is missing, just present the information you have.
    - Be conversational and engaging.

    RESEARCH_DATA:
    { research_data }
    """
)

tour_guide_workflow = SequentialAgent(
    name="tour_guide_workflow",
    description="The main workflow for handling a user's request about an animal.",
    sub_agents=[
        comprehensive_researcher, # Step 1: Gather all data
        response_formatter,       # Step 2: Format the final response
    ]
)

root_agent = Agent(
    name="greeter",
    model=model_name,
    description="The main entry point for the Zoo Tour Guide.",
    instruction="""
    - Let the user know you will help them learn about the animals we have in the zoo.
    - When the user responds, use the 'add_prompt_to_state' tool to save their response.
    After using the tool, transfer control to the 'tour_guide_workflow' agent.
    """,
    tools=[add_prompt_to_state],
    sub_agents=[tour_guide_workflow]
)
4.# Grant the "Vertex AI User" role to your service account
gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SERVICE_ACCOUNT" \
  --role="roles/aiplatform.user"
 https://zoo-tour-guide-1066397835614.europe-west1.run.app/dev-ui/?app=zoo_guide_agent&session=f319ca9f-d048-43ce-9fb8-4d5e0b139856
 
 <img width="1357" height="621" alt="Screenshot 2026-04-08 094422" src="https://github.com/user-attachments/assets/eefaddf4-61fc-4c18-94f6-2c1ce1354764" />
<img width="1337" height="635" alt="Screenshot 2026-04-08 094506" src="https://github.com/user-attachments/assets/56bc7363-bbd2-4ff1-b912-827af98650fb" />
<img width="1330" height="623" alt="Screenshot 2026-04-08 095212" src="https://github.com/user-attachments/assets/343cd1cf-36e9-4e98-a501-7dfed49aded7" />
