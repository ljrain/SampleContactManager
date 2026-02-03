# Export Power Platform Solution (Dual Export, Fixed Version Branch)

## Overview

This GitHub Actions workflow provides a **Federal and DoD-compliant** Application Lifecycle Management (ALM) pattern for exporting Microsoft Power Platform (Dataverse) solutions. It is specifically designed for **GCC, GCC High, and DoD (Flank Speed)** environments.

### Key Characteristics

- ✅ **No Direct Deployment** - Only exports solutions; never imports or deploys
- ✅ **Least Privilege** - Uses Entra ID Application User with minimal permissions
- ✅ **Auditable Change History** - All changes visible as Git diffs
- ✅ **Deterministic Artifacts** - Both managed and unmanaged exports from same snapshot
- ✅ **Explicit Failure Handling** - ZIP validation prevents silent failures
- ✅ **Fixed Version Branching** - Automatic branch naming using solution version

---

## What This Action Does

The **Export Power Platform Solution (Dual Export, Fixed Version Branch)** action automates the complete export and version control process for Power Platform solutions. It performs the following operations:

1. **Retrieves Solution Version** - Queries Dataverse Web API to get the current solution version
2. **Creates Versioned Branch** - Automatically creates a branch named `<solution-name>-<version>` (e.g., `Sample-1_0_0_1`)
3. **Exports Dual Solutions** - Exports both unmanaged and managed versions of the solution
4. **Validates ZIP Integrity** - Ensures exported files are valid ZIP archives
5. **Unpacks Solutions** - Converts ZIP files into readable XML format for source control
6. **Commits Changes** - Commits the unpacked solution files to the versioned branch
7. **Pushes Branch** - Pushes the branch to GitHub for Pull Request creation

**Important:** This workflow **does not** deploy solutions, import solutions, or modify any environments. It is purely an export and version control mechanism.

---

## How It Works

### Step-by-Step Process

#### 1. Manual Trigger
The workflow is triggered manually via `workflow_dispatch` from the GitHub Actions UI. You must provide required inputs such as:
- Environment URL
- Solution name
- Source branch
- Commit message
- Git username

#### 2. Environment Setup
```yaml
environment: MAIN
```
The workflow reads secrets from the GitHub Environment named **MAIN**, which must contain:
- `CLIENT_ID` - Entra ID Application Client ID
- `CLIENT_SECRET` - Application Client Secret
- `TENANT_ID` - Entra ID Tenant ID

#### 3. Checkout Source Branch
The workflow checks out the specified source branch (typically `main`) to use as the base for the new export.

#### 4. Validate Secrets
Before proceeding, the workflow validates that all required secrets are present:
```bash
missing=()
[[ -z "$CLIENT_ID" ]] && missing+=("CLIENT_ID")
[[ -z "$CLIENT_SECRET" ]] && missing+=("CLIENT_SECRET")
[[ -z "$TENANT_ID" ]] && missing+=("TENANT_ID")
```

#### 5. Install Power Platform Tools
Installs Microsoft Power Platform CLI tools required for export and unpack operations.

#### 6. Retrieve Solution Version
**Critical Step:** The workflow queries the Dataverse Web API to retrieve the current solution version **before** export:

```bash
# Authenticates using OAuth2 client credentials flow
TOKEN=$(curl -sS -X POST "https://login.microsoftonline.com/${TENANT_ID}/oauth2/v2.0/token" ...)

# Queries solution version from Dataverse
RESP=$(curl -sS --get "${ENV_URL}/api/data/v9.0/solutions" \
  --data-urlencode '$select=uniquename,version' \
  --data-urlencode "\$filter=uniquename eq '${SOL_NAME}'" ...)
```

The version is stored in two formats:
- `SOLUTION_VERSION` - Original format (e.g., `1.0.0.1`)
- `SOLUTION_VERSION_SAFE` - Branch-safe format (e.g., `1_0_0_1`)

#### 7. Create Fixed Version Branch
The workflow creates a branch with a **fixed, deterministic name** based on the solution name and version:

```bash
SAFE_SOL=$(echo "$SOL" | sed -E 's/[^a-zA-Z0-9_-]+/-/g')
BRANCH="${SAFE_SOL}-${SOLUTION_VERSION_SAFE}"
# Example: Sample-1_0_0_1
```

