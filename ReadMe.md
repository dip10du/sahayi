# Building Sahayi: A Compassionate AI Assistant for Senior Citizens Using Google ADK

## Introduction | Overview

As our population ages, many senior citizens face challenges navigating digital services and accessing emergency assistance quickly. In critical moments—whether facing a medical emergency or safety concern—every second counts, yet complexity in technology can create barriers when help is needed most.

**Sahayi** (meaning "helper" in Sanskrit) is a compassionate AI-powered digital assistant designed specifically for senior citizens. Built using Google's Agent Development Kit (ADK), Sahayi provides:

- **Emergency Response**: Immediate access to nearest police stations and medical facilities
- **Personalized Assistance**: Retrieves user profiles to provide context-aware help
- **Compassionate Interaction**: Uses simple, warm language designed for older adults
- **Automated Notifications**: Alerts emergency contacts when help is needed

**Target Audience**: Developers interested in building AI agents with Google ADK, particularly those focused on healthcare, eldercare, or emergency response applications.

**What You'll Learn**: By the end of this blog, you'll understand how to build a multi-agent system using Google ADK, integrate custom tools with search capabilities, and design compassionate conversational flows for vulnerable user groups.

## Design

Sahayi uses a **multi-agent orchestration architecture** powered by Google's Gemini 2.5 Flash model. The system consists of four specialized agents working together:

### Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│               Cloud Run Environment #1: Sahayi App               │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              Root Agent (Sahayi)                       │    │
│  │        Orchestrator & User Interface                   │    │
│  │   - Greets user and collects name                      │    │
│  │   - Confirms location                                  │    │
│  │   - Routes to appropriate specialist agent             │    │
│  └──────────┬─────────────┬────────────────┬─────────────┘    │
│             │             │                │                   │
│             ▼             ▼                ▼                   │
│      ┌──────────┐  ┌──────────┐    ┌──────────────┐          │
│      │  User    │  │  Police  │    │   Medical    │          │
│      │ Profiler │  │ Locator  │    │   Locator    │          │
│      │  Agent   │  │  Agent   │    │    Agent     │          │
│      └──────────┘  └──────────┘    └──────────────┘          │
│           │              │                 │                   │
└───────────┼──────────────┼─────────────────┼───────────────────┘
            │              │                 │
            │              ▼                 ▼
            │         Google Search    Google Search
            │
            │ HTTPS/REST
            ▼
