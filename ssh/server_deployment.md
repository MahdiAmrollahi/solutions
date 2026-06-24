# Comprehensive Deployment Guide 🚀

This document provides a precise roadmap for where to generate and where to store each SSH key for a **VPS-first deployment** workflow. All key generation happens **directly on the VPS** — your local machine never needs to hold any private keys.

### 🔑 Quick Summary Table

| Key Name              | Generated On | Private Key Location (Secret)          | Public Key Location (Lock)               |
| :-------------------- | :----------- | :------------------------------------- | :--------------------------------------- |
| **vps_deploy_key**    | **VPS**      | **GitHub Secrets** (`SSH_PRIVATE_KEY`) | **VPS** (`~/.ssh/authorized_keys`)       |
| **github_deploy_key** | **VPS**      | **VPS** (`~/.ssh/id_ed25519`)          | **GitHub Repo** (Settings > Deploy Keys) |

> **Why VPS-first?** Your local machine is not a deployment target. By generating keys on the VPS, the private key never leaves the server, you avoid accidentally committing it, and onboarding a new developer only requires sharing a public key.

---

### 1️⃣ Connection A: GitHub Actions ➔ VPS

**Purpose:** Allows GitHub Actions to "log into" your server and run commands (like `git pull`).

#### Step 1 — Generate the key ON the VPS

SSH into your VPS and run:

```bash
ssh-keygen -t ed25519 -f ~/.ssh/vps_deploy_key -C "github-to-vps"
```

You will get two files:

- `~/.ssh/vps_deploy_key` — **private key** (the secret)
- `~/.ssh/vps_deploy_key.pub` — **public key** (the lock)

#### Step 2 — Authorize the public key on the VPS

The VPS must accept logins with this key, so append the **public** key to `authorized_keys`:

```bash
cat ~/.ssh/vps_deploy_key.pub >> ~/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
chmod 700 ~/.ssh
```

#### Step 3 — Register the private key in GitHub Secrets

1. On the **VPS**, display the private key:
   ```bash
   cat ~/.ssh/vps_deploy_key
   ```
2. Copy the **entire output** (including the `-----BEGIN ...-----` and `-----END ...-----` lines).
3. In your **GitHub Repository**, go to **Settings > Secrets and variables > Actions**.
4. Click **New repository secret**:
   - **Name:** `SSH_PRIVATE_KEY`
   - **Value:** paste the copied private key.
5. Click **Add secret**.

> **Security note:** The private key is now stored in GitHub. Anyone with write access to the repo can read it — restrict repo access accordingly.

#### Step 4 — Test from GitHub Actions

A minimal workflow step that proves the connection works:

```yaml
- name: Test VPS connection
  uses: appleboy/ssh-action@v1
  with:
    host: ${{ secrets.VPS_HOST }}
    username: ${{ secrets.VPS_USER }}
    key: ${{ secrets.SSH_PRIVATE_KEY }}
    script: echo "Connected from GitHub Actions to $(hostname)"
```

---

### 2️⃣ Connection B: VPS ➔ GitHub Repo

**Purpose:** Allows the VPS to "download" (pull) the code from your private GitHub repository.

#### Step 1 — Generate the key ON the VPS

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519 -C "vps-to-github"
```

#### Step 2 — Lock down permissions on the VPS

SSH will refuse to use a key that has loose permissions:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/id_ed25519
```

#### Step 3 — Register the public key on GitHub

1. On the **VPS**, copy the public key:
   ```bash
   cat ~/.ssh/id_ed25519.pub
   ```
2. In your **GitHub Repository**, go to **Settings > Deploy keys**.
3. Click **Add deploy key**:
   - **Title:** `Production VPS` (or any descriptive name).
   - **Key:** paste the public key.
4. **Leave "Allow write access" UNCHECKED** — the VPS only needs to read.

> If you have multiple repos on the same VPS, see [`multi_repo_ssh.md`](./multi_repo_ssh.md) for handling multiple deploy keys on one server.

---

### 3️⃣ Finishing Touches on VPS

Run these commands on the **VPS** to verify everything is wired up:

```bash
# Ensure the .ssh directory exists with correct permissions
mkdir -p ~/.ssh
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys

# Test that the VPS can talk to GitHub
ssh -T git@github.com
```

A successful response looks like:

```
Hi <username>/<repo-name>! You've successfully authenticated, but GitHub does not provide shell access.
```

### 💡 Pro Tip: The SSH Config File

On your **VPS**, create `~/.ssh/config` to force SSH to use the correct key for GitHub:

```text
Host github.com
  HostName github.com
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
```

This guarantees that even if you have multiple keys in `~/.ssh/`, the right one is always used for GitHub.

---

### 🔁 Rotating or Revoking Keys

Because everything is on the VPS, rotation is local — no local cleanup needed:

- **Rotate the VPS→GitHub key:** regenerate `~/.ssh/id_ed25519`, replace the public key in **GitHub Repo > Settings > Deploy keys**, and delete the old one.
- **Revoke GitHub→VPS access:** remove the corresponding line from `~/.ssh/authorized_keys` on the VPS **and** rotate `SSH_PRIVATE_KEY` in GitHub Secrets.
