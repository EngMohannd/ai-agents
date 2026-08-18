# CSE 491 / 895 - AI Agents: Homework 1

## Overview
You will extend your resume application to create a multi-expert AI agent system. Specifically, you will: (1) design and implement four specialized AI experts with distinct capabilities - (i) a database read expert for SQL queries, (ii) a database write expert for data modifications, (iii) a content expert that answers questions about the current page, and (iv) an orchestrator agent that analyzes user questions and coordinates multi-step responses across experts using chain-of-thought reasoning; (2) implement database schemas to store expert prompts and track all agent interactions; and (3) enable persistent resume modifications that update both the database and frontend interface in real-time.

## Learning Objectives
- Design and implement specialized AI prompt templates for different task types (database reads, database writes, page content analysis)
- Create database schemas for storing and managing AI agent configurations
- Implement an orchestrator AI prompt template that coordinates between user questions and specialized expert prompts
- Apply few-shot examples and chain-of-thought reasoning in orchestrator prompt design
- Integrate database operations with AI responses to enable dynamic content updates

## Preliminaries

This assignment assumes that you have familiarized yourself with the AI-integrated web application template that was provided as part of Homework 0. If you have not already, please review the tutorial covering the Homework 0 web application, and the associated codebase.

**Important**: Create a new folder called `Homework1` in your Personal Course Repository for this assignment. Copy your entire Homework 0 codebase into this new folder before making any modifications. This ensures you preserve your Homework 0 submission while working on Homework 1.

## Specific Homework Instructions

### Step 1: Prompt Template Design and Experimentation

Design and test the prompt templates you will use for your AI experts. As we covered in the prompt-engineering lecture.

#### 1.1 Create a Master Template

Your prompt templates should be designed as parameterized strings that can accommodate:

- **Variable substitution**: Using placeholders including:
  - `{{role}}`: The expert's role/identity (e.g., "Database Read Expert", "Database Write Expert")
  - `{{domain}}`: The expert's area of expertise (e.g., "PostgreSQL database queries")
  - `{{background_context}}`: Relevant background information for the expert's domain (e.g., database schema details)
  - `{{specific_instructions}}`: Task-specific directions for the expert
  - `{{few_shot_examples}}`: Optional example Q&A pairs for few-shot demonstration
  - `{{request}}`: The actual user query/request
- **Conditional sections**: Template should be configured so that if a section parameter is None or empty, the entire section (including header) is removed rather than showing "Examples: None" or similar

**Common Template Patterns:**

While there's no official standard for prompt templates, successful prompts tend to have certain common patterns:

1. They are brief, containing only the information and instructions strictly required to make a decision
2. They include few-shot demonstrations when helpful
3. They specify the desired response structure

You should create one master template with parameterized inputs that will instantiate each of the experts you're looking to create (e.g. Database Read Expert, Database Write Expert, etc.). An example master template is provided below.

**Example Master Template**:
```
You are a {{role}} with expertise in {{domain}}.

Instructions:
{{specific_instructions}}

Context:
{{background_context}}

Examples:
{{few_shot_examples}}

Request: 
{{request}}
```

#### 1.2 Experiment with Prompt Design

Now that you have a master template, the next step is to determine the right parameter values for each of your four roles (the three "experts" and the "orchestrator"). It's important to keep in mind that these four roles are not separate AI models - they are the same underlying LLM (GPT) but with different prompts. To start, you should experiment with different parameter settings to find the most effective initial prompt configurations. Suggestions follow:

**Database Read Expert:**
- `{{role}}`: "Database Read Expert"
- `{{domain}}`: "PostgreSQL database queries and data analysis"
- `{{specific_instructions}}`: "Use the database schema provided to answer the question below. Respond with only SQL query code. Do not include explanations or markdown formatting."
- `{{few_shot_examples}}`: A set of example user questions and corresponding SQL queries
- `{{background_context}}`: The database schema (users, institutions, positions, experiences, skills tables and their relationships)
- `{{request}}`: The specific request that requires a database read operation.

**Database Write Expert:**
- `{{role}}`: "Database Write Expert"
- `{{domain}}`: "PostgreSQL database modifications and Python database operations"
- `{{specific_instructions}}`: "Use the database schema provided to generate Python code that will modify the database. Respond with only Python code using the db.insertRows, db.query, or other database methods. Do not include explanations or markdown formatting."
- `{{few_shot_examples}}`: A set of example user requests and corresponding Python database operations (e.g., calls to `db.insertRows`)
- `{{background_context}}`: The database schema and available database methods (insertRows, query, etc.)
- `{{request}}`: The specific request that requires a database write operation.

