# Flux Kustomization Error Troubleshooting Guide

## The Problem

Flux was showing an error:

```
kustomize build failed: accumulating resources: accumulation err='accumulating resources from './apps/staging/linkding': read /tmp/kustomization-.../clusters/staging/apps/staging/linkding: is a directory': recursed merging from path '...': may not add resource with an already registered id: Namespace.v1.[noGrp]/linkding.[noNs]
```

**Root Cause:** Flux couldn't authenticate to GitHub via SSH, so it was stuck using an old commit (`858d1be1`) that didn't have the `apps/staging/kustomization.yaml` file. Without this file, kustomize tried to process the directory directly, causing duplicate namespace definitions.

## The Solution

### Step 1: Diagnose the Issue

#### Check Flux Kustomization Status

```bash
flux get kustomizations
```

- **What it does:** Lists all Flux Kustomization resources and their status
- **What to look for:** `READY` status and `REVISION` (commit hash)
- **Red flag:** Old commit hash that doesn't match your latest commit

#### Check Git Source Status

```bash
flux get sources git
```

- **What it does:** Shows GitRepository resources and their sync status
- **What to look for:** `READY` status and authentication errors
- **Red flag:** `failed to checkout` or `unable to authenticate` messages

#### Verify Local Git State

```bash
git log --oneline -5
```

- **What it does:** Shows last 5 commits in one-line format
- **`--oneline`:** Condenses each commit to one line (hash + message)
- **What to check:** Your latest commit should be newer than what Flux shows

```bash
git status
```

- **What it does:** Shows working directory status
- **What to check:** Ensure all changes are committed (`nothing to commit, working tree clean`)

```bash
git log origin/main --oneline -5
```

- **What it does:** Shows commits on remote `main` branch
- **`origin/main`:** References the remote branch (not local)
- **What to check:** Verify your commits are pushed to remote

#### Check Kubernetes Resources

```bash
kubectl get gitrepository flux-system -n flux-system -o yaml
```

- **What it does:** Gets GitRepository resource in YAML format
- **`-o yaml`:** Output format (can also use `-o json`)
- **What to look for:** `status.conditions` for error messages, `status.artifact.revision` for commit hash

```bash
kubectl get secret flux-system -n flux-system -o yaml
```

- **What it does:** Gets the SSH secret used by Flux
- **What to check:** Secret exists and has `identity`, `identity.pub`, and `known_hosts` fields

### Step 2: Verify SSH Authentication

#### Test Local SSH to GitHub

```bash
ssh -T git@github.com
```