**Note:** Unlike other workflows, there is no option to override the branch name. This ensures consistency and traceability.

#### 8. Prepare Folder Structure
Cleans and prepares the output folders:
```bash
mkdir -p out/exported
rm -rf "solutions/${{ inputs.solution_name }}"
mkdir -p "solutions/${{ inputs.solution_name }}"
```

#### 9. Export UNMANAGED Solution
Exports the unmanaged version of the solution using Microsoft's export-solution action:

```yaml
- name: Export UNMANAGED solution
  uses: microsoft/powerplatform-actions/export-solution@v1
  with:
    environment-url: ${{ inputs.environment_url }}
    cloud: UsGov
    solution-name: ${{ inputs.solution_name }}
    solution-output-file: out/exported/${{ inputs.solution_name }}_${{ env.SOLUTION_VERSION_SAFE }}.zip
    managed: false
```

Output: `out/exported/Sample_1_0_0_1.zip`

#### 10. Export MANAGED Solution
Exports the managed version with the same parameters, but with `managed: true`:

```yaml
- name: Export MANAGED solution
  uses: microsoft/powerplatform-actions/export-solution@v1
  with:
    managed: true
    solution-output-file: out/exported/${{ inputs.solution_name }}_${{ env.SOLUTION_VERSION_SAFE }}_managed.zip
```

Output: `out/exported/Sample_1_0_0_1_managed.zip`

#### 11. Validate Exported ZIPs
Validates that both ZIP files are valid archives and not HTML error pages or truncated responses:

```bash
unzip -t "out/exported/${solution_name}_${version}.zip"
unzip -t "out/exported/${solution_name}_${version}_managed.zip"
```

This prevents misleading PAC CLI errors that occur when authentication fails or the API returns HTML instead of a ZIP file.

#### 12. Unpack UNMANAGED Solution
Unpacks the unmanaged ZIP into a structured XML format:

```yaml
- name: Unpack UNMANAGED into repo
  uses: microsoft/powerplatform-actions/unpack-solution@v1
  with:
    solution-file: out/exported/${{ inputs.solution_name }}_${{ env.SOLUTION_VERSION_SAFE }}.zip
    solution-folder: solutions/${{ inputs.solution_name }}/unmanaged
    solution-type: Unmanaged
```

Output: `solutions/Sample/unmanaged/` (contains XML files)

#### 13. Unpack MANAGED Solution
Similarly unpacks the managed ZIP:

```yaml
- name: Unpack MANAGED into repo
  uses: microsoft/powerplatform-actions/unpack-solution@v1
  with:
    solution-file: out/exported/${{ inputs.solution_name }}_${{ env.SOLUTION_VERSION_SAFE }}_managed.zip
    solution-folder: solutions/${{ inputs.solution_name }}/managed
    solution-type: Managed
```

Output: `solutions/Sample/managed/` (contains XML files)

#### 14. Commit Changes
Commits all changes with the solution version appended to the commit message:

```bash
git config user.name "${{ inputs.user_name }}"
git config user.email "${{ github.actor }}@users.noreply.github.com"
git add -A
git commit -m "${{ inputs.commit_message }} (v${SOLUTION_VERSION})"
```

Example commit message: `Export Power Platform solution (v1.0.0.1)`

#### 15. Push Branch
Finally, pushes the versioned branch to the remote repository:

```bash
git push origin "$TARGET_BRANCH"
```

The branch is now available in GitHub for Pull Request creation.

---

## How to Use This Action

### Prerequisites

#### 1. GitHub Environment Configuration

Create a GitHub **Environment** named **`MAIN`** with the following secrets:

| Secret Name | Description | Example |
|-------------|-------------|---------|
| `CLIENT_ID` | Entra ID Application (Service Principal) Client ID | `12345678-1234-1234-1234-123456789012` |
| `CLIENT_SECRET` | Client secret for the App Registration | `abc123def456...` |
| `TENANT_ID` | Entra ID Tenant ID | `87654321-4321-4321-4321-210987654321` |

**To create the environment:**
1. Navigate to your repository → Settings → Environments
2. Click "New environment"
3. Name it **`MAIN`** (case-sensitive)
4. Add the three required secrets

#### 2. Dataverse Configuration