┌──────────────────────────────────────────────────────────────────┐
│         Cloud Run Environment #2: MCP Toolbox Server             │
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │         Toolbox Sync Client (MCP Server)               │    │
│  │    - Exposes 'search-contact-by-name' tool             │    │
│  │    - Handles user profile lookups                      │    │
│  └────────────────────┬───────────────────────────────────┘    │
│                       │                                         │
│                       │ Authenticated Connection                │
│                       ▼                                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │           Google Secret Manager                        │    │
│  │    - Stores Cloud SQL credentials                      │    │
│  │    - Manages database connection secrets               │    │
│  └────────────────────┬───────────────────────────────────┘    │
│                       │                                         │
│                       │ Secured Connection                      │
│                       ▼                                         │
│  ┌────────────────────────────────────────────────────────┐    │
│  │              Cloud SQL Database                        │    │
│  │    - User profiles (name, address, medical history)    │    │
│  │    - Emergency contacts                                │    │
│  │    - Location data                                     │    │
│  └────────────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────┘
```

### Design Rationale

**Why Multi-Agent Architecture?**
- **Separation of Concerns**: Each agent has a single, focused responsibility
- **Tool Compatibility**: ADK requires homogeneous tool types within agent hierarchies
- **Scalability**: Easy to add new specialist agents (reminders, medicine tracking, etc.)

**Why AgentTool Wrapper Pattern?**
Google ADK has a critical constraint: "Multiple tools are supported only when they are all search tools." Since we need both custom tools (user profiler) and search tools (location services), we wrap each specialist agent as an `AgentTool`, allowing the root agent to orchestrate them without tool type conflicts.

**Why Two Separate Cloud Run Environments?**
1. **Sahayi App (Cloud Run #1)**: Hosts the main AI agent application with conversational interface
2. **MCP Toolbox Server (Cloud Run #2)**: Independently scalable microservice hosting custom tools
   - **Benefits**: 
     - Independent scaling and versioning
     - Reusable toolbox across multiple applications
     - Better security isolation (database credentials only in toolbox environment)
     - Easier maintenance and updates

**Why Secret Manager for Database Credentials?**
- **Security**: Cloud SQL credentials never hardcoded in application code
- **Access Control**: Fine-grained IAM permissions for who can access database secrets
- **Rotation**: Easy credential rotation without redeploying applications
- **Audit Trail**: All secret access is logged for compliance

**Why Cloud SQL for User Profiles?**
- **Structured Data**: User profiles, medical history, and emergency contacts fit relational schema
- **ACID Compliance**: Critical for healthcare and emergency contact data integrity
- **Managed Service**: Automatic backups, high availability, and security patches

**Why Name-First Flow?**
For senior citizens experiencing emergencies, having their profile information (medical history, emergency contacts, location) ready before routing to specialists ensures faster, more personalized assistance.

## Prerequisites

Before you begin, ensure you have:

### Software Requirements
- **Python 3.8+** installed
- **Google Cloud Project** with the following APIs enabled:
  - Agent Development Kit (ADK)
  - Cloud Run
  - Cloud SQL
  - Secret Manager
  - Gemini API
- **Google Cloud CLI** configured with appropriate permissions
- **Toolbox Core** for custom tool integration

### Required Python Packages
```bash
pip install google-adk toolbox-core
```

### Google Cloud Services Setup
1. **Cloud Run** (2 services):
   - Service 1: Sahayi agent application
   - Service 2: MCP Toolbox server
2. **Cloud SQL**: PostgreSQL instance for user profiles and emergency contacts
3. **Secret Manager**: Store database credentials securely
4. **IAM Permissions**:
   - Cloud Run Invoker role for service-to-service communication
   - Secret Manager Secret Accessor for toolbox service
   - Cloud SQL Client for database connections

### Knowledge Prerequisites
- Basic understanding of Python
- Familiarity with agent-based systems (helpful but not required)
- Basic Google Cloud concepts (projects, authentication, IAM)
- Understanding of microservices architecture
- Basic SQL for database schema design

### API Access
- Google Cloud Project with Gemini API enabled
- Toolbox service endpoint (will be your Cloud Run URL after deployment)

## Step-by-Step Instructions

### Step 1: Set Up Cloud SQL Database

Create a Cloud SQL PostgreSQL instance:
```bash
gcloud sql instances create sahayi-user-db \
  --database-version=POSTGRES_15 \
  --tier=db-f1-micro \
  --region=us-central1

# Create database
gcloud sql databases create user_profiles \
  --instance=sahayi-user-db

# Create user with password
gcloud sql users create sahayi_app \
  --instance=sahayi-user-db \
  --password=YOUR_SECURE_PASSWORD
```

**Sample Schema**:
```sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  name VARCHAR(255) NOT NULL,
  home_address TEXT,
  phone VARCHAR(20),
  emergency_contact_name VARCHAR(255),
  emergency_contact_phone VARCHAR(20),
  medical_conditions TEXT[],
  medications TEXT[],
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO users (name, home_address, phone, emergency_contact_name, 
                   emergency_contact_phone, medical_conditions, medications)
VALUES ('Robert', '123 Oak Street, Springfield', '(555) 111-2222',
        'Sarah (Daughter)', '(555) 333-4444',
        ARRAY['Hypertension', 'Type 2 Diabetes'],
        ARRAY['Metformin', 'Lisinopril']);
```

### Step 2: Store Database Credentials in Secret Manager

```bash
# Create secret for database password
echo -n "YOUR_SECURE_PASSWORD" | gcloud secrets create sahayi-db-password \
  --data-file=- \
  --replication-policy="automatic"

# Grant Secret Manager access to Cloud Run service account
gcloud secrets add-iam-policy-binding sahayi-db-password \
  --member="serviceAccount:YOUR_PROJECT_NUMBER-compute@developer.gserviceaccount.com" \
  --role="roles/secretmanager.secretAccessor"
