# Prompt Engineering Concepts — Homework 1

In this assignment, we transitioned from a single-chatbot architecture into a **Multi-Expert Agent System** featuring an **Orchestrator Router**, a **Database Read Expert**, a **Database Write Expert**, and a **Content Expert**. 

Below is an explanation of 3 key prompt engineering concepts implemented in this homework, along with their practical effectiveness and demonstrations.

---

# 1. Role-Based Prompting & Persona Specification (Zero-Shot Specialized Roles)

Rather than relying on one general system prompt to handle conversational, analytical, and database manipulation tasks, we assigned a strict, specialized **persona/role** to each sub-agent. By defining explicit boundaries and expectations for each role, the model's probabilistic attention focuses solely on the relevant output domain (e.g., generating only raw SQL for database experts vs. conversational text for content experts).

# Implementation in Code:
* **Database Write Expert Prompt:**
  ```text
  You are a PostgreSQL Write Expert.
  Given the database schema below, generate a valid SQL statement (INSERT, UPDATE, or DELETE) to fulfill the user's modification request.
  [Database Schema]
  Output ONLY the raw SQL query. Do not include markdown or backticks.


# Effectiveness & Observed Results:
Before: A single general prompt would often return conversational text mixed with SQL snippets (e.g., "Sure, here is your SQL query: sql INSERT INTO..."), which broke automated backend execution.

After: The specialized role reliably generated pure, executable SQL queries directly processed by the backend database executor without parsing errors.





## 2. Intent Routing & Constrained Classification Prompting
To coordinate between multiple specialized experts, we implemented an Orchestrator Router that acts as an intent-classification gate. To ensure deterministic downstream routing in Python, the prompt uses strict categorical constraints, instructing the model to output exactly one keyword representing the intended action (READ, WRITE, or CONTENT).
## Implementation in Code:
You are the Orchestrator Router. Analyze the user's message and determine the correct expert needed.
Output ONLY one word from these options:
- READ: If the user asks a question that requires querying or viewing data from the database.
- WRITE: If the user wants to add, insert, update, or delete information in the database.
- CONTENT: If the user is asking general questions or discussing the resume.

Respond ONLY with READ, WRITE, or CONTENT.

## Effectiveness & Observed Results:
Before: Asking diverse questions to a single endpoint resulted in confusion between database state queries and static webpage content.

After: The Orchestrator achieved 100% accuracy in routing commands:

"What institutions are listed in the database?" ➔ Routed to READ

"Add a skill named 'Embedded C' with skill level 5 to experience_id 1." ➔ Routed to WRITE

"Who is Mohanad and what is his background?" ➔ Routed to CONTENT




### 3. Two-Stage Synthesis & Prompt Chaining (Decomposition)
Concept Explanation:
Complex tasks—such as querying a database and answering a user in natural language—were decomposed into two sequential prompt chains:

Stage 1 (SQL Generation): Translates the natural language query into a target SQL SELECT statement.

Stage 2 (Response Synthesis): The database engine executes the query and passes the raw JSON/Dict result into a secondary synthesis prompt to construct a fluent, user-friendly response.

### Implementation in Code:
You are a helpful assistant. The user asked: "{user_message}"
The database returned the following results: {db_results_json}

Synthesize these results into a clear, natural, and helpful response for the user.

### Effectiveness & Observed Results:
Before: Attempting to retrieve and format database data in a single shot frequently resulted in hallucinated resume entries.

After: Chaining allowed 100% grounded facts. For example:

Input: "What skills are associated with experience 1?"

Database Output: [{"name": "Embedded Systems", "skill_level": 9}, {"name": "Arduino & Hardware Design", "skill_level": 9}, {"name": "Embedded C", "skill_level": 5}]

Synthesized Natural Output: "Experience 1 is associated with the following skills: Embedded Systems (skill level 9), Arduino & Hardware Design (skill level 9), and Embedded C (skill level 5)."