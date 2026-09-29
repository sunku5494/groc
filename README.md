# groc

**G**olang **R**emediation for **O**penShift **C**VEs

A Python automation tool for remediating Go CVE vulnerabilities across multiple OpenShift release branches with intelligent fork handling, automatic repository setup, and Go version compatibility management.

## What is groc?

`groc` automates the **complete end-to-end workflow** for CVE remediation in OpenShift repositories:

## Features

✅ **Automated Repository Setup** - Clone, fork, and configure remotes with one command  
✅ **Smart Fallback** - Detects if repository already cloned and uses local copy  
✅ **Multi-Branch Support** - Process multiple release branches in one run  
✅ **Fork Auto-Selection** - Automatically finds compatible OpenShift-sustaining forks  
✅ **Go Version Management** - Handles Go version compatibility automatically  
✅ **Separate Vendor Commits** - Optional separate commits for vendor changes  

## Automated Steps

The script automates the following workflow:

### Setup Phase (when using GitHub URL)

1. **Parse Input** - Detect if URL or local path
2. **Smart Fallback** - Check if `./repo_name` already exists locally
   - If exists → Use local directory (skip clone)
   - If not → Proceed to clone
3. **GitHub Authentication** - Verify `GITHUB_TOKEN` and `GITHUB_ORG` are set
4. **Fork Management** - Check if fork exists in your account
   - If exists → Skip creation
   - If not → Create fork via `gh repo fork`
5. **Repository Clone** - Clone from upstream URL to current directory
6. **Remote Configuration** - Set up git remotes:
   - `upstream` → OpenShift repository (fetch only)
   - `origin` → Your fork (push/pull)
7. **Push Protection** - Set `upstream` push URL to `no_push` (prevents accidents)

### Remediation Phase (when CVE parameters provided)

1. **Branch Checkout** - Checkout and pull latest from upstream branch
2. **Working Branch** - Create unique working branch (auto-increments if exists)
3. **Go Version Detection** - Detect Go version from `.ci-operator.yaml` or `go.mod`
4. **Go Installation** - Install required Go version via goenv
5. **Compatibility Check** - Compare fix Go requirements with repository Go version
6. **Fork Selection** - Find appropriate openshift-sustaining fork if needed
7. **Dependency Update** - Apply fix using `go get` or `go mod replace`
8. **Tidy and Vendor** - Run `go mod tidy` and `go mod vendor`
9. **Git Commit** - Create commit(s) with descriptive message
10. **Multi-Branch** - Repeat for all specified branches

## Next Steps After groc Completes

Once groc finishes, you need to:

### 1. Review Changes
```bash
# Check what was changed
cd installer  # or your repo directory
git log --oneline -5

# Review the diff for each branch
git checkout net-4.16
git show HEAD
```

### 2. Attach OCPBUGS IDs to Commits
If you didn't use `--ocpbugs` flag, amend commits to add JIRA references:
```bash
git commit --amend -m "OCPBUGS-12345: $(git log -1 --format=%B)"
```

Or use `--ocpbugs` flag with groc:
```bash
./groc.py --repo github.com/openshift/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869 \
  --branches 4.16 \
  --ocpbugs 12345,67890
```

### 3. Amend UPSTREAM Tags (for OpenShift Forks)
If the repository is an OpenShift fork, add UPSTREAM commit references:
```bash
# Add UPSTREAM tag to commit message
git commit --amend

# Example commit message format:
# OCPBUGS-12345: Bump golang.org/x/net to v0.38.0 to address CVE-2025-22869
#
# UPSTREAM: <carry>: OpenShift-specific dependency update
```

### 4. Push and Create Pull Requests
```bash
# Push working branch to your fork
git checkout net-4.16
git push origin net-4.16

# Create PR (using GitHub CLI)
gh pr create \
  --base release-4.16 \
  --head sunku5494:net-4.16 \
  --title "OCPBUGS-12345: Fix CVE-2025-22869 in golang.org/x/net" \
  --body "Updates golang.org/x/net to address CVE-2025-22869"

# Repeat for each branch
git checkout net-4.17
git push origin net-4.17
gh pr create --base release-4.17 --head sunku5494:net-4.17 ...
```

## Prerequisites

### Required Software

1. **Python 3.7+** - The script is written in Python 3
2. **Git** - For repository operations
3. **GitHub CLI (`gh`)** - For fork creation and GitHub API interactions
   - Install: https://cli.github.com/manual/installation
   - Authenticate: `gh auth login`