```

**Why Secret Manager?**: Database credentials are encrypted at rest, versioned, and audited. The MCP Toolbox service retrieves credentials at runtime without ever storing them in code.

### Step 3: Deploy MCP Toolbox Server to Cloud Run

Create the toolbox server with Cloud SQL connection:
```bash
# Deploy toolbox server (assumes you have toolbox server code)
gcloud run deploy toolbox-server \
  --image=gcr.io/YOUR_PROJECT/toolbox-server:latest \
  --region=us-central1 \
  --platform=managed \
  --add-cloudsql-instances=YOUR_PROJECT:us-central1:sahayi-user-db \
  --set-secrets=DB_PASSWORD=sahayi-db-password:latest \
  --set-env-vars="DB_HOST=/cloudsql/YOUR_PROJECT:us-central1:sahayi-user-db,DB_USER=sahayi_app,DB_NAME=user_profiles" \
  --allow-unauthenticated
```

**Note your Cloud Run URL**: This will be your `TOOLBOX_URL` (e.g., `https://toolbox-server-xxxxx.run.app`)

### Step 4: Set Up Project Structure for Sahayi Agent

Create your project directory:
```bash
mkdir sahayi
cd sahayi
touch agent.py
```

### Step 5: Import Dependencies and Initialize Toolbox

```python
from google.adk.agents import Agent
from google.adk.tools import AgentTool
from toolbox_core import ToolboxSyncClient
from google.adk.tools import google_search

# Initialize toolbox client pointing to your Cloud Run toolbox service
toolbox = ToolboxSyncClient("https://toolbox-server-xxxxx.run.app/")
tools = [toolbox.load_tool('search-contact-by-name')]
```

**Why this matters**: The Toolbox provides custom business logic (user profile lookup from Cloud SQL) that complements Google's built-in search capabilities. The two Cloud Run services communicate via HTTPS.

### Step 6: Create the Police Locator Agent

```python
police_locator_agent = Agent(
    name="nearest_police_locator",
    model="gemini-2.5-flash",
    description=(
        "This agent helps locate the nearest police station when a user "
        "may be in danger or needs law enforcement support."
    ),
    instruction="""
        You receive inputs such as the citizen's location, address, or landmark.
        Your job is to search and return the nearest police station with 
        address, phone number, and distance when possible.
        Respond only with relevant safety details.
        
        After providing the police station details, gracefully mention:
        "Please don't worry, help is on the way. We will also be reaching 
        out to your emergency contacts to let them know you need assistance. 
        They will be informed shortly."
        
        If information is incomplete, ask for the nearest known landmark or area.
        Do not provide assumptions or unrelated information.
    """.strip(),
    tools=[google_search]
)
```

**Key Design Choice**: The compassionate reassurance message is embedded in the agent's instructions, ensuring consistent emergency communication.

### Step 7: Create the Medical Locator Agent

```python
medical_locator_agent = Agent(
    name="nearest_medical_locator",
    model="gemini-2.5-flash",
    description=(
        "This agent helps locate the nearest hospital, ambulance, or "
        "emergency clinic in case of medical concerns."
    ),
    instruction="""
        You receive inputs such as the citizen's location, address, or landmark.
        Your job is to search and return the nearest hospital, emergency room, 
        or urgent care facility with address, phone number, and distance 
        when possible.
        Prioritize emergency-ready services such as hospitals with ER, 
        ambulance services, or urgent care centers.
        
        After providing the facility details, gracefully mention:
        "We will also be sharing your previous medical conditions with them 
        so they can provide you with the best care. They will reach out to 
        you shortly."
        
        Respond only with relevant medical facility details.
        If information is incomplete, ask for the nearest known landmark or area.
        Do not provide assumptions or unrelated information.
    """.strip(),
    tools=[google_search]
)
```

**Medical Context Sharing**: Automatically informing healthcare providers about existing medical conditions ensures better emergency care.

### Step 8: Create the User Profiler Agent

```python
user_profiler_agent = Agent(
    name="user_profiler_agent",
    model="gemini-2.5-flash",
    description=(
        "This agent retrieves user details based on their name to provide "
        "personalized assistance."
    ),
    instruction=(
        "You receive a user's name as input. "
        "Use the `search-contact-by-name` tool to find the user's details. "
        "If found, return the user's information. "
        "If not found, indicate that the user is not in the system."
    ),
    tools=tools
)
```

### Step 9: Wrap Agents as AgentTools