- **What it does:** Tests SSH connection to GitHub
- **`-T`:** Disables pseudo-terminal allocation (GitHub doesn't provide shell)
- **Expected output:** `Hi <username>! You've successfully authenticated...`
- **Exit code 1 is OK:** GitHub doesn't provide shell, but authentication works

#### Check Your SSH Key

```bash
cat ~/.ssh/id_ed25519 2>/dev/null || cat ~/.ssh/id_rsa 2>/dev/null
```

- **What it does:** Prints your SSH private key
- **`~/.ssh/`:** Home directory's `.ssh` folder (where SSH keys are stored)
- **`2>/dev/null`:** Redirects stderr to `/dev/null` (suppresses errors)
  - `2` = stderr file descriptor
  - `>` = redirect
  - `/dev/null` = discards output
- **`||`:** Logical OR - runs second command if first fails
- **What to check:** Key exists and is readable

```bash
cat ~/.ssh/id_ed25519.pub
```

- **What it does:** Prints your SSH public key
- **`.pub`:** Public key file (safe to share, used for authentication)

#### Get GitHub's SSH Fingerprint

```bash
ssh-keyscan github.com
```

- **What it does:** Retrieves GitHub's SSH host keys
- **Purpose:** Needed for `known_hosts` to verify you're connecting to real GitHub
- **Output:** Multiple key types (rsa, ecdsa, ed25519) for compatibility

### Step 3: Fix the SSH Secret

#### Update Flux Secret with Working SSH Key

```bash
kubectl create secret generic flux-system \
  --from-file=identity=$HOME/.ssh/id_ed25519 \
  --from-file=identity.pub=$HOME/.ssh/id_ed25519.pub \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null)" \
  --dry-run=client -o yaml -n flux-system | kubectl apply -f -
```

**Breaking down the command:**

- **`kubectl create secret generic`:** Creates a generic Kubernetes secret
  - `generic`: Secret type (stores arbitrary key-value pairs)
- **`--from-file=identity=...`:** Adds private key from file
  - `identity`: Key name in secret (Flux expects this name)
  - `$HOME/.ssh/id_ed25519`: Path to your private key
  - `$HOME`: Environment variable for home directory
- **`--from-file=identity.pub=...`:** Adds public key from file
  - `identity.pub`: Public key in secret
- **`--from-literal=known_hosts="$(...)"`:** Adds known_hosts from command output
  - `--from-literal`: Creates secret entry from literal string
  - `$(...)`: Command substitution (runs command and uses output)
  - `ssh-keyscan github.com 2>/dev/null`: Gets GitHub's host keys, suppresses errors
- **`--dry-run=client -o yaml`:** Generates YAML without creating resource
  - `--dry-run=client`: Simulates the operation
  - `-o yaml`: Outputs in YAML format
- **`| kubectl apply -f -`:** Pipes YAML to apply command
  - `|`: Pipe operator (sends stdout of left command to stdin of right)
  - `-f -`: Read from stdin (`-` means stdin)

**Alternative simpler approach:**

```bash
kubectl create secret generic flux-system \
  --from-file=identity=$HOME/.ssh/id_ed25519 \
  --from-file=identity.pub=$HOME/.ssh/id_ed25519.pub \
  --from-literal=known_hosts="$(ssh-keyscan github.com 2>/dev/null)" \
  -n flux-system --dry-run=client -o yaml | kubectl apply -f -
```

### Step 4: Trigger Reconciliation

#### Force Flux to Re-fetch from Git

```bash
flux reconcile source git flux-system
```

- **What it does:** Manually triggers GitRepository reconciliation
- **Purpose:** Forces Flux to check for new commits immediately (instead of waiting for interval)
- **Expected output:** `✔ fetched revision main@sha1:<new-commit-hash>`

#### Verify Everything Works

```bash
flux get kustomizations
flux get sources git
```

- **What to check:** Both should show `READY True` and latest commit hash

## How to Think Through This Independently

### 1. **Start with the Error Message**

- Read the error carefully - it tells you what failed
- In this case: "kustomize build failed" + "duplicate namespace"
- This suggests a kustomization structure problem

### 2. **Check the Current State**

```bash
# What commit is Flux using?
flux get kustomizations

# What commit should it be using?
git log --oneline -1

# Are they different? That's a clue!
```

### 3. **Trace the Dependency Chain**

- Flux Kustomization → GitRepository → GitHub
- If GitRepository fails, Kustomization fails
- Check each link in the chain

### 4. **Use Git Fundamentals**

```bash
# See commit history
git log --oneline -10

# See what commit a hash refers to
git show <hash> --oneline -s

# Compare local vs remote
git log HEAD..origin/main  # Commits on remote not in local
git log origin/main..HEAD  # Commits in local not on remote
```

### 5. **Use Linux Fundamentals**

#### File Descriptors

- `0` = stdin (standard input)
- `1` = stdout (standard output)
- `2` = stderr (standard error)

#### Redirection Operators

```bash
command > file      # Redirect stdout to file (overwrite)
command >> file     # Redirect stdout to file (append)
command 2> file     # Redirect stderr to file
command 2>&1        # Redirect stderr to stdout
command > file 2>&1 # Redirect both stdout and stderr to file
command &> file      # Same as above (bash shorthand)
command 2>/dev/null  # Discard stderr
```

#### Command Chaining

```bash
cmd1 && cmd2  # Run cmd2 only if cmd1 succeeds
cmd1 || cmd2  # Run cmd2 only if cmd1 fails
cmd1 | cmd2   # Pipe stdout of cmd1 to stdin of cmd2
```

#### Command Substitution

```bash
$(command)    # Execute command and use output
`command`     # Older syntax (same thing)
```

### 6. **Debugging Strategy**

1. **Is it a local problem?** → Check your files, git status
2. **Is it a remote problem?** → Check if commits are pushed
3. **Is it a Flux problem?** → Check Flux resources, authentication
4. **Is it a Kubernetes problem?** → Check secrets, permissions

### 7. **Common Patterns**

```bash
# Check if something exists
command 2>/dev/null && echo "exists" || echo "not found"

# Get output or default value
VALUE=$(command 2>/dev/null || echo "default")

# Chain commands with error handling
cmd1 && cmd2 || echo "Failed at cmd1 or cmd2"
```

## Verification Checklist

After fixing, verify:

- [ ] `flux get sources git` shows `READY True` and latest commit
- [ ] `flux get kustomizations` shows `READY True` and latest commit
- [ ] Commit hash matches your latest local commit
- [ ] No error messages in Flux status
- [ ] Resources are actually deployed to cluster

## Key Takeaways

1. **Flux reads from remote Git, not local files** - always push commits
2. **SSH authentication is critical** - if it fails, Flux can't fetch updates
3. **Check the dependency chain** - GitRepository → Kustomization → Resources
4. **Use `flux get` commands** - they show the actual state Flux sees
5. **Compare commit hashes** - if they don't match, something is wrong

## Useful Commands Reference

```bash
# Flux
flux get kustomizations
flux get sources git
flux reconcile source git <name>
flux reconcile kustomization <name>

# Git
git log --oneline -5
git status
git log origin/main --oneline -5
git rev-parse HEAD  # Get current commit hash
git show <hash> --oneline -s

# Kubernetes
kubectl get <resource> -n <namespace> -o yaml
kubectl get secret <name> -n <namespace> -o jsonpath='{.data.key}' | base64 -d

# SSH
ssh -T git@github.com
ssh-keyscan github.com
cat ~/.ssh/id_ed25519.pub
```
