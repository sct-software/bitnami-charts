# SCT GitHub Actions Pipeline Documentation

This document explains the GitHub Actions CI/CD pipelines implemented in the SCT fork of the Bitnami Charts repository. The pipelines automate the building and publishing of selected Helm charts to SCT's private GitHub Packages registry and maintain synchronization with the upstream Bitnami repository.

## Overview

The SCT fork implements two main automated workflows:

1. **SCT CD Pipeline** - Builds and publishes selected Helm charts
2. **SCT Upstream Sync** - Maintains synchronization with the upstream Bitnami repository

## Pipeline Architecture

```mermaid
flowchart TD
    A[Triggers] --> B[SCT CD Pipeline]
    A --> C[SCT Upstream Sync]
    
    subgraph "SCT CD Pipeline"
        B --> D[Setup Charts]
        D --> E[Build Charts]
        E --> F[Publish Charts]
        F --> G[Summary]
    end
    
    subgraph "SCT Upstream Sync"
        C --> H[Check Changes]
        H --> I[Attempt Merge]
        I --> J[Push/Create Issue]
    end
    
    subgraph "Triggers"
        T1[Daily Schedule 2AM UTC]
        T2[Manual Dispatch]
        T3[Push to main]
        T4[Pull Request]
    end
    
    T1 --> B
    T1 --> C
    T2 --> B
    T2 --> C
    T3 --> B
    T4 --> B
    
    style B fill:#e1f5fe
    style C fill:#f3e5f5
```

## 1. SCT CD Pipeline

**File:** `.github/workflows/sct-cd-pipeline.yml`

**Triggers:**

- Daily schedule at 2 AM UTC
- Manual workflow dispatch
- Push to `main` branch
- Pull requests to `main` branch

**Purpose:** Builds and publishes selected Helm charts to GitHub Container Registry

### Pipeline Flow

```mermaid
flowchart LR
    A[Trigger] --> B[Setup Job]
    B --> C[Build Charts Job]
    C --> D[Publish Charts Job]
    D --> E[Summary Job]
    
    subgraph "Setup"
        B1[Define Chart List]
        B2[Set Matrix Strategy]
    end
    
    subgraph "Build"
        C1[Extract Chart Info]
        C2[Package Helm Chart]
        C3[Upload Artifacts]
    end
    
    subgraph "Publish"
        D1[Login to Registry]
        D2[Check Version Exists]
        D3[Download Artifacts]
        D4[Push to Registry]
    end
    
    B --> B1
    B1 --> B2
    C --> C1
    C1 --> C2
    C2 --> C3
    D --> D1
    D1 --> D2
    D2 --> D3
    D3 --> D4
```

### Chart Selection

Currently configured to build these charts:

- `postgresql`
- `redis-cluster`
- `minio`

### Jobs Breakdown

#### 1. Setup Job

```mermaid
flowchart TD
    A[Setup Job] --> B[Set Charts List]
    B --> C[Output Matrix]
    
    C --> D["['postgresql', 'redis-cluster', 'minio']"]
```

#### 2. Build Charts Job

```mermaid
flowchart TD
    A[Matrix Strategy] --> B[PostgreSQL]
    A --> C[Redis Cluster]
    A --> D[MinIO]
    
    B --> E[Extract Chart Info]
    C --> E
    D --> E
    
    E --> F[Update Dependencies]
    F --> G[Package Chart]
    G --> H[Upload Artifact]
```

#### 3. Publish Charts Job

```mermaid
flowchart TD
    A[Publish Job] --> B{On Main Branch?}
    B -->|Yes| C[Login to GHCR]
    B -->|No| D[Skip Publishing]
    
    C --> E[Check Version Exists]
    E --> F{Version Exists?}
    F -->|No| G[Download Artifact]
    F -->|Yes| H[Skip Chart]
    
    G --> I[Push to Registry]
    I --> J[Update Annotations]
```

### Registry Configuration

- **Registry:** `ghcr.io`
- **Namespace:** Repository owner (`sct-software`)
- **Chart URL Format:** `oci://ghcr.io/sct-software/bitnami-chart`

## 2. SCT Upstream Sync

**File:** `.github/workflows/sct-upstream-sync.yml`

**Triggers:**

- Daily schedule at 2 AM UTC
- Manual workflow dispatch

**Purpose:** Automatically synchronizes changes from the upstream Bitnami Charts repository

### Sync Flow

```mermaid
flowchart TD
    A[Trigger Sync] --> B[Checkout Repository]
    B --> C[Configure Git]
    C --> D[Add Upstream Remote]
    D --> E[Check for Changes]
    
    E --> F{Has Changes?}
    F -->|No| G[No Action Needed]
    F -->|Yes| H[Attempt Merge]
    
    H --> I{Merge Successful?}
    I -->|Yes| J[Push Changes]
    I -->|No| K[Create Issue]
    
    J --> L[Close Conflict Issues]
    K --> M[Notify Team]
    
    style G fill:#c8e6c9
    style J fill:#c8e6c9
    style K fill:#ffcdd2
```

