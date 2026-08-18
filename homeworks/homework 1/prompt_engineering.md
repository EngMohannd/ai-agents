# Prompt Engineering Concepts — Homework 1

In this assignment, we transitioned from a single-chatbot architecture into a **Multi-Expert Agent System** featuring an **Orchestrator**, a **Database Read Expert**, a **Database Write Expert**, and a **Content Expert**. 

Below is an explanation of 3 key prompt engineering concepts implemented in this homework, along with their practical effectiveness and demonstrations.

---

## 1. Role-Based Prompting & Persona Specification (Zero-Shot Specialized Roles)

### Concept Explanation:
Rather than relying on one general system prompt to handle conversational, analytical, and database manipulation tasks, we assigned a strict, specialized **persona/role** to each sub-agent. By defining explicit boundaries and expectations for each role, the model's probabilistic attention focuses solely on the relevant output domain (e.g., generating only raw SQL for database experts vs. conversational text for content experts).

### Implementation in Code:
* **Database Write Expert Prompt:**
  ```text
  You are a PostgreSQL Write Expert.
  Given the database schema below, generate a valid SQL statement (INSERT, UPDATE, or DELETE) to fulfill the user's modification request.
  [Database Schema]
  Output ONLY the raw SQL query. Do not include markdown or backticks.