**Content Expert:**
- `{{role}}`: "Content Expert"
- `{{domain}}`: "Current page content analysis and contextual responses"
- `{{specific_instructions}}`: "Analyze the provided page content and answer questions based on what's displayed. Reference specific information from the page when relevant. Provide clear, conversational responses."
- `{{few_shot_examples}}`: None likely needed (zero-shot approach typically works well for content analysis)
- `{{background_context}}`: Current page content (title, URL, cleaned HTML content)
- `{{request}}`: The actual content-related question.

**Orchestrator:**
- `{{role}}`: "Orchestrator AI"
- `{{domain}}`: "Multi-expert coordination and task analysis"
- `{{specific_instructions}}`: "Analyze the user question and respond with a list of function calls to handle_ai_chat_request. Each call should specify the role and message parameters. Format as a Python list of strings."
- `{{few_shot_examples}}`: Examples of task analysis with expert selection and execution plans such as `['handle_ai_chat_request(role="Database Read Expert", message="...")', 'handle_ai_chat_request(role="Database Write Expert", message="...")']`
- `{{background_context}}`: Available experts and their capabilities (Database Read Expert, Database Write Expert, Content Expert)
- `{{request}}`: The actual user request requiring coordination

#### 1.3 Update llm.py to Support Roles

Store the values from step 1.2 as a nested dictionary where the top-level keys correspond to each role. Store this as a global variable in `llm.py`:

```python
LLM_ROLES = {
    "Database Read Expert": {
        "role": "Database Read Expert",
        "domain": "...",
        "specific_instructions": "...",
        "few_shot_examples": "...",
        "background_context": "..."
    },
    # Add other roles following the same pattern
}
```

For now, you can hardcode this dictionary as a global variable in `llm.py`. Later, we will be storing this information in the database and use a call to `db.getLLMRoles()` to fetch this dictionary.

Update the `handle_ai_chat_request` function in `llm.py` to:
1. Take an input parameter for the LLM `role`
2. When the role is not `None`, use the parameters within the dictionary to populate your master template as the system prompt
3. Handle the case where role is `None` (use the existing default behavior)

#### 1.4 Testing and Refinement

1. **Test your templates in your live system** by replacing the hard-coded system prompt in the `@app.route('/chat/ai', methods=['POST'])` function in `flask_app/routes.py` with a newly updated role-dependent call to the Orchestrator AI: `handle_ai_chat_request(role="Orchestrator", ...)`. 

   The orchestrator should return a list of strings, each containing a call to `handle_ai_chat_request(role="...", message="...")`. You can loop through each of the decomposed calls in sequence using a for loop, storing the output in a list of tuples of (`role`, `message`, `response`). When everything has completed, make a final call to an LLM to synthesize across the list of messages so you can return a comprehensive report to the user.

2. **For each of the new roles (excluding the content expert), perform the following tests:**

   (Edit 9/2) Note: You might need to adjust the prompts by providing more information, examples, etc. for the LLM responses to pass the following tests.
   
   **Orchestrator Testing:**
   - Generate feasible `{{request}}` values that would require sequential combinations of the experts above, such as: "Add a new skill 'Python' to the role at MSU if it's not already there." (This would require calling the Database Read Expert to see if the skill exists, followed by the Database Write Expert to add the skill if needed.)
   - Verify the response decomposes into appropriately simple calls and gets the sequence right.
   - Test with: "Check if he has React skills and add it to his first experience if missing"
   - Expected output: `['handle_ai_chat_request(role="Database Read Expert", message="Does he have React skills?")', 'handle_ai_chat_request(role="Database Write Expert", message="Add React skill to first experience at MSU")']`

   **Database Read Expert Testing:**
   - Generate feasible `{{request}}` values, such as: "How long did he work at Michigan State University?"
   - Use the updated `handle_ai_chat_request` with `role='Database Read Expert'` to generate the SQL code.
   - Execute the SQL query using `db.query(<generated_sql>)`.
   - Verify the query returns correct results without errors.
   - Test additional scenarios: "What skills did he use in his research projects?", "Which institution did he work at the longest?"

   **Database Write Expert Testing:**
   - (Edit 9/4) Write operations provided by **Database Write Expert** may include subqueries. For example, to add *"Machine Learning"* to the *Teaching Assistant role at MSU*, you will need the corresponding `experience_id` for that role. The `insertRows` function in `database.py` has been updated to handle such cases. Copy and paste the updated `insertRows` function into your codebase before testing.
   - Generate feasible `{{request}}` values, such as: "Add a new skill 'Machine Learning' to the Teaching Assistant role at MSU"
   - Use the updated `handle_ai_chat_request` with `role='Database Write Expert'` to generate Python code.
   - Execute the returned Python code using `exec(<generated_code>)` (ensure `db` is in scope).
   - Call `db.getResumeData()` to ensure the update persisted in the database.
   - Test additional scenarios: , "Add a new experience 'Published AI Research Paper'..."
   
