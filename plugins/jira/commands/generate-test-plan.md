---
description: Generate test steps for a JIRA issue and optionally create e2e test code
argument-hint: [JIRA issue key] [GitHub PR URLs] [--e2e <test-file-or-location>]
---

## Name
jira:generate-test-plan

## Synopsis
/jira:generate-test-plan [JIRA issue key] [GitHub PR URLs] [--e2e <test-file-or-location>]

## Description
The 'jira:generate-test-plan' command takes a JIRA issue key and optionally a list of PR URLs. It fetches the JIRA issue details, retrieves all related PRs (or uses the provided PR list), analyzes the changes, and generates a comprehensive manual testing guide.

**NEW**: When `--e2e` is provided, the command will also:
- Create a new git branch in the specified repository
- Generate e2e test code based on the JIRA requirements and PR changes
- Follow the repository's existing e2e test patterns and conventions
- Use Ginkgo/Gomega test framework with proper structure
- Intelligently detect or prompt for the correct test directory location

**💡 TIP**: The `--e2e` parameter accepts a test file or location:
- **Test file** (most specific): `/path/to/repo/test/e2e/machines_test.go`
  - Appends test code to this file
  - No prompts, no new file creation
- **Test directory**: `/path/to/repo/test/e2e/`
  - Creates new test file with auto-generated name from JIRA
- **Repo path**: `/path/to/repo` or `../machine-api-operator`
  - Detects test directory, creates new file
- **Repo name**: `machine-api-operator`
  - Searches for repo, may prompt for test directory
  - Creates new test file

**Rule**: If ends with `.go` → append to that file. Otherwise → create new file at that location.

**JIRA Issue Test Guide Generator**

## Implementation

**⚡ Best Practice for `--e2e` parameter**:
- **To append to existing test file**: Specify full path to `.go` file
- **To create new test file in specific location**: Specify test directory path
- **To create new test file (auto-detect location)**: Specify repo path or repo name
- **Rule of thumb**: More specific = more control over where code goes

The command uses:
- curl to fetch JIRA data via REST API: https://issues.redhat.com/rest/api/2/issue/{$1}
- WebFetch to extract PR links from JIRA issue if no PRs provided
- `gh pr view` to fetch PR details for each PR
- Analyzes changes across all PRs to understand implementation
- Generates comprehensive manual test scenarios

## Process Flow:

1. **JIRA Analysis**: Fetch and parse JIRA issue details:
   - Use curl to fetch JIRA issue data: `curl -s "https://issues.redhat.com/rest/api/2/issue/{$1}"`
   - Parse JSON response to extract:
     - Issue summary and description
     - Context and acceptance criteria
     - Steps to reproduce (for bugs)
     - Expected vs actual behavior
   - Extract issue type (Story, Bug, Task, etc.)

2. **PR Discovery**: Get list of PRs to analyze:
   - **If no PRs provided in arguments** ($2, $3, etc. are empty):
     - Use WebFetch on https://issues.redhat.com/browse/{$1}
     - Extract all GitHub PR links from:
       - "Issue Links" section
       - "Development" section
       - PR links in comments
   - **If PRs provided in arguments**:
     - Use only the PRs provided in $2, $3, $4, etc.
     - Ignore any other PRs linked to the JIRA

3. **PR Analysis**: For each PR, fetch and analyze:
   - Use `gh pr view {PR_NUMBER} --repo <your repo> --json title,body,commits,files,labels`
   - Extract:
     - PR title and description
     - Changed files and their diffs
     - Commit messages
     - PR status (merged, open, closed)
   - Read changed files to understand implementation details
   - Use Grep and Glob tools to:
     - Find related test files
     - Locate configuration or documentation
     - Identify dependencies

4. **Change Analysis**: Understand what was changed across all PRs:
   - Identify the overall objective (bug fix, feature, refactor)
   - Determine affected components (API, CLI, operator, control-plane, etc.)
   - Find platform-specific changes (AWS, Azure, KubeVirt, etc.)
   - Map which PR addresses which aspect of the JIRA
   - Identify any dependencies between PRs

