# CSE 491 / 895 - AI Agents: Homework 2 Rubric

This assignment is graded on a 100 point scale. The following rubric will be used when grading your submission and accompanying homework video:

## Functional Requirements:

### Test 1: Semantic Search (25 points)
Ask a question using an abbreviation or nickname (e.g., "Find my MSU experience") and show that:
- The system finds the full name (e.g., "Michigan State University") even though the query uses an abbreviation
- The system response contains accurate information about the requested experiences

### Test 2: Complex Semantic Query (25 points)
Ask a question about skills or experiences using general terms (e.g., "What AI skills do I have?") and show that:
- The system finds related skills using semantic similarity (e.g., "machine learning", "deep learning", etc.)
- The system response contains relevant skills from your database that match the semantic meaning

### Test 3: Human Validation Workflow (25 points)
Ask "Delete all my experiences" and show that:
- The system asks for confirmation before processing the dangerous request
- When you respond "yes", the system processes the deletion request
- When you respond "no", the system cancels the request without processing

### Test 4: Database Schema (25 points)
Show in your database interface or through a query that:
- All tables have `embedding vector(1536)` columns
- All tables have `ivfflat` indexes for similarity search
- New data includes generated embedding vectors

**GRADING POLICY**: 
- Each test uses **"ALL OR NOTHING"** grading. You receive full points only if ALL requirements for that test are demonstrated successfully in your video.
- **No partial credit** is awarded for any test criteria.
- Points are awarded based on **functionality demonstrated**, not code quality or implementation approach.
