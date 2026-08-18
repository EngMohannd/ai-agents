# CSE 491 / 895 - AI Agents: Homework 3

## Overview
Homework 3 continues to extend the multi-expert AI agent system you developed in the prior assignments (Homework 0, 1, and 2) by integrating web crawling capabilities and Agent-to-Agent (A2A) Protocol. More specifically, you will:

1. Develop a specialized `web crawling agent` that analyzes content from URLs (such as those found in the `experiences` table)
2. Use Agent-to-Agent Protocol (A2A) for structured communication between your `orchestrator` and the new web crawling agent
3. Implement a `documents` database table to store chunked and embedded web content for selective context retrieval
4. Enhance your final application to neatly integrate web-scraped information with your existing resume data

## Learning Objectives
- Design and implement a specialized web crawling agent for analyzing web content
- Implement Agent-to-Agent Protocol for structured inter-agent communication
- Create a documents database table for storing chunked and embedded web content
- Design intelligent orchestration logic that determines when web crawling is necessary
- Integrate web-scraped content with existing database and semantic search capabilities
- Apply content analysis techniques to extract meaningful information from web sources

## Preliminaries

This assignment extends Homework 2. Hence, you must have successfully completed Homework 2 before beginning this assignment. Before you start, create a new folder called `Homework3` in your Personal Course Repository. Copy your entire Homework 2 codebase into this new folder before making any modifications. This ensures you preserve your Homework 2 submission while working on Homework 3. **Note**: Homework 2 already includes tools for vector embeddings & semantic similarity search; this feature will be leveraged in Homework 3 to facilitate the enhanced web content analysis and validation.

**Required Dependencies**: Update your `requirements.txt` file to include the following packages needed for web crawling:

```
requests
beautifulsoup4
```

## Specific Homework Instructions

### Step 1: Agent-to-Agent Protocol Implementation

**Overview**: Create a simplified Agent-to-Agent (A2A) Protocol for structured communication between agents. This protocol will enable your orchestrator to communicate with the web crawling agent in a structured, reliable manner.

