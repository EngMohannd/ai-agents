# CSE 491 / 895 - AI Agents: Homework 3 Rubric

This assignment is graded on a 100 point scale. The following rubric will be used when grading your submission and accompanying homework video:

## Functional Requirements:

### Test 1: Web Crawling Agent (50 points)
**Demonstration**: Test your web crawler with a real URL
- Run the web crawler test code from Step 3.2 with a real URL (e.g., https://msu.edu)
- Show the console output displaying the crawler response
- Verify the response contains: `url`, `title`, `chunks_created`, and `status`
- Show that chunks were actually stored in the `documents` table (query the table)

### Test 2: Database Read Expert Enhancement (50 points)
**Demonstration**: Ask a question to your agent that should use web content
- Ask: "What did I work on in the [project name] project?" (use a project that has a URL in your experiences table)
- Show that the response includes information from both resume data and web-crawled content
- Show in console/logs that vector similarity search was performed on the `documents` table
- Demonstrate that the expert successfully combined results from multiple sources

**GRADING POLICY**: 
- Each test uses **"ALL OR NOTHING"** grading. You receive full points only if ALL requirements for that test are demonstrated successfully in your video.
- **No partial credit** is awarded for any test criteria.
- Points are awarded based on **functionality demonstrated**, not code quality or implementation approach.