**Application User Setup:**
1. Register an application in Entra ID (Azure Active Directory)
2. Create a client secret for the application
3. Add the application as a **Dataverse Application User** in your environment
4. Assign minimum required security roles:
   - System Customizer (or custom role with export permissions)
   - Basic User

**Solution Requirements:**
- Solution must be **unmanaged** in the source environment
- Use the solution's **unique name** (not display name) when running the workflow
- Solution version should be properly maintained in Dataverse

#### 3. Permissions

Ensure the repository has the following permissions configured:
```yaml
permissions:
  contents: write  # Required to push branches
```

### Running the Workflow

#### Step 1: Navigate to GitHub Actions
1. Go to your GitHub repository
2. Click the **Actions** tab
3. Select **Export Power Platform Solution (Dual Export, Fixed Version Branch)** from the workflow list

#### Step 2: Trigger the Workflow
1. Click the **Run workflow** button (right side of the page)
2. A form will appear with the following inputs

#### Step 3: Provide Required Inputs

| Input | Description | Example | Required |
|-------|-------------|---------|----------|
| **environment_url** | Full Dataverse environment URL | `https://org12345.crm.dynamics.com` | Yes |
| **solution_name** | Solution **unique name** in Dataverse | `Sample` or `MyCustomSolution` | Yes |
| **source_branch** | Branch to start from | `main` | Yes (default: `main`) |
| **commit_message** | Commit message for traceability | `Export Power Platform solution` | Yes (default provided) |
| **user_name** | Git commit author name | `github-actions[bot]` | Yes (default provided) |

**Important Notes:**
- Use the **unique name** of the solution, not the display name
- For GCC High/DoD environments, the URL should be `.crm.dynamics.com` (UsGov cloud)
- The `target_branch` parameter does **not exist** in this workflow - it's automatically generated

#### Step 4: Monitor Execution
1. After clicking "Run workflow", the workflow will appear in the runs list
2. Click on the run to view live logs
3. Monitor each step's progress
4. The workflow typically takes 2-5 minutes depending on solution size

#### Step 5: Verify Success
When the workflow completes successfully, you should see:
- ✅ All steps completed with green checkmarks
- A new branch created with the format `<solution-name>-<version>`
- The branch visible in your repository's branch list

---

## Directory Structure Created

After the workflow completes successfully, the following directory structure is created in the versioned branch:

```
<repository-root>/
│
├── .github/
│   └── workflows/
│       └── Download_unpack_and_commit_the_solution_to_git.yml
│
├── solutions/
│   └── <solution_name>/              # e.g., "Sample"
│       ├── managed/                   # Managed solution unpacked
│       │   ├── AppModuleSiteMaps/     # App module site maps
│       │   ├── AppModules/            # Model-driven app definitions
│       │   ├── CanvasApps/            # Canvas apps (if any)
│       │   ├── Entities/              # Table definitions
│       │   │   ├── <entity_name>/     # e.g., "ljr_Notices"
│       │   │   │   ├── Entity.xml     # Table schema
│       │   │   │   ├── FormXml/       # Forms
│       │   │   │   │   ├── main/      # Main forms
│       │   │   │   │   ├── quick/     # Quick create forms
│       │   │   │   │   └── card/      # Card forms
│       │   │   │   ├── RibbonDiff.xml # Command bar customizations
│       │   │   │   └── SavedQueries/  # Views
│       │   ├── EntityDataSources/     # Virtual table data sources
│       │   ├── Other/                 # Other solution components
│       │   │   ├── Solution.xml       # Solution metadata
│       │   │   └── Customizations.xml # Customizations manifest
│       │   ├── WebResources/          # Web resources (JS, CSS, HTML, images)
│       │   └── aiskillconfigs/        # AI Builder skills
│       │
│       └── unmanaged/                 # Unmanaged solution unpacked
│           ├── AppModuleSiteMaps/     # (same structure as managed)
│           ├── AppModules/
│           ├── CanvasApps/
│           ├── Entities/
│           ├── EntityDataSources/
│           ├── Other/
│           ├── WebResources/
│           └── aiskillconfigs/
│
└── out/
    └── exported/                      # Temporary export folder (not committed)
```

### Directory Structure Explanation

#### `solutions/<solution_name>/`
Root folder for the solution, named using the solution's unique name.

