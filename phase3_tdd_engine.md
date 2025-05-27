# Phase 3: The TDD Engine - AI-Powered Test-Driven Development

This phase details the setup of an automated Test-Driven Development (TDD) pipeline using GitHub Actions, orchestrated by the "Pacer" LLM and leveraging the other AI Council members for specific tasks. The pipeline will take a feature description, generate tests, generate code to pass those tests, run the tests, and iterate if necessary.

**Disclaimer:** The `tdd_pipeline.yml` provided below uses **mocked AI interactions and file manipulations**. Actual implementation would require:
*   Scripts to make HTTP POST requests to the LiteLLM endpoint for each AI Council member.
*   Robust parsing of LLM responses (e.g., JSON, YAML).
*   Intelligent file creation and modification based on LLM outputs.
*   Actual test execution commands for each language.
*   Logic for handling LLM errors, retries, and potentially human feedback loops.

## 1. Conceptual Repository Setup (Polyglot Project)

A clear repository structure is crucial for managing a polyglot project and for the AI agents to understand where to place files.

```
.
├── .github/
│   └── workflows/
│       └── tdd_pipeline.yml  # Our main CI/CD pipeline
├── src/
│   ├── python/
│   │   ├── app/              # Python application code
│   │   │   └── __init__.py
│   │   └── tests/            # Python tests
│   │       └── __init__.py
│   ├── nodejs/
│   │   ├── services/         # Node.js service code
│   │   │   └── service.js
│   │   └── tests/            # Node.js tests
│   │       └── service.test.js
│   └── java/
│       ├── src/main/java/    # Java source code
│       │   └── com/example/
│       │       └── Main.java
│       └── src/test/java/    # Java tests
│           └── com/example/
│               └── MainTest.java
├── docs/                     # Project documentation
├── scripts/                  # Helper scripts (e.g., for calling LLMs, parsing)
│   └── call_llm.sh           # Example script placeholder
└── README.md
```

This structure separates source code and tests by language, making it easier for both humans and AI to navigate and manage.

## 2. GitHub Actions Workflow: `tdd_pipeline.yml`

This workflow automates the TDD process.