5. **Test Scenario Generation**: Create comprehensive test plan:
   - Map JIRA acceptance criteria to test scenarios
   - For bugs: Use reproduction steps as test cases
   - Generate test scenarios covering:
     - Happy path scenarios (based on acceptance criteria)
     - Edge cases and error handling
     - Platform-specific variations if applicable
     - Regression scenarios
   - For multiple PRs:
     - Create integrated test scenarios
     - Verify PRs work correctly together
     - Test each PR's contribution to the overall solution

6. **Test Guide Creation**: Generate detailed manual testing document:
   - **Filename**: Always use JIRA key format: `test-{JIRA_KEY}.md`
     - Convert JIRA key to lowercase
     - Examples: `test-cntrlplane-205.md`, `test-ocpbugs-12345.md`
   - **Structure**:
     - **JIRA Summary**: Include JIRA key, title, description, acceptance criteria
     - **PR Summary**: List all PRs with titles and how they relate to the JIRA
     - **Prerequisites**:
       - Required infrastructure and tools
       - Environment setup requirements
       - Access requirements
     - **Test Scenarios**:
       - Map each test to JIRA acceptance criteria
       - Numbered test cases with clear steps
       - Expected results with verification commands
       - Platform-specific test variations
     - **Regression Testing**:
       - Related features to verify
       - Areas that might be affected
     - **Success Criteria**:
       - Checklist mapping to JIRA acceptance criteria
     - **Troubleshooting**:
       - Common issues and debug steps
     - **Notes**:
       - Known limitations
       - Links to JIRA and all PRs
       - Critical test cases highlighted

7. **Exclusions**: Apply smart filtering:
   - **Skip PRs that don't require testing**:
     - PRs that only add documentation (.md files only)
     - PRs that only add CI/tooling (.github/, .claude/ directories)
     - PRs marked with labels like "skip-testing" or "docs-only"
   - **Note skipped PRs** in the test guide with reasoning
   - Focus test scenarios on PRs with actual code changes

8. **E2E Test Code Generation** (only if `--e2e` is provided):
   - **Determine target file**:
     - **If `--e2e` value ends with `.go`** (e.g., `test/e2e/machines_test.go`):
       - Use that file directly (append mode)
       - If file doesn't exist: ask user to create it
     - **Otherwise** (directory or repo name):
       - Locate repository (search if needed, may prompt)
       - Detect test directory (may prompt if multiple found)
       - Generate new file name from JIRA summary (e.g., `azure_webhook_defaulting_test.go`)
       - If generated name already exists: ask to append or use different name

   - **Analyze test patterns**:
     - **If appending to existing file**: Read file to understand structure and insertion point
     - **If creating new file**: Find and read existing test files to learn repo conventions

   - **Create git branch**: `test-{jira-key-lowercase}` in the repository

   - **Generate and write e2e test code**:
     - **If appending**: Use Edit tool to insert new test code into existing file
     - **If creating**: Use Write tool to create complete test file
     - **Test structure**: Analyze existing test files in the repository and follow their patterns
     - **Test content**: Based on JIRA acceptance criteria and PR changes
     - **Format**: Run `gofmt` and verify with `go build`

9. **Output**: Display the testing guide and e2e code:
   - Show the file path where the guide was saved
   - If e2e code was generated:
     - Show the repository path
     - Show the branch name
     - Show the test file path
     - Provide commands to run the new tests
   - Provide a summary of:
     - JIRA issue being tested
     - Number of PRs included
     - Number of test scenarios generated (manual)
     - Number of e2e test cases generated (if applicable)
     - Critical test cases to focus on
   - Highlight any PRs that were skipped and why
   - Ask if the user would like any modifications to the test guide or e2e code

## Examples:

1. **Generate test steps for JIRA with auto-discovered PRs**:
   ```
   /jira:generate-test-plan CNTRLPLANE-205
   ```

2. **Generate test steps for JIRA with specific PRs only**:
   ```
   /jira:generate-test-plan CNTRLPLANE-205 https://github.com/openshift/hypershift/pull/6888
   ```

3. **Generate test steps for multiple specific PRs**:
   ```
   /jira:generate-test-plan CNTRLPLANE-205 https://github.com/openshift/hypershift/pull/6888 https://github.com/openshift/hypershift/pull/6889
   ```

