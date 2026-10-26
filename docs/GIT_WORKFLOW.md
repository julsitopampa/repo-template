# Git & GitHub Workflow Guide

A standardized, linear Git and GitHub Pull Request workflow designed for clean history, cryptographic commit signing, automated local quality gates (`mise` + `hk`), and safe CI/CD execution.

---

## 1. Workstation Setup (One-Time Global Configuration)

### 1.1 Global Git Ergonomics & Identity

Configure Git on your workstation so history stays linear and deleted remote branches are automatically pruned:

```bash
# Always rebase instead of creating merge commits when pulling
git config --global pull.rebase true

# Automatically prune deleted remote-tracking branches on fetch/pull
git config --global fetch.prune true

# Automatically handle fixup! / squash! commits during interactive rebase
git config --global rebase.autoSquash true

# Consistent author identity (use your GitHub noreply email to match web merges)
git config --global user.name "julsitopampa"
git config --global user.email "74509738+julsitopampa@users.noreply.github.com"
```

### 1.2 Cryptographic SSH Commit & Tag Signing

Git 2.34+ supports signing commits and tags directly with your SSH key (`~/.ssh/id_ed25519.pub`), giving every commit a `Verified` badge on GitHub:

```bash
# Use SSH keys for commit and tag signing
git config --global gpg.format ssh
git config --global user.signingkey ~/.ssh/id_ed25519.pub
git config --global commit.gpgSign true
git config --global tag.gpgSign true

# Configure local allowed_signers so 'git log --show-signature' verifies locally
mkdir -p ~/.config/git
echo "$(git config --global user.email) namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" > ~/.config/git/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.config/git/allowed_signers

# Register the SSH public key as a Signing Key on GitHub
gh ssh-key add ~/.ssh/id_ed25519.pub --title "Workstation Signing Key" --type signing
```

---

## 2. GitHub Repository Standards

All repositories follow these baseline settings (managed centrally via OpenTofu/Terraform `integrations/github` in `github-management`):

### 2.1 Pull Request & Merge Strategy

- **Merge commits**: Disabled (`allow_merge_commit = false`)
- **Squash merging**: Enabled (`allow_squash_merge = true`, default commit message set to **PR title and description**)
- **Rebase merging**: Enabled (`allow_rebase_merge = true`)
- **Auto-merge**: Enabled (`allow_auto_merge = true`)
- **Always suggest updating pull request branches**: Enabled (`allow_update_branch = true`)
- **Automatically delete head branches**: Enabled (`delete_branch_on_merge = true`)

### 2.2 Security & Actions Permissions

- **Secret Scanning & Push Protection**: Enabled
- **Dependabot Alerts & Security Updates**: Enabled (with 7-day cooldown in `.github/dependabot.yml`)
- **Default `GITHUB_TOKEN` permissions**: Read-only (`default_workflow_permissions = "read"`), with `can_approve_pull_request_reviews = true` for Release Please.

### 2.3 Repository Rulesets

1. **`main-protection` (Target: `~DEFAULT_BRANCH`)**:
   - Restrict deletions & block force pushes (`non_fast_forward`)
   - Require linear history (`required_linear_history`)
   - Require signed commits (`required_signatures`)
   - Require a Pull Request before merging (dismiss stale reviews on push, require review thread resolution)
   - Require status checks to pass (`Validate`)
2. **`tag-protection` (Target: `refs/tags/v*`)**:
   - Restrict deletions & block force pushes on version tags.

---

## 3. Naming & Formatting Conventions

### 3.1 Branch Naming

Every unit of work lives on a short-lived branch created from `main`. Never commit directly to `main`, and never reuse a branch after its PR is merged.

Format: `<type>/<short-kebab-case-description>`

| Prefix      | Purpose                                      | Example                              |
| :---------- | :------------------------------------------- | :----------------------------------- |
| `feat/`     | New feature or capability                    | `feat/cis-level1-hardening`          |
| `fix/`      | Bug fix or configuration repair              | `fix/console-keymap-fr`              |
| `ci/`       | CI/CD workflows, runners, or pipelines       | `ci/pr-label-build-trigger`          |
| `chore/`    | Tooling, dependencies, `mise`, `hk`          | `chore/enforce-conventional-commits` |
| `docs/`     | Documentation only (`README.md`, etc.)       | `docs/update-architecture-guide`     |
| `refactor/` | Restructuring code without changing behavior | `refactor/split-variables`           |

---

### 3.2 Conventional Commits

