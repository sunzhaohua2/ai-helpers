---
description: Verify a bug fix by analyzing the bug report, root cause, and fix implementation
argument-hint: [JIRA bug key]
---

## Name
jira:verify-bug

## Synopsis
```
/jira:verify-bug [JIRA bug key]
```

## Description
The `jira:verify-bug` command analyzes a bug report, identifies the root cause, reviews the fix implementation, and generates verification steps to confirm the bug has been properly resolved.

This command is essential for:
- Verifying that a bug has been properly fixed
- Understanding the root cause of issues
- Reviewing fix quality and completeness
- Creating reproducible verification steps
- Ensuring proper test coverage

## Key Features

- **Root Cause Analysis** – Deep dive into the bug's underlying cause
- **Fix Review** – Analyzes PR changes and validates fix approach
- **Verification Steps** – Generates step-by-step verification instructions
- **Test Coverage Check** – Ensures adequate tests were added
- **Final Verdict** – VERIFIED / FAILED / PARTIALLY VERIFIED status

## Implementation

### High-Level Workflow

1. **Fetch Bug Details**
   - Use MCP JIRA tools to fetch bug issue by key
   - Extract:
     - Bug summary and description
     - Steps to reproduce
     - Expected vs actual behavior
     - Severity and priority
     - Affected versions

2. **Root Cause Investigation**
   - Search codebase for related code using Grep/Glob
   - Read relevant source files
   - Identify the root cause of the bug
   - Document code location (file:line)

3. **Find and Analyze Fix PR(s)**
   - Extract PR links from JIRA issue (Issue Links, Development section)
   - Use `gh pr view` to fetch PR details
   - Review code changes:
     - Does the fix address the root cause?
     - Are there adequate tests?
     - Are there edge cases missed?
   - Check PR status (merged, open, closed)

4. **Generate Verification Steps**
   - Create step-by-step verification instructions based on:
     - Original reproduction steps
     - Fix implementation
     - Test cases added in PR
   - Include expected results and verification commands

5. **Create Verification Report**
   - **Filename**: `verify-{JIRA_KEY}.md` (lowercase)
   - Include:
     - Bug summary
     - Root cause analysis with code references
     - Fix analysis with PR links
     - Verification steps
     - Final verdict (VERIFIED/FAILED/PARTIALLY VERIFIED)

## Usage Examples

### Basic Bug Verification

```bash
/jira:verify-bug OCPBUGS-66244
```

### Example Output Structure

```markdown
# Bug Verification Report: OCPBUGS-66244

## Bug Summary
- **JIRA**: [OCPBUGS-66244](https://issues.redhat.com/browse/OCPBUGS-66244)
- **Title**: Azure webhook should default to marketplace image
- **Severity**: High
- **Status**: Closed

## Root Cause
**Component**: machine-api-operator/pkg/controller/azure

**Root Cause**: The webhook was not defaulting the Image field when empty,
causing machines to fail creation in 4.21+ clusters where installer no longer
creates gallery images.

**Code Location**: `pkg/controller/azure/webhook.go:123`

## Fix Analysis
**PR**: https://github.com/openshift/machine-api-operator/pull/1441

**Fix Approach**: Added webhook defaulting logic to set marketplace image
when Image field is empty.

**Test Coverage**:
- ✅ Added unit tests for webhook defaulting
- ✅ Added e2e test for minimal providerSpec

## Verification Steps

### Prerequisites
- 4.21+ Azure cluster
- `oc` CLI configured

### Test 1: Verify Minimal ProviderSpec
1. Create a Machine with minimal providerSpec (no Image field)
2. Verify webhook defaults Image to marketplace format
3. Verify Machine reaches Running state

**Expected Result**: Machine should have Image.Publisher/Offer/SKU/Version set

**Verification Command**:
    oc get machine <name> -n openshift-machine-api -o yaml | grep -A 5 image

### Test 2: Verify Gallery Image Still Works
1. Create a Machine with Image.ResourceID set
2. Verify webhook preserves ResourceID
3. Verify Machine reaches Running state

**Expected Result**: Machine should preserve ResourceID and reach Running state

**Verification Command**:
    oc get machine <name> -n openshift-machine-api -o yaml | grep -A 5 resourceID

## Verification Result
**Status**: ✅ VERIFIED

**Notes**:
- Fix properly addresses the root cause
- Adequate test coverage added
- Both marketplace and gallery images work correctly

**Recommendations**: None - fix is complete
```

## Arguments

- **$1 – JIRA bug key** *(required)*
  The JIRA bug issue key to verify (e.g., OCPBUGS-66244)

## Smart Features

1. **Automatic PR Discovery**
   - Scans JIRA for all related PRs
   - Analyzes PR status and merge state

2. **Root Cause Deep Dive**
   - Uses Grep/Glob to find related code
   - Reads source files to understand issue

3. **Fix Quality Assessment**
   - Checks if fix addresses root cause
   - Validates test coverage
   - Identifies potential edge cases

4. **Reproducible Verification**
   - Generates clear step-by-step instructions
   - Includes verification commands
   - Maps to original reproduction steps

## Exclusions

- Does NOT create or run tests automatically
- Does NOT update JIRA status (manual action required)
- Does NOT deploy fixes (assumes fix is already merged/deployed)

## Output

The command generates:
1. **Console summary** - Quick overview of verification status
2. **Markdown file** - Detailed verification report (`verify-{jira-key}.md`)
3. **Next steps** - Recommendations for follow-up actions

## Example Workflow

```bash
# Verify a bug fix
/jira:verify-bug OCPBUGS-66244

# Output:
# ✅ Bug OCPBUGS-66244 verified successfully
# 📄 Report saved to: verify-ocpbugs-66244.md
#
# Summary:
# - Root cause: Webhook missing Image defaulting logic
# - Fix PR: #1441 (merged)
# - Test coverage: Good (unit + e2e tests added)
# - Verification: VERIFIED
```

## See Also

- `/jira:generate-test-plan` - Generate manual testing guide for JIRA issues
- `/jira:solve` - Analyze and create PR to solve a JIRA issue