4. **Iterate and improve** based on response quality and consistency. Adjust your prompt parameters if the outputs are not meeting expectations.

### Step 2: Database Schema Design

#### 2.1 Create Expert Roles Database Table

Create a database table to store the variables specified in step 1.2. This will replace the hardcoded `LLM_ROLES` dictionary.

**Create the SQL file** `flask_app/database/create_tables/llm_roles.sql`:
Design a table structure that can store all the role parameters you identified in step 1.2. Consider what columns you'll need to store role names, domains, instructions, examples, and context.

**Create the CSV file** `flask_app/database/initial_data/llm_roles.csv`:
Populate this file with the role configurations you developed in step 1.2. Make sure the CSV columns match your table structure.

**Important**: After creating your new table, you must update the `self.tables` list in `flask_app/utils/database.py` to include your new table name (e.g., `'llm_roles'`). This ensures the table gets created when the application starts up.

**Update the database class** by adding a `getLLMRoles()` method in `flask_app/utils/database.py`:
This method should query your new table and return a dictionary with the same structure as your hardcoded `LLM_ROLES` variable.

**Update `handle_ai_chat_request`** to call the database instead of using the hardcoded `LLM_ROLES` dictionary.


### Step 3: Update Frontend for Real-time Resume Updates

Update the frontend so that the resume is reloaded after every chat request to reflect any database changes made by the Database Write Expert. In `flask_app/static/js/chat-vue.js`, update the `handleAIResponse` function to trigger a resume reload (e.g. call `loadResumeData()`) after each AI response.

### Step 4: Integration and Testing

#### 4.1 Complete Integration

1. **Update your routes.py** to use the orchestrator by default in the `/chat/ai` route.

2. **Implement the orchestrator execution logic** to parse and execute the function calls returned by the orchestrator.
   
3. (Edit 7/9) **Update the chat interface** to display only the results of database operations, not raw outputs. For read operations, show only the information retrieved from the database. For write operations, display one of the following messages:  
- `New <Element Type> added to the <Table> table.`  
- `Element already exists in the <Table> table.`  
- `Operation was unsuccessful.`  


#### 4.2 End-to-End Testing

Test complete workflows:

1. **Database Read Workflow**: "How long did he work at his first job?"
2. **Database Write Workflow**: "Add 'TensorFlow' as a skill to his ... experience"
3. **Complex Workflow**: "Add 'Communication' as a skill for all experiences and postions held at MSU"
4. **Content Analysis**: "What programming languages are mentioned on this resume page?"

Verify that:
- The orchestrator correctly decomposes complex requests
- Database reads return accurate information
- Database writes persist and appear in the refreshed UI

### Step 5: Review the Assignment Rubric
Carefully review the [rubric](documentation/rubric.md) to understand exactly how you will be graded. For this assignment, the rubric outlines the functional requirements that must be demonstrated in a demo video and submitted via the homework submission form at the bottom of the page.

### Step 6: Demo Video Recording

Using a video recording tool of your choice (OBS Studio, Loom, built-in screen recording, etc.) create a video with the following structure:

- **Introduction**:
  - State your full name
  - Say: "I will now demonstrate the functional requirements for CSE 491 Homework 1"

- **For Each Functional Requirement**:
  - **State the requirement**: Clearly announce the rubric criteria you are about to demonstrate (e.g., "I will now demonstrate that the orchestrator can coordinate multiple experts")
  - **Show the functionality**: Demonstrate the rubric requirement clearly and unambiguously in the video.
  

**IMPORTANT**: Please keep the video as short as possible, and ensure clear audio and video quality.

### Step 7: Submit your code

#### Submit Homework 1 Code

1. Ensure all your Homework 1 files are in the `Homework1` folder within your Personal Course Repository.

2. Submit your assignment by navigating to the main directory of your Personal Course Repository and pushing your repo to GitHub:

   ```bash
   git add .
   git commit -m 'submitting Homework 1'
   git push
   ```

3. You have now submitted Homework 1; you can run the same commands to re-submit anytime before the deadline. Please check that your submission was successfully uploaded by navigating to the `Homework1` directory in your Personal Course Repository online.

### Step 8: Submit your Demo Video and Survey
Submit your video and complete the homework survey using [this Google Form](https://docs.google.com/forms/d/e/1FAIpQLSc8UNBUJXTekGxVEWzDLfys6ybjPjaxvdGc6em570uzSXpK-A/viewform?usp=preview).
