# CSE 491 / 895 - AI Agents: Homework 2

## Overview
Homework 2 continues to build upon the multi-expert AI agent system developed in the prior two assignments (Homework 0 and Homework 1). Specifically, you will implement: 

1. vector embeddings of fields in your database to enable semantic similarity matching across database content, and
2. human validation workflows for questionable AI actions.

## Preliminaries

This assignment extends Homework 1's multi-expert AI agent system. Hence, you must have successfully completed Homework 1 before beginning this assignment. Before you start, create a new folder called `Homework2` in your Personal Course Repository. Copy your entire Homework 1 codebase into this new folder before making any modifications. This ensures you preserve your Homework 1 submission while working on Homework 2.

## Specific Homework Instructions

### Step 1: Enabling Semantic Similarity Search using Vector Embeddings

**Overview**: In Homework 1, your AI agents relied on exact string matching to find relevant information in the database when answering questions or updating the resume. This created a significant limitation: if a user asked about "MSU" but your database contained "Michigan State University", the system would fail to make the connection. Vector embeddings solve this problem by converting text into numerical representations that capture semantic meaning.

**What are Vector Embeddings?**
[Vector embeddings](https://platform.openai.com/docs/guides/embeddings) are numerical representations of text that encode semantic information. Each piece of text (word, phrase, sentence, or document) is converted into a high-dimensional vector (typically 1536 dimensions for OpenAI's models) where similar concepts are positioned closer together in the vector space. For example, "Michigan State University", "MSU", and "Spartans" would all have embeddings that are mathematically similar, even though they use different words.

**OpenAI's Embedding API**
For this assignment, we'll use OpenAI's `text-embedding-3-small` model, which provides semantic representations (1536 dimentional vectors) optimized for similarity search.

**What We'll Implement in Step 1:**

<u>1.1: Embedding Service:</u> Create a service that generates vector embeddings using OpenAI's API

<u>1.2: Database Enhancement:</u> Add embedding vector columns to each of our existing tables using PostgreSQL's [pgvector extension](https://github.com/pgvector/pgvector).

<u>1.3: Database Integration:</u> Enhance your AI agents to perform semantic similarity search.


#### 1.0 Prerequisites:

**Overview**: Before implementing vector embeddings, you need to prepare your development environment to support the necessary technologies. This involves updating your Docker configuration to use PostgreSQL with the pgvector extension, and adding required Python dependencies.



**Implementation Guidance**:

1. **Update `.env` file**:
   Change the PostgreSQL version to use pgvector-enabled PostgreSQL:
   ```env
   POSTGRES_VERSION=pgvector/pgvector:pg15  # Change from '13' to 'pgvector/pgvector:pg15'
   ```

2. **Update `requirements.txt`**:
   Add the pgvector Python client:
   ```
   pgvector
   ```

3. **Update `init-db.sh`**:
   Add pgvector extension to your database initialization script:
   ```bash
   # Enable pgvector extension
   echo "Enabling pgvector extension..."
   PGPASSWORD=${DATABASE_PASSWORD} psql -h ${DATABASE_HOST} -U ${DATABASE_USER} -d ${DATABASE_NAME} -c "CREATE EXTENSION IF NOT EXISTS vector;"
   ```

4. **Update `docker-compose.yml`**:
   Change the image references to use `${POSTGRES_VERSION}` directly (not `postgres:${POSTGRES_VERSION}`):
   ```yaml
   postgres:
     image: ${POSTGRES_VERSION}  # Changed from postgres:${POSTGRES_VERSION}
   
   db-init:
     image: ${POSTGRES_VERSION}  # Changed from postgres:${POSTGRES_VERSION}
   ```

**Testing**:

With your updated configuration, restart your Docker environment (Since your database was previously built with Postgres 13, you may need to remove the existing volumes before rebuilding to prevent version conflicts. You can do this by running: `docker-compose down -v`.)

```bash
# Rebuild and start with the new configuration
docker compose up --build
```

Check that the pgvector extension is loaded:
```bash
docker compose exec postgres psql -U postgres -d db -c "SELECT * FROM pg_extension WHERE extname = 'vector';"
```

The command should return one row, confirming the pgvector extension is properly installed.




#### 1.1 Vector Embedding Service

**Overview**: In this section, you'll create a simple service that converts text into numerical vectors called embeddings. These embeddings capture the semantic meaning of text as 1536-dimensional vectors, where similar concepts have similar numerical representations. For example, "MSU" and "Michigan State University" will have very similar embedding vectors, even though they use different words.


**Implementation Guidance**:

Create `flask_app/utils/embeddings.py`:
```python
# Author: Prof. MM Ghassemi <ghassem3@msu.edu>

import os
import requests

def generate_embedding(text: str):
    """Generate embedding for text using OpenAI API"""
    if not text or not text.strip():
        return [0.0] * 1536
    
    try:
        response = requests.post(
            "https://api.openai.com/v1/embeddings",
            headers={
                "Authorization": f"Bearer {os.getenv('OPENAI_API_KEY')}",
                "Content-Type": "application/json"
            },
            json={
                "model": "text-embedding-3-small",
                "input": text.strip()
            }
        )
        return response.json()['data'][0]['embedding']
    except:
        return [0.0] * 1536
```


#### 1.2 Modify Existing Table Creation Statements

**Overview**: Now that you have an embedding service, you need to modify your database schema to store vector embeddings. You'll add a vector column to each existing table and create indexes for fast similarity searches. This transforms your database from exact string matching to semantic similarity search capabilities.



**Implementation Guidance**:

Modify each of the .sql files in the `flask_app/database/create_tables` directory to include:

1. <u>A Vector Column:</u> Add `embedding vector(1536) DEFAULT NULL` to each table. This stores the 1536-dimensional vector representation of each row's text content. Initially NULL until embeddings are generated programmatically

2. <u>An Index</u>: Create an index for similarity search using `CREATE INDEX ... USING ivfflat (embedding vector_cosine_ops)`. Why do we need an index? Without an index, similarity searches would require scanning every row in the table, which becomes very slow as your database grows. The ivfflat index organizes vectors in a way that makes similarity searches much faster. Learn more: [pgvector Index Types](https://github.com/pgvector/pgvector#indexing)

**Example implementation for `institutions.sql`**:

```sql
CREATE TABLE IF NOT EXISTS institutions (
inst_id        SERIAL PRIMARY KEY,
type           varchar(100)  NOT NULL,
name           varchar(100)  NOT NULL,
department     varchar(100)  DEFAULT NULL,
address        varchar(100)  DEFAULT NULL,
city           varchar(100)  DEFAULT NULL,
state          varchar(100)  DEFAULT NULL,
zip            varchar(10)   DEFAULT NULL,
embedding      vector(1536)  DEFAULT NULL
);

-- Create index for similarity search
CREATE INDEX IF NOT EXISTS institutions_embedding_idx ON institutions USING ivfflat (embedding vector_cosine_ops);
```

#### 1.3 Programatically generate embeddings on insert/modify

**Overview**: After updating your schema, you need to modify any database functions that insert or update rows in `flask_app/utils/database.py` to generate the embeddings associated with those rows just-in-time for the update! This includes:
- the `insertRows` method (for new data)
- Any other functions you authored in Homework 1 that are responsible for data modifications (e.g. update)

**Understanding the Workflow**: When a user asks a question, your orchestrator will need to:

1. Analyze the query to identify which database tables are relevant
2. Decide on search strategy - exact matching vs. semantic similarity (See step 1.4)
3. Extract embedding targets - determine what specific terms/phrases from the user input need embeddings
4. Generate embeddings using `generate_embedding()`
5. Execute appropriate queries using either traditional SQL or pgvector operators
6. Synthesize results from multiple sources if needed

**Implementation Guidance**:

1. **Import the embedding function**: Import the `generate_embedding` function from `flask_app/utils/embeddings.py` in your `database.py` file.

2. **Choose which columns to embed**: For each table, combine only the most important text fields into a single embedding:
   - **institutions**: `name` + `department` 
   - **experiences**: `name` + `description`
   - **skills**: `name` only
   - **positions**: `title` + `responsibilities`
   - **users**: `email` only

3. **Generate embeddings**: Use `generate_embedding()` to convert the combined text fields into a vector.

4. **Include the embedding**: Add the embedding vector to the insert/update operation.


#### 1.4 Update AI Agents to use Semantic Search

**Overview**: Now that your database has vector embeddings and your embedding service is working, you need to update your AI agents from Homework 1 (specifically, the orchestrator and database read agent) to effectively use semantic similarity queries! This involves a more complex decision-making process where your orchestrator must determine: (i) which tables are needed to answer the question, (ii) whether to use exact matching (WHERE clauses) or semantic similarity (pgvector operators), and (iii) what specific parts of the user input need to be embedded if we are doing a semantic similarity search. Because this orchestration is more complex than Homework 1's single-step approach, we'll implement the ReAct pattern within the orchestrator to break down the decision-making into sequential reasoning and action steps.


**Implementation Guidance**:

**Implement ReAct Pattern for Orchestrator**: Update your orchestrator to use the ReAct pattern for complex semantic search decisions. The ReAct pattern alternates between:
- Thought: Reasoning about what to do next
- Action: Taking a specific action (API call, database query, etc.)
- Observation: Analyzing the results of the action

I suggest you implement the ReAct orchestrator in `flask_app/utils/llm.py` by modifying the `handle_ai_chat_request` function. Learn more about ReAct: [ReAct: Synergizing Reasoning and Acting in Language Models](https://arxiv.org/abs/2210.03629)

<u>[Optional] Use LangChain</u>: You're welcome to use LangChain for implementing the ReAct agent with tool calling (embedding endpoint + database queries) and multi-agent orchestration.

**Update Database Read Agent Prompts**: Update your database read agent's prompt to support pgvector SQL queries for semantic similarity search. More specifically, the agent will need to learn how to use the `<=>` operator and similarity scoring methods. For comprehensive SQL patterns and examples for semantic similarity search, refer to the [official pgvector documentation](https://github.com/pgvector/pgvector)):

**Testing the Updated Agents**:

Test your updated system with complex queries that require multi-step reasoning; examples to test follow:
   - "Find my MSU experience" → Should find "Michigan State University"
   - "What AI skills do I have?" → Should find skills like "machine learning", "deep learning".

### Step 2: Human Validation Workflow

**Overview**: Your AI chat system currently processes all user messages without oversight. This step adds validation to intercept potentially dangerous requests (like data deletion) and ask for human confirmation before proceeding.

**What We'll Implement in Step 2:**

<u>2.1: Validation System:</u> Implement message risk assessment and session-based confirmation workflow

<u>2.2: Action Execution:</u> Process validated messages or refuse based on user confirmation

#### 2.1 Validation System Implementation

**Overview**: Add validation logic to your existing chat system to intercept dangerous user messages before they reach the LLM.

**How the Validation System Works**: When a user sends a message, your system first checks if there's a pending validation response waiting. If not, it analyzes the new message for dangerous keywords. If the message is risky, it stores the original message in the session and asks the user for confirmation via SocketIO. When the user responds "yes" or "no", the system either processes the original message through the LLM or cancels it entirely. This creates a safety layer that prevents potentially harmful requests from being executed without human oversight.

**Note on Flask Sessions**: Flask sessions are essential for this validation workflow because they allow you to store the original dangerous message between the user's initial request and their "yes"/"no" response. Without sessions, you would lose the original message when the user responds, making it impossible to process their validated request. Sessions persist data across multiple HTTP requests for the same user, enabling the validation conversation flow. For more details, see the [Flask Sessions Documentation](https://flask.palletsprojects.com/en/2.3.x/quickstart/#sessions).

**Implementation Guidance**:

1. **Create `assess_message_risk()` function** in `flask_app/utils/llm.py`:
   - Takes a user message string as input
   - Returns `True` if message contains dangerous keywords (delete, remove, clear, drop, destroy)
   - Use case-insensitive string matching to detect risky intent

2. **Create `request_human_validation()` function** in `flask_app/utils/llm.py`:
   - Store the original message in Flask session under `'pending_validation'` key
   - Emit a validation request message to the user via SocketIO using `process_and_emit_message()`
   - Return a status indicating validation is pending

3. **Update `handle_ai_chat_request()` function** in `flask_app/utils/llm.py`:
   - **First**: Check if session contains `'pending_validation'` data
   - **If validation pending**: Process user's "yes"/"no" response and clear session state
   - **If no validation pending**: Check if new message is risky using `assess_message_risk()`
   - **If risky**: Call `request_human_validation()` and return early (don't process with LLM)
   - **If safe**: Continue with normal LLM processing

**Testing the Complete Validation System**:

Test your complete validation workflow:

1. **Dangerous Delete**: "Delete all my experiences" → Should trigger validation, show warning, require yes/no confirmation
2. **Safe Query**: "Show my MSU experience" → Should execute normally without validation

#### 2.2 Action Execution Implementation

**Overview**: Handle user validation responses and execute or cancel the original message accordingly.

**Implementation Guidance**:

In your updated `handle_ai_chat_request()` function, when processing validation responses:

1. **"Yes" Response**: 
   - Retrieve the original message from session
   - Process it normally through the LLM
   - Emit confirmation message to user
   - Clear the session state

2. **"No" Response**:
   - Emit cancellation message to user
   - Clear the session state
   - Return without processing the original message

3. **Invalid Response**:
   - Emit reminder to respond with 'yes' or 'no'
   - Keep validation state active (don't clear session)

**Testing Action Execution**:

Test both scenarios:
1. **Approved**: "Delete my data" → "yes" → LLM processes the request
2. **Rejected**: "Delete my data" → "no" → Request cancelled, no LLM processing

### Step 3: Review the Assignment Rubric

Carefully review the [rubric](documentation/rubric.md) to understand exactly how you will be graded. The rubric outlines the functional requirements that must be demonstrated in a demo video and submitted via the homework submission form.

### Step 4: Demo Video Recording

Using a video recording tool of your choice, create a video with the following structure:

- **Introduction**:
  - State your full name
  - Say: "I will now demonstrate the functional requirements for CSE 491 Homework 2"

- **For Each Functional Requirement**:
  - **State the requirement**: Clearly announce the rubric criteria you are about to demonstrate
  - **Show the functionality**: Demonstrate the rubric requirement clearly and unambiguously in the video.
  
**IMPORTANT**: Please keep the video as short as possible, and ensure clear audio and video quality.

### Step 5: Submit your code

#### Submit Homework 2 Code

1. Ensure all your Homework 2 files are in the `Homework2` folder within your Personal Course Repository.

2. Submit your assignment by navigating to the main directory of your Personal Course Repository and pushing your repo to GitHub:

   ```bash
   git add .
   git commit -m 'submitting Homework 2'
   git push
   ```

3. You have now submitted Homework 2; you can run the same commands to re-submit anytime before the deadline. Please check that your submission was successfully uploaded by navigating to the `Homework2` directory in your Personal Course Repository online.

### Step 6: Submit your Demo Video and Survey

Submit your video and complete the homework survey using [this Google Form](https://docs.google.com/forms/d/e/1FAIpQLScrZmoZfBHr8ZAwoytaXxtFbTQrMzst2VDWC-RB3EQ6UXC94Q/viewform?usp=header).