**What is Agent-to-Agent Protocol?**
[Agent-to-Agent (A2A) Protocol](https://developers.googleblog.com/en/a2a-a-new-era-of-agent-interoperability/) is a communication framework that allows different AI agents to exchange structured messages. In this assignment, we'll implement a simplified version that enables your orchestrator to request web crawling services from a specialized web crawler agent.

**What We'll Implement in Step 1:**

<u>1.1: A2A Protocol Classes:</u> Create message and protocol classes for structured agent communication

<u>1.2: Protocol Testing:</u> Verify the A2A protocol works correctly with basic message passing

This foundation will enable your orchestrator to coordinate with the web crawling agent using standardized message formats and response handling.

#### 1.1 Create the A2A Protocol Classes

Create `flask_app/utils/a2a_protocol.py` with the following simplified implementation of A2A protocol:

```python
import uuid
from typing import Dict, Any

class A2AMessage:
    def __init__(self, sender: str, recipient: str, action: str, params: Dict[str, Any]):
        self.message_id = str(uuid.uuid4())		# Generate unique ID for this message
        self.sender = sender									# Who sent this message
        self.recipient = recipient 						# Who should receive this message
        self.action = action									# What action to perform (e.g., "crawl_url")
        self.params = params 									# Parameters for the action (e.g., {"url": "https://example.com"})

class A2AProtocol:
    def __init__(self):
        self.pending_requests = {}            # Track requests that haven't been responded to yet
    
    def send_request(self, sender: str, recipient: str, action: str, params: Dict[str, Any]) -> str:
        message = A2AMessage(sender, recipient, action, params) # Create a new message
        self.pending_requests[message.message_id] = message     # Store it as pending until we get a response
        return message.message_id                               # Return the message ID so we can match responses
    
    def send_response(self, message_id: str, sender: str, recipient: str, result: Any) -> A2AMessage:
        response = A2AMessage(sender, recipient, "response", {"result": result})   # Create response message with the result
        if message_id in self.pending_requests: # Remove the original request from pending (it's been answered)
            del self.pending_requests[message_id]
        return response
```

#### 1.2 Test the A2A Protocol

Verify that the simple A2A protocol works by executing the following code in a test script (or within one of your routes within `routes.py`)

```python
from flask_app.utils.a2a_protocol import A2AProtocol

# Test the A2A Protocol
a2a = A2AProtocol()

# Send a request from orchestrator to web crawler
message_id    = a2a.send_request(
    sender    = "orchestrator",                 # Who is sending the request
    recipient = "web_crawler_agent",            # Who should handle it
    action    = "crawl_url",                    # What action to perform
    params    = {"url": "https://example.com"}  # Parameters for the action
)

print(f"Request sent with message ID: {message_id}")
print(f"Pending requests: {len(a2a.pending_requests)}")
```

### Step 2: Database Table Creation

**Overview**: Create a database table to store web-crawled content with embeddings for semantic search. This table will serve as the repository for all web content that your crawler extracts, enabling your Database Read Expert to perform semantic searches across both resume data and web content.

**Why We Need a Documents Table:**
The `documents` table will store web-crawled content in chunks with corresponding vector embeddings. This allows your system to store web content persistently for future queries, perform semantic similarity search across web content, and combine resume data with relevant web content in responses.

**What We'll Implement in Step 2:**

<u>2.1: Documents Table Schema:</u> Create the database table structure with embedding support

<u>2.2: Initial Data Setup:</u> Prepare the table for storing web-crawled content

This foundation will enable your web crawler to store content and your Database Read Expert to retrieve it using semantic search.

#### 2.1 Create the Documents Table

Create `flask_app/database/create_tables/documents.sql`:

```sql
CREATE TABLE IF NOT EXISTS documents (
    document_id   SERIAL PRIMARY KEY,                    -- Unique identifier for each document chunk
    url           varchar(500) NOT NULL,                 -- Source URL of the web page
    title         varchar(500) NOT NULL,                 -- Page title extracted during crawling
    chunk_text    text NOT NULL,                         -- Text content of this specific chunk
    chunk_index   integer NOT NULL,                      -- Order/index of this chunk within the page (0, 1, 2, ...)
    embedding     vector(1536) DEFAULT NULL,             -- Vector embedding for semantic search (1536 dimensions for OpenAI)
    created_at    timestamp DEFAULT CURRENT_TIMESTAMP,   -- When this chunk was created and stored
    FOREIGN KEY (url) REFERENCES experiences(hyperlink)  -- Links to the experience that contains this URL
);
```

**Note**: You will need to update `flask_app/utils/database.py` to ensure that the `documents` table gets created when your application initializes. Follow the same pattern used for other tables in your database setup.

(EDIT 10/21) You need to add the keyword `UNIQUE` to the hyperlink column in the `experiences` table to be able to use it as a foreign key for the `documents` table.

#### 2.2 Create the Initial (Empty) Documents Data Directory

Create `flask_app/database/initial_data/documents.csv` (empty):

```csv
document_id,url,title,chunk_text,chunk_index,embedding,created_at
```

### Step 3: Web Crawling Agent Implementation

**Overview**: Create a web crawling agent that accepts A2A-formatted requests, collects web content, cleans the content, segments it, embeds it, and stores it in the database. This agent will be responsible for extracting meaningful content from web pages and preparing it for semantic search.

**What is a Web Crawling Agent?**
A web crawling agent is a specialized AI component that can fetch web pages, extract their content, clean and process the text, and store it in a structured format. In this assignment, your web crawler will accept requests via A2A Protocol from your orchestrator.

**What We'll Implement in Step 3:**

<u>3.1: Web Crawler Agent Class:</u> Create the main web crawling agent with A2A Protocol integration

<u>3.2: Agent Testing:</u> Verify the web crawler works correctly with sample URLs

This agent will serve as the bridge between your orchestrator's requests and the web content analysis capabilities.

#### 3.1 Create the Web Crawler Agent Class

Create a web crawler agent class in `flask_app/utils/web_crawler.py` by completing the `TODO` lines in the following code skeleton:

**Implementation Skeleton**:

```python
# Author: Prof. MM Ghassemi <ghassem3@msu.edu>

import requests
from bs4 import BeautifulSoup
from .a2a_protocol import A2AProtocol, A2AMessage

class WebCrawlerAgent:
    def __init__(self, a2a_protocol: A2AProtocol = None):
        # TODO: Initialize the A2A protocol and set agent ID
        pass
        
    def handle_a2a_request(self, message: A2AMessage) -> A2AMessage:
        # TODO: Check if action is "crawl_url"
        # TODO: Extract URL from message.params.get("url")
        # TODO: Call _crawl_url() method
        # TODO: Return response using a2a_protocol.send_response()
        pass
    
    def _crawl_url(self, url: str):
        # TODO: Use requests.get() to fetch the web page
        # TODO: Parse HTML content using BeautifulSoup
        # TODO: Extract page title
        # TODO: Extract raw text content
        # TODO: Call LLM to clean up text (remove menus, ads, navigation, etc.)
        # TODO: Segment cleaned text into chunks (500-1000 characters each)
        # TODO: Generate embeddings for each chunk using the embedding service you developed in Homework 2.
        # TODO: Store text chunks and corresponding embeddings in `documents` table you created in Step 2
        # TODO: Return a dictionary with the url, title, chunks_created, status
        pass
```

**Input Requirements**:

- The agent should accept `A2AMessage` objects via a `handle_a2a_request()` method
- The message should have `action="crawl_url"` and `params={"url": "https://example.com"}`

**Processing Requirements**:

- For `action="crawl_url"`, extract the URL from `message.params.get("url")`
- Use `requests.get()` to fetch the web page
- Parse the HTML content using `BeautifulSoup` to extract all text content (e.g. using `soup.get_text()`)
- **Important**: The raw text may contain navigation menus, advertisements, and other noise. I suggest you call an LLM (with an appropriate prompt) to "clean up" the non-content elements, returning a single continuous block of clean text
- **Segment the cleaned text** into chunks (e.g., 500-1000 characters per chunk)
- **Generate embeddings** for each chunk using the embedding service from Homework 2
- **Store chunks and embeddings** in the `documents` table

**Output Requirements**:

- For successful crawling, return a dictionary with:
  - `"url"`: the original URL
  - `"title"`: the page title (or "No title" if none found)
  - `"chunks_created"`: number of text chunks created and stored
  - `"status"`: "success"
  
- For failed crawling, return a dictionary with:
  - `"url"`: the original URL
  - `"error"`: the error message
  - `"status"`: "error"
  
- Always use `a2a_protocol.send_response()` to return the result

  

#### 3.2 Test the Web Crawler

Verify that the web crawler works by executing the following code in a test script (or within a test route in `routes.py`)

```python
from flask_app.utils.a2a_protocol import A2AProtocol, A2AMessage
from flask_app.utils.web_crawler import WebCrawlerAgent

# Initialize components
a2a = A2AProtocol()
crawler = WebCrawlerAgent(a2a)

# Create a test request message
test_message = A2AMessage(
    sender="orchestrator",
    recipient="web_crawler_agent",
    action="crawl_url",
    params={"url": "https://msu.edu"}
)
print(f"Test message: {test_message.action} with URL: {test_message.params['url']}")

# Handle the request
response = crawler.handle_a2a_request(test_message)

print(f"Response: {response.action}")
print(f"Result: {response.params['result']}")
```

**Note**: The web crawler should clean the text, segment it into chunks, generate embeddings, and store everything in the `documents` table. The response indicates how many chunks were created and stored.


### Step 4: Enhance Orchestrator with Web Crawling Integration

**Overview**: Update your orchestrator to determine when web crawling is necessary and how to integrate the results with existing data analysis using A2A Protocol. This step will enable your orchestrator to intelligently decide when to leverage web content and coordinate between database queries and web crawling.

**Why Enhance the Orchestrator?**
Your orchestrator needs to understand when web crawling would add value to a user's query. For example, if a user asks "What did I work on in the BRAINWORKS project?" and there's a URL in the experiences table, the orchestrator should:
- Query the experiences table for BRAINWORKS details
- Check if there's a URL associated with the project
- Use A2A Protocol to request web crawling if a URL exists
- Combine database results with web content for a comprehensive response (You can achieve this by using a separate call to the LLM with the retrieved data)

#### 4.1 Orchestrator Decision Logic

**Overview**: Update your orchestrator to determine when web crawling is necessary and how to integrate the results with existing data analysis using A2A Protocol.

**Implementation Guidance**:

1. **Decision Framework**: Your orchestrator needs to determine when web crawling would add value to a response. Consider: Does the user's question involve specific projects or experiences? Are there URLs in the experiences table that might provide additional context? Would external web content help answer the user's question more completely?
2. **Web Crawling Workflow**: When your orchestrator decides to trigger web crawling, it should understand that the web crawler will store processed content in the `documents` table, and the Database Read Expert will need to subsequently fetch this stored content if it is to be used in a response
3. **Response Synthesis**: Your orchestrator should combine results from classic database queries (resume data), and the web-crawled content to generate better quality responses for the user.


### Step 5: Enhance Database Read Expert with Documents Table Integration

**Overview**: Modify the Database Read Expert to understand when and how to query the `documents` table for additional context from web-crawled content. This enhancement will enable your Database Read Expert to perform semantic searches across both resume data and web content, providing more comprehensive responses.

**Why Enhance the Database Read Expert?**
Your Database Read Expert needs to understand the new `documents` table and how to leverage web-crawled content. For example, when a user asks "What technologies did I use in my research?", the expert should query the traditional resume tables for research experiences, search the documents table for technology mentions using vector similarity, and combine results from both sources for a comprehensive response

#### 5.1 Update Database Read Expert for Documents Table

**Overview**: Modify the Database Read Expert to understand when and how to query the `documents` table for additional context from web-crawled content.

**Implementation Guidance**:

1. **Documents Table Integration**: Your Database Read Expert needs to understand the new `documents` table and how to query it for web-crawled content. Consider: how to identify when web content would be relevant to a user's question, how to combine traditional SQL queries with vector similarity search, how to merge resume data with web-crawled content.
2. **Vector Similarity Search**: Implement semantic search capabilities using the embedding service from Homework 2. 
3. **Query Strategy**: Develop a strategy for when to use different types of queries. Consider: When to query traditional resume tables (users, experiences, skills, etc., when to search the `documents` table for web content, how to combine results from multiple sources
4. **Experience-URL Integration**: For experiences with URLs, your expert should check if an experience has a hyperlink, search the documents table for content from that URL, and combine experience data with relevant web content


### Step 6: Review the Assignment Rubric

Carefully review the [rubric](documentation/rubric.md) to understand exactly how you will be graded. The rubric outlines the functional requirements that must be demonstrated in a demo video and submitted via the homework submission form.

### Step 7: Demo Video Recording

Using a video recording tool of your choice, create a video with the following structure:

- **Introduction**:
  - State your full name
  - Say: "I will now demonstrate the functional requirements for CSE 491 Homework 3"

- **For Each Functional Requirement**:
  - **State the requirement**: Clearly announce the rubric criteria you are about to demonstrate
  - **Show the functionality**: Demonstrate the rubric requirement clearly and unambiguously in the video.
  

**IMPORTANT**: Please keep the video as short as possible, and ensure clear audio and video quality.

### Step 8: Submit your code

#### Submit Homework 3 Code

1. Ensure all your Homework 3 files are in the `Homework3` folder within your Personal Course Repository.

2. Submit your assignment by navigating to the main directory of your Personal Course Repository and pushing your repo to GitHub:

   ```bash
   git add .
   git commit -m 'submitting Homework 3'
   git push
   ```

3. You have now submitted Homework 3; you can run the same commands to re-submit anytime before the deadline. Please check that your submission was successfully uploaded by navigating to the `Homework3` directory in your Personal Course Repository online.

### Step 9: Submit your Demo Video and Survey

Submit your video and complete the homework survey using the provided [Google Form link](https://docs.google.com/forms/d/e/1FAIpQLScUPhT4fd-5-WdOPNKDhlHZuy0U8xDjmon3JZhU92_8ocoOKw/viewform?usp=dialog).
