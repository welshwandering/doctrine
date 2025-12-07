# Prompt Engineering Guide

## RFC 2119 Key Words

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this document are to be interpreted as described in [RFC 2119](https://tools.ietf.org/html/rfc2119).

- **MUST** / **REQUIRED** / **SHALL**: Absolute requirement
- **MUST NOT** / **SHALL NOT**: Absolute prohibition
- **SHOULD** / **RECOMMENDED**: Strong recommendation, may be valid reasons to ignore in particular circumstances
- **SHOULD NOT** / **NOT RECOMMENDED**: Strong discouragement, may be valid reasons to use in particular circumstances
- **MAY** / **OPTIONAL**: Truly optional, up to implementer

---

## Table of Contents

1. [Introduction](#introduction)
2. [Prompt Anatomy](#prompt-anatomy)
3. [Structured Output Techniques](#structured-output-techniques)
4. [Reasoning Techniques](#reasoning-techniques)
5. [Few-Shot Patterns](#few-shot-patterns)
6. [System Prompt Design for Coding](#system-prompt-design-for-coding)
7. [Context Optimization](#context-optimization)
8. [Code-Specific Patterns](#code-specific-patterns)
9. [Meta-Prompting](#meta-prompting)
10. [Evaluation and Iteration](#evaluation-and-iteration)
11. [Common Mistakes and Fixes](#common-mistakes-and-fixes)
12. [Provider-Specific Considerations](#provider-specific-considerations)
13. [Extensive Examples](#extensive-examples)
14. [References](#references)

---

## Introduction

Prompt engineering[^9] is the practice of designing, refining, and optimizing inputs to large language models (LLMs) to achieve desired outputs. This guide provides comprehensive patterns, techniques, and best practices for effective prompt engineering, with special emphasis on coding and software development tasks.

### Scope

This guide:

- **MUST** be used as the authoritative reference for prompt engineering practices
- **SHOULD** be consulted before designing complex prompt systems
- **MAY** be extended with organization-specific patterns and requirements

This guide draws from academic research[^1][^2][^3][^7][^8], industry best practices[^4][^5][^6], and practical experience in software development workflows. All referenced papers and documentation are listed in the [References](#references) section.

### Fundamental Principles

Effective prompts[^4][^5][^9]:

1. **MUST** be clear and unambiguous
2. **MUST** provide sufficient context for the task
3. **SHOULD** specify desired output format explicitly
4. **SHOULD** include constraints and requirements upfront
5. **MAY** include examples to guide the model
6. **SHOULD NOT** include unnecessary information that dilutes focus
7. **MUST NOT** assume implicit knowledge without verification

---

## Prompt Anatomy

### Role-Based Message Structure

Modern LLM APIs use a role-based conversation structure[^12]. Each message **MUST** have a role and content.

#### System Role

The system message[^10]:

- **MUST** define the assistant's behavior and capabilities
- **SHOULD** be placed at the beginning of the conversation
- **SHOULD** establish constraints, rules, and output formats
- **MAY** include personality traits or tone specifications
- **SHOULD NOT** be changed mid-conversation (in most implementations)

**Example System Message:**

```
You are an expert Python developer specializing in data processing pipelines.
You write clean, well-documented code following PEP 8 standards.
You always include type hints and comprehensive docstrings.
When solving problems, you explain your reasoning step-by-step.
```

#### User Role

The user message:

- **MUST** contain the actual request or query
- **SHOULD** be specific and detailed
- **SHOULD** include all necessary context
- **MAY** reference previous messages in the conversation
- **SHOULD NOT** contradict the system message constraints

**Example User Message:**

```
I need a function that parses CSV files and handles errors gracefully.
The function should:
- Accept a file path as input
- Return a pandas DataFrame
- Log errors to a specified logger
- Handle missing files and malformed CSV data

Include comprehensive error handling and type hints.
```

#### Assistant Role

The assistant message:

- **MUST** be used for model responses
- **SHOULD** be used in few-shot examples to show desired output
- **MAY** be used to prime the model's response format
- **MUST NOT** be fabricated for actual model responses (only use in examples)

**Example Assistant Message (in few-shot context):**

```python
import pandas as pd
import logging
from pathlib import Path
from typing import Optional

def parse_csv_file(
    file_path: str,
    logger: logging.Logger
) -> Optional[pd.DataFrame]:
    """
    Parse a CSV file and return a pandas DataFrame.

    Args:
        file_path: Path to the CSV file
        logger: Logger instance for error reporting

    Returns:
        DataFrame if successful, None if parsing fails

    Raises:
        None (errors are logged, not raised)
    """
    try:
        path = Path(file_path)
        if not path.exists():
            logger.error(f"File not found: {file_path}")
            return None

        df = pd.read_csv(path)
        logger.info(f"Successfully parsed {len(df)} rows from {file_path}")
        return df

    except pd.errors.ParserError as e:
        logger.error(f"CSV parsing error in {file_path}: {e}")
        return None
    except Exception as e:
        logger.error(f"Unexpected error parsing {file_path}: {e}")
        return None
```

### Message Ordering and Flow

Conversation flow **MUST** follow these rules:

1. System message (if present) **MUST** come first
2. Messages **MUST** alternate between user and assistant (after system)
3. The conversation **MUST** end with a user message when requesting a response
4. Few-shot examples **SHOULD** be placed after system message, before the actual query

**Correct Flow:**

```
[System] - Set role and constraints
[User] - Example input 1
[Assistant] - Example output 1
[User] - Example input 2
[Assistant] - Example output 2
[User] - Actual query
```

**Incorrect Flow:**

```
[User] - Actual query
[System] - Set role (too late)
[User] - Another query (no assistant response in between)
```

---

## Structured Output Techniques

Structured output techniques[^10] ensure that LLM responses follow specific formats, making them machine-parseable and reliable for downstream processing.

### JSON Mode

JSON mode[^10] **SHOULD** be used when you need structured, machine-readable output.

#### Implementation Requirements

When using JSON mode:

1. **MUST** explicitly request JSON output in the prompt
2. **MUST** specify the exact schema expected
3. **SHOULD** provide an example of the desired JSON structure
4. **SHOULD** enable JSON mode in the API if available
5. **MUST** validate the output against the schema
6. **SHOULD** handle malformed JSON gracefully

#### JSON Prompting Template

```
Return your response as a JSON object with the following schema:

{
  "field_name": "type and description",
  "another_field": "type and description"
}

Requirements:
- Output MUST be valid JSON
- All fields are REQUIRED unless marked optional
- Do not include any text outside the JSON object
```

#### Detailed JSON Example

**Prompt:**

```
Analyze the following code and return a JSON object with this exact schema:

{
  "language": "string - programming language detected",
  "complexity": "string - one of: low, medium, high",
  "issues": [
    {
      "type": "string - issue category",
      "severity": "string - critical, warning, or info",
      "line": "number - line number or null",
      "description": "string - detailed description",
      "suggestion": "string - how to fix"
    }
  ],
  "metrics": {
    "lines_of_code": "number",
    "cyclomatic_complexity": "number",
    "maintainability_score": "number - 0 to 100"
  },
  "summary": "string - overall assessment"
}

Code to analyze:

def process_data(data):
    result = []
    for item in data:
        if item:
            if item > 0:
                if item < 100:
                    result.append(item * 2)
    return result

Output only the JSON object, no additional text.
```

**Expected Output:**

```json
{
  "language": "python",
  "complexity": "medium",
  "issues": [
    {
      "type": "readability",
      "severity": "warning",
      "line": 3,
      "description": "Deeply nested conditionals reduce readability",
      "suggestion": "Combine conditions or use early returns"
    },
    {
      "type": "documentation",
      "severity": "warning",
      "line": 1,
      "description": "Missing docstring and type hints",
      "suggestion": "Add docstring explaining function purpose and type hints for parameters"
    },
    {
      "type": "logic",
      "severity": "info",
      "line": 3,
      "description": "Multiple conditions can be combined",
      "suggestion": "Use: if item and 0 < item < 100:"
    }
  ],
  "metrics": {
    "lines_of_code": 8,
    "cyclomatic_complexity": 4,
    "maintainability_score": 65
  },
  "summary": "Function is functional but would benefit from simplified conditional logic and better documentation"
}
```

#### Schema Enforcement Strategies

**Strategy 1: Inline Schema Definition**

```
Return a JSON object matching this TypeScript interface:

interface CodeReview {
  score: number;           // 0-100
  issues: {
    critical: string[];
    warnings: string[];
    suggestions: string[];
  };
  approved: boolean;
  reviewer_notes: string;
}
```

**Strategy 2: JSON Schema**

```
Return JSON matching this JSON Schema:

{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "required": ["status", "data"],
  "properties": {
    "status": {
      "type": "string",
      "enum": ["success", "error"]
    },
    "data": {
      "type": "object",
      "properties": {
        "items": {
          "type": "array",
          "items": {
            "type": "object",
            "required": ["id", "name"],
            "properties": {
              "id": {"type": "integer"},
              "name": {"type": "string"},
              "description": {"type": "string"}
            }
          }
        }
      }
    },
    "error": {
      "type": "string"
    }
  }
}
```

**Strategy 3: Example-Based**

```
Return JSON in exactly this format:

{
  "status": "completed",
  "results": [
    {"id": 1, "name": "test", "passed": true},
    {"id": 2, "name": "test2", "passed": false}
  ],
  "summary": "2 tests run, 1 passed, 1 failed"
}

Replace the example values with actual analysis results.
```

### XML Tags

XML-style tags[^4] **SHOULD** be used when:

- Output has multiple distinct sections
- Hierarchical structure is important
- You need clear boundaries between content types
- Working with models that perform well with XML (e.g., Claude)

#### XML Structure Requirements

1. Tags **MUST** be properly nested
2. Tag names **SHOULD** be descriptive and lowercase
3. Complex outputs **SHOULD** use nested tags
4. Content **MAY** include attributes for metadata

#### XML Prompting Template

```
Structure your response using XML tags:

<response>
  <section_name>
    Content here
  </section_name>
  <another_section>
    More content
  </another_section>
</response>

Use clear, descriptive tag names.
```

#### Detailed XML Example

**Prompt:**

```
Analyze this pull request and provide feedback using the following XML structure:

<pr_review>
  <summary>
    Brief overview of the PR
  </summary>
  <code_quality>
    <score>0-10</score>
    <assessment>Detailed quality assessment</assessment>
  </code_quality>
  <changes>
    <change>
      <file>filename</file>
      <type>added|modified|deleted</type>
      <impact>low|medium|high</impact>
      <comment>Specific feedback</comment>
    </change>
    <!-- Repeat for each significant change -->
  </changes>
  <concerns>
    <concern priority="high|medium|low">
      Description of concern
    </concern>
  </concerns>
  <recommendations>
    <recommendation>Specific action item</recommendation>
  </recommendations>
  <approval>approved|changes_requested|commented</approval>
</pr_review>

Pull request diff:
[diff content here]
```

**Expected Output:**

```xml
<pr_review>
  <summary>
    This PR refactors the authentication module to use JWT tokens instead of session cookies.
    The changes span 5 files and introduce a new token management service.
  </summary>
  <code_quality>
    <score>8</score>
    <assessment>
      Code is well-structured with good separation of concerns. Type hints are present
      and comprehensive. Test coverage is good but could be improved for edge cases.
      Documentation is clear and helpful.
    </assessment>
  </code_quality>
  <changes>
    <change>
      <file>auth/jwt_handler.py</file>
      <type>added</type>
      <impact>high</impact>
      <comment>
        New JWT handler with token generation and validation. Implementation follows
        best practices for token security including proper expiration handling.
      </comment>
    </change>
    <change>
      <file>auth/middleware.py</file>
      <type>modified</type>
      <impact>high</impact>
      <comment>
        Middleware updated to validate JWT tokens. Good error handling, but consider
        adding rate limiting to prevent brute force attacks on token validation.
      </comment>
    </change>
    <change>
      <file>tests/test_auth.py</file>
      <type>modified</type>
      <impact>medium</impact>
      <comment>
        Tests updated for JWT flow. Good coverage of happy path, but missing tests
        for token expiration and invalid signature scenarios.
      </comment>
    </change>
  </changes>
  <concerns>
    <concern priority="high">
      No rate limiting on token validation endpoint could allow brute force attacks
    </concern>
    <concern priority="medium">
      Secret key for JWT signing is hardcoded in middleware.py - should use environment
      variable or secure configuration management
    </concern>
    <concern priority="low">
      Token refresh mechanism not implemented - users will need to re-authenticate
      frequently with current 1-hour expiration
    </concern>
  </concerns>
  <recommendations>
    <recommendation>
      Add rate limiting middleware to token validation endpoint
    </recommendation>
    <recommendation>
      Move JWT secret to environment variable and document in README
    </recommendation>
    <recommendation>
      Implement token refresh endpoint to improve user experience
    </recommendation>
    <recommendation>
      Add tests for token expiration, invalid signatures, and malformed tokens
    </recommendation>
    <recommendation>
      Consider adding refresh token mechanism for long-lived sessions
    </recommendation>
  </recommendations>
  <approval>changes_requested</approval>
</pr_review>
```

#### Nested XML for Complex Hierarchies

```
<code_review>
  <file path="src/main.py">
    <function name="process_data" line="45">
      <issues>
        <issue severity="error" line="47">
          <description>Division by zero possible</description>
          <suggestion>Add zero check before division</suggestion>
          <code>
            if denominator != 0:
                result = numerator / denominator
          </code>
        </issue>
      </issues>
      <metrics>
        <complexity>8</complexity>
        <lines>23</lines>
        <coverage>85</coverage>
      </metrics>
    </function>
  </file>
</code_review>
```

### Markdown Formatting

Markdown **SHOULD** be used for human-readable output that may include:

- Code blocks
- Lists and tables
- Headers and sections
- Emphasis and formatting
- Mixed content types

#### Markdown Structure Requirements

1. **MUST** use proper markdown syntax
2. **SHOULD** use code fences with language specifiers
3. **SHOULD** use headers for clear section delineation
4. **MAY** combine with other formats (e.g., JSON in code blocks)

#### Markdown Prompting Template

```
Format your response using markdown:

## Section Title

Explanatory text here.

### Subsection

- Bullet points
- For lists

```language
code blocks for examples
```

| Column 1 | Column 2 |
|----------|----------|
| Data     | More data|
```

#### Detailed Markdown Example

**Prompt:**

```
Review this code and provide feedback in markdown format:

# Code Review: [filename]

## Overview
Brief summary of what the code does

## Strengths
- Bullet list of good practices

## Issues Found

### Critical
List any critical issues

### Warnings
List warnings

### Suggestions
List suggestions

## Detailed Analysis

For each function:
### Function: `function_name`
- Purpose: what it does
- Issues: specific problems
- Suggested fix:
```language
code example
```

## Overall Assessment
Final thoughts and recommendation

Code to review:
[code here]
```

**Expected Output:**

```markdown
# Code Review: user_manager.py

## Overview

This module handles user authentication and session management. It provides
functions for user login, logout, and session validation using a custom
token-based system.

## Strengths

- Clear function names that indicate purpose
- Good error handling with specific exception types
- Consistent use of type hints throughout
- Logging implemented for debugging
- Separation of concerns between auth and session management

## Issues Found

### Critical

- **SQL Injection Vulnerability** (line 45): User input concatenated directly
  into SQL query
- **Password Storage** (line 78): Passwords stored in plaintext instead of hashed

### Warnings

- **Deprecated Library** (line 3): Using `md5` for hashing, should use `hashlib`
- **Missing Input Validation** (line 52): Email format not validated
- **Hard-coded Secrets** (line 12): Database credentials in source code

### Suggestions

- Consider using an ORM like SQLAlchemy instead of raw SQL
- Add rate limiting to prevent brute force attacks
- Implement password complexity requirements
- Add session timeout mechanism
- Include unit tests for authentication logic

## Detailed Analysis

### Function: `authenticate_user`

- **Purpose**: Validates user credentials and creates session
- **Issues**:
  - SQL injection vulnerability from string concatenation
  - No rate limiting allows brute force attacks
  - Missing logging for failed attempts
- **Suggested fix**:

```python
def authenticate_user(username: str, password: str) -> Optional[Session]:
    """
    Authenticate user and create session.

    Args:
        username: User's username
        password: User's password (will be hashed)

    Returns:
        Session object if authentication succeeds, None otherwise
    """
    # Use parameterized query to prevent SQL injection
    query = "SELECT * FROM users WHERE username = ? AND password_hash = ?"
    password_hash = hashlib.sha256(password.encode()).hexdigest()

    with db.connection() as conn:
        cursor = conn.execute(query, (username, password_hash))
        user = cursor.fetchone()

    if user:
        logger.info(f"Successful login for user: {username}")
        return create_session(user.id)
    else:
        logger.warning(f"Failed login attempt for user: {username}")
        return None
```

### Function: `create_session`

- **Purpose**: Generates session token and stores in database
- **Issues**:
  - Token generation uses weak randomness
  - No expiration time set
  - Session data stored without encryption
- **Suggested fix**:

```python
import secrets
from datetime import datetime, timedelta

def create_session(user_id: int, duration_hours: int = 24) -> Session:
    """
    Create a new session for authenticated user.

    Args:
        user_id: ID of authenticated user
        duration_hours: Session duration in hours (default 24)

    Returns:
        Session object with secure token
    """
    token = secrets.token_urlsafe(32)  # Cryptographically secure
    expires_at = datetime.now() + timedelta(hours=duration_hours)

    session = Session(
        user_id=user_id,
        token=token,
        expires_at=expires_at,
        created_at=datetime.now()
    )

    db.session.add(session)
    db.session.commit()

    logger.info(f"Created session for user {user_id}, expires at {expires_at}")
    return session
```

## Overall Assessment

**Recommendation**: Changes Required

The code demonstrates good structure and organization, but has critical security
vulnerabilities that must be addressed before deployment. The SQL injection and
plaintext password storage issues are serious security risks.

Priority actions:
1. Fix SQL injection vulnerability immediately
2. Implement proper password hashing (bcrypt or Argon2)
3. Move secrets to environment variables
4. Add rate limiting to authentication endpoints
5. Implement session expiration

Once these issues are resolved, this will be a solid authentication module.
Estimated effort: 1-2 days for security fixes, additional day for testing.
```

### Combining Formats

Formats **MAY** be combined for maximum clarity:

**Prompt:**

```
Provide code review in markdown format, with a JSON summary at the end:

## Review Content
[markdown content]

## Machine-Readable Summary
```json
{
  "approved": boolean,
  "critical_issues": number,
  "score": number
}
```
```

---

## Reasoning Techniques

Reasoning techniques guide the model through complex problem-solving processes by making its thinking explicit.

### Chain-of-Thought Prompting

Chain-of-thought (CoT) prompting[^1] **SHOULD** be used when:

- Problem requires multiple steps
- Intermediate reasoning is valuable
- Debugging the thought process may be necessary
- Transparency in decision-making is important

#### CoT Requirements

1. **MUST** explicitly request step-by-step reasoning
2. **SHOULD** provide examples of desired reasoning format
3. **MAY** use phrases like "Let's think step by step"
4. **SHOULD** separate reasoning from final answer

#### CoT Template

```
[Problem statement]

Let's solve this step by step:

1. [First, identify/analyze...]
2. [Then, consider...]
3. [Next, evaluate...]
4. [Finally, conclude...]

Therefore, [final answer].
```

#### Detailed CoT Example

**Prompt:**

```
Debug the following code and explain your reasoning step by step:

```python
def calculate_average(numbers):
    total = 0
    for num in numbers:
        total += num
    return total / len(numbers)

result = calculate_average([])
print(result)
```

Analyze this code step by step:
1. Identify what the code is supposed to do
2. Trace through the execution with the given input
3. Identify where the error occurs
4. Explain why the error happens
5. Propose a fix
6. Verify the fix would work
```

**Expected Output:**

```
Let me debug this code step by step:

**Step 1: Identify the purpose**
The code is designed to calculate the average of a list of numbers by:
- Summing all numbers in the list
- Dividing the total by the count of numbers

**Step 2: Trace execution with input []**
- Input: empty list []
- Loop iteration: The loop doesn't execute (no elements to iterate)
- total remains 0
- Return statement: 0 / len([])

**Step 3: Identify the error location**
The error occurs at the return statement: `return total / len(numbers)`
When numbers is empty, len(numbers) = 0, causing division by zero.

**Step 4: Explain why the error happens**
Division by zero is mathematically undefined and raises a ZeroDivisionError
in Python. The function doesn't handle the edge case of an empty list.

**Step 5: Propose a fix**
We should check if the list is empty before dividing and handle it appropriately.
Options:
a) Return None for empty list
b) Return 0 for empty list
c) Raise a descriptive error

Option (a) is best as it clearly indicates no average can be calculated.

**Step 6: Verify the fix**

```python
def calculate_average(numbers):
    if not numbers:  # Check for empty list
        return None

    total = 0
    for num in numbers:
        total += num
    return total / len(numbers)

# Test cases to verify
result1 = calculate_average([])      # Returns None ✓
result2 = calculate_average([5])     # Returns 5.0 ✓
result3 = calculate_average([1,2,3]) # Returns 2.0 ✓
```

The fix works correctly:
- Empty list returns None (no error)
- Single element returns that element
- Multiple elements return correct average

**Final Answer**: Add an empty list check before division. Return None when
the list is empty to indicate no average can be calculated.
```

#### CoT for Code Generation

**Prompt:**

```
Create a function to find the longest palindromic substring.

Think through this step by step:
1. What approaches could solve this?
2. What's the time/space complexity of each?
3. Which approach is best for typical use cases?
4. What edge cases need handling?
5. Implement the chosen approach

Then provide the final code.
```

**Expected Output:**

```
Let me work through this step by step:

**Step 1: Possible Approaches**

a) Brute force: Check every substring
b) Expand around center: For each position, expand outward
c) Dynamic programming: Build up solution from smaller subproblems
d) Manacher's algorithm: Linear time specialized algorithm

**Step 2: Complexity Analysis**

a) Brute force:
   - Time: O(n³) - n² substrings, O(n) to check each
   - Space: O(1)

b) Expand around center:
   - Time: O(n²) - n centers, O(n) expansion
   - Space: O(1)

c) Dynamic programming:
   - Time: O(n²)
   - Space: O(n²) for DP table

d) Manacher's:
   - Time: O(n)
   - Space: O(n)

**Step 3: Best Approach**

For typical use cases, expand around center is best because:
- O(n²) time is acceptable for most strings
- O(1) space is excellent
- Code is simple and maintainable
- No special preprocessing needed

Manacher's is faster but complex and rarely needed.

**Step 4: Edge Cases**

- Empty string → return ""
- Single character → return that character
- No palindrome longer than 1 → return first character
- Multiple palindromes of same length → return first found
- Even-length palindromes (e.g., "abba")
- Odd-length palindromes (e.g., "aba")

**Step 5: Implementation**

```python
def longest_palindromic_substring(s: str) -> str:
    """
    Find the longest palindromic substring using expand around center.

    Args:
        s: Input string

    Returns:
        Longest palindromic substring (first if multiple exist)

    Time: O(n²), Space: O(1)
    """
    if not s:
        return ""

    def expand_around_center(left: int, right: int) -> int:
        """Expand around center and return length of palindrome."""
        while left >= 0 and right < len(s) and s[left] == s[right]:
            left -= 1
            right += 1
        return right - left - 1

    start = 0
    max_length = 0

    for i in range(len(s)):
        # Check odd-length palindromes (center is single character)
        len1 = expand_around_center(i, i)
        # Check even-length palindromes (center is between characters)
        len2 = expand_around_center(i, i + 1)

        current_max = max(len1, len2)

        if current_max > max_length:
            max_length = current_max
            # Calculate start position of palindrome
            start = i - (current_max - 1) // 2

    return s[start:start + max_length]


# Test cases
assert longest_palindromic_substring("babad") == "bab"  # or "aba"
assert longest_palindromic_substring("cbbd") == "bb"
assert longest_palindromic_substring("a") == "a"
assert longest_palindromic_substring("") == ""
assert longest_palindromic_substring("racecar") == "racecar"
```

This implementation efficiently handles all edge cases and provides O(n²) time
with O(1) space complexity.
```

### Step-by-Step Decomposition

Step-by-step decomposition **MUST** be used for:

- Complex multi-part problems
- Tasks requiring sequential operations
- Situations where order of operations matters

#### Decomposition Template

```
Break this problem into steps:

Step 1: [First action]
- What to do
- Expected outcome

Step 2: [Second action]
- What to do
- Expected outcome

[Continue for all steps]

Then execute each step in order.
```

#### Detailed Decomposition Example

**Prompt:**

```
Create a database migration system from scratch. Break this into concrete steps:

1. Requirements analysis
2. Design decisions
3. Core components needed
4. Implementation order
5. Testing strategy

For each step, explain what needs to be done and why.
```

**Expected Output:**

```
## Database Migration System: Step-by-Step Plan

### Step 1: Requirements Analysis

**What to do:**
- Define what migrations need to track (schema changes, data changes)
- Identify version tracking mechanism
- Determine rollback requirements
- Establish migration file format
- Define CLI interface requirements

**Why this first:**
Understanding requirements prevents rework and ensures we build what's needed.

**Key decisions:**
- Support both up and down migrations
- Track applied migrations in database table
- Use timestamp-based versioning
- Support SQL and Python-based migrations
- Provide CLI for generation, application, and rollback

### Step 2: Design Decisions

**What to do:**
- Choose migration file naming convention (timestamp_description.py)
- Design migration table schema
- Define migration class interface
- Plan dependency resolution (if migrations depend on each other)
- Decide on transaction handling (per-migration or batch)

**Why this second:**
Design decisions guide implementation and prevent architectural mistakes.

**Key design:**
```python
# Migration interface
class Migration:
    def up(self):
        """Apply migration"""
        pass

    def down(self):
        """Rollback migration"""
        pass

# Migration tracking table
CREATE TABLE schema_migrations (
    id SERIAL PRIMARY KEY,
    version VARCHAR(255) UNIQUE NOT NULL,
    applied_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Step 3: Core Components

**What to do:**
Identify and specify each component:

a) **MigrationLoader**: Discovers and loads migration files
b) **MigrationTracker**: Tracks which migrations are applied
c) **MigrationRunner**: Executes migrations with transaction handling
d) **MigrationGenerator**: Creates new migration files
e) **CLI**: Command-line interface

**Why this third:**
Component identification ensures we don't miss critical pieces.

**Component specs:**
```
MigrationLoader:
- scan_migrations(directory) -> List[Migration]
- load_migration(filename) -> Migration

MigrationTracker:
- is_applied(version) -> bool
- mark_applied(version) -> None
- mark_reverted(version) -> None
- get_applied_migrations() -> List[str]

MigrationRunner:
- apply_migration(migration) -> None
- revert_migration(migration) -> None
- apply_all_pending() -> None

MigrationGenerator:
- create_migration(name) -> str

CLI:
- migrate up [version]
- migrate down [version]
- migrate create [name]
- migrate status
```

### Step 4: Implementation Order

**What to do:**
Implement in dependency order:

**Phase 1: Core Infrastructure**
1. MigrationTracker (no dependencies)
2. MigrationLoader (no dependencies)
3. Base Migration class (no dependencies)

**Phase 2: Execution Layer**
4. MigrationRunner (depends on Tracker + Loader)
5. Transaction handling

**Phase 3: User Interface**
6. MigrationGenerator
7. CLI commands

**Why this order:**
- Foundation first (tracking and loading)
- Execution layer second (uses foundation)
- User interface last (uses all other components)
- Enables testing at each phase

### Step 5: Testing Strategy

**What to do:**

**Unit tests:**
- Test each component in isolation
- Mock dependencies
- Cover edge cases

**Integration tests:**
- Test migration application end-to-end
- Test rollback functionality
- Test error scenarios (failed migration, database errors)

**Test migrations:**
- Create sample migrations for testing
- Test migration ordering
- Test duplicate detection

**Test cases:**
```python
# Unit test example
def test_migration_tracker_marks_applied():
    tracker = MigrationTracker(db_connection)
    tracker.mark_applied("20240101_create_users")
    assert tracker.is_applied("20240101_create_users")

# Integration test example
def test_migration_applies_and_reverts():
    runner = MigrationRunner(db_connection)

    # Apply
    runner.apply_migration(test_migration)
    assert table_exists("test_table")

    # Revert
    runner.revert_migration(test_migration)
    assert not table_exists("test_table")

# Error handling test
def test_failed_migration_rolls_back():
    migration = FailingMigration()
    with pytest.raises(MigrationError):
        runner.apply_migration(migration)
    # Verify database state unchanged
    assert not tracker.is_applied(migration.version)
```

**Why comprehensive testing:**
- Migrations modify production data - errors are costly
- Rollback functionality must be reliable
- Edge cases can cause data corruption

### Implementation Summary

By following these steps in order:
1. We understand what to build (requirements)
2. We know how to build it (design)
3. We identify all pieces needed (components)
4. We build in the right order (implementation)
5. We verify it works correctly (testing)

This systematic approach reduces bugs, enables iterative development, and
produces a maintainable migration system.
```

### Self-Consistency

Self-consistency[^8] **SHOULD** be used when:

- Correctness is critical
- Multiple solution paths exist
- Verification is important

#### Self-Consistency Template

```
Solve this problem using multiple approaches:

## Approach 1: [Method name]
[Solution]

## Approach 2: [Different method]
[Solution]

## Approach 3: [Another method]
[Solution]

## Verification
Compare results. If consistent, high confidence. If different, investigate why.

## Final Answer
[Most reliable solution]
```

#### Detailed Self-Consistency Example

**Prompt:**

```
Calculate the time complexity of this algorithm using multiple approaches:

```python
def find_duplicates(arr):
    result = []
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            if arr[i] == arr[j] and arr[i] not in result:
                result.append(arr[i])
    return result
```

Analyze complexity using:
1. Direct counting method
2. Recurrence relation method
3. Step-by-step execution trace

Verify all approaches agree.
```

**Expected Output:**

```
## Approach 1: Direct Counting Method

**Count operations:**

Outer loop: runs n times (i from 0 to n-1)
Inner loop: runs n-1, n-2, ..., 1 times
- Total iterations: (n-1) + (n-2) + ... + 1 = n(n-1)/2

For each iteration:
- Comparison: O(1)
- `not in` check: O(k) where k = len(result), worst case O(n)
- Append: O(1)

**Calculation:**
- Nested loops: O(n²) iterations
- `not in` check inside: O(n) per iteration
- Total: O(n²) × O(n) = O(n³)

**Result: O(n³)**

---

## Approach 2: Recurrence Relation Method

**Define T(n) = time for array of size n**

T(n) = time for outer loop iteration + T(n-1)

For each outer loop iteration at position i:
- Inner loop runs (n - i - 1) times
- Each iteration has `not in` check: O(n)
- So each outer iteration: O(n) × O(n) = O(n²)

Recurrence:
T(n) = O(n²) + O(n²) + ... (n times) = n × O(n²) = O(n³)

**Result: O(n³)**

---

## Approach 3: Step-by-Step Execution Trace

**Trace with input size n:**

i=0: inner loop (n-1) times, each with O(n) `not in` check = O(n²)
i=1: inner loop (n-2) times, each with O(n) `not in` check = O(n²)
...
i=n-2: inner loop 1 time, with O(n) `not in` check = O(n)

Sum: O(n²) + O(n²) + ... + O(n)
    ≈ n × O(n²)  [dominated by first terms]
    = O(n³)

**Concrete example (n=4):**
- i=0: 3 iterations × O(4) check = 12 operations
- i=1: 2 iterations × O(4) check = 8 operations
- i=2: 1 iteration × O(4) check = 4 operations
- Total: 24 operations ≈ O(4³/2) = O(64/2) = O(32) ✓

**Result: O(n³)**

---

## Verification

All three approaches agree: **O(n³)**

✓ Direct counting: n² iterations × O(n) operation = O(n³)
✓ Recurrence relation: n × O(n²) = O(n³)
✓ Execution trace: confirms O(n³) with concrete example

**Confidence: High** - All methods converge on the same result.

---

## Final Answer

**Time Complexity: O(n³)**

The algorithm has cubic time complexity due to:
1. Nested loops: O(n²)
2. `not in` list search inside loops: O(n)
3. Combined: O(n³)

**Improvement suggestion:**
Use a set for O(1) lookups, reducing to O(n²):

```python
def find_duplicates_optimized(arr):
    seen = set()
    result = set()
    for num in arr:
        if num in seen:
            result.add(num)
        seen.add(num)
    return list(result)
```

This is O(n) time and O(n) space - much better than O(n³).
```

### Tree of Thought

Tree of thought[^3] **SHOULD** be used for:

- Problems with multiple possible solution paths
- Situations requiring exploration of alternatives
- When you need to evaluate trade-offs

#### Tree of Thought Template

```
Problem: [Problem statement]

## Possible Approaches

### Option 1: [Approach name]
**Pros:**
- [Advantage 1]
- [Advantage 2]

**Cons:**
- [Disadvantage 1]
- [Disadvantage 2]

**Viability:** [Score/assessment]

### Option 2: [Approach name]
[Same structure]

### Option 3: [Approach name]
[Same structure]

## Decision Tree

If [condition]: Choose Option 1
Else if [condition]: Choose Option 2
Else: Choose Option 3

## Recommended Solution

Based on [criteria], Option [X] is best because [reasoning].
```

#### Detailed Tree of Thought Example

**Prompt:**

```
Design a caching strategy for a web application. Consider multiple approaches
and evaluate trade-offs:

Requirements:
- Handle 10,000 requests/second
- Cache hit ratio > 80%
- Data can be slightly stale (up to 60 seconds)
- Some data more frequently accessed than others
- Memory limited to 4GB

Use tree of thought to explore options.
```

**Expected Output:**

```
# Caching Strategy: Tree of Thought Analysis

## Problem Statement

Design caching for high-traffic web app with 10K req/s, targeting 80%+ hit rate,
tolerating 60s staleness, with 4GB memory constraint.

---

## Branch 1: Cache Eviction Strategy

### Option 1.1: LRU (Least Recently Used)

**How it works:**
Evict items that haven't been accessed recently.

**Pros:**
+ Simple to implement
+ Works well for temporal locality
+ Predictable behavior
+ Good for most access patterns

**Cons:**
- Doesn't account for access frequency
- Can evict frequently-used items that had temporary access gap
- May not handle burst traffic optimally

**Viability for our use case:**
⭐⭐⭐⭐ (4/5) - Good default choice

**Memory overhead:**
~40-80 bytes per cache entry for doubly-linked list

### Option 1.2: LFU (Least Frequently Used)

**How it works:**
Evict items accessed least frequently.

**Pros:**
+ Keeps frequently-accessed items
+ Better for skewed access patterns (popular items)
+ Aligns with "some data more frequently accessed"

**Cons:**
- More complex implementation
- Slow to adapt to changing access patterns
- Can keep old popular items too long

**Viability for our use case:**
⭐⭐⭐ (3/5) - Good for stable access patterns

**Memory overhead:**
~80-120 bytes per entry for frequency tracking

### Option 1.3: LRU + LFU Hybrid (LRFU/ARC)

**How it works:**
Combine recency and frequency using weighted score or adaptive partitioning.

**Pros:**
+ Best of both worlds
+ Adapts to workload changes
+ Higher hit rates than pure LRU/LFU

**Cons:**
- Complex implementation
- Higher computational overhead
- More memory for metadata

**Viability for our use case:**
⭐⭐⭐⭐⭐ (5/5) - Best for our mixed workload

**Memory overhead:**
~100-150 bytes per entry

---

## Branch 2: Cache Architecture

### Option 2.1: In-Memory (Application-Level)

**Implementation:**
Redis, Memcached, or in-process cache

**Pros:**
+ Extremely fast (microsecond latency)
+ Simple deployment (Redis/Memcached)
+ Easy to reason about

**Cons:**
- Limited to single-server memory (4GB constraint)
- Cache lost on restart (unless Redis persistence)
- Doesn't scale beyond one machine

**Capacity calculation:**
4GB / 150 bytes per entry ≈ 28M entries
At 10K req/s, this is adequate if good hit rate

**Viability:** ⭐⭐⭐⭐ (4/5)

### Option 2.2: Distributed Cache (Multi-Node)

**Implementation:**
Redis Cluster, Memcached distributed, Hazelcast

**Pros:**
+ Scales horizontally
+ Higher total capacity
+ Redundancy/availability

**Cons:**
- Network latency added
- Complexity in consistency
- More operational overhead

**Viability:** ⭐⭐⭐ (3/5) - Overkill for 10K req/s

### Option 2.3: Two-Tier (L1 + L2)

**Implementation:**
- L1: In-process cache (small, fast)
- L2: Redis (larger, shared)

**Pros:**
+ Best latency for hot items (L1)
+ Shared cache benefits (L2)
+ Optimal resource usage

**Cons:**
- Complexity in invalidation
- Cache coherency challenges
- More moving parts

**Viability:** ⭐⭐⭐⭐⭐ (5/5) - Excellent for high traffic

---

## Branch 3: TTL (Time-to-Live) Strategy

### Option 3.1: Fixed TTL

**Implementation:**
All entries expire after 60 seconds

**Pros:**
+ Simple to implement
+ Predictable staleness
+ Guaranteed freshness bound

**Cons:**
- Simultaneous expirations cause load spikes
- Doesn't differentiate by data type

**Viability:** ⭐⭐⭐ (3/5)

### Option 3.2: Variable TTL by Content Type

**Implementation:**
- Static content: 300s
- User data: 60s
- Real-time data: 10s

**Pros:**
+ Optimized per data type
+ Higher hit rate for stable data
+ Reduced load on origin

**Cons:**
- Requires classification logic
- More complex invalidation

**Viability:** ⭐⭐⭐⭐ (4/5)

### Option 3.3: TTL + Active Invalidation

**Implementation:**
60s TTL + event-driven invalidation

**Pros:**
+ Best of both: safety (TTL) + freshness (invalidation)
+ No stale data for critical updates
+ High hit rate

**Cons:**
- Most complex
- Requires event system
- Invalidation logic needed

**Viability:** ⭐⭐⭐⭐⭐ (5/5) - Best for production

---

## Decision Tree

```
Start
├─ Is traffic > 50K req/s?
│  ├─ Yes → Use Option 2.2 (Distributed)
│  └─ No → Continue
│
├─ Is single-server memory sufficient?
│  ├─ No → Use Option 2.2 (Distributed)
│  └─ Yes → Continue
│
├─ Need sub-millisecond latency?
│  ├─ Yes → Use Option 2.3 (Two-Tier)
│  └─ No → Use Option 2.1 (In-Memory)
│
└─ For eviction:
   ├─ Simple workload → Option 1.1 (LRU)
   ├─ Highly skewed → Option 1.2 (LFU)
   └─ Mixed/Unknown → Option 1.3 (Hybrid)
```

---

## Recommended Solution

**Chosen Architecture: Two-Tier Cache with Hybrid Eviction and TTL+Invalidation**

### Rationale

Given our requirements:
- 10K req/s fits in-memory but benefits from two-tier
- Mixed access patterns benefit from hybrid eviction
- 60s staleness tolerance enables TTL, but invalidation improves freshness
- 4GB memory is sufficient for L2, L1 adds performance boost

### Proposed Implementation

**L1 Cache (In-Process):**
- Size: 256MB per app server
- Eviction: LRU (simplicity for small cache)
- TTL: 10 seconds
- Purpose: Ultra-hot items, sub-millisecond access

**L2 Cache (Redis):**
- Size: 4GB
- Eviction: LRFU (allkeys-lru with frequency hints)
- TTL: Variable (10-300s based on content type)
- Purpose: Shared cache, high hit rate

**Invalidation:**
- Pub/sub pattern for critical updates
- Invalidate both L1 and L2 on update events

**Configuration:**

```python
# L1 Cache (in-process)
l1_cache = LRUCache(
    max_size_mb=256,
    default_ttl=10
)

# L2 Cache (Redis)
redis_client = Redis(
    host='cache-cluster',
    maxmemory='4gb',
    maxmemory_policy='allkeys-lru',
    eviction_policy='lrfu'
)

# Cache wrapper
class TwoTierCache:
    def get(self, key):
        # Check L1 first
        value = l1_cache.get(key)
        if value is not None:
            return value

        # Check L2
        value = redis_client.get(key)
        if value is not None:
            # Populate L1 for next access
            l1_cache.set(key, value)
            return value

        # Cache miss
        return None

    def set(self, key, value, ttl=60):
        # Set in both caches
        l1_cache.set(key, value, ttl=min(ttl, 10))
        redis_client.setex(key, ttl, value)

    def invalidate(self, key):
        # Remove from both
        l1_cache.delete(key)
        redis_client.delete(key)
        # Publish invalidation event for other servers
        redis_client.publish('cache:invalidate', key)
```

### Expected Performance

- **Hit rate**: 85-90% (L1: ~40%, L2: ~50%)
- **Latency**:
  - L1 hit: <1ms
  - L2 hit: 1-3ms
  - Miss: 10-50ms (origin fetch)
- **Memory usage**:
  - L1: 256MB × N servers
  - L2: 4GB shared
- **Throughput**: Supports 10K req/s with headroom

### Why This Beats Other Options

✓ Better than pure in-memory: L1 provides speed boost for hot items
✓ Better than distributed: Simpler, lower latency, adequate for traffic
✓ Better than LRU-only: Hybrid handles mixed access patterns
✓ Better than fixed TTL: Variable TTL + invalidation optimizes freshness

This design maximizes hit rate while staying within constraints and provides
a clear upgrade path if traffic grows.
```

---

## Reasoning Techniques: Summary

| Technique | Use When | Benefit |
|-----------|----------|---------|
| Chain-of-Thought | Multi-step problems | Makes reasoning explicit |
| Step-by-Step | Sequential tasks | Ensures proper ordering |
| Self-Consistency | Verification needed | Increases confidence |
| Tree of Thought | Multiple options | Explores alternatives |

All reasoning techniques **SHOULD** be combined with clear formatting and
explicit structure to maximize clarity and verifiability.

For provider-specific guidance on reasoning techniques, consult:
- Anthropic Claude documentation[^4]
- OpenAI GPT best practices[^5]
- Google Gemini prompting guide[^6]

---

## Few-Shot Patterns

Few-shot learning[^2] provides the model with examples of the desired input-output behavior before the actual task.

### When to Use Few-Shot

Few-shot examples **SHOULD** be used when:

- Output format is complex or specific
- Task is ambiguous without examples
- Consistent style/tone is required
- Edge cases need to be demonstrated
- You need to override default behavior

Few-shot examples **SHOULD NOT** be used when:

- Task is simple and self-explanatory
- Instructions alone are sufficient
- Examples would add too much token cost
- Zero-shot works adequately

### Few-Shot Structure

Few-shot prompts **MUST** follow this structure:

```
[System message - role and constraints]

[Example 1 - User message]
[Example 1 - Assistant response]

[Example 2 - User message]
[Example 2 - Assistant response]

[Example N - User message]
[Example N - Assistant response]

[Actual query - User message]
```

### Example Selection Criteria

Examples **SHOULD** be selected based on:

1. **Diversity**: Cover different input types
2. **Difficulty**: Include both simple and complex cases
3. **Edge cases**: Show how to handle unusual inputs
4. **Format**: Demonstrate exact output format desired
5. **Relevance**: Similar to actual use case

Examples **MUST** be:

- Accurate and correct
- Representative of actual usage
- Consistent in format and style
- High quality (not just quantity)

### Optimal Number of Examples

- **Simple tasks**: 1-2 examples
- **Moderate complexity**: 3-5 examples
- **Complex tasks**: 5-10 examples
- **Rarely**: More than 10 examples

### Format Consistency

All examples **MUST** maintain:

- Identical output structure
- Consistent formatting conventions
- Same level of detail
- Uniform style and tone

### Detailed Few-Shot Examples

#### Example 1: Code Review Comments

**Few-Shot Prompt Example:**

```
System: You are a code reviewer. Provide constructive feedback in this format:
- **Issue**: Description
- **Severity**: Critical/Warning/Suggestion
- **Fix**: Specific recommendation

User: Review this code:
```python
x = input()
print(x * 2)
```

---

## References

[^1]: Wei, J., et al. (2022). "Chain-of-Thought Prompting Elicits Reasoning in Large Language Models." *arXiv:2201.11903*. [https://arxiv.org/abs/2201.11903](https://arxiv.org/abs/2201.11903)

[^2]: Brown, T., et al. (2020). "Language Models are Few-Shot Learners." *NeurIPS 2020*. [https://arxiv.org/abs/2005.14165](https://arxiv.org/abs/2005.14165)

[^3]: Yao, S., et al. (2023). "Tree of Thoughts: Deliberate Problem Solving with Large Language Models." *arXiv:2305.10601*. [https://arxiv.org/abs/2305.10601](https://arxiv.org/abs/2305.10601)

[^4]: Anthropic Claude Documentation. "Prompt Engineering Guide." [https://docs.anthropic.com/claude/docs/prompt-engineering](https://docs.anthropic.com/claude/docs/prompt-engineering)

[^5]: OpenAI GPT Best Practices. "Prompt Engineering." [https://platform.openai.com/docs/guides/prompt-engineering](https://platform.openai.com/docs/guides/prompt-engineering)

[^6]: Google AI Studio Documentation. "Prompting Guide for Gemini." [https://ai.google.dev/docs/prompting_intro](https://ai.google.dev/docs/prompting_intro)

[^7]: Zhou, D., et al. (2022). "Large Language Models Are Human-Level Prompt Engineers." *arXiv:2211.01910*. [https://arxiv.org/abs/2211.01910](https://arxiv.org/abs/2211.01910)

[^8]: Wang, X., et al. (2022). "Self-Consistency Improves Chain of Thought Reasoning in Language Models." *arXiv:2203.11171*. [https://arxiv.org/abs/2203.11171](https://arxiv.org/abs/2203.11171)

[^9]: Liu, P., et al. (2023). "Pre-train, Prompt, and Predict: A Systematic Survey of Prompting Methods in Natural Language Processing." *ACM Computing Surveys*. [https://arxiv.org/abs/2107.13586](https://arxiv.org/abs/2107.13586)

[^10]: Anthropic. "Introduction to Prompt Design." *Anthropic Documentation*. [https://docs.anthropic.com/claude/docs/introduction-to-prompt-design](https://docs.anthropic.com/claude/docs/introduction-to-prompt-design)

[^11]: Reynolds, L., & McDonell, K. (2021). "Prompt Programming for Large Language Models: Beyond the Few-Shot Paradigm." *CHI Extended Abstracts 2021*. [https://arxiv.org/abs/2102.07350](https://arxiv.org/abs/2102.07350)

[^12]: Ouyang, L., et al. (2022). "Training language models to follow instructions with human feedback." *arXiv:2203.02155*. [https://arxiv.org/abs/2203.02155](https://arxiv.org/abs/2203.02155)