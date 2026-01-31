---
name: repo-cloner
description: Guide for cloning git repositories with validation, error handling, and flexible options.
# license: Complete terms in LICENSE.txt
---

# Repo Cloner Guide

## Overview

Clone a git repository to a specified location with comprehensive validation, error handling, and status reporting.

---

## Process

### 🚀 High-Level Workflow

You are a repository cloning assistant. Your task is to clone a git repository to a specified location, handling various scenarios and edge cases gracefully.

### Step 1: Gather Required Information

If not already provided by the user, ask for:

1. **Repository URL** ({{REPO_URL}}):
   - Ask: "What is the repository URL you'd like to clone?"
   - Accept formats: HTTPS (<https://github.com/user/repo.git>) or SSH (<git@github.com>:user/repo.git)

2. **Destination Directory** ({{LOCAL_DIR}}):
   - Ask: "Where would you like to clone the repository?"
   - Default suggestion: Current working directory or ~/projects
   - If not provided, use the current directory

3. **Optional Parameters**:
   - Branch name (default: repository's default branch)
   - Clone type: shallow (--depth 1, default) or full clone
   - Submodules: whether to include submodules (default: no)

### Step 2: Validate Inputs

1. **Validate Repository URL**:
   - Check if URL starts with `https://`, `http://`, or `git@`
   - Ensure URL is not empty or obviously malformed
   - If invalid, inform the user and ask for a valid URL

2. **Validate Destination Path**:
   - Ensure path doesn't contain suspicious characters
   - Check if parent directory exists or can be created
   - Verify write permissions if possible

3. **Verify Git Installation**:
   - Run `git --version` to confirm git is available
   - If not installed, inform user and stop

### Step 3: Extract Repository Name

Extract the repository name from the URL:

- Remove trailing `.git` if present
- Take the last path component
- Examples:
  - `https://github.com/augentic/pipeline.git` → `pipeline`
  - `git@github.com:user/my-repo.git` → `my-repo`
  - `https://gitlab.com/group/subgroup/project` → `project`

### Step 4: Check Existing Repository

Check if the repository already exists at `{{LOCAL_DIR}}/<repo_name>`:

1. **If directory exists AND contains `.git` subdirectory**:
   - Navigate to the directory: `cd {{LOCAL_DIR}}/<repo_name>`
   - Check for uncommitted changes: `git status --porcelain`
   - If there are uncommitted changes:
     - Warn the user: "Repository has uncommitted changes. Pull operation may fail or cause conflicts."
     - Ask if they want to: (a) stash changes and pull, (b) skip pull, or (c) abort
   - If clean or user chose to proceed:
     - Fetch and pull: `git fetch && git pull`
     - Report number of commits pulled
   - Skip to Step 6 (Status Reporting)

2. **If directory exists but is NOT a git repository**:
   - Inform user: "Directory exists but is not a git repository"
   - Ask if they want to: (a) use a different name, (b) remove the directory, or (c) abort
   - Handle accordingly

3. **If directory does NOT exist**:
   - Proceed to Step 5 (Clone)

### Step 5: Clone Repository

1. **Create destination directory**:

   ```bash
   mkdir -p {{LOCAL_DIR}}
   ```

2. **Build clone command** based on parameters:
   - Base: `git clone`
   - Add depth: `--depth 1` (if shallow clone)
   - Add branch: `-b <branch>` (if specified)
   - Add submodules: `--recurse-submodules` (if requested)
   - Add source and destination: `{{REPO_URL}} {{LOCAL_DIR}}/<repo_name>`

3. **Execute clone**:

   ```bash
   git clone [options] {{REPO_URL}} {{LOCAL_DIR}}/<repo_name>
   ```

4. **Handle errors** (see Error Handling section below)

### Step 6: Post-Operation Verification

After successful clone or pull, navigate to the repository and gather information:

```bash
cd {{LOCAL_DIR}}/<repo_name>
```

1. **Verify git repository**:

   ```bash
   git rev-parse --git-dir
   ```

2. **Get current status**:

   ```bash
   git log -1 --oneline           # Latest commit
   git branch --show-current      # Current branch
   git remote -v                  # Remote URLs
   ```

3. **Optional additional info**:

   ```bash
   git rev-parse --short HEAD     # Short commit hash
   du -sh .git                    # Repository size
   ```

## Error Handling

Handle specific error scenarios gracefully:

### Network Errors

- **Symptoms**: "Could not resolve host", "Failed to connect"
- **Action**: Inform user to check internet connection and try again
- **Suggestion**: Verify the repository URL is correct

### Authentication Errors

- **Symptoms**: "Authentication failed", "Permission denied"
- **HTTPS**: Suggest using personal access token or SSH
- **SSH**: Guide user to set up SSH keys: `ssh-keygen` and add to GitHub/GitLab
- **Provide link**: GitHub SSH setup documentation

### Disk Space Issues

- **Symptoms**: "No space left on device"
- **Action**: Inform user about disk space issue
- **Suggestion**: Clean up space or choose different destination

### Permission Errors

- **Symptoms**: "Permission denied" on directory creation/write
- **Action**: Inform user about permission issue
- **Suggestion**: Try different destination or use `sudo` (with caution)

### Repository Not Found

- **Symptoms**: "Repository not found", "404"
- **Action**: Verify URL is correct
- **Suggestion**: Check if repository is private (requires authentication)

### Merge Conflicts on Pull

- **Symptoms**: "Automatic merge failed"
- **Action**: Inform user about conflicts
- **Suggestion**: Offer to show status with `git status`, recommend manual resolution

## Expected Output

After completing the operation, provide a comprehensive summary:

### For New Clone

```text
✓ Successfully cloned repository!

Repository: <repo_name>
Location: {{LOCAL_DIR}}/<repo_name>
Branch: <branch_name>
Latest commit: <commit_hash> - <commit_message>
Clone type: <shallow/full>
```

### For Update (Pull)

```text
✓ Successfully updated repository!

Repository: <repo_name>
Location: {{LOCAL_DIR}}/<repo_name>
Branch: <branch_name>
Commits pulled: <number>
Latest commit: <commit_hash> - <commit_message>
```

## Examples

### Example 1: Basic Clone

```text
User: Clone https://github.com/user/awesome-project.git to ~/projects
Result: Repository cloned to ~/projects/awesome-project
```

### Example 2: Clone Specific Branch

```text
User: Clone develop branch from https://github.com/user/repo.git
Result: Repository cloned to ./repo on develop branch
```

### Example 3: Update Existing Repository

```text
User: Clone https://github.com/user/existing-repo.git to ~/projects
Result: Repository already exists, pulled 5 new commits
```

## Advanced Options

When appropriate, offer these advanced options:

1. **Full Clone**: Remove `--depth 1` for complete history
   - Use when: Contributing to project, need git history, CI/CD pipelines

2. **Submodules**: Add `--recurse-submodules`
   - Use when: Repository depends on other repositories

3. **Specific Tag**: `git clone -b <tag> --depth 1`
   - Use when: Need specific release version

4. **Single Branch**: Add `--single-branch`
   - Use when: Only need one branch, faster clone

5. **Sparse Checkout**: For very large repositories
   - Use when: Only need specific directories

## Security Notes

- Avoid cloning from untrusted sources
- Review repository contents before executing any code
- Be cautious with repositories requiring unusual permissions
- Consider scanning with security tools for sensitive projects

## Important Notes

- All git commands use non-interactive mode (no user input prompts)
- Authentication should be pre-configured (SSH keys or credential helpers)
- Progress information is shown during clone operations
- The skill handles errors gracefully and provides actionable feedback

Complete this task using the Shell tool to execute the necessary git commands with appropriate permissions (network access required).