#### `managed/` vs `unmanaged/`
- **`unmanaged/`** - Contains the unmanaged solution unpacked as XML
  - Used for source control and change tracking
  - Developers work with this version in development environments
  - Shows all solution components with full metadata
  
- **`managed/`** - Contains the managed solution unpacked as XML
  - Used for deployment validation and parity checking
  - Represents what will be deployed to Test/Prod environments
  - Contains managed properties and deployment settings

#### Key Folders Within Each Solution

**`AppModuleSiteMaps/`**
- Site maps for model-driven apps
- Defines navigation structure

**`AppModules/`**
- Model-driven app definitions
- Contains app configuration and assets

**`CanvasApps/`**
- Canvas app packages
- Only present if solution contains canvas apps

**`Entities/`**
- Table (entity) definitions
- Contains all table metadata including:
  - Schema definitions (`Entity.xml`)
  - Forms (`FormXml/`)
  - Views (`SavedQueries/`)
  - Command bar customizations (`RibbonDiff.xml`)

**`EntityDataSources/`**
- Virtual table data sources
- External data connections

**`Other/`**
- Contains critical solution files:
  - `Solution.xml` - Solution metadata, version, publisher info
  - `Customizations.xml` - Master customizations manifest
- Connection references
- Environment variables
- Other miscellaneous components

**`WebResources/`**
- JavaScript, CSS, HTML files
- Images and other static resources
- Organized by web resource unique name

**`aiskillconfigs/`**
- AI Builder skill configurations
- Only present if solution uses AI Builder

#### `out/exported/` (Temporary)
- Contains the exported ZIP files temporarily during workflow execution
- Files are typically cleaned up or not committed
- Example files:
  - `Sample_1_0_0_1.zip` (unmanaged)
  - `Sample_1_0_0_1_managed.zip` (managed)

### File Formats

All unpacked solution components are in **XML format**, which provides:
- ✅ Human-readable structure
- ✅ Git-friendly diff capabilities
- ✅ Line-by-line change tracking
- ✅ Merge conflict resolution support
- ✅ Code review compatibility

---

## Complete Workflow: From Export to Pull Request

This section provides a **complete end-to-end workflow** showing what happens after the action completes successfully and how to create a Pull Request for review and approval.

### Workflow Overview

```
┌─────────────────────────────────────┐
│  1. Make Changes in Dev Environment │
│     (Power Platform Maker Portal)   │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  2. Run Export GitHub Action        │
│     (Manual Trigger)                │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  3. Action Creates Versioned Branch │
│     (e.g., Sample-1_0_0_1)          │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  4. Create Pull Request             │
│     (Manual Step - Required)        │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  5. Review Changes                  │
│     (Team Review Process)           │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  6. Approve and Merge PR            │
│     (After Approval)                │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│  7. Changes Now in Main Branch      │
│     (Ready for Deployment)          │
└─────────────────────────────────────┘
```

### Detailed Steps

#### Step 1: Make Changes in Development Environment

1. Open the Power Platform Maker Portal
2. Navigate to your development environment
3. Open your solution (e.g., "Sample Solution")
4. Make necessary changes:
   - Add/modify tables
   - Create/update forms
   - Modify business rules
   - Update apps
5. **Increment the solution version** in Solution Properties
   - Example: `1.0.0.1` → `1.0.0.2`
   - This ensures proper version tracking
6. Save and publish all customizations

#### Step 2: Run the Export GitHub Action

1. Navigate to GitHub → **Actions** tab
2. Select **Export Power Platform Solution (Dual Export, Fixed Version Branch)**
3. Click **Run workflow**
4. Fill in the required inputs:
   ```
   environment_url: https://orgXXXXX.crm.dynamics.com
   solution_name: Sample
   source_branch: main
   commit_message: Export solution after adding new table
   user_name: github-actions[bot]
   ```
5. Click **Run workflow** (green button)

#### Step 3: Monitor Action Execution

1. Click on the workflow run to view logs
2. Monitor progress through each step:
   - ✅ Checkout source branch
   - ✅ Validate secrets
   - ✅ Get solution version
   - ✅ Create versioned branch
   - ✅ Export UNMANAGED solution
   - ✅ Export MANAGED solution
   - ✅ Validate exported ZIPs
   - ✅ Unpack solutions
   - ✅ Commit changes
   - ✅ Push branch

3. **Wait for successful completion** (typically 2-5 minutes)

