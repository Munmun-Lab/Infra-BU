# Configure SSH Keys in Bitbucket & SourceTree (Mac/Windows)

> Complete implementation guide with screenshots, commands, and troubleshooting steps.

![SSH Banner](https://images.unsplash.com/photo-1555949963-aa79dcee981c?q=80\&w=1200\&auto=format\&fit=crop)

---

## Overview

This guide explains how to:

1. Generate SSH keys
2. Add SSH keys to Bitbucket
3. Configure SourceTree to use SSH
4. Clone and push repositories securely
5. Troubleshoot common SSH issues

---

# Prerequisites

Before starting, ensure:

* Git is installed
* SourceTree is installed
* Bitbucket account is available
* Terminal or Git Bash access is available

---

# Step 1: Check Existing SSH Keys

Open Terminal (Mac/Linux) or Git Bash (Windows):

```bash
ls -al ~/.ssh
```

Look for files like:

```bash
id_rsa
id_rsa.pub
id_ed25519
id_ed25519.pub
```

If no keys exist, create a new one.

---

![Generate SSH Key](https://images.unsplash.com/photo-1515879218367-8466d910aaa4?q=80\&w=1200\&auto=format\&fit=crop)

# Step 2: Generate SSH Key

Recommended modern SSH key:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
```

If `ed25519` is unsupported:

```bash
ssh-keygen -t rsa -b 4096 -C "your_email@example.com"
```

Press Enter for default location:

```bash
/Users/username/.ssh/id_ed25519
```

Optional:

* Enter passphrase for extra security
* Or press Enter to skip

---

# Step 3: Start SSH Agent

## Mac/Linux

```bash
eval "$(ssh-agent -s)"
```

Add key:

```bash
ssh-add ~/.ssh/id_ed25519
```

---

## Windows (Git Bash)

Start agent:

```bash
eval $(ssh-agent -s)
```

Add key:

```bash
ssh-add ~/.ssh/id_ed25519
```

---

# Step 4: Copy Public Key

View the public key:

```bash
cat ~/.ssh/id_ed25519.pub
```

Copy the complete output.

Example:

```text
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIExampleKey your_email@example.com
```

---

![Bitbucket SSH Configuration](https://images.unsplash.com/photo-1555066931-4365d14bab8c?q=80\&w=1200\&auto=format\&fit=crop)

# Step 5: Add SSH Key to Bitbucket

## In Bitbucket

1. Login to Bitbucket
2. Click Profile Icon
3. Go to:

```text
Personal Settings → SSH Keys
```

4. Click:

```text
Add Key
```

5. Paste copied public key
6. Give label name
7. Save

---

# Step 6: Test SSH Connection

Run:

```bash
ssh -T git@bitbucket.org
```

Expected message:

```text
authenticated via ssh key.
You can use git to connect to Bitbucket.
```

If prompted:

```text
Are you sure you want to continue connecting?
```

Type:

```bash
yes
```

---

![SourceTree Setup](https://images.unsplash.com/photo-1516321318423-f06f85e504b3?q=80\&w=1200\&auto=format\&fit=crop)

# Step 7: Configure SourceTree

## Open SourceTree

Go to:

```text
SourceTree → Settings / Preferences
```

---

## Git Configuration

Under Git:

* Use Embedded Git OR System Git
* Recommended:

```text
System Git
```

---

## SSH Client

Recommended:

### Mac/Linux

```text
OpenSSH
```

### Windows

Choose either:

```text
OpenSSH
```

OR

```text
PuTTY / Pageant
```

OpenSSH is simpler.

---

# Step 8: Clone Repository Using SSH

In Bitbucket repository:

Click:

```text
Clone
```

Select:

```text
SSH
```

Example SSH URL:

```bash
git@bitbucket.org:workspace/repository.git
```

---

## Clone via SourceTree

1. Click Clone
2. Paste SSH URL
3. Select local folder
4. Click Clone

---

# Step 9: Verify Git Remote

Inside repository:

```bash
git remote -v
```

Expected:

```bash
origin  git@bitbucket.org:workspace/repository.git (fetch)
origin  git@bitbucket.org:workspace/repository.git (push)
```

---

![Git Push Workflow](https://images.unsplash.com/photo-1618477388954-7852f32655ec?q=80\&w=1200\&auto=format\&fit=crop)

# Step 10: Push Changes

```bash
git add .
git commit -m "Initial commit"
git push origin main
```

Or:

```bash
git push origin master
```

Depending on branch name.

---

# Common SSH Issues & Fixes

## Permission Denied (publickey)

Error:

```text
Permission denied (publickey)
```

Fix:

```bash
ssh-add ~/.ssh/id_ed25519
```

Verify key added:

```bash
ssh-add -l
```

---

## Wrong Remote URL

Check:

```bash
git remote -v
```

If HTTPS exists instead of SSH:

```bash
git remote set-url origin git@bitbucket.org:workspace/repository.git
```

---

## SSH Agent Not Running

Start again:

```bash
eval "$(ssh-agent -s)"
```

---

## Multiple SSH Keys

Create config file:

```bash
nano ~/.ssh/config
```

Example:

```text
Host bitbucket.org
  HostName bitbucket.org
  User git
  IdentityFile ~/.ssh/id_ed25519
```

Save and exit.

---

# Useful Commands

## Generate SSH Key

```bash
ssh-keygen -t ed25519 -C "email@example.com"
```

## View Public Key

```bash
cat ~/.ssh/id_ed25519.pub
```

## Add SSH Key

```bash
ssh-add ~/.ssh/id_ed25519
```

## Test Connection

```bash
ssh -T git@bitbucket.org
```

## Check Remote

```bash
git remote -v
```

---

# Best Practices

* Use SSH instead of HTTPS for Git operations
* Use separate SSH keys for work/personal accounts if needed
* Never share private keys
* Backup SSH keys securely
* Use passphrase for sensitive environments

---

# Recommended Git Ignore

Example `.gitignore`:

```gitignore
.DS_Store
.idea/
.vscode/
*.log
```

---

![DevOps Workflow](https://images.unsplash.com/photo-1605379399642-870262d3d051?q=80\&w=1200\&auto=format\&fit=crop)

# Conclusion

SSH authentication provides a secure and password-free method for working with Bitbucket repositories. Proper SSH configuration simplifies Git operations in SourceTree and improves developer workflow efficiency.