4. **Append to existing test file** ⚡ (most specific):
   ```
   /jira:generate-test-plan OCPBUGS-66244 --e2e /Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/machines_test.go
   ```
   **Benefits**:
   - ✅ Zero ambiguity - uses exactly this file
   - ✅ Fastest - no directory detection needed
   - ✅ Appends test code to existing file
   - ✅ No prompts (if file exists)

   **Example**: Appends new test following the patterns in machines_test.go

5. **Create new file in specific test directory**:
   ```
   /jira:generate-test-plan OCPBUGS-66244 --e2e /Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/
   ```
   - Creates new file: `azure_webhook_defaulting_test.go` (name from JIRA)
   - In directory: `/Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/`

6. **Create new file by repo path** (auto-detect test dir):
   ```
   /jira:generate-test-plan OCPBUGS-66244 --e2e /Users/zhsun/go/src/github.com/openshift/machine-api-operator
   ```
   - Detects test directory (may prompt if multiple found)
   - Creates new test file

7. **Create new file by repo name** (searches and auto-detects):
   ```
   /jira:generate-test-plan OCPBUGS-66244 --e2e machine-api-operator
   ```
   - Searches for repo (may prompt if not found)
   - Detects test directory (may prompt)
   - Creates new test file

8. **Relative path to test file**:
   ```
   /jira:generate-test-plan OCPBUGS-66244 --e2e ../machine-api-operator/test/e2e/webhook_test.go
   ```
   Appends to existing file using relative path

## Arguments:

- **$1**: JIRA issue key (required) - e.g., CNTRLPLANE-205, OCPBUGS-12345
- **$2, $3, ..., $N**: Optional GitHub PR URLs
  - If provided: Only these PRs will be analyzed
  - If omitted: All PRs linked to the JIRA will be discovered and analyzed
- **--e2e <test-file-or-location>**: Optional flag to generate e2e test code
  - Accepts multiple formats (ordered by specificity):
    1. **Test file path** ⚡ (most specific, appends to file):
       - Absolute: `/Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/machines_test.go`
       - Relative: `../machine-api-operator/test/e2e/webhook_test.go`
       - **If path ends with `.go`**: Appends test code to this file
       - **If file doesn't exist**: Prompts to create it
    2. **Test directory path** (creates new file):
       - Absolute: `/Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/`
       - Creates new file with name from JIRA summary
    3. **Repository path** (auto-detects test dir):
       - Absolute: `/Users/zhsun/go/src/github.com/openshift/machine-api-operator`
       - Relative: `../machine-api-operator`, `.` (current dir)
       - Detects test directory (may prompt if multiple)
    4. **Repo name** 🔍 (searches everything):
       - Format: `machine-api-operator` or `openshift/cluster-api-operator`
       - Searches common locations, may prompt for directory
  - When provided:
    - Creates a git branch named `test-{jira-key}`
    - Generates e2e test code following repository conventions
    - **Appends** to existing file OR **creates** new file based on path specificity
  - Can be combined with PR URL arguments
  - **💡 Best practice**:
    - Use `.go` file path to append to existing test file (zero prompts)
    - Use directory path to create new file with auto-generated name
    - Use repo name for quick testing (may need interaction)

## Smart Features:

1. **Automatic PR Discovery**:
   - Scans JIRA issue for all related PRs
   - Identifies PRs in "Issue Links", "Development" section, and comments

2. **Selective PR Testing**:
   - Allows manual override to test specific PRs only
   - Useful when JIRA has many PRs but only some need testing

3. **Context-Aware Test Generation**:
   - Bug fixes: Focus on reproduction steps and verification
   - Features: Focus on acceptance criteria and user workflows
   - Refactors: Focus on regression and functional equivalence

4. **Multi-PR Integration**:
   - Understands how multiple PRs work together
   - Creates integration test scenarios
   - Identifies dependencies and testing order

5. **Build/Deploy Section Exclusion**:
   - Does NOT include build or deployment steps
   - Assumes environment is already set up
   - Focuses purely on testing procedures

6. **Cleanup Section Exclusion**:
   - Does NOT include cleanup steps
   - Focuses on test execution and verification

7. **E2E Test Pattern Recognition** (when --e2e is used):
   - Analyzes existing test files to understand repository conventions
   - Adapts to different test frameworks (Ginkgo/Gomega, standard Go testing)
   - Follows existing import patterns and helper functions
   - Maintains consistency with repository's test style