#### Step 4: Verify Branch Creation

After the action completes:

1. Navigate to your repository
2. Click on the **Branches** dropdown or go to the **Branches** page
3. Verify the new branch exists:
   ```
   Sample-1_0_0_2
   ```
4. Note the branch naming pattern: `<solution-name>-<version>`

#### Step 5: Create Pull Request (REQUIRED)

**⚠️ IMPORTANT:** The workflow does **NOT** automatically create a Pull Request. You **MUST** create one manually.

##### Option A: Using GitHub Web Interface

1. Navigate to your repository on GitHub
2. You should see a yellow banner saying:
   ```
   Sample-1_0_0_2 had recent pushes X minutes ago
   [Compare & pull request]
   ```
3. Click **Compare & pull request**
4. Fill in the Pull Request form:

   **Title:**
   ```
   Export Sample solution v1.0.0.2
   ```

   **Description (Example):**
   ```markdown
   ## Summary
   Export of Sample solution from development environment
   
   ## Changes
   - Added new table: ljr_Customer
   - Updated ljr_Notices main form
   - Added new view for ljr_NoticeAlert
   
   ## Solution Details
   - **Solution Name:** Sample
   - **Version:** 1.0.0.2
   - **Environment:** https://orgXXXXX.crm.dynamics.com
   - **Export Date:** 2026-02-03
   
   ## Checklist
   - [x] Solution exported successfully
   - [x] Both managed and unmanaged versions included
   - [x] ZIP validation passed
   - [ ] Changes reviewed
   - [ ] Ready for deployment
   
   ## Reviewers
   @team-lead @solution-architect
   ```

5. Select reviewers from your team
6. Set appropriate labels (e.g., `power-platform`, `export`, `review-required`)
7. Click **Create pull request**

##### Option B: Using GitHub CLI

```bash
# Navigate to your local repository
cd /path/to/repository

# Fetch the new branch
git fetch origin

# Create pull request using GitHub CLI
gh pr create \
  --base main \
  --head Sample-1_0_0_2 \
  --title "Export Sample solution v1.0.0.2" \
  --body "Export of Sample solution from development environment" \
  --reviewer @team-lead,@solution-architect \
  --label power-platform,export
```

##### Option C: Using Git and Browser

```bash
# Navigate to repository
cd /path/to/repository

# Fetch the new branch
git fetch origin

# Checkout the branch locally (optional, for inspection)
git checkout Sample-1_0_0_2

# View changes (optional)
git diff main...Sample-1_0_0_2

# Open browser and navigate to:
# https://github.com/<owner>/<repo>/compare/main...Sample-1_0_0_2
```

#### Step 6: Review Changes in Pull Request

**Review Process:**

1. **Examine the Files Changed tab**
   - Review XML diffs in `solutions/Sample/unmanaged/`
   - Check for unexpected changes
   - Verify new components are included

2. **Key Files to Review:**
   - `solutions/Sample/unmanaged/Other/Solution.xml`
     - Verify version number is correct
     - Check publisher information
   - `solutions/Sample/unmanaged/Entities/`
     - Review new/modified tables
   - `solutions/Sample/unmanaged/WebResources/`
     - Check for new web resources
   - `solutions/Sample/unmanaged/Other/Customizations.xml`
     - Verify all components are listed

3. **Verify Managed vs Unmanaged Parity**
   - Both `managed/` and `unmanaged/` folders should have updates
   - Managed version should reflect deployment-ready state

4. **Check for Issues:**
   - ❌ Uncommitted temporary files
   - ❌ Personal development artifacts
   - ❌ Sensitive information (passwords, API keys)
   - ❌ Debug configurations

5. **Add Review Comments**
   - Comment on specific lines if questions arise
   - Request changes if issues are found
   - Approve when satisfied

#### Step 7: Approval Process

**Federal/DoD Compliance Note:**
- In Fed/DoD environments, Pull Requests typically require:
  - At least **2 approvals** from authorized reviewers
  - **No outstanding change requests**
  - **All checks passing** (if CI/CD checks are configured)

1. **Request Reviews:**
   - Assign to Solution Architect
   - Assign to Technical Lead
   - Assign to Security Reviewer (if required)

2. **Address Feedback:**
   - If changes are requested, make corrections in the development environment
   - Re-run the export workflow
   - Update the PR branch (or create a new PR)