```yaml
# .github/workflows/tdd_pipeline.yml
name: AI TDD Engine

on:
  workflow_dispatch:
    inputs:
      feature_description:
        description: 'Detailed description of the new feature or improvement'
        required: true
        type: string
      target_languages:
        description: 'Comma-separated list of languages to implement (e.g., python,nodejs,java)'
        required: true
        type: string
        default: 'python'
      pr_branch_name:
        description: 'Name for the new branch to create the PR from (e.g., feature/new-widget)'
        required: true
        type: string
  pull_request:
    branches:
      - main # Or your development branch
    # Optional: Trigger on specific labels or comments in a PR for refinement loops

env:
  LITELLM_ENDPOINT_URL: ${{ secrets.LITELLM_API_ENDPOINT }} # URL for your LiteLLM Azure Container App
  LITELLM_API_KEY: ${{ secrets.LITELLM_API_KEY }} # If LiteLLM itself is protected by a key

jobs:
  generate_tests:
    runs-on: ubuntu-latest
    outputs:
      test_plan_id: ${{ steps.pacer_call.outputs.plan_id }}
      languages: ${{ github.event.inputs.target_languages || 'python' }} # Default if not from dispatch
      pr_branch: ${{ github.event.inputs.pr_branch_name || format('ai-tdd/{0}-{1}', github.run_id, github.run_attempt) }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ github.head_ref || github.ref }} # Checkout PR branch or current ref

      - name: Create new branch for AI work (if workflow_dispatch)
        if: github.event_name == 'workflow_dispatch'
        run: |
          git checkout -b ${{ github.event.inputs.pr_branch_name }}
          git push --set-upstream origin ${{ github.event.inputs.pr_branch_name }}
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}

      - name: "Debug: Initial branch info"
        run: |
          echo "Running on branch: $(git rev-parse --abbrev-ref HEAD)"
          echo "Target languages: ${{ github.event.inputs.target_languages || 'python' }}"
          echo "PR Branch Name: ${{ github.event.inputs.pr_branch_name || 'N/A' }}"

      - name: "[MOCK] Call Pacer (LLM) for Test Plan"
        id: pacer_call
        run: |
          echo "Feature Description: ${{ github.event.inputs.feature_description }}"
          echo "Target Languages: ${{ github.event.inputs.target_languages }}"
          echo "Simulating Pacer LLM: Generating test plan..."
          # Real implementation: call_llm.sh pacer-mistral-7b --prompt "Create a test plan for feature: ${{ github.event.inputs.feature_description }} for languages ${{ github.event.inputs.target_languages }}"
          mkdir -p .ai_artifacts/pacer
          echo '{"plan_id": "plan_123", "tests_to_generate": [{"language": "python", "description": "Test for user login"}, {"language": "nodejs", "description": "Test for API endpoint"}]}' > .ai_artifacts/pacer/test_plan.json
          echo "plan_id=plan_$(date +%s)" >> $GITHUB_OUTPUT # Unique ID for this plan
          echo "Test plan generated: .ai_artifacts/pacer/test_plan.json"

      - name: "[MOCK] Call Maven (LLM) for Context/RAG"
        id: maven_call
        run: |
          echo "Simulating Maven LLM: Retrieving relevant context, code snippets, docs for plan_id ${{ steps.pacer_call.outputs.plan_id }}..."
          # Real implementation: call_llm.sh maven-phi-3-mini --prompt "Gather context for test plan ${{ steps.pacer_call.outputs.plan_id }}" --input_files "src/**"
          mkdir -p .ai_artifacts/maven
          echo "Relevant context for plan_123..." > .ai_artifacts/maven/context.txt
          echo "Context gathered: .ai_artifacts/maven/context.txt"

      - name: "[MOCK] Call Builder (LLM) to Generate Tests"
        id: builder_generate_tests
        run: |
          echo "Simulating Builder LLM: Generating test files based on plan and context..."
          # Real implementation: Loop through test_plan.json, call Builder for each test
          # call_llm.sh builder-llama-3-8b --prompt "Generate test code for: Test for user login (Python)" --context_file .ai_artifacts/maven/context.txt
          mkdir -p .ai_artifacts/builder/tests/python/tests .ai_artifacts/builder/tests/nodejs/tests
          echo "def test_user_login(): assert True" > .ai_artifacts/builder/tests/python/tests/test_login.py
          echo "console.log('Mock Node.js test for API endpoint');" > .ai_artifacts/builder/tests/nodejs/tests/test_api.js
          echo "Test files generated in .ai_artifacts/builder/tests/"
          # In a real scenario, these files would be placed in the correct src/LANG/tests/ path and committed.
          # For this mock, we use artifacts.

      - name: Upload Test Plan and Generated Tests
        uses: actions/upload-artifact@v4
        with:
          name: ai-generated-tests-${{ steps.pacer_call.outputs.plan_id }}
          path: |
            .ai_artifacts/pacer/test_plan.json
            .ai_artifacts/maven/context.txt
            .ai_artifacts/builder/tests/

  generate_code:
    needs: generate_tests
    runs-on: ubuntu-latest
    outputs:
      pr_branch: ${{ needs.generate_tests.outputs.pr_branch }}
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.generate_tests.outputs.pr_branch }} # Checkout the branch created/used in generate_tests

      - name: Download Test Plan and Generated Tests
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/

      - name: "[MOCK] Call Builder (LLM) to Generate Code"
        id: builder_generate_code
        run: |
          echo "Simulating Builder LLM: Generating source code to pass the tests..."
          # Real implementation: Loop through tests, call Builder to generate implementation code
          # e.g., call_llm.sh builder-llama-3-8b --prompt "Generate Python code to pass test_login.py" --test_file .ai_artifacts/builder/tests/python/tests/test_login.py --context_file .ai_artifacts/context.txt
          mkdir -p .ai_artifacts/builder/src/python/app .ai_artifacts/builder/src/nodejs/services
          echo "def user_login_feature(): return True" > .ai_artifacts/builder/src/python/app/login.py
          echo "function apiEndpointFeature() { console.log('API logic'); return true; }" > .ai_artifacts/builder/src/nodejs/services/api.js
          echo "Source code generated in .ai_artifacts/builder/src/"
          # Real: Place files in actual src/LANG/app_or_service path and commit.

      - name: Upload Generated Source Code
        uses: actions/upload-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/builder/src/

  run_tests:
    needs: [generate_tests, generate_code]
    runs-on: ubuntu-latest
    outputs:
      test_outcome: ${{ steps.determine_outcome.outputs.outcome }} # success or failure
      pr_branch: ${{ needs.generate_code.outputs.pr_branch }}
    strategy:
      matrix:
        language: ${{ fromJson(format('["%s"]', replace(needs.generate_tests.outputs.languages, ',', '","'))) }} # Dynamically create matrix from input
      fail-fast: false # Allow all language tests to run even if one fails
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.generate_code.outputs.pr_branch }}

      - name: Download Generated Tests
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts-tests/ # Separate path to avoid conflicts if src is also downloaded

      - name: Download Generated Source Code
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts-src/

      - name: "[MOCK] Setup Environment for ${{ matrix.language }}"
        run: |
          echo "Setting up environment for ${{ matrix.language }}..."
          # Real: Install Python, Node.js, JDK, Maven/Gradle, dependencies etc.
          # if [ "${{ matrix.language }}" == "python" ]; then pip install -r src/python/requirements.txt; fi
          # if [ "${{ matrix.language }}" == "nodejs" ]; then cd src/nodejs && npm install; cd ../..; fi

      - name: "[MOCK] Copy generated files to workspace for ${{ matrix.language }}"
        # This step simulates committing the files to the branch.
        # In a real workflow, generate_tests and generate_code would commit their files.
        run: |
          echo "Copying generated files for ${{ matrix.language }}..."
          # Example for Python:
          if [ "${{ matrix.language }}" == "python" ]; then
            mkdir -p src/python/tests src/python/app
            cp -r .ai_artifacts-tests/builder/tests/python/tests/* src/python/tests/ || echo "No Python tests found in artifact"
            cp -r .ai_artifacts-src/builder/src/python/app/* src/python/app/ || echo "No Python src found in artifact"
          fi
          # Example for Node.js:
          if [ "${{ matrix.language }}" == "nodejs" ]; then
            mkdir -p src/nodejs/tests src/nodejs/services
            cp -r .ai_artifacts-tests/builder/tests/nodejs/tests/* src/nodejs/tests/ || echo "No Node.js tests found in artifact"
            cp -r .ai_artifacts-src/builder/src/nodejs/services/* src/nodejs/services/ || echo "No Node.js src found in artifact"
          fi
          # Add similar blocks for Java, etc.
          echo "Listing files:"
          ls -R src/

      - name: "[MOCK] Run ${{ matrix.language }} Tests"
        run: |
          echo "Running ${{ matrix.language }} tests..."
          # Real implementation:
          # if [ "${{ matrix.language }}" == "python" ]; then pytest src/python/tests/; fi
          # if [ "${{ matrix.language }}" == "nodejs" ]; then cd src/nodejs && npm test; cd ../..; fi
          # if [ "${{ matrix.language }}" == "java" ]; then mvn -f src/java/pom.xml test; fi
          if [ "${{ matrix.language }}" == "python" ] && [ ! -f src/python/tests/test_login.py ]; then echo "Python test file missing!"; exit 1; fi
          if [ "${{ matrix.language }}" == "nodejs" ] && [ ! -f src/nodejs/tests/test_api.js ]; then echo "Node.js test file missing!"; exit 1; fi
          # Simulate test failure for demonstration occasionally
          # if (( RANDOM % 2 == 0 )); then echo "Mock test failure!"; exit 1; fi

      - name: "[MOCK] Determine Test Outcome"
        id: determine_outcome
        # This step would aggregate results if matrix strategy was used for individual test files
        # For now, it assumes success if the previous step didn't exit with error
        run: |
          echo "Tests for ${{ matrix.language }} completed."
          echo "outcome=success" >> $GITHUB_OUTPUT # Default to success for mock
          # In reality, parse test reports to determine success/failure.

  refine_or_review:
    needs: [generate_tests, run_tests] # Depends on generate_tests for branch name
    runs-on: ubuntu-latest
    # Only run if tests failed OR if all tests passed (for review)
    # This logic is complex for matrix outcomes; simplified here.
    # A more robust solution might involve a separate job to aggregate matrix 'run_tests' outcomes.
    # For this mock, we'll assume 'run_tests' provides an overall outcome, or just run if any test job failed.
    # if: needs.run_tests.outputs.test_outcome == 'failure' || always() # 'always()' ensures it runs for review if tests pass
    if: always() # Simplified: always run this job to decide next steps.
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          ref: ${{ needs.run_tests.outputs.pr_branch }} # Use the branch from run_tests

      - name: Download All Artifacts (if needed for context)
        uses: actions/download-artifact@v4
        with:
          name: ai-generated-tests-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/tests
        continue-on-error: true
      - uses: actions/download-artifact@v4
        with:
          name: ai-generated-src-${{ needs.generate_tests.outputs.test_plan_id }}
          path: .ai_artifacts/src
        continue-on-error: true

      - name: "[MOCK] Decide Next Step based on Test Outcomes"
        id: decision
        run: |
          # In a real scenario, you'd check the actual outcomes of all matrix jobs in 'run_tests'.
          # This requires more complex scripting to fetch job conclusions via GitHub API or by passing structured outputs.
          # For this MOCK, we'll simulate a failure sometimes to show the refine path.
          # This is a placeholder for more robust outcome aggregation.
          # Assume 'needs.run_tests.outputs.test_outcome' gives an overall status.
          # However, 'needs.run_tests.outputs' isn't directly accessible due to matrix.
          # Let's simulate:
          if (( RANDOM % 3 == 0 )); then # Roughly 33% chance of simulated failure
            echo "Simulated test failure detected from 'run_tests' job."
            echo "action=refine" >> $GITHUB_OUTPUT
          else
            echo "Simulated test success from 'run_tests' job."
            echo "action=review" >> $GITHUB_OUTPUT
          fi

      - name: "[MOCK] Call Artisan (LLM) to Refine Code (if tests failed)"
        if: steps.decision.outputs.action == 'refine'
        run: |
          echo "Simulating Artisan LLM: Refining code based on test failures..."
          # Real: call_llm.sh artisan-codegemma-7b --prompt "Refine code in [file] due to test failure: [error]" --code_file ... --test_output ...
          echo "Refined code would be generated and committed here."
          # This might loop back to 'run_tests' or require human intervention.
          # For now, we'll just log it.
          echo "Code refinement requested. In a full loop, would re-run tests or create PR for review."

      - name: "[MOCK] Call Guardian (LLM) for Code Review (if tests passed)"
        if: steps.decision.outputs.action == 'review'
        run: |
          echo "Simulating Guardian LLM: Performing code review and security check..."
          # Real: call_llm.sh guardian-codellama-7b --prompt "Review code for quality, security, and best practices" --code_files "src/**"
          mkdir -p .ai_artifacts/guardian
          echo "Guardian Review: Looks good, but consider edge cases for login. No major security issues found." > .ai_artifacts/guardian/review_comments.txt
          echo "Review comments generated: .ai_artifacts/guardian/review_comments.txt"

      - name: "[MOCK] Commit files and Create/Update Pull Request"
        # This step would normally commit any new/modified files from LLMs (tests, src, review comments)
        # and then create or update a Pull Request.
        run: |
          echo "Branch for PR: ${{ needs.run_tests.outputs.pr_branch }}"
          # Mock creating/updating files
          git config --global user.name "AI TDD Bot"
          git config --global user.email "ai-tdd-bot@example.com"
          
          # Simulate adding review comments to a file in the PR
          if [ -f ".ai_artifacts/guardian/review_comments.txt" ]; then
            mkdir -p CODE_REVIEW
            cp .ai_artifacts/guardian/review_comments.txt CODE_REVIEW/guardian_review.md
            git add CODE_REVIEW/guardian_review.md || echo "No review file to add"
          fi

          # Simulate adding generated code/tests (in a real scenario, this happens in earlier jobs)
          # This is simplified; actual file paths from artifacts would be used.
          if [ -d "src" ]; then # If 'src' was populated by 'run_tests' mock copy
             git add src/
          fi

          git commit -m "AI TDD Cycle: ${{ github.event.inputs.feature_description }} (run ${{ github.run_id }})" || echo "No changes to commit."
          git push origin ${{ needs.run_tests.outputs.pr_branch }} --force || echo "Failed to push changes or no changes to push."
          
          echo "If this were a real run for workflow_dispatch, a PR would be created/updated now."
          echo "Example: gh pr create --title 'AI TDD: ${{ github.event.inputs.feature_description }}' --body 'Automated PR by AI TDD Engine. Review comments in CODE_REVIEW/guardian_review.md' --base main --head ${{ needs.run_tests.outputs.pr_branch }}"
          # For PR triggered runs, it would add comments or push new commits.
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }} # GITHUB_TOKEN has permissions to create PRs in the same repo
          # For cross-repo PRs or more complex scenarios, a PAT might be needed.
```

