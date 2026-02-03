Power Platform Solution Export (Fed / DoD ALM Pattern)
This repository implements a Federal / DoD–compliant Application Lifecycle Management (ALM) pattern for Microsoft Power Platform (Dataverse) solutions using GitHub Actions.
It is designed for GCC, GCC High, and DoD (Flank Speed) environments where auditability, least‑privilege access, deterministic artifacts, and human review are mandatory.

✅ What This Repository Does
This repo contains a GitHub Actions workflow that:

Exports a Dataverse solution from a Power Platform environment
Exports both:

Unmanaged (source-of-truth for development & review)
Managed (deployment-aligned reference)


Validates exported ZIPs to prevent silent corruption
Unpacks solutions into human‑readable XML source
Commits changes to a new Git branch
Requires a Pull Request for review before merge

🚫 This repository does not deploy solutions
All imports/deployments are intentionally out‑of‑scope and must be handled by separate, approved pipelines.

Why This Pattern Exists (Fed / DoD Context)
Federal and DoD environments require:

✅ Human‑reviewable change history
✅ Explicit change approval (PR‑based)
✅ Separation of authoring and deployment
✅ Deterministic, replayable artifacts
✅ No opaque binaries as system of record
✅ Least‑privilege service access
✅ Traceability for audits (CM / RM controls)

This pattern satisfies those requirements by treating Git as the system of record and Dataverse as the source of truth, while avoiding platform‑managed pipelines that obscure changes.

Security & Compliance Characteristics
✅ No Direct Deployment

Workflow never imports solutions into any environment
Eliminates accidental promotion risks

✅ Least Privilege

Uses a Service Principal (Application User) limited to solution export permissions
No environment admin privileges required

✅ Auditable Changes

Every change results in:

A Git commit
A branch
A Pull Request


Full history is preserved and reviewable

✅ Deterministic Artifacts

Managed and unmanaged exports are produced from the same source snapshot
Prevents version drift

✅ Explicit Failure for Corrupt Exports

ZIP validation (unzip -t) runs before unpack
Prevents misleading Central Directory corrupt PAC CLI failures
Common Fed issue when exports return HTML/JSON due to auth errors


Prerequisites
1. GitHub Environment
A GitHub Environment named MAIN must exist and contain the following secrets:





















Secret NameDescriptionCLIENT_IDEntra ID Application (Service Principal) Client IDCLIENT_SECRETClient secret for the App RegistrationTENANT_IDEntra ID Tenant ID
Access to this environment should be restricted and audited.

2. Dataverse Configuration

App Registration added to Dataverse as an Application User
Assigned minimum required roles to export solutions
Solution must be unmanaged in the source environment
Workflow uses the solution UNIQUE NAME (not display name)


Running the Workflow

Navigate to GitHub → Actions
Select Export Power Platform Solution (Managed + Unmanaged)
Click Run workflow
Provide inputs:

Workflow Inputs

































InputDescriptionenvironment_urlDataverse environment URL (e.g. https://org.crm.dynamics.com)solution_nameUnique solution name in Dataversesource_branchBranch to start from (typically main)target_branchLeave blank to auto‑generatecommit_messageCommit message for audit traceabilityuser_nameGit author name (default: GitHub Actions bot)

Branching Behavior
If target_branch is left blank, the workflow auto‑creates a branch:
<solution-name>-YYYYMMDD-HHMM

Example:
claims-intake-20260203-1512

This aligns exports to change windows, tickets, or release cycles.

Repository Directory Structure (After Merge)
Once the PR is approved and merged into main, the repo will contain:
solutions/
└── <solution-name>/
    ├── unmanaged/
    │   ├── Entities/
    │   ├── Workflows/
    │   ├── OptionSets/
    │   ├── WebResources/
    │   └── solution.xml
    └── managed/
        ├── Entities/
        ├── Workflows/
        ├── OptionSets/
        ├── WebResources/
        └── solution.xml

Key Notes

Unmanaged

Primary source-of-truth
Used for diffs, reviews, and merges


Managed

Mirrors managed solution state
Used for validation and deployment alignment


ZIP files are NOT committed

Treated as transient build artifacts only