4. **goenv** - For Go version management
   - Install: https://github.com/go-nv/goenv#installation
5. **Go toolchain** - Various Go versions will be installed via goenv as needed
6. **curl** - For fetching module and fork information
7. **PyYAML** - Python YAML library
   ```bash
   pip install PyYAML
   ```

### Required Environment Variables

Set these environment variables before using the script:

```bash
# GitHub Personal Access Token (required for clone/fork operations)
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxx

# Your GitHub username/organization (required for fork creation)
export GITHUB_ORG=your-github-username
```

**Creating a GitHub Token:**
1. Go to https://github.com/settings/tokens
2. Generate new token (classic)
3. Required scopes: `repo`, `workflow`

## Command-Line Arguments

| Argument | Required | Description |
|----------|----------|-------------|
| `--repo` | Yes | Repository path or GitHub URL (e.g., `github.com/openshift/installer`) |
| `--vuln-pkg` | No* | Vulnerable Go module name (e.g., `golang.org/x/net`) |
| `--fixed-version` | No* | Fixed version of the module (e.g., `v0.38.0`) |
| `--cve-ids` | No* | Comma-separated CVE IDs (e.g., `CVE-2025-22869,CVE-2025-22870`) |
| `--branches` | No* | Comma-separated release versions (e.g., `4.15,4.16,4.17`) |
| `--ocpbugs` | No | Comma-separated JIRA OCPBUGS IDs for commit message prefix |
| `--separate-vendor-commits` | No | Create separate commits for vendor/ and go.mod/go.sum |

## Quick Start

### 1. Setup Environment (One-Time)

```bash
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxx
export GITHUB_ORG=sunku5494
```

### 2. Repository Setup Only

Clone repository, create fork, configure remotes:

```bash
./groc.py --repo github.com/openshift/baremetal-runtimecfg
```

**Output:**
- ✅ Creates fork in your GitHub account (if doesn't exist)
- ✅ Clones repository to `./baremetal-runtimecfg`
- ✅ Configures `upstream` and `origin` remotes
- ✅ Enables upstream push protection

### 3. Full CVE Remediation

Apply CVE fix across multiple branches and create commits:

```bash
./groc.py \
  --repo github.com/openshift/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869 \
  --branches 4.15,4.16,4.17
```

**What it does:**
- ✅ Applies the fix to each branch
- ✅ Runs `go mod tidy` and `go mod vendor`
- ✅ Creates git commits with proper CVE references
- ✅ Leaves you ready to review and push


## Usage Examples

### Repository Setup Only

Set up a repository for later CVE remediation:

```bash
# Setup environment
export GITHUB_TOKEN=ghp_xxxxxxxxxxxxx
export GITHUB_ORG=sunku5494

# Clone, fork, and configure
./groc.py --repo github.com/openshift/baremetal-runtimecfg
```

### Local Repository - Single Branch

Apply CVE fix to an existing local repository:

```bash
./groc.py \
  --repo /home/user/repos/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869 \
  --branches 4.16
```

### Auto-Clone with Multiple Branches

Clone repository, create fork, and apply CVE fix across multiple branches:

```bash
./groc.py \
  --repo github.com/openshift/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869,CVE-2025-22870 \
  --branches 4.15,4.16,4.17
```

**Commit message format:**
```
OCPBUGS-12345,OCPBUGS-67890: Bump golang.org/x/net to v0.38.0 to address CVE-2025-22869
```

### Separate Vendor Commits

Create separate commits for vendor/ changes and go.mod/go.sum changes:

```bash
./groc.py \
  --repo github.com/openshift/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869 \
  --branches 4.16 \
  --separate-vendor-commits
```

**Creates two commits:**
1. `Vendor Changes` - Updates vendor/ directory
2. `go.mod Changes` - Updates go.mod and go.sum

### Include Master/Main Branch

Apply fix to master along with release branches:

```bash
./groc.py \
  --repo github.com/openshift/installer \
  --vuln-pkg golang.org/x/net \
  --fixed-version v0.38.0 \
  --cve-ids CVE-2025-22869 \
  --branches master,4.16,4.17
```

## Best Practices

1. **Set environment variables in your shell profile** (`.bashrc`, `.zshrc`) for permanent setup
2. **Always review changes** before pushing to your fork
3. **Test in a single branch** before running on multiple branches
4. **Use `--separate-vendor-commits`** for easier PR review
   