## 3. Explanations for Key Aspects

*   **Secret Management:**
    *   `LITELLM_API_ENDPOINT` and `LITELLM_API_KEY` (if LiteLLM is secured) are stored as GitHub encrypted secrets.
    *   `GITHUB_TOKEN` is automatically available and used for actions like checking out code and potentially creating PRs (within the same repository).

*   **Prompt Engineering:**
    *   **Pacer:** Needs clear feature descriptions and target languages to generate a comprehensive test plan. Prompts should ask for structured output (e.g., JSON list of test cases with language and description).
    *   **Maven:** Needs the test plan (or specific parts) to fetch relevant context. Prompts should guide it to look for existing code, documentation, or patterns.
    *   **Builder (Tests):** Needs a specific test case description from Pacer's plan and context from Maven. Prompts should specify the testing framework (if known) and desired level of detail.
    *   **Builder (Code):** Needs the generated test file(s) and context. Prompts should instruct it to write code that makes the tests pass, adhering to project conventions.
    *   **Artisan:** Needs the failing test output and the relevant code. Prompts should focus on fixing the error or optimizing the existing code.
    *   **Guardian:** Needs the generated code and tests. Prompts should ask for review based on coding standards, security vulnerabilities, and best practices. Output should be structured for easy parsing (e.g., list of comments with severity and file/line numbers).
    *   **General:** Use few-shot prompting (providing examples) if initial results are poor. Clearly define roles and expected output formats for each LLM.