```python
police_tool = AgentTool(agent=police_locator_agent)
medical_tool = AgentTool(agent=medical_locator_agent)
user_tool = AgentTool(agent=user_profiler_agent)
```

**Critical Step**: This wrapper pattern solves the ADK tool compatibility constraint, allowing mixed tool types in the orchestrator.

### Step 10: Create the Root Orchestrator Agent

```python
root_agent = Agent(
    name="sahayi_agent",
    model="gemini-2.5-flash",
    description=(
        "You are the Orchestrator Agent for a Senior Citizen Digital Helper. "
        "Your job is to understand the user's message and route it to the "
        "correct internal agent. Always respond clearly, calmly, and in "
        "simple language suitable for older adults."
    ),
    instruction="""
        You are a compassionate assistant for senior citizens. 
        Always prioritize their comfort and safety.

        STEP 1 - Greeting and Name Collection (DO THIS FIRST):
        If the user's name is not known, warmly greet them and ask:
        "Hello! I'm here to help you. May I know your name so I can assist 
        you better?"

        If the user seems confused or hesitant, reassure them gently:
        "Don't worry, I'm here to help. What would you like me to call you?"

        Once the name is provided, acknowledge it warmly and with care:
        "Thank you, [name]. It's wonderful to talk with you. I'm here to help 
        you with anything you need."

        STEP 2 - Get User Details and Confirm Location:
        After you have the user's name, use the `user_tool` to get their details. 
        Once you have their information, gently confirm their current location 
        using their home address (not latitude/longitude):
        "Just to make sure I can help you properly, are you currently at 
        [home address from their profile]? Or are you somewhere else right now?"

        If they are at a different location, kindly ask:
        "Could you tell me your current address or location? This will help 
        me assist you better."

        STEP 3 - Route to Appropriate Agent:
        After confirming their identity and location, understand what they need:
           - Police Locator Agent (`police_tool`) - If they mention danger, 
             assault, theft, suspicious activity, feeling unsafe, or personal threat.
           - Medical Locator Agent (`medical_tool`) - If they mention illness, 
             injury, breathing issues, chest pain, fall, bleeding, confusion, 
             dizziness, weakness, or inability to move.
           - For other requests (reminders, bills, medicine, messages, cab booking), 
             acknowledge warmly and let them know you're working on adding that support.

        IMPORTANT - Emergency Handling:
        If at ANY point the user mentions symptoms or situations that could be 
        emergencies (medical or safety), IMMEDIATELY:
        1. Stay calm and reassuring: "I understand, [name]. Let me help you right 
           away. Please don't worry, I'm here for you."
        2. Confirm their current location if not already done
        3. Route to the appropriate emergency agent (police_tool or medical_tool)
        4. Gracefully inform them: "I'm finding the nearest [police station/hospital] 
           for you right now. We'll also be reaching out to your emergency contacts 
           to let them know you need help."

        Always speak with warmth, patience, and compassion. Use simple, clear language.
    """.strip(),
    tools=[user_tool, police_tool, medical_tool]
)
```

**Instruction Design Philosophy**: 
- **Sequential steps** guide the conversation flow
- **Emergency detection** embedded at every stage
- **Compassionate language** reduces anxiety in stressful situations
- **Plain addresses** instead of coordinates for senior-friendly communication

### Step 11: Test Locally

Run the agent locally with ADK's web interface:
```bash
adk web sahayi/agent.py
```

This starts a local development server where you can test the conversation flow before deploying.

### Step 12: Deploy Sahayi Agent to Cloud Run

Set environment variables:
```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export SERVICE_NAME="sahayi"
export APP_NAME="sahayi"
export AGENT_PATH="sahayi/agent.py"
```

Deploy to Google Cloud for production use:
```bash
adk deploy cloud_run \
  --project=$GOOGLE_CLOUD_PROJECT \
  --region=$GOOGLE_CLOUD_LOCATION \
  --service_name=$SERVICE_NAME \
  --app_name=$APP_NAME \
  --with_ui $AGENT_PATH
```

**What happens during deployment?**
1. ADK builds a containerized version of your agent
2. Pushes the container to Google Container Registry
3. Deploys to Cloud Run with automatic scaling
4. Creates a web UI for interacting with the agent
5. Returns a public URL (e.g., `https://sahayi-xxxxx.run.app`)

