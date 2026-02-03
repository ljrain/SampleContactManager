# Power Platform Solution Export
### Federal / DoD / GCC / GCC High ALM Pattern

This repository implements a **Federal and DoD‑compliant Application Lifecycle Management (ALM) pattern** for Microsoft Power Platform (Dataverse) solutions using **GitHub Actions**.

It is designed for **GCC, GCC High, and DoD (Flank Speed)** environments and prioritizes:

- Auditability
- Least privilege
- Deterministic artifacts
- Human review and approval
- Clear separation of authoring vs deployment

The workflow **exports** solutions only.  
It **does not deploy** or modify environments.

---

## Why This Pattern Exists (Fed / DoD Context)

Federal and DoD environments typically require:

- Human review of all changes before promotion
- Clear, reviewable change history
- Separation of development and deployment responsibilities
- No opaque or environment‑bound artifacts
- Deterministic and repeatable builds
- Evidence suitable for audits (CM / RM controls)

Default Power Platform patterns (manual exports, opaque ZIP files, in‑product pipelines) do not always meet these expectations.

This repository treats **Git as the system of record** and **Dataverse as the source of truth**, with explicit review gates between them.

---

## What the Workflow Does

The GitHub Actions workflow performs the following steps:

1. **Manual trigger** (`workflow_dispatch`)
2. **Exports the solution twice** from Dataverse:
   - Unmanaged (for source control & diffing)
   - Managed (for deployment validation / parity)
3. **Validates ZIP integrity**
   - Prevents misleading `Central Directory corrupt` PAC errors
   - Catches HTML / JSON / truncated responses early
4. **Unpacks both solutions** into readable XML
5. **Copies unpacked output into a structured repo folder**
6. **Commits changes to a new branch**
7. **Pushes the branch for Pull Request review**

🚫 **No imports, deployments, or environment mutations occur**

---

## Security & Compliance Characteristics

### ✅ No Direct Deployment
- Workflow **never imports** solutions
- Prevents accidental promotion to Test / Prod

### ✅ Least Privilege
- Uses an **Entra ID Application User**
- Requires only permissions to **export solutions**
- No environment admin elevation

### ✅ Auditable Change History
- All changes visible as Git diffs
- Pull Requests enforce human review
- Commit metadata provides attribution and timestamps

### ✅ Deterministic Artifacts
- Managed and Unmanaged exports come from the **same snapshot**
- Prevents version drift between artifacts

### ✅ Explicit Failure on Bad Exports
- ZIP validation prevents silent failures
- Avoids confusing PAC CLI errors caused by HTML / auth responses

---

## Prerequisites

### 1. GitHub Environment

A GitHub **Environment** named **`MAIN`** must exist and contain the following secrets:

| Secret Name      | Description |
|------------------|-------------|
| `CLIENT_ID`      | Entra ID App (Service Principal) Client ID |
| `CLIENT_SECRET` | Client secret for the App Registration |
| `TENANT_ID`      | Entra ID Tenant ID |

> Environment scoping ensures access is controlled and auditable.

---

### 2. Dataverse Configuration

- App Registration added as a **Dataverse Application User**
- Assigned **minimum roles required to export solutions**
- Solution must be **unmanaged** in the source environment
- Workflow uses the **solution unique name** (not display name)

---

## Running the Workflow

1. Navigate to **GitHub → Actions**
2. Select **Export Power Platform Solution (Managed + Unmanaged)**
3. Click **Run workflow**
4. Provide required inputs

### Workflow Inputs

| Input | Description |
|------|------------|
| `environment_url` | Dataverse environment URL (e.g. `https://org.crm.dynamics.com`) |
| `solution_name` | **Unique** Dataverse solution name |
| `source_branch` | Branch to start from (typically `main`) |
| `target_branch` | Optional – leave blank to auto‑generate |
| `commit_message` | Commit message for audit traceability |
| `user_name` | Git author name (default: GitHub Actions bot) |

---

## Branching Behavior

If `target_branch` is left blank, the workflow creates a branch using:

``
