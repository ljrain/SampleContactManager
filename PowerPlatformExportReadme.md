Power Platform Solution Export (Managed + Unmanaged)
This repository uses a GitHub Actions workflow to export a Dataverse solution from a Power Platform environment, store it in source control, and keep both unmanaged and managed representations in a consistent, reviewable structure.
The workflow is designed for repeatable ALM, PR-based reviews, and safe automation (including protection against corrupted ZIP artifacts).

What This Workflow Does
At a high level, the workflow:

Exports the solution twice from Dataverse:

Unmanaged (for development / diff / source control)
Managed (for deployment artifacts)


Validates that the exported ZIPs are real, intact ZIP files
Unpacks both ZIPs into readable folders
Commits the unpacked source into a new Git branch
Pushes the branch so it can be reviewed and merged via PR

The workflow is run manually using workflow_dispatch.

When to Use This Workflow
Use this workflow when you want to:

Capture the current state of a Dataverse solution in Git
Review solution changes via pull request
Keep managed and unmanaged versions aligned
Avoid committing opaque .zip files to source control
Standardize ALM without Power Platform Pipelines


Prerequisites
1. GitHub Environment
The workflow expects a GitHub Environment named:
MAIN

This environment must contain the following secrets:





















Secret NameDescriptionCLIENT_IDEntra App (Service Principal) Client IDCLIENT_SECRETClient secret for the App RegistrationTENANT_IDEntra Tenant ID
The service principal must be configured as an Application User in Dataverse with sufficient permissions to export solutions.

2. Solution Requirements

The solution must be an unmanaged solution in the source environment
You must use the solution unique name, not the display name


How to Run the Workflow

Go to GitHub → Actions
Select Export Power Platform Solution (Managed + Unmanaged)
Click Run workflow
Provide the inputs:

Workflow Inputs

































InputDescriptionenvironment_urlDataverse environment URL (e.g. https://org.crm.dynamics.com)solution_nameUnique name of the Dataverse solutionsource_branchBranch to start from (typically main)target_branchOptional; leave blank to auto‑generatecommit_messageCommit message for the exportuser_nameGit author name (default is GitHub Actions bot)

Branching Behavior
If you leave target_branch blank, the workflow automatically creates a branch using:
<solution-name>-YYYYMMDD-HHMM

Example:
coreforms-20260203-1512

This makes it easy to identify what was exported and when.

What Gets Committed
Only unpacked solution source is committed — ZIP files are not committed.
The workflow explicitly:

Exports ZIPs
Validates them
Unpacks them
Copies unpacked source into /solutions
Deletes ZIP artifacts before committing


Directory Structure (After Merge to main)
After the PR is merged, your repository will contain:
solutions/
└── <solution-name>/
    ├── unmanaged/
    │   ├── Entities/
    │   ├── Workflows/
    │   ├── OptionSets/
    │   ├── WebResources/
    │   ├── CustomControls/
    │   └── solution.xml
    │
    └── managed/
        ├── Entities/
        ├── Workflows/
        ├── OptionSets/
        ├── WebResources/
        ├── CustomControls/
        └── solution.xml

Key Points About This Structure


unmanaged/

Primary source-of-truth for development
Used for Git diffs, reviews, and merges
What you expect to change frequently



managed/

Mirrors the managed export
Useful for validation, auditing, and promotion checks
Typically changes only via releases



Both folders are generated from the same Dataverse solution version



Why ZIP Validation Exists
Before unpacking, the workflow runs:
Shellunzip -t <solution>.zip``Show more lines
This prevents failures like:
Central Directory corrupt

Which usually means:

The export returned HTML or JSON (401 / 403 / 404)
The ZIP was truncated
A downstream step would have failed with an unclear PAC error

Instead, the workflow fails early with a clear signal.

What This Workflow Does Not Do

❌ It does not import solutions
❌ It does not deploy to downstream environments
❌ It does not use Power Platform Pipelines
❌ It does not commit ZIP files

This workflow is export + source control only by design.

Typical ALM Flow Using This Pattern

Developer modifies solution in Dataverse
Run this workflow
Open PR from auto-generated branch → main
Review solution diffs
Merge to main
Downstream pipelines (or manual steps) deploy from source if needed


Common Extensions
This workflow is often extended to:

Automatically open a Pull Request
Trigger validation (Solution Checker)
Feed managed ZIPs into release pipelines
Support multiple clouds (Commercial / GCC / GCC High / DoD)
Enforce branch naming or review policies