**Architecture Summary After Deployment**:
- **Cloud Run Service 1** (toolbox-server): Handles database queries via Secret Manager → Cloud SQL
- **Cloud Run Service 2** (sahayi): Hosts the AI agent, calls toolbox-server for user profiles, uses Google Search for locations

## Result / Demo

### Sample Interaction Flow

**User**: "Hi"

**Sahayi**: "Hello! I'm here to help you. May I know your name so I can assist you better?"

**User**: "My name is Robert"

**Sahayi**: "Thank you, Robert. It's wonderful to talk with you. I'm here to help you with anything you need. Just to make sure I can help you properly, are you currently at 123 Oak Street, Springfield? Or are you somewhere else right now?"

**User**: "Yes, I'm at home. I'm having chest pain and feeling dizzy"

**Sahayi**: "I understand, Robert. Let me help you right away. Please don't worry, I'm here for you. I'm finding the nearest hospital for you right now. We'll also be reaching out to your emergency contacts to let them know you need help."

*[Routes to medical_locator_agent]*

**Medical Locator**: "Here are the nearest emergency facilities:

1. **Springfield General Hospital Emergency Room**
   - Address: 456 Medical Drive, Springfield
   - Phone: (555) 123-4567
   - Distance: 1.2 miles
   - 24/7 Emergency Services Available

2. **Urgent Care Plus**
   - Address: 789 Health Avenue, Springfield
   - Phone: (555) 234-5678
   - Distance: 0.8 miles
   - Open until 10 PM

We will also be sharing your previous medical conditions with them so they can provide you with the best care. They will reach out to you shortly."

### Key Outcomes
✅ **Name-first personalization** before emergency routing  
✅ **Location confirmation** using familiar home address  
✅ **Immediate emergency detection** based on symptoms  
✅ **Compassionate reassurance** throughout the interaction  
✅ **Automated contact notification** (communicated to user)  
✅ **Medical history sharing** for better care  

### Infrastructure Highlights
✅ **Two Cloud Run Services**: Independent scaling and security isolation  
✅ **Secret Manager Integration**: Database credentials never exposed in code  
✅ **Cloud SQL Backend**: Reliable, structured storage for user profiles  
✅ **Service-to-Service Communication**: Toolbox microservice architecture  
✅ **Automatic Scaling**: Both services scale based on demand  
✅ **Built-in UI**: ADK provides web interface out of the box  

## What's Next?

### Expand Sahayi's Capabilities
- **Medicine Reminders**: Integrate with calendar and notification systems
- **Bill Payment Assistance**: Connect to banking APIs for senior-friendly transactions
- **Transportation Booking**: Add cab/ride-sharing integration for medical appointments
- **Fall Detection**: Integrate with wearable devices for automatic emergency triggers

### Enhance the Architecture
- **Voice Interface**: Add speech-to-text and text-to-speech for hands-free operation
- **Multilingual Support**: Expand to regional languages for broader accessibility
- **Real-time Contact Notification**: Actually trigger SMS/email to emergency contacts
- **Integration with Healthcare Systems**: Fetch real-time medical records with consent

### Related Resources
- [Google ADK Documentation](https://cloud.google.com/adk)
- [Building Multi-Agent Systems](https://cloud.google.com/blog/topics/developers-practitioners/building-multi-agent-systems)
- [Gemini API for Healthcare](https://cloud.google.com/healthcare-api)

### Challenge Yourself
Try building a "medication conflict checker" agent that analyzes prescribed medicines and warns about dangerous drug interactions using medical knowledge bases.

## Call to Action

To learn more about Google Cloud services and to create impact for the work you do, get around to these steps right away:

• Register for [Code Vipassana sessions](https://rsvp.withgoogle.com/events/cv)  
• Join the meetup group [Datapreneur Social](https://www.meetup.com/datapreneur-social/)  
• Sign up to become [Google Cloud Innovator](https://cloud.google.com/innovators?utm_source=cloud_sfdc&utm_medium=email&utm_campaign=FY23-1H-vipassana-innovators&utm_content=joininnovators&utm_term=-)

---

**Tags**: #GoogleADK #AI #AgentDevelopment #Eldercare #EmergencyResponse #Gemini #MultiAgentSystems #CompassionateAI