3. **Obtain Approvals:**
   - Wait for all required approvals
   - Ensure all conversations are resolved

#### Step 8: Merge Pull Request

Once approved:

1. Navigate to the Pull Request
2. Click **Merge pull request**
3. Choose merge strategy (typically **Squash and merge** or **Create a merge commit**)
4. Confirm the merge
5. **Delete the branch** (optional but recommended):
   ```
   Sample-1_0_0_2 [Delete branch]
   ```

#### Step 9: Post-Merge Actions

After merging:

1. **Verify main branch updated:**
   ```bash
   git checkout main
   git pull origin main
   ```

2. **Tag the release (optional but recommended):**
   ```bash
   git tag -a v1.0.0.2 -m "Sample solution version 1.0.0.2"
   git push origin v1.0.0.2
   ```

3. **Document in Release Notes:**
   - Create a GitHub Release
   - Attach managed ZIP file (if needed for deployment)
   - Document changes and deployment instructions

4. **Prepare for Deployment:**
   - The solution is now ready for import into Test/Staging/Production
   - Use the managed solution files from `solutions/Sample/managed/`
   - Pack the solution if needed:
     ```bash
     pac solution pack \
       --zipfile Sample_1.0.0.2_managed.zip \
       --folder solutions/Sample/managed \
       --packagetype Managed
     ```

---

## Why Pull Requests Are Required

### Federal/DoD Compliance Considerations

The Pull Request step is **mandatory** and critical for several reasons:

#### 1. **Human Review Gate**
- Federal regulations require human oversight of all changes
- Prevents accidental deployment of untested changes
- Ensures at least 2 sets of eyes review every change

#### 2. **Audit Trail**
- Pull Requests create an immutable audit record
- Shows who approved changes and when
- Demonstrates compliance with change management procedures

#### 3. **Separation of Duties**
- Developer who exports ≠ Developer who approves
- Enforces principle of least privilege
- Prevents single-person control over production assets

#### 4. **Change Documentation**
- PR description documents what changed and why
- Reviewers can ask questions and request clarification
- Creates searchable history for future reference

#### 5. **Risk Mitigation**
- Catches potential security issues before merge
- Identifies unintended changes or deletions
- Validates that solution version increments correctly

#### 6. **Compliance Evidence**
- PR history serves as evidence for audits
- Demonstrates due diligence in change management
- Shows approval chain for sensitive changes

---

## Troubleshooting

### Common Issues and Solutions

#### Issue 1: "Missing required secrets"

**Error Message:**
```
Missing required secrets: CLIENT_ID, CLIENT_SECRET, TENANT_ID
```

**Solution:**
1. Verify the GitHub Environment named **`MAIN`** exists
2. Check that all three secrets are configured in the environment
3. Ensure secret names match exactly (case-sensitive)
4. Verify you have permission to access the environment

#### Issue 2: "Unable to retrieve solution version"

**Error Message:**
```
Unable to retrieve solution version for SolutionName
```

**Solution:**
1. Verify the solution name is the **unique name**, not display name
2. Check that the solution exists in the specified environment
3. Ensure the application user has permissions to read solution metadata
4. Verify the environment URL is correct

#### Issue 3: ZIP validation fails

**Error Message:**
```
unzip: cannot find zipfile directory
```

**Solution:**
1. Check authentication credentials are valid
2. Verify the application user has export permissions
3. Ensure the environment URL is accessible
4. Check for network/firewall issues blocking access

#### Issue 4: Branch already exists

**Error Message:**
```
fatal: A branch named 'Sample-1_0_0_1' already exists.
```

**Solution:**
1. If the previous export failed, delete the branch:
   ```bash
   git push origin --delete Sample-1_0_0_1
   ```
2. Or increment the solution version in Dataverse before re-exporting

#### Issue 5: Permission denied on push

**Error Message:**
```
remote: Permission to repo denied
```

**Solution:**
1. Verify repository permissions include `contents: write`
2. Check that GitHub Actions has permission to create branches
3. Verify the GITHUB_TOKEN has appropriate permissions

---

## Best Practices

### 1. Solution Version Management
- **Always increment** the solution version before export
- Use semantic versioning: `major.minor.patch.build`
- Document version changes in PR descriptions