All commit messages and Pull Request titles must follow the [Conventional Commits](https://www.conventionalcommits.org/) specification:

```text
<type>(<optional-scope>): <short imperative description>

[optional body explaining WHY the change was made, wrapped at 72 chars]

[optional footer(s), e.g., Closes #12 or BREAKING CHANGE: ...]
```

#### Allowed `<type>` values:

- `feat`: A new feature (triggers a **minor** version bump in Release Please)
- `fix`: A bug fix (triggers a **patch** version bump in Release Please)
- `feat!` / `fix!` (or `BREAKING CHANGE:` in footer): Breaking change (triggers a **major** version bump)
- `ci`: CI/CD workflow or pipeline changes
- `chore`: Build scripts, tooling (`mise`, `hk`), dependency updates
- `docs`: Documentation changes only
- `refactor`: Code change that neither fixes a bug nor adds a feature
- `style`: Formatting or whitespace changes (`packer fmt`, `yamlfmt`, `prettier`)
- `test`: Adding or updating tests
- `perf`: Performance improvements
- `build`: Changes affecting build systems or external dependencies
- `revert`: Reverting a previous commit

#### Description Style Rules:

- Use the **imperative, lowercase mood**: `"add feature"` (not `"Added feature"` or `"Adds feature"`).
- Keep the subject line under **72 characters** with no trailing period.

---

### 3.3 Pull Request Structure

A Pull Request represents **one logical change**.

- **PR Title**: Must follow Conventional Commits (`<type>(<scope>): <description>`), enforced both locally and by the CI `Validate` job.
- **PR Body**: Uses `.github/pull_request_template.md` (`## Summary` and `## Verification`).

---

## 4. Local Quality Gates (`mise` + `hk`)

`hk` runs automatically on `git commit` (`pre-commit` and `commit-msg` hooks):

- **`betterleaks`**: Scans staged changes for hardcoded secrets/credentials.
- **`pin-github-action`**: Enforces full-length commit SHAs on all GitHub Actions `uses:` lines.
- **`actionlint`**: Validates GitHub Actions workflow syntax and runner labels.
- **`zizmor`**: Static security analysis for GitHub Actions workflows (template injection, permissions, unpinned refs).
- **`prettier`**: Formats and lints YAML, Markdown, and JSON files (using `.editorconfig`).
- **`shellcheck` & `shfmt`**: Lints and formats shell scripts.
- **`check-conventional-commit`**: Validates that commit messages adhere to Conventional Commits.

### Useful `hk` Commands

```bash
# Run all read-only checks across the entire repository
hk check --all

# Run auto-formatters across the entire repository
hk fix --all

# Manually run pre-commit checks on staged files
hk run pre-commit
```

---

## 5. Daily Developer Workflow (Step-by-Step)

### Step 1: Start from a Clean, Up-to-Date `main`

Never start work on an old branch. Always return to `main` and pull the latest changes:

```bash
git checkout main
git pull --ff-only origin main
git checkout -b feat/my-feature
```

### Step 2: Make Atomic Commits

Stage related changes and commit using a Conventional Commit message:

```bash
git status -s
git add <file1> <file2>
git commit -m "feat(scope): short imperative description"
```

> **Fixing the Previous Commit (`--amend`):**
>
> ```bash
> git add <fixed-file>
> git commit --amend --no-edit
> ```

> **Fixing an Earlier Commit on Your Branch (`--fixup`):**
>
> ```bash
> git add <fixed-file>
> git commit --fixup <sha-of-earlier-commit>
> git rebase -i --autosquash origin/main
> ```

### Step 3: Push & Open a Pull Request

```bash
git push -u origin HEAD

# Single-commit branch:
gh pr create --fill

# Multi-commit branch (explicit title & summary):
gh pr create \
  --title "feat(scope): overall summary of the feature" \
  --body "## Summary
- First logical change
- Second logical change"
```

### Step 4: Monitor CI & Trigger Optional Heavy Builds

```bash
gh pr checks --watch

# Trigger privileged homelab build on self-hosted-arc before merging:
gh pr edit --add-label test-build
gh pr checks --watch
```

### Step 5: Keep Your Branch Updated with `main`

Never merge `main` into your feature branch. Always rebase onto `origin/main`:

```bash
git fetch origin
git rebase origin/main
git push --force-with-lease
```

### Step 6: Merge the PR & Clean Up

- **Rebase and merge** _(preserves clean, atomic commits individually on `main`)_:
  ```bash
  gh pr merge --rebase --delete-branch
  ```
- **Squash and merge** _(condenses multiple iteration commits into 1 commit on `main`)_:
  ```bash
  gh pr merge --squash --delete-branch
  ```

Pull the updated `main` branch locally:

```bash
git pull --ff-only origin main
```

---

## 6. Cleaning Up History Before Merging (Interactive Rebase Cheat Sheet)

```bash
# 1. Create a safety backup tag first
git tag backup-branch

# 2. Start interactive rebase against origin/main
git rebase -i origin/main
```

In the editor (oldest commit at top, newest at bottom):

- `pick` (`p`): Keep the commit as-is.
- `reword` (`r`): Keep the commit changes, edit its commit message.
- `fixup` (`f`): Meld this commit into the commit above it and discard its message.
- `squash` (`s`): Meld this commit into the commit above it and combine messages.
- `drop` (`d`): Delete the commit completely.

Verify zero unintended file changes, force-push, and delete the backup tag:

```bash
git diff backup-branch HEAD
git push --force-with-lease
git tag -d backup-branch
```