*   **LLM Output Parsing:**
    *   LLMs should be prompted to return structured data (JSON, YAML).
    *   The scripts (e.g., `call_llm.sh` placeholder) would need to parse this output. For example, if Pacer returns a JSON array of tests, the workflow needs to iterate over it.
    *   Robust error handling is needed for malformed LLM responses.

*   **State Management Between AI Calls:**
    *   **GitHub Artifacts:** Used in the mock to pass generated files (test plans, context, test code, source code, review comments) between jobs.
    *   **Committing to Branch:** In a real implementation, each job that generates or modifies files (tests, source code) would commit those changes to the PR branch. This makes the state persistent and visible. The subsequent jobs would pull the latest changes from the branch.
    *   **Files with IDs:** Using unique IDs (e.g., `plan_id`) can help correlate artifacts and logs across different stages.

*   **Polyglot Test Execution:**
    *   The `run_tests` job uses a `matrix` strategy based on the `target_languages` input.
    *   Each matrix instance would:
        1.  Set up the specific environment for that language (e.g., `actions/setup-python`, `actions/setup-node`, `actions/setup-java`).
        2.  Install dependencies (`pip install`, `npm install`, `mvn dependency:resolve`).
        3.  Run the tests using the language-specific test runner (`pytest`, `npm test`, `mvn test`).
        4.  Parse test results to determine success/failure.