### 2. Commit Messages
- Use descriptive commit messages
- Include ticket/work item numbers
- Example: `Export solution after fixing bug #123`

### 3. Branch Hygiene
- Delete merged branches promptly
- Don't reuse version-based branch names
- Keep branch names synchronized with solution versions

### 4. Review Process
- Require at least 2 approvals for production-bound changes
- Use a checklist in PR description
- Tag specific reviewers based on component changes

### 5. Documentation
- Document what changed in Power Platform
- Link to related work items or tickets
- Include screenshots for UI changes

### 6. Security
- Never commit secrets or credentials
- Review web resources for sensitive data
- Use connection references for external connections

### 7. Testing
- Test in a non-production environment first
- Validate export in a clean environment
- Verify managed solution imports successfully

---

## Additional Resources

### Microsoft Documentation
- [Power Platform GitHub Actions](https://learn.microsoft.com/en-us/power-platform/alm/devops-github-actions)
- [Solution Concepts](https://learn.microsoft.com/en-us/power-platform/alm/solution-concepts-alm)
- [Application Lifecycle Management](https://learn.microsoft.com/en-us/power-platform/alm/)

### GitHub Documentation
- [GitHub Environments](https://docs.github.com/en/actions/deployment/targeting-different-environments/using-environments-for-deployment)
- [GitHub Actions Secrets](https://docs.github.com/en/actions/security-guides/encrypted-secrets)
- [Pull Requests](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests)

### Related Workflows
- See `PowerPlatformExportReadme.md` for general ALM pattern documentation
- See `.github/workflows/` for other available workflows

---

## Frequently Asked Questions

### Q: Why doesn't the workflow automatically create a Pull Request?

**A:** Federal and DoD compliance requires explicit human action to initiate the review process. Automatically creating PRs could bypass approval gates or create noise. The manual PR creation step ensures conscious, deliberate review.

### Q: Can I customize the branch naming pattern?

**A:** No. The "Fixed Version Branch" workflow uses a deterministic naming pattern (`<solution-name>-<version>`) specifically for compliance and traceability. This ensures branch names are unique, sortable, and tied to specific solution versions.

### Q: What's the difference between managed and unmanaged exports?

**A:** 
- **Unmanaged** - Used for development and source control. Contains all metadata and can be modified.
- **Managed** - Used for deployment to Test/Prod. Locked down and cannot be modified in target environment.

### Q: Can I export multiple solutions in one workflow run?

**A:** No. This workflow is designed to export one solution at a time for clarity and auditability. To export multiple solutions, run the workflow multiple times or create solution dependencies.

### Q: What happens if the solution version doesn't change between exports?

**A:** The workflow will attempt to create a branch with the same name, which will fail. Always increment the solution version before exporting to avoid conflicts.

### Q: Do I need to pack the solution before deploying?

**A:** The solution is already unpacked for source control. To deploy, you may need to pack it back into a ZIP file using the Power Platform CLI:
```bash
pac solution pack --zipfile output.zip --folder solutions/Sample/managed --packagetype Managed
```

### Q: Can I use this with GCC High or DoD environments?

**A:** Yes! The workflow is specifically designed for Federal environments and uses the `cloud: UsGov` parameter for GCC High and DoD (Flank Speed) compatibility.

### Q: What if my solution has dependencies on other solutions?

**A:** Dependencies are included in the solution export automatically. When importing the managed solution, Dataverse will enforce dependency requirements. Document dependencies in your PR description.

---

## Summary

The **Export Power Platform Solution (Dual Export, Fixed Version Branch)** workflow provides a secure, auditable, and Federal-compliant method for version controlling Power Platform solutions. By exporting both managed and unmanaged versions, validating integrity, and requiring Pull Request review, it ensures that all changes are tracked, reviewed, and approved before integration.

**Key Takeaways:**
- ✅ Fully automated export and unpacking process
- ✅ Fixed version-based branch naming for consistency
- ✅ Dual export (managed + unmanaged) for parity validation
- ✅ ZIP validation prevents silent failures
- ✅ **Manual Pull Request creation required for approval gate**
- ✅ Federal/DoD compliant ALM pattern
- ✅ Complete audit trail from export to merge

**Remember:** Always create a Pull Request after the workflow completes successfully. The PR is the approval gate that ensures compliance and enables team review.
