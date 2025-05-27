# Phase 4: Polyglot Code Generation Demonstration - User Profile Display

This phase demonstrates the AI Council's capability in handling a polyglot feature request, specifically a "User Profile Display." We'll show example prompts for generating tests (Guardian), code (Builder, guided by Pacer's plan), and refining code (Artisan) for both Python/FastAPI backend and TypeScript/React frontend components. We'll also discuss how Guardian handles polyglot test generation and how linters/formatters are managed.

## 1. Example Feature: User Profile Display

**Feature Request:**
"As a user, I want to view my user profile information, including my username, email, full name, and join date. The backend should expose an API endpoint to fetch this data, and the frontend should display it clearly on a profile page."

**Technical Stack:**
*   **Backend:** Python 3.9+, FastAPI, Pytest
*   **Frontend:** TypeScript, React, Jest, React Testing Library

## 2. Example Prompts for the AI Council

These prompts are conceptual and would be sent to the respective LLMs via the LiteLLM proxy. They include bracketed placeholders like `[User Story]` or `[Test File Content]` which would be dynamically filled by the orchestrating GitHub Actions workflow.

### 2.1. Guardian: Generating Tests

Guardian is prompted first to generate tests based on Pacer's breakdown of the feature. It needs separate, targeted prompts for each language/framework.

**Prompt for Python/Pytest (Backend API Tests):**

```
Role: Guardian (Test Generation Specialist)
Task: Generate Pytest tests for a FastAPI backend endpoint.
Feature: [User Story: "As a user, I want to view my user profile information, including my username, email, full name, and join date. The backend should expose an API endpoint to fetch this data..."]

Details:
1.  The API endpoint should be `/api/v1/users/me/profile`.
2.  It should be a GET request.
3.  Assume the user is authenticated (mock authentication if necessary).
4.  The expected successful response (200 OK) should be a JSON object containing:
    *   `username`: string
    *   `email`: string (valid email format)
    *   `full_name`: string
    *   `join_date`: string (ISO 8601 date format, e.g., "YYYY-MM-DD")
5.  Generate tests for:
    *   Successful retrieval of a user profile.
    *   User not authenticated (expect 401/403 error, mock this).
    *   User profile not found (expect 404 error, if applicable for your data model).
    *   Validate data types and presence of all fields in a successful response.
6.  Use standard Pytest conventions. Mock any database interactions or external service calls.
7.  Output the test code in a single Python file named `test_user_profile_api.py`.
8.  Ensure tests are self-contained and runnable. Include necessary imports (e.g., `FastAPI`, `TestClient`, `pytest`).

Example (minimal) structure for a test:
```python
from fastapi.testclient import TestClient
from main_app import app # Assuming your FastAPI app instance is in main_app.py

client = TestClient(app)

def test_read_user_profile_success():
    # Mock authentication if needed
    response = client.get("/api/v1/users/me/profile" #, headers={"Authorization": "Bearer fake-token"}
    )
    assert response.status_code == 200
    data = response.json()
    assert "username" in data
    assert "email" in data
    # ... more assertions
```
```

**Prompt for TypeScript/Jest (Frontend Component Tests):**

```
Role: Guardian (Test Generation Specialist)
Task: Generate Jest and React Testing Library tests for a React frontend component.
Feature: [User Story: "...the frontend should display it clearly on a profile page."]

Details:
1.  The component will be named `UserProfileDisplay.tsx`.
2.  It will receive user profile data as props:
    ```typescript
    interface UserProfileData {
      username: string;
      email: string;
      fullName: string;
      joinDate: string; // Display as human-readable, e.g., "January 1, 2024"
    }
    ```
3.  Generate tests to:
    *   Render the username, email, full name, and formatted join date correctly from props.
    *   Display a loading state (e.g., "Loading profile...") when `isLoading` prop is true.
    *   Display an error message (e.g., "Failed to load profile.") when `error` prop is provided.
    *   Ensure accessibility (e.g., appropriate roles, aria-labels if complex).
4.  Use Jest and React Testing Library (`@testing-library/react`).
5.  Output the test code in a single TypeScript file named `UserProfileDisplay.test.tsx`.
6.  Ensure tests are self-contained. Import the component to be tested.

Example (minimal) structure for a test:
```typescript
import { render, screen } from '@testing-library/react';
import UserProfileDisplay, { UserProfileData } from './UserProfileDisplay'; // Assuming component path

describe('UserProfileDisplay', () => {
  const mockProfile: UserProfileData = {
    username: 'testuser',
    email: 'test@example.com',
    fullName: 'Test User',
    joinDate: '2024-01-15T10:00:00Z', // Will need formatting
  };

  it('renders user profile information correctly', () => {
    render(<UserProfileDisplay profile={mockProfile} isLoading={false} error={null} />);
    expect(screen.getByText(mockProfile.username)).toBeInTheDocument();
    expect(screen.getByText(mockProfile.email)).toBeInTheDocument();
    // ... more assertions for fullName and formatted joinDate
  });
});
```
```

### 2.2. Pacer (Planning) / Builder (Code Generation)

Pacer would break down the feature into backend and frontend tasks. Builder then receives prompts to generate code to pass the tests created by Guardian.

**Prompt for Python/FastAPI (Backend Code Generation):**

```
Role: Builder (Code Generation Specialist)
Task: Generate Python FastAPI code to implement the user profile API endpoint.
Feature: [User Story: Backend portion for User Profile Display]
Associated Tests: [Content of `test_user_profile_api.py` generated by Guardian]

Details:
1.  Implement the API endpoint `/api/v1/users/me/profile` using FastAPI.
2.  The endpoint should handle GET requests.
3.  It must pass all tests in the provided Pytest file (`test_user_profile_api.py`).
4.  Define a Pydantic model for the response with fields: `username`, `email`, `full_name`, `join_date`.
5.  For now, you can return mock/hardcoded data that satisfies the tests. Do not implement actual database logic or authentication checks; focus on the API structure and test passability.
    *   Mock user data: `{"username": "testuser", "email": "test@example.com", "full_name": "Test User", "join_date": "2023-05-15"}`
6.  Structure the code clearly, potentially in a dedicated router file (e.g., `routers/user_profile.py`) and a models file (e.g., `models/user.py`).
7.  Output the Python code files. Name the main FastAPI application file `main_app.py` if creating a new one, or specify how to integrate into an existing structure.

Example Response Model (in `models/user.py`):
```python
from pydantic import BaseModel, EmailStr
from datetime import date

class UserProfileResponse(BaseModel):
    username: str
    email: EmailStr
    full_name: str
    join_date: date # FastAPI will serialize this to ISO 8601 string
```
```

**Prompt for TypeScript/React (Frontend Code Generation):**

```
Role: Builder (Code Generation Specialist)
Task: Generate TypeScript React code for the `UserProfileDisplay` component.
Feature: [User Story: Frontend portion for User Profile Display]
Associated Tests: [Content of `UserProfileDisplay.test.tsx` generated by Guardian]

Details:
1.  Implement the `UserProfileDisplay.tsx` React component.
2.  The component must pass all tests in the provided Jest/RTL test file (`UserProfileDisplay.test.tsx`).
3.  It should accept props: `profile: UserProfileData | null`, `isLoading: boolean`, `error: string | null`.
    ```typescript
    interface UserProfileData {
      username: string;
      email: string;
      fullName: string;
      joinDate: string; // ISO 8601 string from API
    }
    ```
4.  Display the profile information. Format the `joinDate` (e.g., from "2024-01-15T10:00:00Z" to "January 15, 2024").
5.  Handle `isLoading` and `error` states as per the tests.
6.  Use functional components and hooks.
7.  Output the TypeScript React component file (`UserProfileDisplay.tsx`).

Example component structure:
```typescript
import React from 'react';

interface UserProfileData { /* ... as above ... */ }

interface UserProfileDisplayProps {
  profile: UserProfileData | null;
  isLoading: boolean;
  error: string | null;
}

const UserProfileDisplay: React.FC<UserProfileDisplayProps> = ({ profile, isLoading, error }) => {
  if (isLoading) return <div>Loading profile...</div>;
  if (error) return <div>{error}</div>;
  if (!profile) return <div>No profile data.</div>;

  const formatDate = (isoDate: string) => {
    return new Date(isoDate).toLocaleDateString('en-US', {
      year: 'numeric', month: 'long', day: 'numeric'
    });
  };

  return (
    // JSX to display username, email, fullName, formatted joinDate
  );
};

export default UserProfileDisplay;
```
```

### 2.3. Artisan: Refining Code

Artisan is called if tests fail, or for a general quality pass even if tests pass.

**Prompt for Artisan (Code Refinement - e.g., if a Python test failed):**

```
Role: Artisan (Code Refinement Specialist)
Task: Refine Python FastAPI code to fix failing tests and improve quality.
Feature: [User Story: Backend portion for User Profile Display]
Code Files: [Content of `routers/user_profile.py`, `models/user.py`]
Test File: [Content of `test_user_profile_api.py`]
Test Output/Error: [Specific error message from Pytest, e.g., "AssertionError: Expected status code 200 but got 404 on /api/v1/users/me/profile"]

Details:
1.  Analyze the provided code and the failing test output.
2.  Identify the root cause of the test failure.
3.  Modify the Python code to make the test pass.
4.  Additionally, review the code for:
    *   Adherence to FastAPI best practices.
    *   Clarity and readability.
    *   Proper error handling (even if not explicitly in the failing test).
    *   Efficient data handling (though current focus is mock data).
5.  Output the refined Python code files. Explain the changes made.
```

**Prompt for Artisan (General Frontend Refinement - even if tests pass):**

```
Role: Artisan (Code Refinement Specialist)
Task: Refine the `UserProfileDisplay.tsx` React component for best practices and robustness.
Feature: [User Story: Frontend portion for User Profile Display]
Code File: [Content of `UserProfileDisplay.tsx`]
Test File: [Content of `UserProfileDisplay.test.tsx`]

Details:
1.  Review the React component for:
    *   React best practices (e.g., hook usage, component composition, memoization if applicable).
    *   TypeScript best practices (e.g., type safety, clarity of interfaces).
    *   Accessibility improvements beyond basic test coverage.
    *   Potential edge cases or UI inconsistencies.
    *   Code style and readability.
    *   Consider internationalization (i18n) needs for date formatting if applicable (though current prompt uses `toLocaleDateString`).
2.  Output the refined `UserProfileDisplay.tsx` file. Explain any significant changes or recommendations.
```

## 3. Guardian's Polyglot Test Generation

Guardian handles polyglot test generation by:

1.  **Targeted Prompts:** As seen above, Guardian receives distinct prompts for each language and testing framework. The prompt includes specific instructions, expected file names, and examples relevant to that language's ecosystem (Pytest for Python, Jest/RTL for TypeScript/React).
2.  **Specialized Knowledge:** Guardian (as an LLM fine-tuned or prompted for code understanding) has knowledge of different programming languages, their common testing frameworks, and idiomatic test structures.
3.  **File Naming Conventions:** The prompts explicitly request specific file names that align with common conventions for each language:
    *   Python: `test_*.py` or `*_test.py` (e.g., `test_user_profile_api.py`).
    *   TypeScript/JavaScript (Jest): `*.test.ts`, `*.spec.ts`, `*.test.tsx`, `*.spec.tsx` (e.g., `UserProfileDisplay.test.tsx`).
    The TDD workflow would then know where to place these files within the repository structure (e.g., `src/python/tests/`, `src/nodejs/tests/`).
4.  **Separate Outputs:** Guardian generates each set of tests as a separate output (e.g., one block of code for Python tests, another for TypeScript tests). The GitHub Actions workflow is responsible for routing these to the correct locations in the workspace or artifacts.

## 4. Managing Linters and Formatters in the TDD Workflow

Integrating linters and formatters is crucial for maintaining code quality, consistency, and readability, especially with AI-generated code.

**General Strategy:**
*   Linters and formatters are run automatically in the GitHub Actions workflow after code generation (and potentially after refinement).
*   They can be configured to `--check` for violations (fail the build) or to automatically fix issues and commit changes. The latter is often preferred for formatters.

**Language-Specific Configurations:**

*   **Python (Black & Flake8/Ruff):**
    *   **Configuration:**
        *   `pyproject.toml`: Configure Black (line length, target Python versions) and Flake8 (or Ruff, which is faster and can replace Flake8, isort).
        ```toml
        # pyproject.toml example
        [tool.black]
        line-length = 88
        target-version = ['py39']

        [tool.flake8] # Or [tool.ruff]
        max-line-length = 88
        extend-ignore = "E203,W503" # Example ignores
        # For Ruff:
        # [tool.ruff]
        # line-length = 88
        # select = ["E", "F", "W", "I"] # Select rules
        ```
    *   **CI Steps (in `tdd_pipeline.yml` under `run_tests` or a dedicated linting job):**
        ```yaml
        - name: Setup Python
          uses: actions/setup-python@v4
          with:
            python-version: '3.9'
        - name: Install Python Linters
          run: pip install black flake8 # or ruff
        - name: Run Black Formatter Check
          run: black --check . --config pyproject.toml
        - name: Run Flake8 Linter # or Ruff
          run: flake8 . --config pyproject.toml # or ruff check . --config pyproject.toml
        # Optional: Auto-fix and commit (more complex, requires PAT or app with write perms)
        # - name: Auto-format with Black (if check fails)
        #   if: failure() # Or a specific condition based on black --check output
        #   run: |
        #     black . --config pyproject.toml
        #     git config --global user.name 'GitHub Actions Linter'
        #     git config --global user.email 'actions@github.com'
        #     git commit -am "Chore: Auto-format Python code with Black" || echo "No changes to commit"
        #     git push
        ```

*   **JavaScript/TypeScript (ESLint & Prettier):**
    *   **Configuration:**
        *   `.eslintrc.js` (or `.json`, `.yaml`): Configure ESLint rules, plugins (e.g., `@typescript-eslint/eslint-plugin`, `eslint-plugin-react`, `eslint-plugin-jest`).
        *   `.prettierrc.js` (or `.json`, `.yaml`): Configure Prettier formatting options (tab width, semi-colons, print width).
        *   `package.json`: Add linting/formatting scripts.
        ```json
        // package.json (scripts section)
        "scripts": {
          "lint": "eslint 'src/**/*.{js,jsx,ts,tsx}' --quiet",
          "format:check": "prettier --check 'src/**/*.{js,jsx,ts,tsx,json,css,md}'",
          "format:write": "prettier --write 'src/**/*.{js,jsx,ts,tsx,json,css,md}'"
        }
        ```
    *   **CI Steps (in `tdd_pipeline.yml` under `run_tests` or a dedicated linting job):**
        ```yaml
        - name: Setup Node.js
          uses: actions/setup-node@v4
          with:
            node-version: '18' # Or your project's version
        - name: Install JS/TS Dependencies (including linters)
          # Assuming linters are in devDependencies
          run: npm ci # or yarn install --frozen-lockfile
        - name: Run ESLint
          run: npm run lint
        - name: Run Prettier Check
          run: npm run format:check
        # Optional: Auto-fix and commit (similar to Python example)
        # - name: Auto-format with Prettier (if check fails)
        #   if: failure()
        #   run: |
        #     npm run format:write
        #     # Git commit and push logic here
        ```

**Workflow Integration:**

1.  **Placement:** Linting/formatting steps should ideally run *before* tests, as syntactic errors or major formatting issues can sometimes prevent tests from running correctly. However, they can also run in parallel or after tests if the primary goal is to enforce standards on code that is already functionally tested.
2.  **Failure Policy:** Decide if linter/formatter violations should fail the build (`--check` mode). For formatters, it's common to have a separate step that auto-fixes and commits, or to provide instructions for developers to fix locally.
3.  **Feedback to LLMs:** If linters/formatters fail on AI-generated code, this feedback could potentially be passed back to Artisan for a refinement iteration, prompting it to adhere to specific style guides. This adds complexity but could improve the quality of direct AI output.
    *   Example prompt addition for Artisan: "Additionally, ensure the generated code passes `flake8` with the provided `pyproject.toml` configuration and `black` formatting."

By incorporating these practices, the TDD engine can ensure that the AI-generated code is not only functional (passes tests) but also clean, consistent, and maintainable across different languages.
```