### Conflict Resolution

```mermaid
sequenceDiagram
    participant GHA as GitHub Actions
    participant Repo as SCT Repository
    participant Upstream as Bitnami Repository
    participant Issues as GitHub Issues
    
    GHA->>Upstream: Fetch latest changes
    GHA->>Repo: Attempt merge
    
    alt Merge Successful
        GHA->>Repo: Push merged changes
        GHA->>Issues: Close conflict issues
    else Merge Conflicts
        GHA->>Issues: Create/Update conflict issue
        GHA->>Issues: Add resolution instructions
    end
```

### Issue Management

When merge conflicts occur, the workflow:

1. **Creates an issue** with label `upstream-sync-conflict`
2. **Provides resolution steps** in the issue description
3. **Updates existing issues** if conflicts persist
4. **Automatically closes issues** when conflicts are resolved

## Workflow Triggers Summary

```mermaid
gantt
    title Workflow Execution Schedule
    dateFormat  HH:mm
    axisFormat %H:%M
    
    section Daily
    SCT CD Pipeline        :02:00, 1h
    SCT Upstream Sync      :02:00, 1h
    
    section On-Demand
    Manual CD Pipeline     :crit, 00:00, 24h
    Manual Upstream Sync   :crit, 00:00, 24h
    
    section Git Events
    Push to Main          :active, 00:00, 24h
    Pull Request          :active, 00:00, 24h
```

## Environment Variables and Secrets

### Required Environment Variables

| Variable | Value | Purpose |
|----------|--------|---------|
| `REGISTRY` | `ghcr.io` | Container registry URL |
| `REGISTRY_NAMESPACE` | `${{ github.repository_owner }}` | Registry namespace |

### Required Secrets

| Secret | Purpose | Used In |
|--------|---------|---------|
| `GITHUB_TOKEN` | Repository and registry access | Both workflows |

### Permissions

- **SCT CD Pipeline:** Default permissions
- **SCT Upstream Sync:**
  - `contents: write` - Push merged changes
  - `pull-requests: write` - Create conflict issues

## Chart Installation

To install charts from the SCT registry:

```bash
# Login to GitHub Container Registry
echo $GITHUB_TOKEN | helm registry login ghcr.io -u $GITHUB_USERNAME --password-stdin

# Install a chart
helm install my-postgresql oci://ghcr.io/sct-software/bitnami-chart/postgresql
helm install my-redis oci://ghcr.io/sct-software/bitnami-chart/redis-cluster
helm install my-minio oci://ghcr.io/sct-software/bitnami-chart/minio
```

## Monitoring and Troubleshooting

### Pipeline Status

Both workflows provide detailed summaries in the GitHub Actions interface:

- **Build status** for each chart
- **Publishing results** with registry URLs
- **Merge conflict details** with resolution steps

### Common Issues

#### 1. Chart Build Failures

```mermaid
flowchart TD
    A[Build Failure] --> B{Error Type}
    B -->|Dependency Issue| C[Update Chart.lock]
    B -->|Syntax Error| D[Fix Chart.yaml/templates]
    B -->|Version Conflict| E[Update Chart Version]
    
    C --> F[Re-run Pipeline]
    D --> F
    E --> F
```

#### 2. Upstream Sync Conflicts

```mermaid
flowchart TD
    A[Sync Conflict] --> B[Check Created Issue]
    B --> C[Clone Repository]
    C --> D[Add Upstream Remote]
    D --> E[Attempt Manual Merge]
    E --> F[Resolve Conflicts]
    F --> G[Commit and Push]
    G --> H[Issue Auto-Closes]
```

### Resolution Steps

1. **For Build Issues:**
   - Check workflow logs in GitHub Actions
   - Validate charts locally with `helm lint`
   - Test packaging with `helm package`

2. **For Sync Conflicts:**
   - Follow instructions in the auto-created issue
   - Resolve conflicts locally
   - Push resolved changes to main branch

## Maintenance

### Adding New Charts

To add charts to the build pipeline:

1. Edit the setup job in `sct-cd-pipeline.yml`
2. Update the charts array in the "Set charts list" step
3. Ensure the chart exists in the `bitnami/` directory

### Updating Schedule

Both workflows run daily at 2 AM UTC. To change the schedule, update the cron expression in the workflow files:

```yaml
schedule:
  - cron: '0 2 * * *'  # Daily at 2 AM UTC
```

---

*This documentation reflects the current state of the SCT CI/CD pipelines as of August 14, 2025.*