8. **Intelligent and Interactive Test Directory Selection**:
   - **Flexible repository location**: Accepts absolute paths, relative paths, or repo names
   - **Smart directory detection**: Scans for existing e2e test directories
   - **User confirmation**: When multiple options exist, prompts user to choose
   - **Automatic creation**: Creates directories when needed with user approval
   - **Clear communication**: Shows user exactly where tests will be placed
   - Generates meaningful test file names based on JIRA summary

9. **Interactive Workflow**:
   - Asks user for input when there are ambiguous choices
   - Provides clear default options
   - Confirms actions before creating directories or branches
   - Shows full paths to avoid confusion

10. **Repository Pattern Following**:
   - Analyzes existing test files in the repository
   - Follows the same test structure, naming conventions, and patterns
   - Adapts to repository-specific best practices
   - Follows CLAUDE.md preferences when applicable (Eventually/Expect/Should style)

## Example Workflow:

### Manual Test Plan Only:
```bash
# Auto-discover all PRs from JIRA
/jira:generate-test-plan CNTRLPLANE-205

# Test only specific PRs
/jira:generate-test-plan CNTRLPLANE-205 https://github.com/openshift/hypershift/pull/6888

# Test multiple specific PRs
/jira:generate-test-plan OCPBUGS-12345 https://github.com/openshift/hypershift/pull/1234 https://github.com/openshift/hypershift/pull/1235
```

### With E2E Test Code Generation:
```bash
# Generate manual test plan + e2e code
/jira:generate-test-plan OCPBUGS-66244 --e2e machine-api-operator

# Interactive prompts example:
# 🔍 Searching for repository 'machine-api-operator'...
# ✅ Found: /Users/zhsun/go/src/github.com/openshift/machine-api-operator
#
# 🔍 Detecting test directories...
# ✅ Found existing test directory: test/e2e/
# Using: /Users/zhsun/go/src/github.com/openshift/machine-api-operator/test/e2e/
#
# 📝 Creating branch: test-ocpbugs-66244
# 📝 Generating e2e test file: azure_webhook_defaulting_test.go
#
# ✅ Test plan generated: test-ocpbugs-66244.md
# ✅ E2E test code created:
#    Repository: /Users/zhsun/go/src/github.com/openshift/machine-api-operator
#    Branch: test-ocpbugs-66244
#    Test file: test/e2e/azure_webhook_defaulting_test.go
#
# To run the e2e tests:
#   cd /Users/zhsun/go/src/github.com/openshift/machine-api-operator
#   make test-e2e
```

The command will provide a comprehensive manual testing guide that QE or developers can use to thoroughly test the JIRA issue implementation. When `--e2e` is specified, it also generates ready-to-run e2e test code following the repository's conventions.

**Interactive Mode**: The command will ask for user confirmation when:
- Repository location is ambiguous
- Multiple test directories exist
- New directories need to be created

---

### 📊 Comparison: `--e2e` Parameter Specificity

| Format | Example | Behavior | Prompts? | Speed | Use Case |
|--------|---------|----------|----------|-------|----------|
| **Test file (.go)** ⚡ | `/path/to/repo/test/e2e/machines_test.go` | Appends to existing file | None* | Fastest | Add tests to existing file |
| **Test directory** 📁 | `/path/to/repo/test/e2e/` | Creates new file | None | Fast | New test file, specific location |
| **Repo path** 📂 | `/path/to/repo` | Finds test dir, creates file | Maybe** | Fast | New test file, auto-detect dir |
| **Relative path** 🔄 | `../machine-api-operator` | Same as repo path | Maybe** | Fast | Local development |
| **Repo name** 🔍 | `machine-api-operator` | Searches, detects, creates | Likely*** | Medium | Quick testing |

\* Only prompts if file doesn't exist
\*\* Prompts if multiple test directories exist
\*\*\* May prompt for repo location, test directory, and file name

**Key Insight**:
- **Most specific** (`.go` file) = appends to existing file, no prompts
- **Medium** (directory) = creates new file, minimal prompts
- **Least specific** (repo name) = searches and auto-detects, may need input

**Recommendation**:
- To add to existing test file: Use full `.go` file path
- To create new test file: Use test directory path or repo path