*   **Error Handling & Iteration:**
    *   `continue-on-error: true` can be used on steps if partial failure is acceptable or handled by later steps.
    *   Job dependencies (`needs: [...]`) control the execution flow.
    *   The `refine_or_review` job demonstrates a conditional step. If tests fail, Artisan is called.
    *   A true iterative loop (e.g., Code -> Test -> Refine -> Test) would be more complex, potentially involving:
        *   Re-running the `run_tests` job after Artisan makes changes.
        *   Using workflow dispatch or specific PR comments/labels to trigger refinement loops.
        *   Setting a maximum number of retries to prevent infinite loops.

*   **Importance of Human Review:**
    *   **Crucial Final Step:** AI-generated code and tests **must** be reviewed by human developers before merging.
    *   The pipeline aims to automate the TDD boilerplate and provide a strong starting point, not replace human oversight.
    *   Guardian's output can feed into the PR description or as comments to aid human reviewers.
    *   The final step should be a clear PR ready for human assessment.

*   **Managing GitHub Actions Costs:**
    *   **Self-Hosted Runners:** For GPU-intensive LLM calls (if running models locally, not via API) or long-running jobs, self-hosted runners can be more cost-effective than GitHub-hosted runners. However, this pipeline calls external LLM APIs, so compute is on Azure.
    *   **Concurrency Limits:** Be mindful of concurrency limits on GitHub-hosted runners for your account tier.
    *   **Optimize Workflows:**
        *   Ensure jobs only run when necessary.
        *   Use caching for dependencies (`actions/cache`).
        *   Mocking (as done here) is cheap; actual LLM API calls will incur costs on Azure based on token usage per model.
        *   Implement scale-to-zero for your Azure ML Endpoints and LiteLLM Container App to save costs when idle.
    *   **Artifact Storage:** Artifacts also consume storage, which has associated costs. Regularly clean up old artifacts if space becomes an issue.

*   **Mocked Interactions Clarification:**
    *   The YAML workflow uses `echo` statements and basic file creation (e.g., `echo "def test_user_login(): assert True" > ...`) to simulate what real scripts interacting with LLMs would do.
    *   **Actual Implementation:** Each "[MOCK] Call..." step would be replaced by a script execution (e.g., `bash ./scripts/call_llm.sh <model_name> --prompt "..." --output_file ...`) that handles:
        1.  Constructing the correct JSON payload for the LiteLLM endpoint.
        2.  Making an HTTP POST request to the `LITELLM_ENDPOINT_URL`.
        3.  Handling the response (e.g., extracting the LLM's message).
        4.  Parsing the LLM's message (e.g., if it's JSON or contains code).
        5.  Writing the relevant content to files in the workspace (e.g., test files, source code files).
        6.  Committing these new/modified files to the current branch so subsequent jobs can use them.

This TDD engine provides a powerful framework for AI-assisted development. The key is iterative refinement of prompts, robust parsing, and always keeping a human in the loop for final validation.
```
