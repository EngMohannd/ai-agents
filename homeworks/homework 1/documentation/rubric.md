# CSE 491 / 895 - AI Agents: Homework 1 Rubric

This assignment is graded on a 100 point scale. The following rubric will be used when grading your submission and accompanying homework video:

## Functional Requirements:

### Test 1: Database Read Expert (25 points)
Ask the question "How long did they work at [institution name]?" in the chat interface and show that:
- The system generates a valid SQL query (you can print the sql to the console)
- The system response contains accurate duration information

### Test 2: Database Write Expert (25 points)
Ask to "Add '[skill name]' as a skill to the [experience]" and show that:
- The system generates valid Python/database code (again, print to the console)
- The new skill appears in the resume interface after the LLM responds.
- The skill persists when you reload the page

### Test 3: Orchestrator Coordination (25 points)
Ask "Check if he has [skill name] and add it to all experiences at [position] if missing" and show that:
- The orchestrator returns a list of function calls in proper sequence
- The sequence includes both a Database Read Expert call and Database Write Expert call
- The actual execution follows the planned sequence

### Test 4: Database Schema (25 points)
Show in your database interface or through a query that:
- The `llm_roles` table exists and contains your expert configurations

**GRADING POLICY**: 
- Each test uses **"ALL OR NOTHING"** grading. You receive full points only if ALL requirements for that test are demonstrated successfully in your video.
- **No partial credit** is awarded for any test criteria.
- Points are awarded based on **functionality demonstrated**, not code quality or implementation approach.
